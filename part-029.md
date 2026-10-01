# ภาค 29: JOIN Performance และ Optimization

## ทำไม JOIN Performance ถึงสำคัญ?

เมื่อตารางมีข้อมูลหลายล้านแถว การ JOIN ที่ไม่มีประสิทธิภาพอาจทำให้ query ช้าเป็นนาทีหรือชั่วโมง บทนี้จะสอนวิธีทำให้ JOIN เร็วขึ้น

```
ตัวอย่างปัญหา:
- ตาราง orders: 10 ล้านแถว
- ตาราง order_items: 50 ล้านแถว
- JOIN ที่ไม่ optimize: ใช้เวลา 10 นาที
- หลัง optimize: ใช้เวลา 0.5 วินาที

ประเด็นหลัก:
1. ประเภท Join Algorithm ที่ DB เลือกใช้
2. Index บน JOIN columns
3. ลำดับการ JOIN
4. ขนาด result set ในแต่ละขั้น
```

---

## Join Algorithms

Database engines มีหลาย algorithms ในการ execute JOIN:

### 1. Nested Loop Join (NLJ)

```
Concept:
FOR each row in Table A (outer):
    FOR each row in Table B (inner):
        IF row_A.key == row_B.key:
            output combined row

Cost: O(n × m) — n rows in A, m rows in B
เหมาะสำหรับ:
- ตารางเล็ก
- มี index บน inner table
- Join บน few rows

ตัวอย่าง:
employees (50 rows) JOIN departments (10 rows)
= 50 × 10 = 500 comparisons → OK!

orders (1M rows) JOIN order_items (5M rows)
= 1M × 5M = 5 trillion comparisons → BAD!
```

### 2. Hash Join

```
Concept:
Phase 1 (Build): 
    สร้าง hash table จากตารางเล็ก (inner)
    FOR each row in Table B:
        insert into hash_table[hash(key)] = row

Phase 2 (Probe):
    FOR each row in Table A (outer):
        lookup hash_table[hash(key)]
        output matches

Cost: O(n + m) — linear!
เหมาะสำหรับ:
- ตารางใหญ่ไม่มี index
- Equi-join (= only)
- Full table scans

Memory requirement: ต้องการ memory สำหรับ hash table
```

### 3. Merge Join (Sort-Merge Join)

```
Concept:
Phase 1: Sort both tables on join key
Phase 2: Merge (like merge sort)
    pointer_A = start of sorted A
    pointer_B = start of sorted B
    WHILE not end of either:
        IF key_A == key_B: output, advance both
        IF key_A < key_B: advance A
        IF key_A > key_B: advance B

Cost: O(n log n + m log m) for sort + O(n + m) for merge
เหมาะสำหรับ:
- ข้อมูลที่ sort แล้ว (มี index sorted)
- Range joins (>, <, BETWEEN)
- Large tables สองตาราง
```

```sql
-- ดู Query Plan: EXPLAIN ANALYZE
EXPLAIN ANALYZE
SELECT e.first_name, d.dept_name
FROM employees e
JOIN departments d ON e.dept_id = d.dept_id;

-- ตัวอย่าง Output (PostgreSQL):
-- Hash Join  (cost=1.12..2.62 rows=50 width=23) (actual time=0.052..0.089 rows=50)
--   Hash Cond: (e.dept_id = d.dept_id)
--   ->  Seq Scan on employees e  (cost=0.00..1.50 rows=50 width=16)
--   ->  Hash  (cost=1.10..1.10 rows=10 width=15)
--         Buckets: 1024  Batches: 1  Memory Usage: 9kB
--         ->  Seq Scan on departments d  (cost=0.00..1.10 rows=10 width=15)
-- Planning Time: 0.1 ms
-- Execution Time: 0.2 ms
```

---

## Understanding EXPLAIN Output

### ตัวอย่างที่ 1: Reading EXPLAIN ANALYZE

```sql
-- Basic EXPLAIN
EXPLAIN SELECT e.first_name, d.dept_name
FROM employees e
JOIN departments d ON e.dept_id = d.dept_id;

/* Output คล้ายนี้:
Hash Join  (cost=1.12..2.62 rows=50 width=23)
  Hash Cond: (e.dept_id = d.dept_id)
  ->  Seq Scan on employees e  (cost=0.00..1.50 rows=50 width=16)
  ->  Hash  (cost=1.10..1.10 rows=10 width=15)
        ->  Seq Scan on departments d  (cost=0.00..1.10 rows=10 width=15)

อ่านผล:
- cost=start_cost..total_cost
- rows = estimated rows output
- width = estimated bytes per row
- Seq Scan = full table scan (ช้าสำหรับตารางใหญ่)
- Index Scan = ใช้ index (เร็วกว่า)
*/

-- EXPLAIN ANALYZE = รัน query จริง + แสดง actual times
EXPLAIN ANALYZE SELECT e.first_name, d.dept_name
FROM employees e
JOIN departments d ON e.dept_id = d.dept_id;

/* Output เพิ่มเติม:
Hash Join  (cost=1.12..2.62 rows=50 width=23) (actual time=0.052..0.089 rows=50 loops=1)
                                                ^^^^^^^^^^^^^^ actual execution info
Planning Time: 0.1 ms
Execution Time: 0.2 ms
*/
```

### ตัวอย่างที่ 2: EXPLAIN บน Complex JOIN

```sql
-- EXPLAIN สำหรับ 4-table JOIN
EXPLAIN ANALYZE
SELECT 
    o.order_id, c.first_name, p.product_name, oi.quantity
FROM orders o
JOIN customers c ON o.customer_id = c.customer_id
JOIN order_items oi ON o.order_id = oi.order_id
JOIN products p ON oi.product_id = p.product_id
WHERE o.status = 'completed';

/* ตัวอย่าง output:
Hash Join  (cost=...)  ← orders JOIN customers
  Hash Cond: (o.customer_id = c.customer_id)
  ->  Hash Join  (cost=...)  ← result JOIN order_items  
        Hash Cond: (oi.order_id = o.order_id)
        ->  Seq Scan on order_items oi
        ->  Hash
              ->  Seq Scan on orders o
                    Filter: ((status)::text = 'completed')
  ->  Hash
        ->  Hash Join  ← customers JOIN products (indirect)
              ...

Query Plan อ่านจาก INSIDE OUT (ในสุดก่อน)
*/
```

---

## Indexes สำหรับ JOIN

### ตัวอย่างที่ 3: สร้าง Index บน JOIN Columns

```sql
-- *** Index บน Foreign Key columns ช่วยมาก ***

-- ตรวจสอบ indexes ที่มีอยู่แล้ว (PostgreSQL)
SELECT 
    tablename,
    indexname,
    indexdef
FROM pg_indexes
WHERE tablename IN ('employees', 'orders', 'order_items')
ORDER BY tablename, indexname;

-- สร้าง indexes สำหรับ JOIN columns
-- Foreign Key: employees.dept_id → departments.dept_id
CREATE INDEX idx_employees_dept_id ON employees(dept_id);

-- Foreign Key: orders.customer_id → customers.customer_id
CREATE INDEX idx_orders_customer_id ON orders(customer_id);

-- Foreign Key: order_items.order_id → orders.order_id
CREATE INDEX idx_order_items_order_id ON order_items(order_id);

-- Foreign Key: order_items.product_id → products.product_id
CREATE INDEX idx_order_items_product_id ON order_items(product_id);

-- Composite index สำหรับ WHERE + JOIN
CREATE INDEX idx_orders_status_date ON orders(status, order_date);

-- ตรวจสอบหลังสร้าง index
EXPLAIN ANALYZE
SELECT o.order_id, c.first_name
FROM orders o
JOIN customers c ON o.customer_id = c.customer_id
WHERE o.status = 'completed';
/* ตอนนี้จะเห็น Index Scan แทน Seq Scan */
```

### ตัวอย่างที่ 4: Index Scan vs Seq Scan

```sql
-- Seq Scan (Sequential Scan) = อ่านทุก rows (ช้า)
-- Index Scan = ใช้ index ค้นหา (เร็ว)
-- Index Only Scan = ข้อมูลทั้งหมดอยู่ใน index แล้ว (เร็วที่สุด)
-- Bitmap Index Scan = ใช้ index + bitmap (กลาง)

-- ตัวอย่าง: ดูว่า query ใช้ index หรือไม่
EXPLAIN ANALYZE
SELECT e.emp_id, e.first_name
FROM employees e
WHERE e.dept_id = 1;  -- ถ้ามี index บน dept_id จะเป็น Index Scan

-- ถ้าไม่มี index:
-- Seq Scan on employees  (cost=0.00..1.50 rows=50)

-- ถ้ามี index:
-- Index Scan using idx_employees_dept_id on employees
--   Index Cond: (dept_id = 1)
```

---

## JOIN Order Matters!

### ตัวอย่างที่ 5: ลำดับ JOIN ส่งผลต่อ Performance

```sql
-- หลักการ: JOIN ตารางเล็กหรือตาราง filtered ก่อน
-- ลด intermediate result set

-- *** ไม่ดี: JOIN ตารางใหญ่ก่อน ***
-- สมมติ orders มี 1M rows, customers มี 100K rows
-- ใน real scenario
SELECT o.order_id, c.first_name
FROM orders o                    -- 1M rows
JOIN customers c ON o.customer_id = c.customer_id  -- join กับ 100K
WHERE c.city = 'Bangkok';        -- filter หลัง join

-- *** ดีกว่า: Filter ก่อน JOIN ***
SELECT o.order_id, c.first_name
FROM (
    SELECT customer_id, first_name 
    FROM customers 
    WHERE city = 'Bangkok'  -- filter customers ก่อน: 100K → อาจเหลือ 5K
) c
JOIN orders o ON c.customer_id = o.customer_id;  -- join กับ 5K แทน 100K

-- หมายเหตุ: ใน practice SQL Optimizer จัดการเองได้ดี
-- แต่บางครั้งต้อง hint optimizer ด้วย subquery
```

### ตัวอย่างที่ 6: Filter Early — ลด Rows ก่อน JOIN

```sql
-- BAD: JOIN ทุกอย่างก่อน แล้วค่อย filter
SELECT 
    o.order_id, c.first_name, p.product_name, oi.quantity
FROM orders o
JOIN customers c ON o.customer_id = c.customer_id
JOIN order_items oi ON o.order_id = oi.order_id
JOIN products p ON oi.product_id = p.product_id
WHERE o.status = 'completed'
  AND o.order_date >= '2024-06-01'
  AND c.city = 'Bangkok'
  AND p.category = 'Electronics';

-- GOOD: Filter ใน subqueries ก่อน
SELECT 
    o.order_id, c.first_name, p.product_name, oi.quantity
FROM (
    SELECT order_id, customer_id 
    FROM orders 
    WHERE status = 'completed' AND order_date >= '2024-06-01'
) o  -- filtered orders
JOIN (
    SELECT customer_id, first_name 
    FROM customers 
    WHERE city = 'Bangkok'
) c ON o.customer_id = c.customer_id  -- filtered customers
JOIN order_items oi ON o.order_id = oi.order_id
JOIN (
    SELECT product_id, product_name 
    FROM products 
    WHERE category = 'Electronics'
) p ON oi.product_id = p.product_id;  -- filtered products

-- Note: สำหรับ modern databases, optimizer มักทำ "predicate pushdown"
-- ซึ่งเทียบเท่ากับ GOOD version โดยอัตโนมัติ
```

---

## Common JOIN Performance Pitfalls

### ตัวอย่างที่ 7: Pitfall 1 — Function บน JOIN Column (ทำลาย Index)

```sql
-- BAD: ใช้ function บน JOIN column — index ถูก bypass!
SELECT e.first_name, d.dept_name
FROM employees e
JOIN departments d ON UPPER(e.dept_id::TEXT) = UPPER(d.dept_id::TEXT);
-- index บน dept_id ถูก bypass เพราะใช้ UPPER()

-- BAD: date function
SELECT o.order_id, c.first_name
FROM orders o
JOIN customers c ON o.customer_id = c.customer_id
WHERE YEAR(o.order_date) = 2024;  -- function ทำลาย index!

-- GOOD: ไม่ใช้ function บน indexed column
SELECT o.order_id, c.first_name
FROM orders o
JOIN customers c ON o.customer_id = c.customer_id
WHERE o.order_date BETWEEN '2024-01-01' AND '2024-12-31';  -- range ใช้ index ได้

-- GOOD: เปลี่ยน function ไปฝั่งที่ไม่มี index
SELECT e.first_name, d.dept_name
FROM employees e
JOIN departments d ON e.dept_id = d.dept_id;
-- ไม่ใช้ function เลย
```

### ตัวอย่างที่ 8: Pitfall 2 — Implicit Type Conversion

```sql
-- BAD: type ไม่ตรงกัน ทำให้ implicit conversion
-- สมมติ orders.customer_id เป็น INT แต่ customers.id เป็น BIGINT
SELECT o.order_id, c.first_name
FROM orders o
JOIN customers c ON o.customer_id = c.customer_id::INT;
-- การ cast ทำลาย index บน c.customer_id

-- GOOD: ทำให้ type ตรงกันในโครงสร้างตาราง
-- หรือ cast ฝั่งที่ไม่มี index
SELECT o.order_id, c.first_name
FROM orders o
JOIN customers c ON o.customer_id::BIGINT = c.customer_id;
-- cast ฝั่ง o ซึ่งน่าจะเล็กกว่า
```

### ตัวอย่างที่ 9: Pitfall 3 — Cartesian Product โดยไม่ตั้งใจ

```sql
-- BAD: ลืม ON clause!
SELECT e.first_name, d.dept_name
FROM employees e
JOIN departments d;  -- ERROR: syntax error หรือ CROSS JOIN!

-- BAD: ON clause ผิดทำให้ rows เพิ่มขึ้นมาก
SELECT e.first_name, d.dept_name
FROM employees e
JOIN departments d ON d.budget > 0;  -- ทุก employee match กับทุก dept ที่มี budget!
-- ได้ 50 employees × 10 departments = 500 rows แทนที่จะได้ 50!

-- GOOD: JOIN condition ที่ถูกต้อง
SELECT e.first_name, d.dept_name
FROM employees e
JOIN departments d ON e.dept_id = d.dept_id;
-- ได้ 50 rows ✓
```

### ตัวอย่างที่ 10: Pitfall 4 — Unnecessary JOINs

```sql
-- BAD: JOIN ตารางที่ไม่จำเป็น
SELECT DISTINCT e.emp_id, e.first_name, e.dept_id
FROM employees e
JOIN departments d ON e.dept_id = d.dept_id;  -- ไม่ได้ใช้ d เลย!
-- dept_id อยู่ใน employees อยู่แล้ว ไม่ต้อง JOIN!

-- GOOD: ไม่ JOIN ถ้าไม่ต้องการข้อมูลจาก table นั้น
SELECT e.emp_id, e.first_name, e.dept_id
FROM employees e;  -- ไม่ต้อง JOIN departments!

-- อีกตัวอย่าง:
-- BAD: JOIN เพื่อตรวจสอบ existence
SELECT DISTINCT o.customer_id
FROM orders o
JOIN customers c ON o.customer_id = c.customer_id;  -- ไม่ได้ใช้ c columns เลย!

-- GOOD: ใช้ IN หรือ EXISTS แทน
SELECT DISTINCT o.customer_id
FROM orders o
WHERE o.customer_id IN (SELECT customer_id FROM customers);

-- หรือถ้า FK constraint ถูก enforce อยู่ก็ไม่ต้อง check เลย:
SELECT DISTINCT o.customer_id FROM orders o;
```

---

## Query Rewriting Techniques

### ตัวอย่างที่ 11: ย้าย Aggregation ก่อน JOIN

```sql
-- BAD: JOIN ก่อน แล้วค่อย aggregate (ข้อมูลเยอะเกินไปใน memory)
SELECT 
    c.customer_id,
    c.first_name,
    COUNT(oi.item_id) AS total_items,
    SUM(oi.quantity * oi.unit_price - oi.discount) AS total_revenue
FROM customers c
JOIN orders o ON c.customer_id = o.customer_id      -- join ทั้งหมด
JOIN order_items oi ON o.order_id = oi.order_id    -- ก่อน aggregate
WHERE o.status = 'completed'
GROUP BY c.customer_id, c.first_name;

-- GOOD: Aggregate ก่อน แล้วค่อย JOIN (ลด rows ที่ต้อง join)
WITH order_summary AS (
    SELECT 
        o.customer_id,
        COUNT(oi.item_id) AS total_items,
        SUM(oi.quantity * oi.unit_price - oi.discount) AS total_revenue
    FROM orders o
    JOIN order_items oi ON o.order_id = oi.order_id
    WHERE o.status = 'completed'
    GROUP BY o.customer_id
)
SELECT 
    c.customer_id,
    c.first_name,
    COALESCE(os.total_items, 0) AS total_items,
    COALESCE(os.total_revenue, 0) AS total_revenue
FROM customers c
LEFT JOIN order_summary os ON c.customer_id = os.customer_id;
-- ตอนนี้ join กับ aggregated result แทน raw rows
```

### ตัวอย่างที่ 12: ใช้ Semi-join แทน Full JOIN

```sql
-- ต้องการ: customers ที่มี completed orders
-- BAD: Full JOIN + DISTINCT
SELECT DISTINCT c.customer_id, c.first_name, c.city
FROM customers c
JOIN orders o ON c.customer_id = o.customer_id
WHERE o.status = 'completed';

-- GOOD: Semi-join ด้วย EXISTS
SELECT c.customer_id, c.first_name, c.city
FROM customers c
WHERE EXISTS (
    SELECT 1 
    FROM orders o 
    WHERE o.customer_id = c.customer_id 
      AND o.status = 'completed'
);

-- GOOD: Semi-join ด้วย IN (ระวัง NULL)
SELECT c.customer_id, c.first_name, c.city
FROM customers c
WHERE c.customer_id IN (
    SELECT DISTINCT customer_id 
    FROM orders 
    WHERE status = 'completed'
);
```

### ตัวอย่างที่ 13: Partition Pruning

```sql
-- ใน partitioned tables, WHERE clause บน partition key
-- ทำให้ DB scan เฉพาะ partition ที่ต้องการ
-- (สมมติ orders ถูก partition ตาม year)

-- GOOD: WHERE บน partition key
EXPLAIN ANALYZE
SELECT o.order_id, c.first_name
FROM orders o
JOIN customers c ON o.customer_id = c.customer_id
WHERE o.order_date >= '2024-01-01'  -- partition pruning!
  AND o.order_date < '2025-01-01';

-- Planner จะ scan เฉพาะ partition ปี 2024
```

---

## Before/After Optimization Examples

### ตัวอย่างที่ 14: Slow Query — Before Optimization

```sql
-- BEFORE: Slow query สำหรับ Sales Report
-- ปัญหา:
-- 1. ไม่มี index บน JOIN columns
-- 2. Aggregation หลัง full JOIN
-- 3. ใช้ function บน filtered column

-- SLOW VERSION
SELECT 
    TO_CHAR(o.order_date, 'YYYY-MM') AS month,
    UPPER(c.city) AS city,  -- function!
    COUNT(o.order_id) AS orders,
    SUM(oi.quantity * oi.unit_price - oi.discount) AS revenue
FROM orders o
JOIN customers c ON CAST(o.customer_id AS VARCHAR) = CAST(c.customer_id AS VARCHAR)  -- type cast!
JOIN order_items oi ON o.order_id = oi.order_id
JOIN products p ON oi.product_id = p.product_id
WHERE EXTRACT(YEAR FROM o.order_date) = 2024  -- function!
  AND o.status = 'completed'
GROUP BY month, city
ORDER BY month, revenue DESC;
```

### ตัวอย่างที่ 15: After Optimization

```sql
-- สร้าง indexes ก่อน
CREATE INDEX IF NOT EXISTS idx_orders_customer_id ON orders(customer_id);
CREATE INDEX IF NOT EXISTS idx_orders_status_date ON orders(status, order_date);
CREATE INDEX IF NOT EXISTS idx_order_items_order_id ON order_items(order_id);
CREATE INDEX IF NOT EXISTS idx_order_items_product_id ON order_items(product_id);
CREATE INDEX IF NOT EXISTS idx_customers_city ON customers(city);

-- FAST VERSION: หลัง optimization
WITH filtered_orders AS (
    -- Filter ก่อน JOIN
    SELECT order_id, customer_id, order_date, total_amount
    FROM orders
    WHERE status = 'completed'
      AND order_date BETWEEN '2024-01-01' AND '2024-12-31'  -- range แทน function!
),
aggregated_items AS (
    -- Aggregate ก่อน JOIN กับ customers
    SELECT 
        o.order_id,
        o.customer_id,
        TO_CHAR(o.order_date, 'YYYY-MM') AS month,
        SUM(oi.quantity * oi.unit_price - oi.discount) AS order_revenue
    FROM filtered_orders o
    JOIN order_items oi ON o.order_id = oi.order_id
    GROUP BY o.order_id, o.customer_id, month
)
SELECT 
    ai.month,
    c.city,  -- ไม่ใช้ UPPER()
    COUNT(ai.order_id) AS orders,
    SUM(ai.order_revenue) AS revenue
FROM aggregated_items ai
JOIN customers c ON ai.customer_id = c.customer_id  -- ไม่มี type cast!
GROUP BY ai.month, c.city
ORDER BY ai.month, revenue DESC;
```

---

## EXPLAIN Output Analysis

### ตัวอย่างที่ 16: ตีความ EXPLAIN Output

```sql
-- ตัวอย่าง EXPLAIN ANALYZE output และการตีความ
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT 
    c.first_name,
    d.dept_name,
    e.salary
FROM employees e
JOIN customers c ON e.emp_id = c.customer_id  -- intentional bad join for demo
JOIN departments d ON e.dept_id = d.dept_id;

/*
EXPLAIN output keywords:
- Seq Scan: Full table scan — ไม่มี index หรือ table เล็กเกินไปสำหรับ index
- Index Scan: ใช้ index
- Index Only Scan: ข้อมูลทั้งหมดใน index (เร็วมาก)
- Hash Join: ใช้ hash table
- Nested Loop: ใช้ nested loops
- Merge Join: ใช้ sort + merge
- Sort: ต้อง sort ข้อมูล
- Aggregate: GROUP BY, COUNT, SUM, etc.
- Filter: WHERE condition
- Rows Removed by Filter: กี่ rows ที่ถูก filter ออก

cost=X..Y
- X: startup cost (ก่อนส่ง row แรก)
- Y: total cost

actual time=X..Y
- X: time to first row
- Y: time to last row

rows=N: estimated rows
actual rows=N: actual rows
loops=N: กี่ครั้งที่ node ถูก execute
*/
```

### ตัวอย่างที่ 17: เปรียบเทียบ Query Plans

```sql
-- Query 1: ไม่มี index
EXPLAIN ANALYZE
SELECT o.order_id, c.first_name
FROM orders o
JOIN customers c ON o.customer_id = c.customer_id
WHERE o.status = 'completed';
/*
Hash Join  (cost=4.75..7.44 rows=43 width=13) (actual time=0.110..0.198)
  Hash Cond: (o.customer_id = c.customer_id)
  ->  Seq Scan on orders o  (cost=0.00..2.45 rows=43)
        Filter: ((status)::text = 'completed')
        Rows Removed by Filter: 27
  ->  Hash  (cost=2.60..2.60 rows=60 width=9)
        ->  Seq Scan on customers c
Planning Time: 0.4 ms
Execution Time: 0.3 ms
*/

-- Query 2: หลังสร้าง index
CREATE INDEX idx_orders_status ON orders(status);
CREATE INDEX idx_orders_customer ON orders(customer_id);

EXPLAIN ANALYZE
SELECT o.order_id, c.first_name
FROM orders o
JOIN customers c ON o.customer_id = c.customer_id
WHERE o.status = 'completed';
/*
Hash Join  (cost=2.75..5.00 rows=43 width=13) (actual time=0.089..0.145)
  Hash Cond: (o.customer_id = c.customer_id)
  ->  Index Scan using idx_orders_status on orders o  (cost=0.14..2.00)
        Index Cond: ((status)::text = 'completed')
  ->  Hash  (cost=2.60..2.60 rows=60)
        ->  Seq Scan on customers c  (tiny table, Seq Scan is OK)
Planning Time: 0.5 ms
Execution Time: 0.2 ms
*/
-- Index ช่วยลดเวลาลงได้ (~33% สำหรับตารางเล็ก, ใหญ่กว่ามากสำหรับตารางจริง)
```

---

## Statistics และ Query Planner

### ตัวอย่างที่ 18: ANALYZE เพื่ออัพเดต Statistics

```sql
-- PostgreSQL ใช้ statistics เพื่อ estimate rows
-- ต้อง ANALYZE เพื่ออัพเดต statistics หลังเพิ่มข้อมูลมาก

ANALYZE employees;
ANALYZE orders;
ANALYZE order_items;
ANALYZE customers;
ANALYZE products;

-- หรือ ANALYZE ทุกตารางพร้อมกัน
VACUUM ANALYZE;  -- VACUUM + ANALYZE

-- ดู statistics
SELECT 
    tablename,
    attname AS column_name,
    n_distinct,
    correlation,
    most_common_vals::TEXT
FROM pg_stats
WHERE tablename IN ('orders', 'employees')
  AND attname IN ('status', 'dept_id', 'customer_id')
ORDER BY tablename, attname;
```

### ตัวอย่างที่ 19: Optimizer Hints (MySQL)

```sql
-- MySQL: ใช้ INDEX HINT เพื่อ force optimizer
-- ปกติไม่แนะนำ แต่บางครั้งจำเป็น

-- Force index
SELECT o.order_id, c.first_name
FROM orders o USE INDEX (idx_orders_customer_id)
JOIN customers c ON o.customer_id = c.customer_id
WHERE o.status = 'completed';

-- Ignore index
SELECT o.order_id, c.first_name
FROM orders o IGNORE INDEX (idx_orders_status)
JOIN customers c ON o.customer_id = c.customer_id
WHERE o.status = 'completed';

-- Force join order
SELECT o.order_id, c.first_name
FROM orders o STRAIGHT_JOIN customers c  -- MySQL: force left-to-right join
ON o.customer_id = c.customer_id;
```

---

## Optimization Best Practices

### ตัวอย่างที่ 20: Best Practices Summary

```sql
-- ✅ สร้าง index บน JOIN columns (FK columns)
CREATE INDEX idx_emp_dept ON employees(dept_id);
CREATE INDEX idx_orders_cust ON orders(customer_id);
CREATE INDEX idx_oi_order ON order_items(order_id);
CREATE INDEX idx_oi_product ON order_items(product_id);

-- ✅ Composite index สำหรับ WHERE + JOIN
CREATE INDEX idx_orders_status_cust ON orders(status, customer_id);

-- ✅ Filter early ใน WHERE clause
-- ❌ BAD: WHERE ที่มี function
WHERE YEAR(order_date) = 2024
-- ✅ GOOD: Range comparison
WHERE order_date BETWEEN '2024-01-01' AND '2024-12-31'

-- ✅ Aggregate ก่อน JOIN ถ้าเป็นไปได้
-- ✅ ใช้ EXISTS สำหรับ semi-join
-- ✅ หลีกเลี่ยง DISTINCT โดยไม่จำเป็น
-- ✅ ใช้ EXPLAIN ANALYZE เพื่อ verify
-- ✅ ANALYZE ตารางหลังเพิ่มข้อมูลมาก

-- ❌ อย่า JOIN ตารางที่ไม่ได้ใช้ข้อมูล
-- ❌ อย่าใช้ SELECT * ใน JOIN queries
-- ❌ อย่าใช้ OR บน indexed columns ใน JOIN condition
-- ❌ อย่าใช้ NOT IN กับ subquery ที่อาจมี NULL
```

---

## แบบฝึกหัดภาค 29

**ข้อ 1:** ใช้ EXPLAIN ANALYZE เพื่อดู query plan ของ JOIN ระหว่าง employees และ departments

```sql
-- เฉลย
EXPLAIN ANALYZE
SELECT e.emp_id, e.first_name, d.dept_name, d.location
FROM employees e
JOIN departments d ON e.dept_id = d.dept_id
ORDER BY d.dept_name, e.emp_id;

-- อ่านผล:
-- ดู Seq Scan vs Index Scan
-- ดู actual rows vs estimated rows
-- ดู execution time
```

**ข้อ 2:** สร้าง indexes ที่เหมาะสมสำหรับ query ที่ JOIN orders + customers + order_items

```sql
-- เฉลย
-- สร้าง indexes
CREATE INDEX IF NOT EXISTS idx_orders_cust_id ON orders(customer_id);
CREATE INDEX IF NOT EXISTS idx_orders_status ON orders(status);
CREATE INDEX IF NOT EXISTS idx_oi_order_id ON order_items(order_id);
CREATE INDEX IF NOT EXISTS idx_oi_product_id ON order_items(product_id);

-- ตรวจสอบด้วย EXPLAIN
EXPLAIN ANALYZE
SELECT 
    c.first_name, 
    o.order_date, 
    COUNT(oi.item_id) AS items
FROM orders o
JOIN customers c ON o.customer_id = c.customer_id
JOIN order_items oi ON o.order_id = oi.order_id
WHERE o.status = 'completed'
GROUP BY c.first_name, o.order_id, o.order_date;
```

**ข้อ 3:** เขียน query ที่ Aggregate ก่อน JOIN เพื่อ improve performance

```sql
-- เฉลย
WITH order_totals AS (
    SELECT 
        order_id,
        customer_id,
        COUNT(*) AS item_count,
        SUM(quantity * unit_price - discount) AS revenue
    FROM order_items
    GROUP BY order_id, customer_id
    -- aggregate ก่อน join ลด intermediate rows
)
SELECT 
    c.customer_id,
    c.first_name || ' ' || c.last_name AS customer,
    SUM(ot.item_count) AS total_items,
    SUM(ot.revenue) AS total_revenue,
    COUNT(*) AS total_orders
FROM customers c
LEFT JOIN orders o ON c.customer_id = o.customer_id
LEFT JOIN order_totals ot ON o.order_id = ot.order_id
GROUP BY c.customer_id, customer;
```

**ข้อ 4:** แก้ไข query ที่ใช้ function บน JOIN column

```sql
-- ต้นฉบับที่ช้า (function บน column):
SELECT e.first_name, d.dept_name
FROM employees e
JOIN departments d ON LOWER(d.dept_name) = LOWER(e.dept_name_fk);

-- เฉลย: filter ด้านที่ไม่มี index แทน
SELECT e.first_name, d.dept_name
FROM employees e
JOIN departments d ON e.dept_id = d.dept_id
WHERE d.dept_name = 'Engineering';  -- ไม่ใช้ function บน indexed column

-- หรือถ้าต้องทำ case-insensitive
CREATE INDEX idx_dept_name_lower ON departments(LOWER(dept_name));
-- แล้วใช้
WHERE LOWER(d.dept_name) = 'engineering';  -- ใช้ functional index
```

**ข้อ 5:** เขียน Semi-join ด้วย EXISTS แทน JOIN + DISTINCT

```sql
-- ต้นฉบับ (ช้ากว่า):
SELECT DISTINCT c.customer_id, c.first_name
FROM customers c
JOIN orders o ON c.customer_id = o.customer_id
WHERE o.status = 'completed';

-- เฉลย: EXISTS
SELECT c.customer_id, c.first_name
FROM customers c
WHERE EXISTS (
    SELECT 1 FROM orders o
    WHERE o.customer_id = c.customer_id
      AND o.status = 'completed'
);
```

**ข้อ 6:** อธิบายความแตกต่างระหว่าง Nested Loop, Hash Join, และ Merge Join

```sql
-- เฉลย: อธิบายผ่าน use cases

-- Nested Loop: เหมาะสำหรับ
-- - ตารางเล็กมาก (< 1000 rows)
-- - Inner table มี index ที่ดี
-- - Selective WHERE condition
SELECT e.first_name, d.dept_name
FROM employees e  -- 50 rows
JOIN departments d ON e.dept_id = d.dept_id  -- 10 rows
-- → Likely Nested Loop

-- Hash Join: เหมาะสำหรับ
-- - ตารางใหญ่ไม่มี useful index
-- - Equi-join เท่านั้น (=)
-- - Available memory เยอะ
SELECT o.order_id, c.first_name
FROM orders o  -- 1M rows
JOIN customers c ON o.customer_id = c.customer_id  -- 100K rows
-- → Likely Hash Join

-- Merge Join: เหมาะสำหรับ
-- - ทั้งสองตาราง sort แล้ว (มี index sorted)
-- - Range joins
-- - อ่านทั้งสองตารางแบบ sequential
SELECT o.order_id, oi.item_id
FROM orders o
JOIN order_items oi ON o.order_id = oi.order_id
ORDER BY o.order_id
-- → Possible Merge Join if both have clustered index on order_id
```

**ข้อ 7:** ใช้ EXPLAIN เพื่อเปรียบเทียบ 2 versions ของ query เดียวกัน

```sql
-- เฉลย
-- Version 1: ไม่ optimal
EXPLAIN ANALYZE
SELECT 
    COUNT(*) AS total_customers_with_orders
FROM customers c
JOIN orders o ON c.customer_id = o.customer_id
WHERE o.status = 'completed';

-- Version 2: Semi-join
EXPLAIN ANALYZE
SELECT 
    COUNT(*) AS total_customers_with_orders
FROM customers c
WHERE EXISTS (
    SELECT 1 FROM orders o
    WHERE o.customer_id = c.customer_id
    AND o.status = 'completed'
);

-- เปรียบเทียบ Planning Time + Execution Time
-- Version 2 มักเร็วกว่าสำหรับตารางใหญ่
```

**ข้อ 8:** ระบุปัญหาใน query ต่อไปนี้และแก้ไข

```sql
-- Query ที่มีปัญหา:
SELECT o.order_id, c.first_name
FROM orders o, customers c
WHERE o.customer_id = c.customer_id
  AND DATE_FORMAT(o.order_date, '%Y') = '2024'
  AND LENGTH(c.first_name) > 3;

-- เฉลย: แก้ไข
SELECT o.order_id, c.first_name
FROM orders o
JOIN customers c ON o.customer_id = c.customer_id  -- explicit JOIN
WHERE o.order_date >= '2024-01-01'                 -- range แทน function
  AND o.order_date < '2025-01-01'
  AND LENGTH(c.first_name) > 3;                    -- OK (on non-indexed col)
-- หมายเหตุ: LENGTH() บน first_name อาจไม่มี index
-- แต่ถ้า first_name ไม่มี index อยู่แล้ว ก็ไม่แย่ลง
```

**ข้อ 9:** สร้าง Composite Index ที่เหมาะสมสำหรับ query ที่ใช้บ่อย

```sql
-- เฉลย
-- Query ที่ใช้บ่อย:
-- WHERE status = 'completed' AND order_date BETWEEN x AND y

-- Composite index: status + order_date (query uses both)
CREATE INDEX idx_orders_status_date 
ON orders(status, order_date);

-- ทดสอบ
EXPLAIN ANALYZE
SELECT order_id, customer_id, total_amount
FROM orders
WHERE status = 'completed'
  AND order_date BETWEEN '2024-01-01' AND '2024-06-30';
-- ควรใช้ Index Scan บน composite index

-- อีกตัวอย่าง: covering index
CREATE INDEX idx_emp_dept_salary 
ON employees(dept_id, salary)  -- dept_id บน WHERE/JOIN, salary ใน SELECT
INCLUDE (first_name, last_name);  -- PostgreSQL: covering columns

EXPLAIN ANALYZE
SELECT first_name, last_name, salary
FROM employees
WHERE dept_id = 1
ORDER BY salary DESC;
-- Index Only Scan → เร็วที่สุด!
```

**ข้อ 10:** เขียน Optimized version ของ Complex Business Query

```sql
-- Original (ช้า):
SELECT c.city, p.category, 
       SUM(oi.quantity * oi.unit_price) AS revenue
FROM customers c
JOIN orders o ON c.customer_id = o.customer_id
JOIN order_items oi ON o.order_id = oi.order_id
JOIN products p ON oi.product_id = p.product_id
WHERE EXTRACT(YEAR FROM o.order_date) = 2024
  AND o.status = 'completed'
GROUP BY c.city, p.category;

-- เฉลย: Optimized version
WITH completed_2024_orders AS (
    -- Filter orders ก่อน
    SELECT order_id, customer_id
    FROM orders
    WHERE status = 'completed'
      AND order_date >= '2024-01-01'  -- range แทน EXTRACT()
      AND order_date < '2025-01-01'
),
order_revenue AS (
    -- Aggregate items ก่อน join กับ customers
    SELECT 
        o.customer_id,
        p.category,
        SUM(oi.quantity * oi.unit_price - oi.discount) AS revenue
    FROM completed_2024_orders o
    JOIN order_items oi ON o.order_id = oi.order_id
    JOIN products p ON oi.product_id = p.product_id
    GROUP BY o.customer_id, p.category
)
SELECT 
    c.city,
    orv.category,
    SUM(orv.revenue) AS total_revenue
FROM order_revenue orv
JOIN customers c ON orv.customer_id = c.customer_id
GROUP BY c.city, orv.category
ORDER BY c.city, total_revenue DESC;
```

---

## สรุปภาค 29

1. **Join Algorithms** — Nested Loop (เล็ก), Hash Join (ใหญ่), Merge Join (sorted)
2. **EXPLAIN ANALYZE** — เครื่องมือสำคัญสำหรับ diagnose performance
3. **Indexes** — สร้างบน FK columns, WHERE columns, composite สำหรับ multi-column
4. **Filter Early** — ลด rows ก่อน JOIN ด้วย subquery หรือ CTE
5. **Aggregate Before JOIN** — ลด intermediate data size
6. **Avoid Functions on Join Columns** — ทำลาย index
7. **Semi-join** — EXISTS เร็วกว่า JOIN + DISTINCT
8. **Avoid Unnecessary JOINs** — ถ้าไม่ใช้ข้อมูลจากตาราง ก็ไม่ต้อง JOIN

**ในภาคสุดท้าย** จะทำ Practical JOIN Projects เพื่อประยุกต์ใช้ทุกอย่างที่เรียนมา!
