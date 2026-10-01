# Part 066: Query Optimization Techniques

## เทคนิคการ Optimize Query

---

## บทนำ

Query Optimization คือกระบวนการปรับปรุง SQL queries ให้ database engine สามารถ execute ได้เร็วขึ้น โดยมักเกี่ยวข้องกับการเขียน query ที่ "index-friendly" และหลีกเลี่ยงรูปแบบที่ทำให้ planner ไม่สามารถใช้ index ได้

---

## 1. Sargable Predicates - WHERE Clauses ที่ Index-friendly

**Sargable** มาจาก "Search ARGument ABLE" หมายถึง WHERE clause ที่ database สามารถใช้ index เพื่อค้นหาได้

### กฎ Sargable

```sql
-- ตัวอย่างที่ 1: ✅ Sargable - ใช้ index ได้
SELECT * FROM orders WHERE customer_id = 1001;
SELECT * FROM orders WHERE order_date >= '2024-01-01';
SELECT * FROM orders WHERE amount BETWEEN 1000 AND 5000;
SELECT * FROM customers WHERE last_name LIKE 'สมช%';  -- prefix only
SELECT * FROM employees WHERE department_id IN (1, 2, 3);
SELECT * FROM products WHERE price IS NULL;
SELECT * FROM products WHERE price IS NOT NULL;

-- ตัวอย่างที่ 2: ❌ Non-Sargable - ไม่ใช้ index
SELECT * FROM orders WHERE YEAR(order_date) = 2024;  -- function on column
SELECT * FROM customers WHERE UPPER(email) = 'USER@EXAMPLE.COM';  -- function
SELECT * FROM products WHERE price * 1.07 > 1000;  -- arithmetic on column
SELECT * FROM customers WHERE last_name LIKE '%ชาย';  -- leading wildcard
SELECT * FROM customers WHERE last_name LIKE '%ชาย%';  -- both wildcards
```

---

## 2. Index Usage Killers (สิ่งที่ทำลาย Index)

### 2.1 Functions บน Indexed Columns

```sql
-- ตัวอย่างที่ 3: ❌ Function ครอบ column
-- ปัญหา: YEAR() ครอบ order_date → ไม่ใช้ index
SELECT * FROM orders WHERE YEAR(order_date) = 2024;

-- ✅ วิธีแก้: rewrite เป็น range
SELECT * FROM orders 
WHERE order_date >= '2024-01-01' AND order_date < '2025-01-01';
```

```sql
-- ตัวอย่างที่ 4: ❌ Function ครอบ text column
SELECT * FROM employees WHERE LOWER(email) = 'john@example.com';
-- หรือ
SELECT * FROM employees WHERE UPPER(first_name) = 'JOHN';

-- ✅ วิธีแก้ Option 1: Expression Index
CREATE INDEX idx_emp_lower_email ON employees(LOWER(email));
SELECT * FROM employees WHERE LOWER(email) = 'john@example.com';  -- ใช้ index!

-- ✅ วิธีแก้ Option 2: Store lowercase ไว้ใน column
-- ✅ วิธีแก้ Option 3: citext type (PostgreSQL)
CREATE EXTENSION citext;
ALTER TABLE employees ALTER COLUMN email TYPE citext;
SELECT * FROM employees WHERE email = 'John@example.com';  -- case-insensitive
```

```sql
-- ตัวอย่างที่ 5: ❌ Date function บน indexed column
SELECT * FROM sales WHERE DATE(created_at) = '2024-01-15';
-- DATE() ครอบ created_at → Seq Scan

-- ✅ วิธีแก้:
SELECT * FROM sales 
WHERE created_at >= '2024-01-15' AND created_at < '2024-01-16';
-- หรือ
SELECT * FROM sales 
WHERE created_at >= '2024-01-15 00:00:00' AND created_at <= '2024-01-15 23:59:59';
-- หรือ (PostgreSQL)
SELECT * FROM sales WHERE created_at::date = '2024-01-15';  -- cast ไม่ครอบ!
-- NOTE: ใน PostgreSQL การ cast ยังใช้ index ได้ในบางกรณี ขึ้นอยู่กับ type
```

```sql
-- ตัวอย่างที่ 6: ❌ MONTH() / DAY()
SELECT * FROM birthdays WHERE MONTH(birth_date) = 12;
-- MONTH() ครอบ column → ไม่ใช้ index

-- ✅ วิธีแก้: Expression index (PostgreSQL)
CREATE INDEX idx_birthdays_month ON birthdays(EXTRACT(MONTH FROM birth_date));
SELECT * FROM birthdays WHERE EXTRACT(MONTH FROM birth_date) = 12;
```

### 2.2 Implicit Type Conversions

```sql
-- ตัวอย่างที่ 7: ❌ Type mismatch - MySQL
-- สมมติ customer_id เป็น INT แต่ query ใส่เป็น string
SELECT * FROM orders WHERE customer_id = '1001';
-- MySQL แปลง '1001' → INT แต่บางครั้ง planner ไม่ใช้ index!

-- ✅ วิธีแก้: ใช้ type ที่ถูกต้อง
SELECT * FROM orders WHERE customer_id = 1001;  -- INT literal
```

```sql
-- ตัวอย่างที่ 8: ❌ Type mismatch ใน PostgreSQL
-- account_number เป็น VARCHAR แต่ query เปรียบเทียบกับ INT
SELECT * FROM accounts WHERE account_number = 12345;
-- PostgreSQL แปลง account_number → INT แล้ว compare
-- ไม่ใช้ index บน account_number!

-- ✅ วิธีแก้:
SELECT * FROM accounts WHERE account_number = '12345';  -- string literal
```

```sql
-- ตัวอย่างที่ 9: ❌ Charset/Collation mismatch (MySQL)
-- table: utf8mb4_unicode_ci แต่ column: latin1_swedish_ci
-- การ join จะทำให้ไม่ใช้ index

-- ✅ วิธีแก้: ให้ charset ตรงกัน
ALTER TABLE users CONVERT TO CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
```

### 2.3 Leading Wildcards

```sql
-- ตัวอย่างที่ 10: ❌ Leading Wildcard
SELECT * FROM customers WHERE name LIKE '%สมชาย%';
SELECT * FROM products WHERE description LIKE '%laptop%';
-- B-tree index ไม่สามารถใช้ได้กับ leading wildcard

-- ✅ วิธีแก้ Option 1: Full-text Search
-- PostgreSQL:
SELECT * FROM products WHERE to_tsvector('english', description) @@ to_tsquery('english', 'laptop');

-- MySQL:
SELECT * FROM products WHERE MATCH(description) AGAINST('laptop');

-- ✅ วิธีแก้ Option 2: pg_trgm (trigram index) สำหรับ LIKE '%...%'
CREATE EXTENSION pg_trgm;
CREATE INDEX idx_products_desc_trgm ON products USING GIN(description gin_trgm_ops);
SELECT * FROM products WHERE description LIKE '%laptop%';  -- ใช้ trigram index!
```

---

## 3. Rewriting Queries เพื่อ Index Usage

### 3.1 OR → UNION

```sql
-- ตัวอย่างที่ 11: ❌ OR อาจไม่ใช้ index (MySQL เก่า)
SELECT * FROM employees 
WHERE department_id = 1 OR department_id = 5;

-- ✅ วิธีแก้ Option 1: IN (PostgreSQL/MySQL 8+)
SELECT * FROM employees WHERE department_id IN (1, 5);

-- ✅ วิธีแก้ Option 2: UNION ALL
SELECT * FROM employees WHERE department_id = 1
UNION ALL
SELECT * FROM employees WHERE department_id = 5;
-- แต่ละ query ใช้ index แยกกัน
```

### 3.2 NOT IN → NOT EXISTS / LEFT JOIN

```sql
-- ตัวอย่างที่ 12: ❌ NOT IN กับ NULL values (บัค!)
SELECT * FROM orders 
WHERE customer_id NOT IN (SELECT customer_id FROM blacklisted_customers);
-- ถ้า blacklisted_customers มี NULL → ไม่ได้ผลลัพธ์อะไรเลย!

-- ✅ วิธีแก้ Option 1: NOT EXISTS
SELECT * FROM orders o
WHERE NOT EXISTS (
    SELECT 1 FROM blacklisted_customers bc
    WHERE bc.customer_id = o.customer_id
);
-- ปลอดภัยกว่า และมักใช้ index ได้ดีกว่า

-- ✅ วิธีแก้ Option 2: LEFT JOIN + IS NULL
SELECT o.*
FROM orders o
LEFT JOIN blacklisted_customers bc ON o.customer_id = bc.customer_id
WHERE bc.customer_id IS NULL;
```

### 3.3 Subquery → JOIN

```sql
-- ตัวอย่างที่ 13: ❌ Correlated subquery (ช้ามาก!)
SELECT emp_id, name, salary
FROM employees e
WHERE salary > (
    SELECT AVG(salary) 
    FROM employees 
    WHERE department_id = e.department_id  -- correlated! execute ทุก row
);

-- ✅ วิธีแก้: Window function หรือ JOIN
-- Option 1: Window function (PostgreSQL)
SELECT emp_id, name, salary
FROM (
    SELECT emp_id, name, salary,
           AVG(salary) OVER (PARTITION BY department_id) AS dept_avg_salary
    FROM employees
) t
WHERE salary > dept_avg_salary;

-- Option 2: JOIN กับ aggregation
SELECT e.emp_id, e.name, e.salary
FROM employees e
JOIN (
    SELECT department_id, AVG(salary) AS avg_salary
    FROM employees
    GROUP BY department_id
) dept_avg ON e.department_id = dept_avg.department_id
WHERE e.salary > dept_avg.avg_salary;
```

```sql
-- ตัวอย่างที่ 14: ❌ Scalar subquery ใน SELECT
SELECT 
    o.order_id,
    o.total_amount,
    (SELECT name FROM customers WHERE customer_id = o.customer_id) AS customer_name
FROM orders o
WHERE o.order_date >= '2024-01-01';
-- Scalar subquery execute 1 ครั้งต่อ row!

-- ✅ วิธีแก้: JOIN
SELECT 
    o.order_id,
    o.total_amount,
    c.name AS customer_name
FROM orders o
JOIN customers c ON o.customer_id = c.customer_id
WHERE o.order_date >= '2024-01-01';
```

### 3.4 DISTINCT → EXISTS

```sql
-- ตัวอย่างที่ 15: ❌ DISTINCT (ช้าสำหรับ large tables)
SELECT DISTINCT c.customer_id, c.name
FROM customers c
JOIN orders o ON c.customer_id = o.customer_id;
-- DISTINCT ต้อง sort/deduplicate ทุก row

-- ✅ วิธีแก้: EXISTS
SELECT customer_id, name
FROM customers c
WHERE EXISTS (
    SELECT 1 FROM orders o 
    WHERE o.customer_id = c.customer_id
);
-- หยุดค้นหาทันทีที่พบ 1 row (short-circuit)
```

---

## 4. Before/After Optimization Examples

### ตัวอย่างที่ 16: Date Range Query

```sql
-- ❌ Before (Slow)
SELECT * FROM transactions 
WHERE YEAR(created_at) = 2024 AND MONTH(created_at) = 6;
-- Full Scan เพราะ function ครอบ column

-- ✅ After (Fast)
SELECT * FROM transactions 
WHERE created_at >= '2024-06-01' AND created_at < '2024-07-01';
-- Range scan → ใช้ index บน created_at
```

**EXPLAIN Before:**
```
Seq Scan on transactions (cost=0.00..45000.00 rows=50000 width=100)
  Filter: ((YEAR(created_at) = 2024) AND (MONTH(created_at) = 6))
  Execution Time: 3500 ms
```

**EXPLAIN After:**
```
Index Scan using idx_transactions_created_at on transactions
  (cost=0.56..234.56 rows=50000 width=100)
  Index Cond: ((created_at >= '2024-06-01') AND (created_at < '2024-07-01'))
  Execution Time: 12 ms
```

---

### ตัวอย่างที่ 17: String Search Optimization

```sql
-- ❌ Before (Slow)
SELECT * FROM products WHERE LOWER(name) LIKE '%laptop%';

-- ✅ After Option 1: Trigram index + LOWER()
CREATE INDEX idx_products_name_trgm ON products USING GIN(LOWER(name) gin_trgm_ops);
SELECT * FROM products WHERE LOWER(name) LIKE '%laptop%';

-- ✅ After Option 2: Full-text Search
SELECT * FROM products 
WHERE to_tsvector('english', name) @@ plainto_tsquery('english', 'laptop');
```

---

### ตัวอย่างที่ 18: Aggregation Optimization

```sql
-- ❌ Before (ช้า - Subquery ใน FROM)
SELECT dept, avg_salary
FROM (
    SELECT department_id AS dept, AVG(salary) AS avg_salary
    FROM employees
    GROUP BY department_id
) subq
WHERE avg_salary > 50000;

-- ✅ After: ใช้ HAVING แทน
SELECT department_id, AVG(salary) AS avg_salary
FROM employees
GROUP BY department_id
HAVING AVG(salary) > 50000;
-- HAVING กรองหลัง GROUP BY → ไม่ต้องมี subquery
```

---

### ตัวอย่างที่ 19: Pagination Optimization

```sql
-- ❌ Before (OFFSET ใหญ่ = ช้ามาก)
SELECT * FROM products ORDER BY product_id LIMIT 20 OFFSET 100000;
-- ต้องอ่าน + ทิ้ง 100,000 rows ก่อน!

-- ✅ After: Keyset Pagination
-- First page:
SELECT * FROM products ORDER BY product_id LIMIT 20;
-- ได้ last_id = 200 จาก page แรก

-- Next page:
SELECT * FROM products WHERE product_id > 200 ORDER BY product_id LIMIT 20;
-- ใช้ index! ไม่ต้อง offset

-- ✅ After สำหรับ arbitrary page: OFFSET น้อยลง
-- Store last_seen_id ใน session
```

---

### ตัวอย่างที่ 20: Multiple OR Conditions

```sql
-- ❌ Before (อาจช้า)
SELECT * FROM logs
WHERE severity = 'ERROR' 
   OR severity = 'CRITICAL'
   OR severity = 'FATAL';

-- ✅ After Option 1: IN
SELECT * FROM logs WHERE severity IN ('ERROR', 'CRITICAL', 'FATAL');

-- ✅ After Option 2: Partial Index
CREATE INDEX idx_logs_high_severity ON logs(created_at DESC)
WHERE severity IN ('ERROR', 'CRITICAL', 'FATAL');
SELECT * FROM logs 
WHERE severity IN ('ERROR', 'CRITICAL', 'FATAL')
ORDER BY created_at DESC;
```

---

### ตัวอย่างที่ 21: Implicit Conversion Fix

```sql
-- ❌ Before (MySQL: type mismatch)
-- phone เป็น VARCHAR แต่ compare กับ INT
SELECT * FROM customers WHERE phone = 0812345678;
-- MySQL แปลง VARCHAR → INT ทุก row → ไม่ใช้ index

-- ✅ After:
SELECT * FROM customers WHERE phone = '0812345678';
```

---

### ตัวอย่างที่ 22: Avoiding SELECT *

```sql
-- ❌ Before (ดึงข้อมูลมากกว่าที่ต้องการ)
SELECT * FROM employees WHERE department_id = 5;
-- ดึงทุก column แม้ต้องการแค่ name, email

-- ✅ After: ระบุ columns ที่ต้องการ
SELECT emp_id, first_name, last_name, email
FROM employees WHERE department_id = 5;
-- ถ้ามี covering index (dept_id, emp_id, first_name, last_name, email)
-- จะเป็น Index Only Scan!
```

---

### ตัวอย่างที่ 23: EXISTS vs COUNT

```sql
-- ❌ Before (นับทุก row)
SELECT customer_id FROM customers
WHERE (SELECT COUNT(*) FROM orders WHERE customer_id = c.customer_id) > 0;

-- ✅ After: EXISTS (หยุดทันทีที่เจอ 1 row)
SELECT customer_id FROM customers c
WHERE EXISTS (SELECT 1 FROM orders WHERE customer_id = c.customer_id);
```

---

### ตัวอย่างที่ 24: LIKE Prefix Optimization

```sql
-- ❌ Before (ช้า - leading wildcard)
SELECT * FROM articles WHERE title LIKE '%SQL%';

-- ✅ After Option 1: pg_trgm (PostgreSQL)
CREATE INDEX idx_articles_title_trgm ON articles USING GIN(title gin_trgm_ops);
SELECT * FROM articles WHERE title LIKE '%SQL%';  -- เร็วขึ้น

-- ✅ After Option 2: Full-text search (ดีกว่าสำหรับ text)
SELECT * FROM articles 
WHERE to_tsvector('english', title) @@ to_tsquery('english', 'SQL');
```

---

### ตัวอย่างที่ 25: Correlated Subquery Fix

```sql
-- ❌ Before (Correlated subquery - execute ทุก row!)
SELECT p.product_id, p.name,
    (SELECT category_name FROM categories WHERE category_id = p.category_id) AS cat_name
FROM products p
WHERE p.is_active = TRUE;

-- ✅ After: JOIN
SELECT p.product_id, p.name, c.category_name
FROM products p
JOIN categories c ON p.category_id = c.category_id
WHERE p.is_active = TRUE;
```

---

## 5. Covering Index Strategy

```sql
-- ตัวอย่างที่ 26: สร้าง covering index สำหรับ common query
-- Query ที่รันบ่อย:
SELECT order_id, order_date, total_amount, status
FROM orders
WHERE customer_id = ? AND order_date >= ?
ORDER BY order_date DESC
LIMIT 50;

-- ✅ Covering index
CREATE INDEX idx_orders_customer_date_cover
ON orders(customer_id, order_date DESC)
INCLUDE (order_id, total_amount, status);
-- customer_id: key column (WHERE)
-- order_date DESC: key column (WHERE range + ORDER BY)
-- order_id, total_amount, status: included columns (SELECT)
```

---

## 6. Join Optimization

### ตัวอย่างที่ 27: Join Order Matters

```sql
-- PostgreSQL เลือก join order อัตโนมัติตาม statistics
-- แต่เราสามารถ force ได้ในกรณีพิเศษ

-- ดูว่า planner เลือก join order ไหน:
SET enable_hashjoin = OFF;
SET enable_mergejoin = OFF;
-- ทำให้เห็นว่าถ้าใช้ Nested Loop อย่างเดียว จะใช้ order ไหน

-- Reset:
RESET enable_hashjoin;
RESET enable_mergejoin;
```

```sql
-- ตัวอย่างที่ 28: เพิ่ม Index สำหรับ JOIN
-- Join ที่ช้า:
SELECT o.*, p.name
FROM order_items oi
JOIN orders o ON oi.order_id = o.order_id
JOIN products p ON oi.product_id = p.product_id
WHERE o.customer_id = 1001;

-- ✅ Indexes ที่ต้องมี:
CREATE INDEX idx_orders_customer ON orders(customer_id);  -- สำหรับ WHERE
CREATE INDEX idx_order_items_order ON order_items(order_id);  -- สำหรับ JOIN
CREATE INDEX idx_order_items_product ON order_items(product_id);  -- สำหรับ JOIN
-- products.product_id เป็น PK ไม่ต้องสร้างเพิ่ม
```

---

## 7. Query Hints (ใช้เมื่อจำเป็นเท่านั้น!)

```sql
-- PostgreSQL: ไม่มี query hints โดยตรง
-- แต่สามารถ disable บาง plan types ชั่วคราว:
SET enable_seqscan = OFF;  -- บังคับใช้ index scan
SET enable_hashjoin = OFF;  -- disable hash join
SET join_collapse_limit = 1;  -- ป้องกัน join reordering

-- คำเตือน: ใช้เฉพาะ session scope! อย่า SET ใน postgresql.conf
-- ใช้สำหรับ testing เท่านั้น ไม่ใช่ production fix

-- ✅ วิธีที่ถูกต้อง: แก้ที่ root cause (index, statistics)
```

```sql
-- MySQL: Index Hints
-- บังคับใช้ index เฉพาะ:
SELECT * FROM employees USE INDEX (idx_emp_dept_salary)
WHERE department_id = 5 AND salary > 50000;

-- Force index:
SELECT * FROM orders FORCE INDEX (idx_orders_customer)
WHERE customer_id = 1001;

-- Ignore index:
SELECT * FROM orders IGNORE INDEX (idx_old_unused_index)
WHERE order_date >= '2024-01-01';

-- คำเตือน: Query hints ผูกติดกับ schema
-- ถ้าลบ index จะเกิด error!
```

---

## 8. ตัวอย่างสมบูรณ์: Before/After พร้อม EXPLAIN

### ตัวอย่างที่ 29: Dashboard Query

```sql
-- Dashboard: ยอดขายวันนี้แยกตามหมวดหมู่
-- ❌ Before:
SELECT 
    c.category_name,
    COUNT(oi.item_id) AS items_sold,
    SUM(oi.quantity * oi.unit_price) AS revenue
FROM order_items oi
JOIN orders o ON oi.order_id = o.order_id
JOIN products p ON oi.product_id = p.product_id
JOIN categories c ON p.category_id = c.category_id
WHERE DATE(o.created_at) = CURRENT_DATE
GROUP BY c.category_name
ORDER BY revenue DESC;

-- ปัญหา: DATE() ครอบ created_at → Seq Scan

-- ✅ After:
SELECT 
    c.category_name,
    COUNT(oi.item_id) AS items_sold,
    SUM(oi.quantity * oi.unit_price) AS revenue
FROM order_items oi
JOIN orders o ON oi.order_id = o.order_id
JOIN products p ON oi.product_id = p.product_id
JOIN categories c ON p.category_id = c.category_id
WHERE o.created_at >= CURRENT_DATE 
  AND o.created_at < CURRENT_DATE + INTERVAL '1 day'
GROUP BY c.category_name
ORDER BY revenue DESC;

-- Indexes ที่ต้องมี:
CREATE INDEX idx_orders_created ON orders(created_at, order_id);
CREATE INDEX idx_order_items_order ON order_items(order_id, product_id);
CREATE INDEX idx_products_category ON products(category_id, product_id);
```

---

### ตัวอย่างที่ 30: User Search Optimization

```sql
-- ❌ Before: Case-insensitive search ที่ช้า
SELECT * FROM users 
WHERE LOWER(first_name) = LOWER('john') 
   OR LOWER(last_name) = LOWER('john');

-- ✅ After: Expression indexes + UNION
CREATE INDEX idx_users_lower_fname ON users(LOWER(first_name));
CREATE INDEX idx_users_lower_lname ON users(LOWER(last_name));

SELECT * FROM users WHERE LOWER(first_name) = LOWER('john')
UNION
SELECT * FROM users WHERE LOWER(last_name) = LOWER('john');
```

---

## 9. Common Optimization Checklist

```
การ Optimize Query - รายการตรวจสอบ:

□ 1. WHERE clause ใช้ functions บน indexed columns หรือไม่?
     ถ้าใช้ → rewrite หรือสร้าง expression index

□ 2. Type mismatch ระหว่าง column และ literal value หรือไม่?
     ถ้ามี → ใช้ literal ที่ type ตรงกัน

□ 3. LIKE '%...' leading wildcard หรือไม่?
     ถ้ามี → full-text search หรือ trigram index

□ 4. SELECT * ที่ไม่จำเป็น?
     ถ้ามี → ระบุ columns ที่ต้องการ (อาจเป็น covering index)

□ 5. Correlated subquery หรือไม่?
     ถ้ามี → rewrite เป็น JOIN หรือ window function

□ 6. NOT IN ที่อาจมี NULL?
     ถ้ามี → เปลี่ยนเป็น NOT EXISTS หรือ LEFT JOIN IS NULL

□ 7. OR ระหว่าง columns ต่างกัน?
     ถ้ามี → พิจารณา UNION ALL หรือ IN

□ 8. DISTINCT ที่ไม่จำเป็น?
     ถ้ามี → ตรวจว่า JOIN ทำให้เกิด duplicates หรือไม่ → แก้ที่ join

□ 9. ORDER BY + LIMIT โดยไม่มี index?
     ถ้ามี → สร้าง index ตาม ORDER BY column

□ 10. Large OFFSET ใน pagination?
      ถ้ามี → ใช้ keyset pagination แทน
```

---

## แบบฝึกหัด (10 ข้อ)

**ข้อ 1:** Rewrite query ต่อไปนี้ให้ใช้ index ได้:
```sql
SELECT * FROM sales WHERE YEAR(sale_date) = 2024 AND MONTH(sale_date) = 3;
```

**เฉลยข้อ 1:**
```sql
SELECT * FROM sales 
WHERE sale_date >= '2024-03-01' AND sale_date < '2024-04-01';
-- Range query ที่ใช้ index บน sale_date ได้
```

---

**ข้อ 2:** Query ต่อไปนี้มีปัญหาอะไร และแก้อย่างไร:
```sql
SELECT * FROM customers WHERE phone + 0 = 812345678;
```

**เฉลยข้อ 2:**
`phone + 0` เป็น arithmetic บน phone column (VARCHAR) ทำให้ database ต้องแปลงทุก row เป็น number ก่อน compare → ไม่ใช้ index แก้ไขโดย:
```sql
SELECT * FROM customers WHERE phone = '0812345678';
-- หรือ
SELECT * FROM customers WHERE phone = CAST(812345678 AS VARCHAR);
```

---

**ข้อ 3:** Rewrite correlated subquery ต่อไปนี้ให้เร็วขึ้น:
```sql
SELECT product_id, name, price,
    (SELECT AVG(review_score) FROM reviews WHERE product_id = p.product_id) AS avg_score
FROM products p;
```

**เฉลยข้อ 3:**
```sql
SELECT p.product_id, p.name, p.price, r.avg_score
FROM products p
LEFT JOIN (
    SELECT product_id, AVG(review_score) AS avg_score
    FROM reviews
    GROUP BY product_id
) r ON p.product_id = r.product_id;
-- Compute avg_score ครั้งเดียวต่อ product ไม่ใช่ทุก row!
```

---

**ข้อ 4:** อธิบายว่าทำไม `NOT IN (subquery)` อาจให้ผลลัพธ์ผิดเมื่อ subquery มี NULL?

**เฉลยข้อ 4:**
`NOT IN` ใช้ logic `x != v1 AND x != v2 AND ... AND x != NULL` เนื่องจาก `x != NULL` เป็น UNKNOWN (ไม่ใช่ TRUE) ดังนั้น `NOT IN` ที่มี NULL ใน subquery จะไม่คืน rows ใดเลย! แก้โดยใช้ `NOT EXISTS` หรือเพิ่ม `WHERE col IS NOT NULL` ใน subquery

---

**ข้อ 5:** เขียน query ที่ใช้ Keyset Pagination แทน OFFSET สำหรับตาราง `posts` (post_id, created_at, title)

**เฉลยข้อ 5:**
```sql
-- Page 1:
SELECT post_id, created_at, title
FROM posts
ORDER BY created_at DESC, post_id DESC
LIMIT 20;
-- บันทึก last row: created_at = '2024-01-15 10:30:00', post_id = 1234

-- Page 2 (keyset):
SELECT post_id, created_at, title
FROM posts
WHERE (created_at, post_id) < ('2024-01-15 10:30:00', 1234)
ORDER BY created_at DESC, post_id DESC
LIMIT 20;
-- ใช้ composite index (created_at DESC, post_id DESC)
```

---

**ข้อ 6:** Query ต่อไปนี้จะ optimize ได้อย่างไร:
```sql
SELECT DISTINCT customer_id FROM orders WHERE status = 'completed';
```

**เฉลยข้อ 6:**
```sql
-- Option 1: GROUP BY (บางครั้งเร็วกว่า DISTINCT)
SELECT customer_id FROM orders WHERE status = 'completed' GROUP BY customer_id;

-- Option 2: ถ้าต้องการแค่รู้ว่า "customer มี order ไหม"
-- ใช้ index (status, customer_id) เป็น covering index
CREATE INDEX idx_orders_status_customer ON orders(status, customer_id);
SELECT DISTINCT customer_id FROM orders WHERE status = 'completed';
-- Index Only Scan + deduplication
```

---

**ข้อ 7:** อธิบาย Sargable predicate พร้อมตัวอย่าง 5 ข้อที่ sargable และ 5 ข้อที่ไม่ sargable

**เฉลยข้อ 7:**
**Sargable (index-friendly):**
1. `WHERE emp_id = 1001` - equality
2. `WHERE salary BETWEEN 40000 AND 80000` - range
3. `WHERE created_at >= '2024-01-01'` - range comparison
4. `WHERE name LIKE 'Smith%'` - prefix
5. `WHERE dept_id IN (1, 2, 3)` - IN list

**Non-Sargable (ไม่ใช้ index):**
1. `WHERE YEAR(created_at) = 2024` - function on column
2. `WHERE salary * 1.1 > 50000` - arithmetic on column
3. `WHERE LOWER(name) = 'smith'` - function without expression index
4. `WHERE name LIKE '%Smith%'` - leading wildcard
5. `WHERE created_at::text = '2024-01-01'` - cast บน column

---

**ข้อ 8:** Rewrite query ต่อไปนี้ให้ดีขึ้น:
```sql
SELECT * FROM employees e
WHERE (SELECT COUNT(*) FROM attendance a WHERE a.emp_id = e.emp_id AND a.date = CURRENT_DATE) = 0;
```

**เฉลยข้อ 8:**
```sql
-- Option 1: NOT EXISTS (เร็วกว่า COUNT เพราะ short-circuit)
SELECT * FROM employees e
WHERE NOT EXISTS (
    SELECT 1 FROM attendance a 
    WHERE a.emp_id = e.emp_id AND a.date = CURRENT_DATE
);

-- Option 2: LEFT JOIN IS NULL
SELECT e.*
FROM employees e
LEFT JOIN attendance a ON e.emp_id = a.emp_id AND a.date = CURRENT_DATE
WHERE a.emp_id IS NULL;
```

---

**ข้อ 9:** สร้าง Expression Index สำหรับ query: `WHERE (first_name || ' ' || last_name) LIKE 'John Sm%'`

**เฉลยข้อ 9:**
```sql
-- สร้าง expression index บน full name
CREATE INDEX idx_employees_full_name 
ON employees((first_name || ' ' || last_name));

-- Query ที่ใช้ index:
SELECT * FROM employees 
WHERE (first_name || ' ' || last_name) LIKE 'John Sm%';
-- Prefix LIKE + Expression Index = ใช้ index ได้!

-- สำหรับ case-insensitive:
CREATE INDEX idx_employees_full_name_ci 
ON employees(LOWER(first_name || ' ' || last_name));
SELECT * FROM employees 
WHERE LOWER(first_name || ' ' || last_name) LIKE 'john sm%';
```

---

**ข้อ 10:** ออกแบบ optimization strategy สำหรับ query report ที่ช้า:
```sql
SELECT c.country, p.category, SUM(oi.quantity * oi.price) as revenue
FROM order_items oi
JOIN orders o ON oi.order_id = o.order_id
JOIN customers c ON o.customer_id = c.customer_id
JOIN products p ON oi.product_id = p.product_id
WHERE YEAR(o.created_at) = 2024
GROUP BY c.country, p.category
ORDER BY revenue DESC;
```

**เฉลยข้อ 10:**
```sql
-- 1. Fix sargable predicate
-- แก้ YEAR() → range

-- 2. สร้าง indexes ที่จำเป็น
CREATE INDEX idx_orders_created ON orders(created_at, customer_id, order_id);
CREATE INDEX idx_order_items_order ON order_items(order_id, product_id)
INCLUDE (quantity, price);
CREATE INDEX idx_customers_country ON customers(customer_id) INCLUDE (country);
CREATE INDEX idx_products_category ON products(product_id) INCLUDE (category);

-- 3. Query ที่ optimize แล้ว
SELECT c.country, p.category, SUM(oi.quantity * oi.price) as revenue
FROM orders o
JOIN customers c ON o.customer_id = c.customer_id
JOIN order_items oi ON oi.order_id = o.order_id
JOIN products p ON oi.product_id = p.product_id
WHERE o.created_at >= '2024-01-01' AND o.created_at < '2025-01-01'
GROUP BY c.country, p.category
ORDER BY revenue DESC;

-- 4. ถ้า query นี้รันบ่อย → Materialized View
CREATE MATERIALIZED VIEW mv_annual_sales AS
SELECT ...;
REFRESH MATERIALIZED VIEW mv_annual_sales;
```

---

## สรุป

ใน Part 066 เราได้เรียนรู้:

1. **Sargable predicates** - WHERE clauses ที่ index-friendly
2. **Index killers**: Functions on columns, type mismatch, leading wildcards
3. **30+ Before/After** optimization examples
4. **OR → UNION**, **NOT IN → NOT EXISTS**, **Subquery → JOIN**
5. **Covering Index Strategy** - ลด I/O ด้วย index ที่มีทุก columns
6. **Join Optimization** - indexes สำหรับ join conditions
7. **Query Hints** - ใช้เมื่อจำเป็น และข้อควรระวัง
8. **Optimization Checklist** - รายการตรวจสอบก่อน deploy

ใน Part 067 เราจะเรียนรู้เกี่ยวกับ Table Statistics และ Query Planner อย่างละเอียด
