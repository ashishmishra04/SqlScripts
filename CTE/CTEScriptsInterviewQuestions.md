# SQL CTE: 20 Most Asked Interview Questions (Senior Level) — Northwind Edition

Write the SQL yourself first, then check the answers at the bottom. Every question runs against the Microsoft **Northwind** sample database created by `instnwnd.sql`, so the primary dialect is **T-SQL (SQL Server)**. Where a question is about PostgreSQL behavior, a PostgreSQL version is also given; those blocks assume the common PostgreSQL port of Northwind, which uses snake_case names (`orders`, `order_details`, `order_id`, `unit_price`, and so on).

**How the original examples map to Northwind**

| Original concept | Northwind equivalent |
|---|---|
| `employees.department` | `Categories.CategoryName` (via `Products.CategoryID`) |
| `employees.salary` | `Products.UnitPrice` (Northwind's `Employees` table has no salary column) |
| `employees.manager_id` | `Employees.ReportsTo` |
| `orders.amount` | Order revenue: `SUM(od.UnitPrice * od.Quantity * (1 - od.Discount))` from `[Order Details]` |
| `orders.status = 'COMPLETED'` | `Orders.ShippedDate IS NOT NULL` (21 orders are unshipped) |

**Org chart (`Employees.ReportsTo`)**: Andrew Fuller (2) is the Vice President, Sales and reports to no one. Nancy Davolio (1), Janet Leverling (3), Margaret Peacock (4), Steven Buchanan (5), and Laura Callahan (8) report to Fuller. Michael Suyama (6), Robert King (7), and Anne Dodsworth (9) report to Buchanan.

**Safety note**: Section C modifies data. Wrap each statement in `BEGIN TRAN; ... ROLLBACK;` so your Northwind copy stays intact, or rerun `instnwnd.sql` afterwards.

---

# QUESTIONS

## Section A: CTE Basics and Readability

**Q1.** Write a CTE that returns the product name and unit price of every product in the **Beverages** category, then filter that CTE for prices above 20.

**Q2.** Rewrite this nested subquery using a CTE:
```sql
SELECT ProductName, UnitPrice
FROM dbo.Products
WHERE UnitPrice > (SELECT AVG(UnitPrice) FROM dbo.Products);
```

**Q3.** Using two chained CTEs, return products priced above their own category's average unit price, with the category name.

**Q4.** Write one query that uses the same CTE twice: show each category's total sales revenue and its percentage of company-wide revenue.

**Q5.** Use a CTE with an explicit column list to return the 2 most expensive products per category, breaking ties by `ProductID`.

**Q6.** Using CTEs on `Orders` and `[Order Details]`, return customers who have more than 15 shipped orders and shipped revenue above 30000, along with their order count and total.

## Section B: Recursive CTEs

**Q7.** Starting from Andrew Fuller (`EmployeeID` 2), return everyone who reports to him directly or indirectly, with each person's level (Fuller = 0).

**Q8.** Return each employee's full reporting path as a string, such as `Andrew Fuller > Steven Buchanan > Michael Suyama`.

**Q9.** For each manager, return the total number of people who report to them directly or indirectly.

**Q10.** Generate every date from 1998-01-01 to 1998-01-31 without using a calendar table, then show the number of orders placed on each date, including days with zero orders.

**Q11.** Each row in `Categories.Description` is a comma-separated list (for example, `Soft drinks, coffees, teas, beers, and ales`). Using a recursive CTE, split every description into one row per item, numbered in order.

**Q12.** Treat the org chart as an undirected graph: each `ReportsTo` link can be travelled in both directions, which creates cycles (Davolio → Fuller → Davolio). Write a recursive CTE that lists every route starting at Nancy Davolio (`EmployeeID` 1), prevents revisiting an employee, and caps the number of hops at 5.

## Section C: CTEs in Data Modification

**Q13.** Make a working copy with `SELECT * INTO dbo.Orders_Work FROM dbo.Orders;`. Delete duplicate rows from `Orders_Work` where `(CustomerID, OrderDate)` repeats, keeping the row with the lowest `OrderID`. Write the version for SQL Server and the version for PostgreSQL. (Why not run this against `dbo.Orders` directly?)

**Q14.** Give a 5% price increase to products in the bottom price quartile, using a CTE with `NTILE(4)`. Write it for SQL Server, then note what changes in PostgreSQL.

**Q15.** Insert one summary row per category (product count, active product count, average unit price, and total units in stock) into a table `dbo.CategorySummary`, using a CTE.

**Q16.** Move all `[Order Details]` rows that belong to orders placed before 1997-01-01 out of `[Order Details]` and into `dbo.OrderDetails_Archive` in one statement. Write it for SQL Server and for PostgreSQL. Both tables have the same columns.

## Section D: Performance, Scope, and Concepts

**Q17.** When would you choose a CTE, a subquery, or a temp table? Give a concrete Northwind scenario for each.

**Q18.** In PostgreSQL 12+, is a CTE always computed once? Show how to force each behavior using Northwind order revenue. What is the SQL Server equivalent?

**Q19.** Why does this batch fail, and how do you fix it?
```sql
WITH big_customers AS (
    SELECT CustomerID
    FROM dbo.Orders
    GROUP BY CustomerID
    HAVING COUNT(*) > 20
)
SELECT CustomerID FROM big_customers;

SELECT c.CompanyName
FROM dbo.Customers c
JOIN big_customers b ON b.CustomerID = c.CustomerID;
```

**Q20.** You build a calendar of every day from the first to the last Northwind order date with a recursive CTE, and SQL Server fails with Msg 530. Why, how do you fix it, and what happens when a recursion cap is reached?

---
---

# ANSWERS

## Section A: CTE Basics and Readability

**A1.**
```sql
WITH beverages AS (
    SELECT p.ProductName, p.UnitPrice
    FROM dbo.Products p
    JOIN dbo.Categories c ON c.CategoryID = p.CategoryID
    WHERE c.CategoryName = N'Beverages'
)
SELECT ProductName, UnitPrice
FROM beverages
WHERE UnitPrice > 20
ORDER BY UnitPrice DESC;
```
The CTE isolates the business step, and the outer query applies the second filter. The result is Côte de Blaye (263.50) and Ipoh Coffee (46.00).

**A2.**
```sql
WITH overall_avg AS (
    SELECT AVG(UnitPrice) AS AvgPrice
    FROM dbo.Products
)
SELECT p.ProductName, p.UnitPrice
FROM dbo.Products p
CROSS JOIN overall_avg a
WHERE p.UnitPrice > a.AvgPrice
ORDER BY p.UnitPrice DESC;
```
The average unit price is about 28.87 across 77 products, and 25 products are above it. The CTE names that value, so the logic is easier to read and reuse. Since `overall_avg` returns one row, `CROSS JOIN` is safe here.

**A3.**
```sql
WITH category_avg AS (
    SELECT CategoryID, AVG(UnitPrice) AS AvgPrice
    FROM dbo.Products
    GROUP BY CategoryID
),
above_avg AS (
    SELECT p.ProductName, p.CategoryID, p.UnitPrice, ca.AvgPrice
    FROM dbo.Products p
    JOIN category_avg ca ON ca.CategoryID = p.CategoryID
    WHERE p.UnitPrice > ca.AvgPrice
)
SELECT c.CategoryName, a.ProductName, a.UnitPrice,
       CAST(a.AvgPrice AS DECIMAL(10, 2)) AS CategoryAvg
FROM above_avg a
JOIN dbo.Categories c ON c.CategoryID = a.CategoryID
ORDER BY c.CategoryName, a.UnitPrice DESC;
```
The second CTE depends on the first. This chaining pattern is the main reason CTEs improve readability over nested subqueries. The query returns 27 products.

**A4.**
```sql
WITH category_revenue AS (
    SELECT c.CategoryName,
           SUM(od.UnitPrice * od.Quantity * (1 - od.Discount)) AS Revenue
    FROM dbo.[Order Details] od
    JOIN dbo.Products   p ON p.ProductID  = od.ProductID
    JOIN dbo.Categories c ON c.CategoryID = p.CategoryID
    GROUP BY c.CategoryName
)
SELECT CategoryName,
       ROUND(Revenue, 2) AS Revenue,
       ROUND(100.0 * Revenue / (SELECT SUM(Revenue) FROM category_revenue), 2) AS PctOfRevenue
FROM category_revenue
ORDER BY Revenue DESC;
```
A CTE can be referenced several times in the same statement. Here, `category_revenue` is used in both the main query and the scalar subquery. SQL Server inlines CTEs, so it evaluates the definition once per reference. Beverages leads at about 21.16% of roughly 1.27 million in revenue, followed by Dairy Products at about 18.53%. `Discount` is a `real`, so the revenue expression is a float; round it for display.

**A5.**
```sql
WITH ranked (ProductName, CategoryID, UnitPrice, rn) AS (
    SELECT ProductName, CategoryID, UnitPrice,
           ROW_NUMBER() OVER (PARTITION BY CategoryID ORDER BY UnitPrice DESC, ProductID)
    FROM dbo.Products
)
SELECT c.CategoryName, r.ProductName, r.UnitPrice, r.rn
FROM ranked r
JOIN dbo.Categories c ON c.CategoryID = r.CategoryID
WHERE r.rn <= 2
ORDER BY c.CategoryName, r.rn;
```
The column list `(ProductName, CategoryID, UnitPrice, rn)` names the outputs, including the window column that has no alias in the inner query. It is supported in SQL Server, PostgreSQL, MySQL 8+, and SQLite. The `ProductID` tiebreaker makes the result deterministic. The query returns 16 rows, such as Côte de Blaye and Ipoh Coffee for Beverages, and Thüringer Rostbratwurst and Mishi Kobe Niku for Meat/Poultry.

**A6.**
```sql
WITH order_totals AS (
    SELECT o.OrderID, o.CustomerID,
           SUM(od.UnitPrice * od.Quantity * (1 - od.Discount)) AS OrderTotal
    FROM dbo.Orders o
    JOIN dbo.[Order Details] od ON od.OrderID = o.OrderID
    WHERE o.ShippedDate IS NOT NULL
    GROUP BY o.OrderID, o.CustomerID
),
cust_totals AS (
    SELECT CustomerID,
           COUNT(*)        AS OrderCount,
           SUM(OrderTotal) AS TotalAmount
    FROM order_totals
    GROUP BY CustomerID
)
SELECT CustomerID, OrderCount, ROUND(TotalAmount, 2) AS TotalAmount
FROM cust_totals
WHERE OrderCount > 15
  AND TotalAmount > 30000
ORDER BY TotalAmount DESC;
```
Filtering on aggregates needs `HAVING` or an outer query. The trap here is grain: joining `Orders` to `[Order Details]` produces one row per order line, so `COUNT(*)` on the joined rows counts lines, not orders. The first CTE rolls up to one row per order, and the second rolls up to one row per customer. The result is 5 customers: QUICK (28 orders, about 110,277), SAVEA (31, about 104,362), ERNSH (28, about 94,976), HUNGO (19, about 49,980), and RATTC (17, about 49,842).

## Section B: Recursive CTEs

**A7.**
```sql
WITH team AS (
    -- Anchor: Fuller, level 0
    SELECT EmployeeID, FirstName, LastName, ReportsTo, 0 AS Lvl
    FROM dbo.Employees
    WHERE EmployeeID = 2

    UNION ALL

    -- Recursive: anyone whose manager is already in the team
    SELECT e.EmployeeID, e.FirstName, e.LastName, e.ReportsTo, t.Lvl + 1
    FROM dbo.Employees e
    JOIN team t ON e.ReportsTo = t.EmployeeID
)
SELECT EmployeeID, FirstName + N' ' + LastName AS EmployeeName, Lvl
FROM team
ORDER BY Lvl, EmployeeID;
```
The anchor starts the walk, and each recursive step moves one level down the hierarchy. Fuller is level 0, his five direct reports are level 1, and Buchanan's three reports are level 2. SQL Server does not accept the `RECURSIVE` keyword; it detects recursion from the self-reference. PostgreSQL, MySQL, and SQLite require `WITH RECURSIVE`.

**A8.**
```sql
WITH chain AS (
    SELECT EmployeeID,
           CAST(FirstName + N' ' + LastName AS NVARCHAR(500)) AS Path
    FROM dbo.Employees
    WHERE ReportsTo IS NULL

    UNION ALL

    SELECT e.EmployeeID,
           CAST(c.Path + N' > ' + e.FirstName + N' ' + e.LastName AS NVARCHAR(500))
    FROM dbo.Employees e
    JOIN chain c ON e.ReportsTo = c.EmployeeID
)
SELECT EmployeeID, Path
FROM chain
ORDER BY Path;
```
Each step appends the employee's name to the parent's path. SQL Server uses `+` for concatenation; PostgreSQL, Oracle, and SQLite use `||`. The `CAST` matters more in SQL Server than elsewhere: the anchor and recursive members must return exactly the same data type and length, or the query fails with a type-mismatch error.

**A9.**
```sql
WITH reports AS (
    -- Anchor: each direct report, paired with their manager
    SELECT ReportsTo AS RootID, EmployeeID
    FROM dbo.Employees
    WHERE ReportsTo IS NOT NULL

    UNION ALL

    -- Recursive: the reports of each person found so far
    SELECT r.RootID, e.EmployeeID
    FROM reports r
    JOIN dbo.Employees e ON e.ReportsTo = r.EmployeeID
)
SELECT m.EmployeeID AS ManagerID,
       m.FirstName + N' ' + m.LastName AS ManagerName,
       COUNT(*) AS TotalReports
FROM reports r
JOIN dbo.Employees m ON m.EmployeeID = r.RootID
GROUP BY m.EmployeeID, m.FirstName, m.LastName;
```
The `RootID` column carries the original manager through every level. Each row in `reports` is one (manager, descendant) pair, so counting by manager gives the total number of direct and indirect reports. The result is Andrew Fuller with 8 and Steven Buchanan with 3.

**A10.**
```sql
WITH dates AS (
    SELECT CAST('1998-01-01' AS DATE) AS dt
    UNION ALL
    SELECT DATEADD(DAY, 1, dt)
    FROM dates
    WHERE dt < '1998-01-31'
)
SELECT d.dt, COUNT(o.OrderID) AS OrderCount
FROM dates d
LEFT JOIN dbo.Orders o
       ON o.OrderDate >= d.dt
      AND o.OrderDate <  DATEADD(DAY, 1, d.dt)
GROUP BY d.dt
ORDER BY d.dt;
```
The `WHERE` clause in the recursive member is the stop condition. `OrderDate` is a `datetime`, so the join uses a half-open range rather than `CAST(o.OrderDate AS DATE) = d.dt`, which keeps the predicate index-friendly. `COUNT(o.OrderID)` counts only matched rows, so days with no orders show 0. The result is 31 rows: 22 days with orders (55 orders in total) and 9 weekend days with 0. In PostgreSQL, use `dt + 1`; in MySQL, use `DATE_ADD(dt, INTERVAL 1 DAY)`.

**A11.**
```sql
WITH split AS (
    -- Anchor: the whole description plus a trailing comma, no item yet
    SELECT CategoryID,
           CAST(N'' AS NVARCHAR(200)) AS Item,
           CAST(REPLACE(CAST(Description AS NVARCHAR(MAX)), N', and ', N', ') + N','
                AS NVARCHAR(MAX)) AS Rest,
           0 AS ItemNo
    FROM dbo.Categories

    UNION ALL

    -- Recursive: cut the first item off the front of Rest
    SELECT CategoryID,
           CAST(LTRIM(RTRIM(LEFT(Rest, CHARINDEX(N',', Rest) - 1))) AS NVARCHAR(200)),
           CAST(STUFF(Rest, 1, CHARINDEX(N',', Rest), N'') AS NVARCHAR(MAX)),
           ItemNo + 1
    FROM split
    WHERE Rest <> N''
)
SELECT c.CategoryName, s.ItemNo, s.Item
FROM split s
JOIN dbo.Categories c ON c.CategoryID = s.CategoryID
WHERE s.ItemNo > 0
ORDER BY c.CategoryName, s.ItemNo;
```
This replaces the original Fibonacci question, which has no natural Northwind data, but tests the same skill: carrying state from one iteration to the next. Each row holds the item just extracted and the remaining text, and the recursion stops when nothing is left. `Description` is `ntext`, which cannot be concatenated or passed to most string functions, so it is cast to `NVARCHAR(MAX)` first. The `REPLACE` removes the Oxford-comma "and" so `beers, and ales` becomes two clean items. The query returns 20 items. On SQL Server 2022+, `STRING_SPLIT(..., ',', 1)` does this without recursion and returns an ordinal column; mentioning that trade-off is a good senior-level point.

**A12.**
```sql
WITH links AS (
    -- Each reporting line in both directions
    SELECT EmployeeID AS FromID, ReportsTo AS ToID
    FROM dbo.Employees
    WHERE ReportsTo IS NOT NULL
    UNION ALL
    SELECT ReportsTo, EmployeeID
    FROM dbo.Employees
    WHERE ReportsTo IS NOT NULL
),
routes AS (
    -- Anchor: direct links out of Davolio
    SELECT l.ToID AS CurrentID,
           CAST('/1/' + CAST(l.ToID AS VARCHAR(10)) + '/' AS VARCHAR(200)) AS Path,
           1 AS Hops
    FROM links l
    WHERE l.FromID = 1

    UNION ALL

    SELECT l.ToID,
           CAST(r.Path + CAST(l.ToID AS VARCHAR(10)) + '/' AS VARCHAR(200)),
           r.Hops + 1
    FROM routes r
    JOIN links l ON l.FromID = r.CurrentID
    WHERE r.Hops < 5                                                       -- hard cap on depth
      AND CHARINDEX('/' + CAST(l.ToID AS VARCHAR(10)) + '/', r.Path) = 0   -- never revisit
)
SELECT Path, Hops
FROM routes
ORDER BY Hops, Path;
```
There are two protections against the cycle. The `CHARINDEX` check stops a route from visiting an employee twice, and the hop cap is a safety net in case the path check misses something. Wrapping every ID in `/` delimiters avoids the substring false match the original `LIKE` version warned about (for example, ID 1 matching inside 10). The result is 8 routes, one to each other employee, with the deepest at 3 hops (such as `/1/2/5/6/`). Without the revisit check, routes would bounce Davolio → Fuller → Davolio until the hop cap stopped them.

## Section C: CTEs in Data Modification

**A13.**

You cannot run this against `dbo.Orders` directly: `[Order Details]` has a foreign key to `Orders`, so the delete fails, and the "duplicates" are real orders (for example, BOTTM placed orders 10410 and 10411 on 1997-01-10). The working copy has no foreign keys.

SQL Server (a CTE can be the target of `DELETE`):
```sql
WITH dupes AS (
    SELECT OrderID,
           ROW_NUMBER() OVER (
               PARTITION BY CustomerID, OrderDate
               ORDER BY OrderID) AS rn
    FROM dbo.Orders_Work
)
DELETE FROM dupes
WHERE rn > 1;
```

PostgreSQL (`DELETE` cannot target a CTE directly, so delete by key):
```sql
WITH dupes AS (
    SELECT order_id,
           ROW_NUMBER() OVER (
               PARTITION BY customer_id, order_date
               ORDER BY order_id) AS rn
    FROM orders_work
)
DELETE FROM orders_work
WHERE order_id IN (SELECT order_id FROM dupes WHERE rn > 1);
```
The statement deletes 7 rows, one from each of 7 duplicate pairs. Always run the `SELECT` version first to confirm which rows will go.

**A14.**

SQL Server (update through the CTE):
```sql
BEGIN TRAN;

WITH quartiles AS (
    SELECT ProductID, UnitPrice,
           NTILE(4) OVER (ORDER BY UnitPrice, ProductID) AS Quartile
    FROM dbo.Products
)
UPDATE quartiles
SET UnitPrice = ROUND(UnitPrice * 1.05, 2)
WHERE Quartile = 1;

SELECT @@ROWCOUNT AS RowsUpdated;   -- 20
ROLLBACK;
```
The CTE assigns each product a quartile, and the update applies the increase only to quartile 1. SQL Server lets you update through a CTE that contains a window function, as long as you only change base-table columns. `NTILE(4)` over 77 rows produces groups of 20, 19, 19, and 19, so 20 products change. The `ProductID` tiebreaker keeps the quartile boundary deterministic when prices tie.

PostgreSQL uses `UPDATE ... FROM` with a join to the CTE:
```sql
WITH quartiles AS (
    SELECT product_id,
           NTILE(4) OVER (ORDER BY unit_price, product_id) AS quartile
    FROM products
)
UPDATE products p
SET unit_price = ROUND((p.unit_price * 1.05)::numeric, 2)
FROM quartiles q
WHERE q.product_id = p.product_id
  AND q.quartile = 1;
```
In the PostgreSQL port, `unit_price` is a `real`, and `ROUND(value, 2)` only exists for `numeric`, so the cast is required.

**A15.**
```sql
CREATE TABLE dbo.CategorySummary (
    CategoryName      NVARCHAR(15) NOT NULL,
    ProductCount      INT          NOT NULL,
    ActiveProducts    INT          NOT NULL,
    AvgUnitPrice      MONEY        NULL,
    TotalUnitsInStock INT          NULL
);

WITH category_stats AS (
    SELECT c.CategoryName,
           COUNT(*)                                          AS ProductCount,
           SUM(CASE WHEN p.Discontinued = 0 THEN 1 ELSE 0 END) AS ActiveProducts,
           AVG(p.UnitPrice)                                  AS AvgUnitPrice,
           SUM(p.UnitsInStock)                               AS TotalUnitsInStock
    FROM dbo.Products p
    JOIN dbo.Categories c ON c.CategoryID = p.CategoryID
    GROUP BY c.CategoryName
)
INSERT INTO dbo.CategorySummary (CategoryName, ProductCount, ActiveProducts, AvgUnitPrice, TotalUnitsInStock)
SELECT CategoryName, ProductCount, ActiveProducts, AvgUnitPrice, TotalUnitsInStock
FROM category_stats;
```
The statement inserts 8 rows. `Discontinued` is a `bit`, so it cannot be summed directly; the `CASE` turns it into a count. Putting `WITH` before `INSERT` works in SQL Server, PostgreSQL, MySQL 8+, and SQLite. Some engines also accept `INSERT INTO ... WITH ... SELECT`, but the leading form is the most portable.

**A16.**

Create the archive table first. `SELECT ... INTO` with `WHERE 1 = 0` copies the columns but none of the constraints:
```sql
SELECT * INTO dbo.OrderDetails_Archive FROM dbo.[Order Details] WHERE 1 = 0;
```

SQL Server (the `OUTPUT` clause plays the role of `RETURNING`):
```sql
WITH old_orders AS (
    SELECT OrderID
    FROM dbo.Orders
    WHERE OrderDate < '19970101'
)
DELETE od
OUTPUT deleted.OrderID, deleted.ProductID, deleted.UnitPrice,
       deleted.Quantity, deleted.Discount
INTO dbo.OrderDetails_Archive (OrderID, ProductID, UnitPrice, Quantity, Discount)
FROM dbo.[Order Details] AS od
WHERE od.OrderID IN (SELECT OrderID FROM old_orders);
```

PostgreSQL (data-modifying CTE):
```sql
WITH moved AS (
    DELETE FROM order_details od
    USING orders o
    WHERE o.order_id = od.order_id
      AND o.order_date < DATE '1997-01-01'
    RETURNING od.*
)
INSERT INTO order_details_archive
SELECT * FROM moved;
```
Both versions move 405 rows from 152 orders in one atomic statement: if the insert fails, the delete is rolled back as well. `[Order Details]` is the right table to archive from, because nothing references it; deleting from `Orders` would fail on the foreign key. In SQL Server, the `OUTPUT ... INTO` target cannot have enabled triggers, foreign keys, or check constraints, which is another reason to build it with `SELECT ... INTO`.

## Section D: Performance, Scope, and Concepts

**A17.**

| Option | Choose it when | Northwind scenario |
|---|---|---|
| **CTE** | The logic is used once or a few times in one statement, and you want readable steps | Ranking products by price within each category and keeping the top 2 (Q5) |
| **Subquery** | A single, small scalar or existence check is needed | `WHERE UnitPrice > (SELECT AVG(UnitPrice) FROM dbo.Products)` |
| **Temp table** | Results are reused across several statements, need indexes, or are large enough that a statistics-driven plan matters | Computing per-order revenue from `[Order Details]` once into `#order_totals`, indexing it on `CustomerID` and `EmployeeID`, then running customer, sales-rep, and monthly reports against it |

The main trade-off is scope. CTEs last only for one statement, while temp tables last for the session and can be indexed and have statistics. Northwind is small (2,155 order lines), so every option is fast in practice; in an interview, describe the choice as if the table had tens of millions of rows.

**A18.**
```sql
-- PostgreSQL 12+: force a single evaluation, even if referenced several times
WITH cust_revenue AS MATERIALIZED (
    SELECT o.customer_id,
           SUM(od.unit_price * od.quantity * (1 - od.discount)) AS total
    FROM orders o
    JOIN order_details od ON od.order_id = o.order_id
    GROUP BY o.customer_id
)
SELECT a.customer_id, a.total, b.total AS total_again
FROM cust_revenue a
JOIN cust_revenue b ON b.customer_id = a.customer_id;

-- PostgreSQL 12+: allow the optimizer to inline it into the main query
WITH cust_revenue AS NOT MATERIALIZED (
    SELECT o.customer_id,
           SUM(od.unit_price * od.quantity * (1 - od.discount)) AS total
    FROM orders o
    JOIN order_details od ON od.order_id = o.order_id
    GROUP BY o.customer_id
)
SELECT customer_id, total
FROM cust_revenue
WHERE customer_id = 'ALFKI';
```
No. Since PostgreSQL 12, a CTE referenced once is inlined by default, which lets the `customer_id = 'ALFKI'` filter push down into it. A CTE referenced more than once is materialized by default. `MATERIALIZED` and `NOT MATERIALIZED` override this.

SQL Server has no equivalent hint: it always inlines a CTE, so a CTE referenced twice is usually computed twice. To compute it once, load it into a temp table (`SELECT ... INTO #cust_revenue`) and reference that. Always confirm with the execution plan.

**A19.**
The batch fails at the second `SELECT` with "Invalid object name 'big_customers'" (Msg 208). A CTE exists only for the statement that defines it, and the second `SELECT` is a new statement.

Fix with a temp table, which persists for the session:
```sql
SELECT CustomerID
INTO #big_customers
FROM dbo.Orders
GROUP BY CustomerID
HAVING COUNT(*) > 20;

SELECT CustomerID FROM #big_customers;

SELECT c.CompanyName
FROM dbo.Customers c
JOIN #big_customers b ON b.CustomerID = c.CustomerID;
```
Use a temp table when the result is needed across statements. Use a view when the definition should be permanent. A related T-SQL gotcha: the statement before a `WITH` must end with a semicolon, or SQL Server reports a syntax error near `WITH`. That is why many T-SQL scripts write `;WITH`.

**A20.**
```sql
WITH calendar AS (
    SELECT CAST(MIN(OrderDate) AS DATE) AS dt,
           CAST(MAX(OrderDate) AS DATE) AS last_dt
    FROM dbo.Orders

    UNION ALL

    SELECT DATEADD(DAY, 1, dt), last_dt
    FROM calendar
    WHERE dt < last_dt
)
SELECT dt
FROM calendar
OPTION (MAXRECURSION 1000);
```
Northwind orders run from 1996-07-04 to 1998-05-06, which needs 671 recursive steps. SQL Server's default limit is 100, so without the hint the query fails with Msg 530 ("The maximum recursion 100 has been exhausted before statement completion"). `OPTION (MAXRECURSION 1000)` raises the cap, and the query returns 672 dates. Note that the anchor can use aggregates such as `MIN` and `MAX`, but the recursive member cannot.

When the cap is reached, SQL Server terminates the statement with an error; rows already sent to the client may still appear, so do not treat partial output as a result. `MAXRECURSION 0` removes the limit, which is risky. The best practice is a stop condition in the query itself (here, `WHERE dt < last_dt`), with `MAXRECURSION` set just above the expected depth as a safety net. PostgreSQL has no built-in cap, so the stop condition is the only protection. MySQL offers `cte_max_recursion_depth`, which defaults to 1000.
