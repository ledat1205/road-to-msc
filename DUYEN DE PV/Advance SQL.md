## 1. Top N Per Category (Window Functions)

**Scenario:** Find the top 2 highest-paid employees in each department. _Key Technique:_ `DENSE_RANK()` or `ROW_NUMBER()` inside a CTE.

SQL

```
WITH RankedSalary AS (
    SELECT 
        employee_name,
        department_id,
        salary,
        DENSE_RANK() OVER (PARTITION BY department_id ORDER BY salary DESC) as rnk
    FROM employees
)
SELECT * FROM RankedSalary 
WHERE rnk <= 2;
```

> **Interview Tip:** Use `DENSE_RANK()` if you want to include all employees who tied for 2nd place. Use `ROW_NUMBER()` if you need exactly 2 rows regardless of ties.

---

## 2. Gaps and Islands (Advanced Sequencing)

**Scenario:** Find "islands" of consecutive days a user logged in. _Key Technique:_ Subtracting a sequence (`ROW_NUMBER`) from the date to create a constant "grouping" value.

SQL

```
WITH DateGroups AS (
    SELECT 
        user_id,
        login_date,
        -- If dates are consecutive, the difference stays constant
        login_date - INTERVAL '1 day' * ROW_NUMBER() OVER(PARTITION BY user_id ORDER BY login_date) as grp
    FROM user_logins
)
SELECT 
    user_id, 
    MIN(login_date) as start_streak, 
    MAX(login_date) as end_streak,
    COUNT(*) as consecutive_days
FROM DateGroups
GROUP BY user_id, grp
HAVING COUNT(*) >= 3; -- Streaks of 3+ days
```

---

## 3. Handling Delta Changes (LEAD/LAG)

**Scenario:** Calculate the percentage growth in revenue compared to the previous month. _Key Technique:_ `LAG()` to pull data from the previous row without a self-join.

SQL

```
SELECT 
    month,
    revenue,
    LAG(revenue) OVER (ORDER BY month) as prev_month_revenue,
    (revenue - LAG(revenue) OVER (ORDER BY month)) / NULLIF(LAG(revenue) OVER (ORDER BY month), 0) * 100 as growth_pct
FROM monthly_sales;
```

> **Interview Tip:** Always use `NULLIF(..., 0)` when dividing to avoid "Division by zero" errors—this shows you have a production-ready mindset.

---

## 4. De-duplication (Data Cleaning)

**Scenario:** You have a table with duplicate `order_id` entries. Keep only the most recent entry for each ID. _Key Technique:_ CTE with `ROW_NUMBER()`.

SQL

```
WITH DeDup AS (
    SELECT 
        *,
        ROW_NUMBER() OVER (PARTITION BY order_id ORDER BY updated_at DESC) as row_num
    FROM orders
)
-- In an interview, you'd explain you'd DELETE where row_num > 1
SELECT * FROM DeDup 
WHERE row_num = 1;
```

---

## 5. Pivoting Data (CASE WHEN)

**Scenario:** Transform a narrow table (rows for each quarter) into a wide table (columns for each quarter).

SQL

```
SELECT 
    product_id,
    SUM(CASE WHEN quarter = 'Q1' THEN revenue ELSE 0 END) as q1_rev,
    SUM(CASE WHEN quarter = 'Q2' THEN revenue ELSE 0 END) as q2_rev,
    SUM(CASE WHEN quarter = 'Q3' THEN revenue ELSE 0 END) as q3_rev,
    SUM(CASE WHEN quarter = 'Q4' THEN revenue ELSE 0 END) as q4_rev
FROM sales
GROUP BY product_id;
```

---

## 6. Recursive CTEs (Hierarchies)

**Scenario:** Find all subordinates of a specific Manager in an organizational chart.

SQL

```
WITH RECURSIVE OrgChart AS (
    -- Anchor member: The Manager
    SELECT employee_id, name, manager_id, 1 as level
    FROM employees 
    WHERE employee_id = 101 
    
    UNION ALL
    
    -- Recursive member: Find people reporting to those in OrgChart
    SELECT e.employee_id, e.name, e.manager_id, oc.level + 1
    FROM employees e
    JOIN OrgChart oc ON e.manager_id = oc.employee_id
)
SELECT * FROM OrgChart;
```

---