# SQL Interview Q&A — Comprehensive Guide for SDE-2

---

## 1. Joins

### Setup Tables Used Throughout

```sql
CREATE TABLE employees (
    emp_id   INT PRIMARY KEY,
    name     VARCHAR(50),
    dept_id  INT
);

CREATE TABLE departments (
    dept_id   INT PRIMARY KEY,
    dept_name VARCHAR(50)
);

INSERT INTO employees VALUES
(1, 'Alice',   10),
(2, 'Bob',     20),
(3, 'Charlie', 30),
(4, 'Diana',   NULL);

INSERT INTO departments VALUES
(10, 'Engineering'),
(20, 'Marketing'),
(40, 'Finance');
```

**Quick reference:**

```
employees                   departments
+--------+---------+---------+   +---------+-------------+
| emp_id | name    | dept_id |   | dept_id | dept_name   |
+--------+---------+---------+   +---------+-------------+
| 1      | Alice   | 10      |   | 10      | Engineering |
| 2      | Bob     | 20      |   | 20      | Marketing   |
| 3      | Charlie | 30      |   | 40      | Finance     |
| 4      | Diana   | NULL    |   +---------+-------------+
+--------+---------+---------+
```

---

### Q1: What is an INNER JOIN?

Returns only rows where there is a match in **both** tables.

```
  Table A       Table B
  +-----+       +-----+
  |     |       |     |
  |  +--+--+--+-+--+  |
  |  |  |XXXXX|  |  |  
  |  +--+--+--+-+--+  |
  |     |       |     |
  +-----+       +-----+
        ^^^^^^^^^^^
        Result set
```

```sql
SELECT e.name, d.dept_name
FROM employees e
INNER JOIN departments d ON e.dept_id = d.dept_id;
```

| name  | dept_name   |
|-------|-------------|
| Alice | Engineering |
| Bob   | Marketing   |

Charlie (dept_id=30) has no matching department. Diana (dept_id=NULL) has no match either.

---

### Q2: What is a LEFT JOIN (LEFT OUTER JOIN)?

Returns **all rows from the left table** and matched rows from the right. Unmatched right-side columns become NULL.

```
  Table A       Table B
  +-----+       +-----+
  |XXXXX|       |     |
  |XX+--+--+--+-+--+  |
  |XX|XX|XXXXX|  |  |  
  |XX+--+--+--+-+--+  |
  |XXXXX|       |     |
  +-----+       +-----+
```

```sql
SELECT e.name, d.dept_name
FROM employees e
LEFT JOIN departments d ON e.dept_id = d.dept_id;
```

| name    | dept_name   |
|---------|-------------|
| Alice   | Engineering |
| Bob     | Marketing   |
| Charlie | NULL        |
| Diana   | NULL        |

---

### Q3: What is a RIGHT JOIN (RIGHT OUTER JOIN)?

Returns **all rows from the right table** and matched rows from the left. Unmatched left-side columns become NULL.

```
  Table A       Table B
  +-----+       +-----+
  |     |       |XXXXX|
  |  +--+--+--+-+--+XX|
  |  |  |XXXXX|XX|XX|XX|
  |  +--+--+--+-+--+XX|
  |     |       |XXXXX|
  +-----+       +-----+
```

```sql
SELECT e.name, d.dept_name
FROM employees e
RIGHT JOIN departments d ON e.dept_id = d.dept_id;
```

| name  | dept_name   |
|-------|-------------|
| Alice | Engineering |
| Bob   | Marketing   |
| NULL  | Finance     |

---

### Q4: What is a FULL OUTER JOIN?

Returns all rows from **both** tables. Where there's no match, NULLs fill in.

```
  Table A       Table B
  +-----+       +-----+
  |XXXXX|       |XXXXX|
  |XX+--+--+--+-+--+XX|
  |XX|XX|XXXXX|XX|XX|XX|
  |XX+--+--+--+-+--+XX|
  |XXXXX|       |XXXXX|
  +-----+       +-----+
```

```sql
SELECT e.name, d.dept_name
FROM employees e
FULL OUTER JOIN departments d ON e.dept_id = d.dept_id;
```

| name    | dept_name   |
|---------|-------------|
| Alice   | Engineering |
| Bob     | Marketing   |
| Charlie | NULL        |
| Diana   | NULL        |
| NULL    | Finance     |

> **Note:** MySQL doesn't support FULL OUTER JOIN directly. Use `LEFT JOIN UNION RIGHT JOIN`.

---

### Q5: What is a CROSS JOIN?

Produces the **Cartesian product** — every row of A paired with every row of B.  
Result size = rows(A) × rows(B).

```sql
SELECT e.name, d.dept_name
FROM employees e
CROSS JOIN departments d;
```

Produces 4 × 3 = **12 rows**. Rarely used intentionally; useful for generating test data or calendar grids.

---

### Q6: What is a SELF JOIN?

A table joined with **itself**. Common for hierarchical data (manager-employee) or comparing rows within the same table.

```sql
CREATE TABLE staff (
    id         INT PRIMARY KEY,
    name       VARCHAR(50),
    manager_id INT
);

INSERT INTO staff VALUES
(1, 'CEO',     NULL),
(2, 'VP Eng',  1),
(3, 'Alice',   2),
(4, 'Bob',     2);

SELECT s.name AS employee, m.name AS manager
FROM staff s
LEFT JOIN staff m ON s.manager_id = m.id;
```

| employee | manager |
|----------|---------|
| CEO      | NULL    |
| VP Eng   | CEO     |
| Alice    | VP Eng  |
| Bob      | VP Eng  |

---

### Q7: When to use which JOIN?

| Scenario | JOIN Type |
|----------|-----------|
| Only matching data from both tables | INNER JOIN |
| All records from the primary table, optional secondary | LEFT JOIN |
| Check if something exists in another table | LEFT JOIN + WHERE right.key IS NULL (anti-join) |
| Merge two datasets completely | FULL OUTER JOIN |
| Compare rows within the same table | SELF JOIN |
| Generate all combinations | CROSS JOIN |

### Common mistakes with JOINs

1. **Forgetting the ON clause** — turns an INNER JOIN into a CROSS JOIN in some dialects.
2. **Joining on nullable columns** — NULL ≠ NULL in SQL, so rows with NULLs won't match.
3. **Accidental Cartesian product** — missing join condition in multi-table queries.
4. **Using RIGHT JOIN unnecessarily** — almost always rewritable as LEFT JOIN by swapping table order, which is more readable.
5. **Not aliasing tables** — makes the query unreadable in multi-join queries.

---

## 2. Indexing

### Q8: How does a B-tree index work internally?

A B-tree (balanced tree) is the most common index structure in relational databases.

```
                        [50]                    <- Root node
                       /    \
                [20, 30]    [70, 90]            <- Internal nodes
               /   |   \   /   |   \
            [10] [25] [35] [60] [80] [95,99]   <- Leaf nodes (point to rows)
```

**Key properties:**
- **Balanced:** All leaf nodes are at the same depth → O(log N) lookups.
- **Sorted:** Leaf nodes are linked in order → efficient range scans.
- **Each node** holds multiple keys (fits a disk page, typically 4-16 KB).

**How a lookup works (find id = 25):**
1. Start at root → 25 < 50 → go left
2. Internal node [20, 30] → 20 ≤ 25 < 30 → go to middle child
3. Leaf node [25] → found! Follow pointer to actual row.

Disk I/O = tree height ≈ 3-4 for millions of rows.

---

### Q9: Clustered vs Non-clustered Index

| Property | Clustered Index | Non-clustered Index |
|----------|----------------|---------------------|
| Physical order | Data rows stored in index order | Separate structure pointing to data |
| Per table | Only **one** (usually the PK) | **Multiple** allowed |
| Leaf nodes contain | Actual data rows | Pointers (row locators) to data |
| Range queries | Very fast (sequential I/O) | May need "bookmark lookups" |
| Insert performance | Slower if not sequential | Independent of data order |

```
Clustered Index (on id):
  B-tree leaf → [actual row data]

Non-clustered Index (on email):
  B-tree leaf → [email value | pointer to row in clustered index]
```

> **MySQL InnoDB**: The primary key IS the clustered index. Secondary indexes store the PK value as the row locator, causing a "double lookup."

---

### Q10: What is a Composite Index and why does column order matter?

A composite index indexes multiple columns together. The **leftmost prefix rule** applies:

```sql
CREATE INDEX idx_dept_salary ON employees(dept_id, salary);
```

This index is useful for:
- `WHERE dept_id = 10` — YES (uses leftmost column)
- `WHERE dept_id = 10 AND salary > 50000` — YES (uses both)
- `WHERE salary > 50000` — NO (skips leftmost column)
- `ORDER BY dept_id, salary` — YES
- `ORDER BY salary, dept_id` — NO (wrong column order)

**Rule of thumb for column ordering:**
1. Equality columns first (`WHERE status = 'ACTIVE'`)
2. Range columns last (`WHERE created_at > '2024-01-01'`)
3. High-cardinality columns first (usually)

---

### Q11: What is a Covering Index?

An index that contains **all columns** the query needs — the database never reads the actual table row.

```sql
CREATE INDEX idx_covering ON orders(customer_id, order_date, total_amount);

-- This query is "covered" — no table lookup needed:
SELECT order_date, total_amount
FROM orders
WHERE customer_id = 123;
```

In EXPLAIN output, you'll see: `Using index` (MySQL) or `Index Only Scan` (PostgreSQL).

**Benefit:** Dramatically faster because leaf nodes of the index contain all needed data.

---

### Q12: When do indexes hurt performance?

1. **Write-heavy tables:** Every INSERT/UPDATE/DELETE must also update every index.
2. **Low-cardinality columns:** Index on `gender` (M/F) is often slower than a full table scan.
3. **Small tables:** Full scan of 100 rows is faster than index traversal overhead.
4. **Heavily updated columns:** Each update rewrites the index entry.
5. **Too many indexes:** Each additional index adds storage and write overhead.

**Guideline:** Profile with EXPLAIN before and after adding an index.

---

### Q13: How to check if a query uses an index?

```sql
-- MySQL
EXPLAIN SELECT * FROM orders WHERE customer_id = 123;

-- PostgreSQL
EXPLAIN ANALYZE SELECT * FROM orders WHERE customer_id = 123;
```

**Key columns in EXPLAIN output (MySQL):**

| Column | What to look for |
|--------|-----------------|
| type | `ref` or `range` (good) vs `ALL` (full scan, bad) |
| possible_keys | Which indexes the optimizer considered |
| key | Which index it actually chose |
| rows | Estimated rows to examine |
| Extra | `Using index` = covering index, `Using filesort` = bad for ORDER BY |

---

## 3. Query Optimization

### Q14: How to read EXPLAIN ANALYZE output?

```sql
EXPLAIN ANALYZE
SELECT d.dept_name, COUNT(*) as emp_count
FROM employees e
JOIN departments d ON e.dept_id = d.dept_id
GROUP BY d.dept_name;
```

Read EXPLAIN output **bottom-up** (innermost operation executes first):

```
Sort  (cost=... rows=... actual time=0.5..0.5 rows=2 loops=1)
  -> HashAggregate  (cost=... rows=2)
       -> Hash Join  (cost=... rows=4)
            -> Seq Scan on employees  (actual rows=4)
            -> Hash  (cost=... rows=3)
                 -> Seq Scan on departments  (actual rows=3)
```

**Key metrics:**
- `actual time`: First row time .. last row time (milliseconds)
- `rows`: Actual rows processed (compare with `estimated rows` for plan accuracy)
- `loops`: How many times this node executed

**Red flags:**
- `Seq Scan` on large tables → missing index
- Estimated rows ≪ actual rows → stale statistics, run `ANALYZE`
- `Sort` with high cost → consider index for ORDER BY

---

### Q15: Common query optimization techniques?

**1. Avoid SELECT * — select only needed columns:**
```sql
-- BAD
SELECT * FROM orders WHERE status = 'ACTIVE';

-- GOOD
SELECT order_id, total_amount FROM orders WHERE status = 'ACTIVE';
```
Reduces I/O and allows covering indexes.

**2. Use LIMIT for large result sets:**
```sql
SELECT order_id, total FROM orders
ORDER BY created_at DESC
LIMIT 20 OFFSET 0;
```

**3. Avoid functions on indexed columns:**
```sql
-- BAD: index on created_at is useless
SELECT * FROM orders WHERE YEAR(created_at) = 2024;

-- GOOD: range scan uses the index
SELECT * FROM orders WHERE created_at >= '2024-01-01' AND created_at < '2025-01-01';
```

**4. Use EXISTS instead of IN for correlated subqueries:**
```sql
-- Often slower
SELECT * FROM employees WHERE dept_id IN (SELECT dept_id FROM departments WHERE active = 1);

-- Often faster (short-circuits on first match)
SELECT * FROM employees e
WHERE EXISTS (SELECT 1 FROM departments d WHERE d.dept_id = e.dept_id AND d.active = 1);
```

**5. Avoid implicit type conversion:**
```sql
-- BAD: phone is VARCHAR, implicit cast disables index
SELECT * FROM users WHERE phone = 9876543210;

-- GOOD:
SELECT * FROM users WHERE phone = '9876543210';
```

---

### Q16: Subquery vs JOIN — which is faster?

**General rule:** JOINs are usually faster because the optimizer can choose the best join strategy. Subqueries (especially correlated) can execute once per outer row.

```sql
-- Correlated subquery: executes inner query N times
SELECT name, (SELECT dept_name FROM departments d WHERE d.dept_id = e.dept_id)
FROM employees e;

-- Equivalent JOIN: single pass
SELECT e.name, d.dept_name
FROM employees e
LEFT JOIN departments d ON e.dept_id = d.dept_id;
```

**Exceptions where subqueries can be faster:**
- `EXISTS` with early termination
- Subquery in `WHERE` with good indexes and small result sets
- Modern optimizers often rewrite subqueries into joins anyway

---

## 4. Transactions and Concurrency

### Q17: Explain ACID properties with examples.

**A — Atomicity:** All or nothing. If a bank transfer debits account A but fails before crediting account B, the entire transaction rolls back.

```sql
BEGIN;
UPDATE accounts SET balance = balance - 500 WHERE id = 1;  -- debit
UPDATE accounts SET balance = balance + 500 WHERE id = 2;  -- credit
-- If anything fails here, ROLLBACK undoes both
COMMIT;
```

**C — Consistency:** The database moves from one valid state to another. Constraints (FK, CHECK, UNIQUE) are enforced. No half-transferred money.

**I — Isolation:** Concurrent transactions don't interfere with each other. The level of isolation is configurable (see Q18).

**D — Durability:** Once committed, data survives crashes. Write-ahead log (WAL) ensures recovery.

---

### Q18: Explain transaction isolation levels and their anomalies.

| Isolation Level | Dirty Read | Non-Repeatable Read | Phantom Read | Performance |
|----------------|------------|---------------------|--------------|-------------|
| READ UNCOMMITTED | Possible | Possible | Possible | Fastest |
| READ COMMITTED | Not possible | Possible | Possible | Fast |
| REPEATABLE READ | Not possible | Not possible | Possible | Moderate |
| SERIALIZABLE | Not possible | Not possible | Not possible | Slowest |

**Dirty Read:** Reading uncommitted data from another transaction.
```
T1: UPDATE accounts SET balance = 0 WHERE id = 1;   -- not committed yet
T2: SELECT balance FROM accounts WHERE id = 1;       -- reads 0 (dirty!)
T1: ROLLBACK;                                         -- balance is actually unchanged
```

**Non-Repeatable Read:** Same query returns different data within one transaction.
```
T1: SELECT balance FROM accounts WHERE id = 1;  -- returns 1000
T2: UPDATE accounts SET balance = 500 WHERE id = 1; COMMIT;
T1: SELECT balance FROM accounts WHERE id = 1;  -- returns 500 (different!)
```

**Phantom Read:** A range query returns different rows because another transaction inserted/deleted rows.
```
T1: SELECT COUNT(*) FROM orders WHERE status = 'PENDING';  -- returns 5
T2: INSERT INTO orders(...) VALUES (...); COMMIT;
T1: SELECT COUNT(*) FROM orders WHERE status = 'PENDING';  -- returns 6 (phantom!)
```

**Defaults:** PostgreSQL = READ COMMITTED, MySQL InnoDB = REPEATABLE READ.

---

### Q19: What are deadlocks and how to handle them?

A deadlock occurs when two transactions each hold a lock the other needs.

```
T1: Locks row A, wants row B
T2: Locks row B, wants row A
→ Both wait forever
```

**Detection:** Databases maintain a wait-for graph. When a cycle is detected, one transaction is chosen as the **victim** and rolled back.

**Prevention strategies:**
1. **Consistent lock ordering:** Always lock resources in the same order (e.g., by primary key ascending).
2. **Lock timeout:** `SET innodb_lock_wait_timeout = 5;`
3. **Keep transactions short:** Minimize the time locks are held.
4. **Use optimistic locking:** Version columns instead of SELECT FOR UPDATE.

```sql
-- Optimistic locking pattern
UPDATE orders
SET status = 'PROCESSED', version = version + 1
WHERE id = 123 AND version = 5;
-- If affected_rows = 0 → concurrent modification detected, retry
```

---

### Q20: What is connection pooling (HikariCP)?

Opening a database connection is expensive (~5-10ms for TCP + authentication). A connection pool pre-creates and reuses connections.

**HikariCP** is the fastest Java connection pool (default in Spring Boot).

```yaml
# application.yml
spring:
  datasource:
    hikari:
      maximum-pool-size: 10        # max connections in pool
      minimum-idle: 5              # min idle connections kept alive
      connection-timeout: 30000    # max wait for a connection (ms)
      idle-timeout: 600000         # max idle time before eviction (ms)
      max-lifetime: 1800000        # max connection lifetime (ms)
```

**Sizing formula (from HikariCP wiki):**
```
pool_size = (core_count * 2) + effective_spindle_count
```
For a 4-core machine with SSD: pool_size ≈ 10.

**Common mistakes:**
- Pool too large → database overwhelmed, context switching overhead.
- Pool too small → threads wait for connections, throughput drops.
- Not closing connections (returning to pool) → pool exhaustion.

---

## 5. Window Functions

### Q21: ROW_NUMBER vs RANK vs DENSE_RANK

```sql
CREATE TABLE scores (name VARCHAR(20), score INT);
INSERT INTO scores VALUES
('Alice', 95), ('Bob', 90), ('Charlie', 90), ('Diana', 85), ('Eve', 80);
```

```sql
SELECT name, score,
    ROW_NUMBER() OVER (ORDER BY score DESC) AS row_num,
    RANK()       OVER (ORDER BY score DESC) AS rnk,
    DENSE_RANK() OVER (ORDER BY score DESC) AS dense_rnk
FROM scores;
```

| name    | score | row_num | rnk | dense_rnk |
|---------|-------|---------|-----|-----------|
| Alice   | 95    | 1       | 1   | 1         |
| Bob     | 90    | 2       | 2   | 2         |
| Charlie | 90    | 3       | 2   | 2         |
| Diana   | 85    | 4       | **4** | **3**   |
| Eve     | 80    | 5       | 5   | 4         |

- **ROW_NUMBER:** Always unique, sequential (1, 2, 3, 4, 5).
- **RANK:** Ties get the same rank, **skips** next (1, 2, 2, **4**, 5).
- **DENSE_RANK:** Ties get the same rank, **no skip** (1, 2, 2, **3**, 4).

---

### Q22: LAG and LEAD functions

```sql
SELECT
    month,
    revenue,
    LAG(revenue, 1)  OVER (ORDER BY month) AS prev_month_rev,
    LEAD(revenue, 1) OVER (ORDER BY month) AS next_month_rev,
    revenue - LAG(revenue, 1) OVER (ORDER BY month) AS month_over_month_change
FROM monthly_revenue;
```

| month   | revenue | prev_month_rev | next_month_rev | month_over_month_change |
|---------|---------|---------------|----------------|------------------------|
| 2024-01 | 10000   | NULL          | 12000          | NULL                   |
| 2024-02 | 12000   | 10000         | 11000          | 2000                   |
| 2024-03 | 11000   | 12000         | 15000          | -1000                  |
| 2024-04 | 15000   | 11000         | NULL           | 4000                   |

---

### Q23: Running totals and moving averages

```sql
-- Running total
SELECT
    order_date,
    amount,
    SUM(amount) OVER (ORDER BY order_date) AS running_total
FROM orders;

-- Running total per customer
SELECT
    customer_id,
    order_date,
    amount,
    SUM(amount) OVER (PARTITION BY customer_id ORDER BY order_date) AS customer_running_total
FROM orders;

-- 3-month moving average
SELECT
    month,
    revenue,
    AVG(revenue) OVER (
        ORDER BY month
        ROWS BETWEEN 2 PRECEDING AND CURRENT ROW
    ) AS moving_avg_3m
FROM monthly_revenue;
```

**Frame specification:**
- `ROWS BETWEEN 2 PRECEDING AND CURRENT ROW` → current row + 2 rows before
- `ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW` → all rows up to current (default for running total)
- `RANGE BETWEEN INTERVAL '7' DAY PRECEDING AND CURRENT ROW` → date-based window

---

## 6. Practice Problems

### Problem 1: Find the 2nd highest salary

**Setup:**
```sql
CREATE TABLE employees (id INT, name VARCHAR(50), salary INT);
INSERT INTO employees VALUES
(1,'Alice',90000),(2,'Bob',80000),(3,'Charlie',90000),
(4,'Diana',70000),(5,'Eve',60000);
```

**Approach 1: Subquery**
```sql
SELECT MAX(salary) AS second_highest
FROM employees
WHERE salary < (SELECT MAX(salary) FROM employees);
-- Result: 80000
```

**Approach 2: DENSE_RANK**
```sql
SELECT salary AS second_highest FROM (
    SELECT salary, DENSE_RANK() OVER (ORDER BY salary DESC) AS rnk
    FROM employees
) ranked
WHERE rnk = 2;
-- Result: 80000
```

**Approach 3: LIMIT + OFFSET**
```sql
SELECT DISTINCT salary
FROM employees
ORDER BY salary DESC
LIMIT 1 OFFSET 1;
-- Result: 80000
```

**Approach 4: Generalized Nth highest**
```sql
SELECT salary FROM (
    SELECT DISTINCT salary FROM employees ORDER BY salary DESC LIMIT N
) AS top_n
ORDER BY salary ASC
LIMIT 1;
```

---

### Problem 2: Employees earning more than their manager

**Setup:**
```sql
CREATE TABLE emp (id INT, name VARCHAR(50), salary INT, manager_id INT);
INSERT INTO emp VALUES
(1,'CEO',150000,NULL),(2,'VP',120000,1),
(3,'Alice',130000,2),(4,'Bob',80000,2);
```

**Query:**
```sql
SELECT e.name AS employee, e.salary AS emp_salary,
       m.name AS manager, m.salary AS mgr_salary
FROM emp e
JOIN emp m ON e.manager_id = m.id
WHERE e.salary > m.salary;
```

| employee | emp_salary | manager | mgr_salary |
|----------|-----------|---------|------------|
| Alice    | 130000    | VP      | 120000     |

**Explanation:** Self-join links each employee to their manager, then filters where employee's salary exceeds manager's.

---

### Problem 3: Find duplicate records

**Setup:**
```sql
CREATE TABLE contacts (id INT, email VARCHAR(100), name VARCHAR(50));
INSERT INTO contacts VALUES
(1,'a@x.com','Alice'),(2,'b@x.com','Bob'),
(3,'a@x.com','Alice2'),(4,'c@x.com','Charlie'),(5,'b@x.com','Bob2');
```

**Query (find duplicate emails):**
```sql
SELECT email, COUNT(*) AS cnt
FROM contacts
GROUP BY email
HAVING COUNT(*) > 1;
```

| email   | cnt |
|---------|-----|
| a@x.com | 2   |
| b@x.com | 2   |

**To see all rows with duplicate emails:**
```sql
SELECT c.*
FROM contacts c
WHERE c.email IN (
    SELECT email FROM contacts GROUP BY email HAVING COUNT(*) > 1
);
```

---

### Problem 4: Running total query

**Setup:**
```sql
CREATE TABLE transactions (id INT, txn_date DATE, amount DECIMAL(10,2));
INSERT INTO transactions VALUES
(1,'2024-01-01',100),(2,'2024-01-02',200),
(3,'2024-01-03',150),(4,'2024-01-04',300);
```

**Query:**
```sql
SELECT
    id,
    txn_date,
    amount,
    SUM(amount) OVER (ORDER BY txn_date) AS running_total
FROM transactions;
```

| id | txn_date   | amount | running_total |
|----|-----------|--------|---------------|
| 1  | 2024-01-01 | 100.00 | 100.00        |
| 2  | 2024-01-02 | 200.00 | 300.00        |
| 3  | 2024-01-03 | 150.00 | 450.00        |
| 4  | 2024-01-04 | 300.00 | 750.00        |

---

### Problem 5: Department-wise top 3 salaries

**Setup:**
```sql
CREATE TABLE emp_salary (id INT, name VARCHAR(50), dept VARCHAR(30), salary INT);
INSERT INTO emp_salary VALUES
(1,'Alice','Eng',90000),(2,'Bob','Eng',85000),(3,'Charlie','Eng',80000),
(4,'Diana','Eng',75000),(5,'Eve','Sales',70000),(6,'Frank','Sales',65000),
(7,'Grace','Sales',60000),(8,'Hank','Sales',55000);
```

**Query:**
```sql
SELECT dept, name, salary FROM (
    SELECT
        dept, name, salary,
        DENSE_RANK() OVER (PARTITION BY dept ORDER BY salary DESC) AS rnk
    FROM emp_salary
) ranked
WHERE rnk <= 3;
```

| dept  | name    | salary |
|-------|---------|--------|
| Eng   | Alice   | 90000  |
| Eng   | Bob     | 85000  |
| Eng   | Charlie | 80000  |
| Sales | Eve     | 70000  |
| Sales | Frank   | 65000  |
| Sales | Grace   | 60000  |

**Why DENSE_RANK?** If two people share rank 3, both are included. RANK would skip rank 4, potentially missing valid results. ROW_NUMBER would arbitrarily exclude one.

---

### Problem 6: Find gaps in sequential IDs

**Setup:**
```sql
CREATE TABLE orders (order_id INT);
INSERT INTO orders VALUES (1),(2),(3),(5),(6),(9),(10);
-- Gaps: 4, 7, 8
```

**Query (find start of each gap):**
```sql
SELECT
    order_id + 1 AS gap_start,
    next_id - 1 AS gap_end
FROM (
    SELECT order_id, LEAD(order_id) OVER (ORDER BY order_id) AS next_id
    FROM orders
) with_next
WHERE next_id - order_id > 1;
```

| gap_start | gap_end |
|-----------|---------|
| 4         | 4       |
| 7         | 8       |

**Explanation:** LEAD gives us the next order_id. If the difference is > 1, there's a gap.

---

### Problem 7: Pivot rows to columns

**Setup:**
```sql
CREATE TABLE sales (product VARCHAR(20), quarter VARCHAR(5), revenue INT);
INSERT INTO sales VALUES
('Widget','Q1',100),('Widget','Q2',150),('Widget','Q3',200),('Widget','Q4',250),
('Gadget','Q1',300),('Gadget','Q2',350),('Gadget','Q3',400),('Gadget','Q4',450);
```

**Query (conditional aggregation):**
```sql
SELECT
    product,
    SUM(CASE WHEN quarter = 'Q1' THEN revenue END) AS Q1,
    SUM(CASE WHEN quarter = 'Q2' THEN revenue END) AS Q2,
    SUM(CASE WHEN quarter = 'Q3' THEN revenue END) AS Q3,
    SUM(CASE WHEN quarter = 'Q4' THEN revenue END) AS Q4
FROM sales
GROUP BY product;
```

| product | Q1  | Q2  | Q3  | Q4  |
|---------|-----|-----|-----|-----|
| Widget  | 100 | 150 | 200 | 250 |
| Gadget  | 300 | 350 | 400 | 450 |

---

### Problem 8: Year-over-year comparison using LAG

**Setup:**
```sql
CREATE TABLE yearly_revenue (year INT, revenue BIGINT);
INSERT INTO yearly_revenue VALUES
(2021,1000000),(2022,1200000),(2023,1500000),(2024,1350000);
```

**Query:**
```sql
SELECT
    year,
    revenue,
    LAG(revenue) OVER (ORDER BY year) AS prev_year_revenue,
    ROUND(
        (revenue - LAG(revenue) OVER (ORDER BY year)) * 100.0
        / LAG(revenue) OVER (ORDER BY year), 2
    ) AS yoy_growth_pct
FROM yearly_revenue;
```

| year | revenue  | prev_year_revenue | yoy_growth_pct |
|------|---------|-------------------|----------------|
| 2021 | 1000000 | NULL              | NULL           |
| 2022 | 1200000 | 1000000           | 20.00          |
| 2023 | 1500000 | 1200000           | 25.00          |
| 2024 | 1350000 | 1500000           | -10.00         |

---

### Problem 9: Finding consecutive days with activity

**Setup:**
```sql
CREATE TABLE logins (user_id INT, login_date DATE);
INSERT INTO logins VALUES
(1,'2024-01-01'),(1,'2024-01-02'),(1,'2024-01-03'),
(1,'2024-01-05'),(1,'2024-01-06'),
(2,'2024-01-01'),(2,'2024-01-03');
```

**Find users with 3+ consecutive login days:**

```sql
WITH numbered AS (
    SELECT
        user_id,
        login_date,
        login_date - INTERVAL ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY login_date) DAY AS grp
    FROM (SELECT DISTINCT user_id, login_date FROM logins) dl
)
SELECT user_id, MIN(login_date) AS streak_start, MAX(login_date) AS streak_end,
       COUNT(*) AS streak_length
FROM numbered
GROUP BY user_id, grp
HAVING COUNT(*) >= 3;
```

| user_id | streak_start | streak_end | streak_length |
|---------|-------------|------------|---------------|
| 1       | 2024-01-01  | 2024-01-03 | 3             |

**Technique:** Subtracting a sequential row number from the date creates identical "group" values for consecutive dates. Non-consecutive dates produce different groups.

---

### Problem 10: Recursive CTE for organizational hierarchy

**Setup:**
```sql
CREATE TABLE org (
    emp_id INT PRIMARY KEY,
    name   VARCHAR(50),
    mgr_id INT
);

INSERT INTO org VALUES
(1,'CEO',NULL),
(2,'CTO',1),(3,'CFO',1),
(4,'Eng Lead',2),(5,'Sr Dev',4),(6,'Jr Dev',5),
(7,'Fin Lead',3);
```

**Query — full org tree with depth:**
```sql
WITH RECURSIVE org_tree AS (
    -- Anchor: start from the CEO (no manager)
    SELECT emp_id, name, mgr_id, 0 AS depth,
           CAST(name AS VARCHAR(500)) AS path
    FROM org
    WHERE mgr_id IS NULL

    UNION ALL

    -- Recursive: find direct reports
    SELECT o.emp_id, o.name, o.mgr_id, t.depth + 1,
           CAST(CONCAT(t.path, ' > ', o.name) AS VARCHAR(500))
    FROM org o
    JOIN org_tree t ON o.mgr_id = t.emp_id
)
SELECT
    REPEAT('  ', depth) || name AS org_chart,
    depth,
    path
FROM org_tree
ORDER BY path;
```

| org_chart      | depth | path                              |
|----------------|-------|-----------------------------------|
| CEO            | 0     | CEO                               |
|   CFO          | 1     | CEO > CFO                         |
|     Fin Lead   | 2     | CEO > CFO > Fin Lead              |
|   CTO          | 1     | CEO > CTO                         |
|     Eng Lead   | 2     | CEO > CTO > Eng Lead              |
|       Sr Dev   | 3     | CEO > CTO > Eng Lead > Sr Dev     |
|         Jr Dev | 4     | CEO > CTO > Eng Lead > Sr Dev > Jr Dev |

**How recursive CTEs work:**
1. **Anchor member** runs once → produces base rows (CEO).
2. **Recursive member** runs repeatedly, joining new results with the CTE, until no new rows are produced.
3. Results from all iterations are combined with UNION ALL.

**Use cases in real world:**
- Category trees (ecommerce)
- BOM (bill of materials) explosion
- Folder/file hierarchies
- Social network friend-of-friend queries
