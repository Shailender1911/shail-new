# File 9: SQL Deep Practice

> Airtel R1 typically has 1-2 SQL problems. These cover the patterns that come up.
> Memorize the templates; the variations are minor.

---

## SECTION 1 — TOP-N PER GROUP (most asked)

### Q1. Second highest salary
```sql
-- With duplicates handled (DENSE_RANK)
SELECT salary
FROM (
  SELECT salary, DENSE_RANK() OVER (ORDER BY salary DESC) AS rk
  FROM employees
) t
WHERE rk = 2;

-- Without duplicates (subquery — Airtel often asks this version)
SELECT MAX(salary) AS second_max
FROM employees
WHERE salary < (SELECT MAX(salary) FROM employees);

-- Nth highest (parameterized)
SELECT salary
FROM (
  SELECT salary, DENSE_RANK() OVER (ORDER BY salary DESC) AS rk
  FROM employees
) t
WHERE rk = N;
```

### Q2. Highest-paid employee per department
```sql
SELECT department_id, employee_id, salary
FROM (
  SELECT *,
         ROW_NUMBER() OVER (PARTITION BY department_id ORDER BY salary DESC) AS rn
  FROM employees
) t
WHERE rn = 1;
```

### Q3. Top 3 paid per department
```sql
SELECT department_id, employee_id, salary
FROM (
  SELECT *,
         DENSE_RANK() OVER (PARTITION BY department_id ORDER BY salary DESC) AS rk
  FROM employees
) t
WHERE rk <= 3;
```

**Window function vocab to drop:**
- `ROW_NUMBER()` — unique sequential, ties broken arbitrarily.
- `RANK()` — same rank for ties, gaps after (1, 2, 2, 4).
- `DENSE_RANK()` — same rank for ties, no gaps (1, 2, 2, 3).

---

## SECTION 2 — JOINS

### Q4. Employees who never made a sale
```sql
SELECT e.employee_id, e.name
FROM employees e
LEFT JOIN sales s ON s.employee_id = e.employee_id
WHERE s.id IS NULL;

-- Equivalent with NOT EXISTS (often faster on indexed FK)
SELECT e.employee_id, e.name
FROM employees e
WHERE NOT EXISTS (
  SELECT 1 FROM sales s WHERE s.employee_id = e.employee_id
);
```

### Q5. Customers with ≥3 orders in last 30 days
```sql
SELECT c.customer_id, c.name, COUNT(*) AS order_count
FROM customers c
JOIN orders o ON o.customer_id = c.customer_id
WHERE o.created_at >= NOW() - INTERVAL 30 DAY
GROUP BY c.customer_id, c.name
HAVING COUNT(*) >= 3;
```

### Q6. Self-join — manager hierarchy
```sql
SELECT e.name AS employee, m.name AS manager
FROM employees e
LEFT JOIN employees m ON m.employee_id = e.manager_id;
```

---

## SECTION 3 — AGGREGATES + GROUPING

### Q7. Department-wise average salary, only depts with avg > 50k
```sql
SELECT department_id, AVG(salary) AS avg_sal
FROM employees
GROUP BY department_id
HAVING AVG(salary) > 50000;
```
**Trap they test:** WHERE filters rows BEFORE grouping; HAVING filters AFTER.

### Q8. Running total of monthly sales
```sql
SELECT
  DATE_FORMAT(order_date, '%Y-%m') AS month,
  SUM(amount) AS monthly_sales,
  SUM(SUM(amount)) OVER (ORDER BY DATE_FORMAT(order_date, '%Y-%m')) AS running_total
FROM orders
GROUP BY DATE_FORMAT(order_date, '%Y-%m')
ORDER BY month;
```

### Q9. Month-over-month growth
```sql
SELECT
  month,
  monthly_sales,
  LAG(monthly_sales, 1) OVER (ORDER BY month) AS prev_month,
  (monthly_sales - LAG(monthly_sales, 1) OVER (ORDER BY month)) * 100.0
    / NULLIF(LAG(monthly_sales, 1) OVER (ORDER BY month), 0) AS growth_pct
FROM monthly_sales_view;
```

### Q10. Top customer's % of total revenue
```sql
SELECT
  customer_id,
  total,
  total * 100.0 / SUM(total) OVER () AS pct_of_total
FROM (
  SELECT customer_id, SUM(amount) AS total
  FROM orders GROUP BY customer_id
) t
ORDER BY total DESC
LIMIT 1;
```

---

## SECTION 4 — DUPLICATES + DEDUP

### Q11. Find duplicate emails
```sql
SELECT email, COUNT(*) AS cnt
FROM users
GROUP BY email
HAVING COUNT(*) > 1;
```

### Q12. Delete duplicates, keep lowest id
```sql
DELETE u1
FROM users u1
JOIN users u2 ON u2.email = u1.email AND u2.id < u1.id;

-- Equivalent with window function (Postgres / MySQL 8+)
WITH ranked AS (
  SELECT id, ROW_NUMBER() OVER (PARTITION BY email ORDER BY id) AS rn
  FROM users
)
DELETE FROM users WHERE id IN (SELECT id FROM ranked WHERE rn > 1);
```

---

## SECTION 5 — DATE/TIME

### Q13. Daily active users for last 7 days
```sql
SELECT DATE(login_at) AS day, COUNT(DISTINCT user_id) AS dau
FROM logins
WHERE login_at >= CURRENT_DATE - INTERVAL 7 DAY
GROUP BY DATE(login_at)
ORDER BY day;
```

### Q14. Users active in 3 consecutive days
```sql
WITH user_days AS (
  SELECT DISTINCT user_id, DATE(login_at) AS d
  FROM logins
),
gaps AS (
  SELECT user_id, d,
         d - INTERVAL ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY d) DAY AS grp
  FROM user_days
)
SELECT user_id
FROM gaps
GROUP BY user_id, grp
HAVING COUNT(*) >= 3;
```
**Pattern:** "consecutive days" → subtract a row number to collapse a streak into a single group.

---

## SECTION 6 — INDEXING SCENARIOS (interview talking points)

### Q15. Why is this query slow?
```sql
SELECT * FROM orders WHERE DATE(created_at) = '2024-01-15';
```
**Answer:** `DATE()` function on indexed column kills the index. Rewrite as range:
```sql
SELECT * FROM orders
WHERE created_at >= '2024-01-15 00:00:00'
  AND created_at <  '2024-01-16 00:00:00';
```

### Q16. When is composite index `(a, b)` used?
- ✅ `WHERE a = ?` (uses prefix)
- ✅ `WHERE a = ? AND b = ?` (uses both)
- ❌ `WHERE b = ?` (skips a — index unusable)
- ✅ `WHERE a = ? ORDER BY b` (uses index for sort)

### Q17. EXPLAIN output — what to look for
- **type:** `const` > `eq_ref` > `ref` > `range` > `index` > `ALL` (worst).
- **rows:** estimated rows scanned. Lower is better.
- **key:** index used (NULL = no index).
- **Extra:** `Using index` (covering index — best); `Using filesort` / `Using temporary` (red flags).

### Q18. Covering index
```sql
-- Index covers all selected columns; engine never reads the table
CREATE INDEX idx_orders_cover ON orders(customer_id, status, created_at, total);

SELECT customer_id, status, created_at, total
FROM orders
WHERE customer_id = 123 AND status = 'PAID';
-- EXPLAIN shows "Using index"
```

---

## SECTION 7 — REAL-WORLD PATTERNS (your projects)

### Q19. Application stage transitions in last 24h (DLS-style)
```sql
SELECT application_id, stage, created_at
FROM application_tracker
WHERE active = 1
  AND created_at >= NOW() - INTERVAL 24 HOUR
ORDER BY application_id, created_at;
```

### Q20. Mandate retry candidates (loan-repayment-style)
```sql
SELECT m.mandate_id, m.last_attempt_at, COUNT(a.id) AS attempts
FROM mandates m
LEFT JOIN mandate_attempts a ON a.mandate_id = m.mandate_id AND a.status = 'FAILED'
WHERE m.status = 'PENDING_RETRY'
  AND m.last_attempt_at < NOW() - INTERVAL 1 HOUR
GROUP BY m.mandate_id, m.last_attempt_at
HAVING COUNT(a.id) < 5;
```

---

## INTERVIEW TALKING POINTS (drop when relevant)

- **"How do you optimize a slow query?"** → EXPLAIN first; identify type (`ALL` = full scan); add covering index; rewrite to keep index usable; avoid functions on indexed columns; check stats are up to date (`ANALYZE TABLE`).
- **"When would you denormalize?"** → Read-heavy + join-heavy + low-write workloads (e.g. reporting); accept duplication for read latency.
- **"GROUP BY vs window function?"** → GROUP BY collapses rows; window keeps row + adds aggregate column. Window is right when you need both the detail and the aggregate.
- **"Subquery vs JOIN?"** → Modern optimizers (MySQL 8+, Postgres) often plan them identically. Pick the one more readable. EXISTS often beats IN for large subqueries.
- **"WHERE vs HAVING?"** → WHERE filters rows before aggregation; HAVING filters groups after. Use WHERE when you can — it filters earlier.

---

## TEMPLATE FOR LIVE SQL CODING

> "Let me think through this. The output needs [X]. Source data is in [tables]. I'll need a join on [key], then [aggregate / window]. Let me write a first version… *(write)* …let me trace it: for input row Y, this produces Z. Edge case: if there are duplicates, ROW_NUMBER picks one arbitrarily — if we want deterministic, add a tiebreaker. To make this fast, an index on `(join_key, filter_col)` would help."

The "let me trace it" + "edge case" + "to make this fast" pattern is what scores points beyond just-a-correct-query.
