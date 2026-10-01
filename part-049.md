# Part 49: Subquery Optimization

## 49.1 ทำความเข้าใจการทำงานของ Subquery

เมื่อ MySQL รัน query ที่มี subquery มีหลายแผน (execution plan) ที่อาจเกิดขึ้น:

```
1. Materialization: คำนวณ subquery ครั้งเดียว เก็บในตาราง temp
2. Merge/Pushdown: รวม subquery เข้ากับ outer query
3. Independent Execution: รัน subquery แยกก่อน
4. Correlated Execution: รัน subquery ซ้ำทุกแถว (ช้าที่สุด)
```

---

## 49.2 EXPLAIN เพื่อดู Execution Plan

### ตัวอย่างที่ 1: EXPLAIN พื้นฐาน

```sql
-- รัน EXPLAIN ก่อนทุก query ที่น่าสงสัย
EXPLAIN SELECT c.first_name, c.last_name
FROM   customers c
WHERE  c.customer_id IN (
    SELECT customer_id FROM orders WHERE total_amount > 20000
);
```

ผลลัพธ์ที่ต้องสังเกต:
```
id | select_type        | table    | type  | key     | rows | Extra
1  | PRIMARY            | c        | ALL   | NULL    | 10   |
2  | DEPENDENT SUBQUERY | orders   | ALL   | NULL    | 15   | Using where
   ↑ อันตราย: DEPENDENT SUBQUERY หมายถึงรัน subquery ทุกแถว
```

### ตัวอย่างที่ 2: EXPLAIN FORMAT=JSON

```sql
-- รายละเอียดมากขึ้น
EXPLAIN FORMAT=JSON
SELECT c.first_name
FROM   customers c
WHERE  c.customer_id IN (
    SELECT customer_id FROM orders GROUP BY customer_id HAVING COUNT(*) > 1
);
-- ดู attached_condition, using_index, cost_info
```

### ตัวอย่างที่ 3: EXPLAIN ANALYZE (MySQL 8.0.18+)

```sql
-- รันจริงและวัดเวลา
EXPLAIN ANALYZE
SELECT c.first_name
FROM   customers c
WHERE  EXISTS (
    SELECT 1 FROM orders WHERE customer_id = c.customer_id
);
-- แสดง actual time, actual rows, loops
```

---

## 49.3 select_type ใน EXPLAIN

```sql
-- ประเภท select_type ที่สำคัญ:
-- PRIMARY       - outer query หลัก
-- SUBQUERY      - subquery ที่ไม่ขึ้นกับ outer (รันครั้งเดียว)
-- DEPENDENT SUBQUERY - correlated subquery (รันทุกแถว) ⚠️
-- DERIVED       - subquery ใน FROM clause
-- MATERIALIZED  - subquery ที่ถูก materialize เป็น temp table

-- ตัวอย่าง SUBQUERY (ดี):
EXPLAIN SELECT * FROM employees
WHERE salary > (SELECT AVG(salary) FROM employees);
-- select_type = SUBQUERY (รันครั้งเดียว)

-- ตัวอย่าง DEPENDENT SUBQUERY (ช้า):
EXPLAIN SELECT * FROM employees e1
WHERE salary > (SELECT AVG(salary) FROM employees e2 WHERE e2.department_id = e1.department_id);
-- select_type = DEPENDENT SUBQUERY (รันทุกแถว)
```

---

## 49.4 เมื่อ MySQL Optimize Subquery เป็น JOIN

### ตัวอย่างที่ 4: IN → Semi-join Optimization

```sql
-- MySQL 8 มักแปลง IN subquery เป็น semi-join อัตโนมัติ

-- Query นี้:
SELECT * FROM products p
WHERE p.product_id IN (SELECT product_id FROM order_items);

-- MySQL อาจ optimize เป็น:
-- SELECT p.* FROM products p SEMI JOIN order_items oi ON oi.product_id = p.product_id;
-- (ใช้ materialization หรือ join strategy)

EXPLAIN SELECT * FROM products p
WHERE p.product_id IN (SELECT product_id FROM order_items);
-- ดู select_type: อาจเห็น MATERIALIZED แทน SUBQUERY
```

### ตัวอย่างที่ 5: EXISTS → Semi-join

```sql
-- EXISTS ก็ถูก optimize เป็น semi-join เช่นกัน
EXPLAIN SELECT p.product_name FROM products p
WHERE EXISTS (SELECT 1 FROM order_items WHERE product_id = p.product_id);
-- ดู type: จะเห็น semi-join ถ้า optimizer เลือก
```

---

## 49.5 Rewriting Correlated เป็น Non-correlated

### ตัวอย่างที่ 6: Before/After Pattern 1

```sql
-- ❌ BEFORE: Correlated (ช้า)
EXPLAIN
SELECT first_name, last_name, salary
FROM   employees e1
WHERE  salary > (
    SELECT AVG(salary)
    FROM   employees e2
    WHERE  e2.department_id = e1.department_id
);
-- DEPENDENT SUBQUERY → รัน 10 ครั้ง (10 พนักงาน)

-- ✓ AFTER: Non-correlated (เร็ว)
EXPLAIN
SELECT e.first_name, e.last_name, e.salary
FROM   employees e
JOIN (
    SELECT department_id, AVG(salary) AS avg_sal
    FROM   employees
    GROUP  BY department_id
) AS dept_avg ON dept_avg.department_id = e.department_id
             AND e.salary > dept_avg.avg_sal;
-- DERIVED + JOIN → รัน 2 ครั้งรวม
```

### ตัวอย่างที่ 7: Before/After Pattern 2

```sql
-- ❌ BEFORE: Multiple Correlated ใน SELECT
EXPLAIN
SELECT
    e.first_name,
    e.salary,
    (SELECT AVG(salary) FROM employees WHERE department_id = e.department_id) AS dept_avg,
    (SELECT MAX(salary) FROM employees WHERE department_id = e.department_id) AS dept_max,
    (SELECT COUNT(*)    FROM employees WHERE department_id = e.department_id) AS dept_count
FROM employees e;
-- รัน subquery 3 ชุด × 10 แถว = 30 subquery executions!

-- ✓ AFTER: One JOIN
EXPLAIN
SELECT
    e.first_name,
    e.salary,
    ds.avg_sal  AS dept_avg,
    ds.max_sal  AS dept_max,
    ds.emp_count AS dept_count
FROM employees e
JOIN (
    SELECT department_id,
           AVG(salary) AS avg_sal,
           MAX(salary) AS max_sal,
           COUNT(*)    AS emp_count
    FROM   employees
    GROUP  BY department_id
) AS ds ON ds.department_id = e.department_id;
-- รัน derived table 1 ครั้ง + 1 join
```

### ตัวอย่างที่ 8: Before/After Pattern 3

```sql
-- ❌ BEFORE: Correlated COUNT
EXPLAIN
SELECT c.first_name, c.last_name
FROM   customers c
WHERE  (SELECT COUNT(*) FROM orders WHERE customer_id = c.customer_id) > 2;
-- DEPENDENT SUBQUERY ทุกแถว

-- ✓ AFTER: JOIN + HAVING
EXPLAIN
SELECT c.first_name, c.last_name
FROM   customers c
JOIN (
    SELECT customer_id, COUNT(*) AS cnt
    FROM   orders
    GROUP  BY customer_id
    HAVING COUNT(*) > 2
) AS multi_orders ON multi_orders.customer_id = c.customer_id;
-- DERIVED + JOIN
```

---

## 49.6 Materialization ของ Subquery

### ตัวอย่างที่ 9: Understand Materialization

```sql
-- Subquery ใน FROM ถูก materialize เป็น temp table
EXPLAIN SELECT *
FROM (
    SELECT department_id, AVG(salary) AS avg_sal
    FROM   employees
    GROUP  BY department_id
) AS dept_avg
WHERE avg_sal > 80000;
-- select_type = DERIVED → materialized temp table

-- MySQL optimizer จะสร้าง temp table สำหรับ derived table
-- แล้ว query outer query บน temp table นั้น
```

### ตัวอย่างที่ 10: Force Materialization

```sql
-- ใช้ derived table เพื่อ force materialization และ filter ก่อน
SELECT e.*
FROM   employees e
JOIN (
    SELECT DISTINCT customer_id
    FROM   orders
    WHERE  total_amount > 10000
      AND  status = 'completed'
    -- ← filter ก่อน materialize ลด size ของ temp table
) AS big_customers ON big_customers.customer_id = e.employee_id;
```

---

## 49.7 Common Optimization Patterns

### ตัวอย่างที่ 11: Pattern - EXISTS แทน COUNT > 0

```sql
-- ❌ ช้า: COUNT ต้องนับทุกแถว
SELECT first_name FROM customers c
WHERE (SELECT COUNT(*) FROM orders WHERE customer_id = c.customer_id) > 0;

-- ✓ เร็ว: EXISTS short-circuit ทันทีที่เจอ match
SELECT first_name FROM customers c
WHERE EXISTS (SELECT 1 FROM orders WHERE customer_id = c.customer_id);
```

### ตัวอย่างที่ 12: Pattern - IN กับ Index

```sql
-- ตรวจว่ามี index บน column ที่ใช้ใน subquery
SHOW INDEX FROM orders;
-- ต้องมี index บน customer_id

-- ถ้าไม่มี index:
CREATE INDEX idx_orders_cust ON orders(customer_id);

-- หลังสร้าง index, IN subquery จะเร็วขึ้นมาก
EXPLAIN SELECT first_name FROM customers
WHERE customer_id IN (SELECT customer_id FROM orders WHERE status = 'completed');
```

### ตัวอย่างที่ 13: Pattern - Derived Table Filter First

```sql
-- ❌ Filter หลัง JOIN (ดึงข้อมูลมาก)
SELECT c.first_name, o.total_amount
FROM   customers c
JOIN   orders o ON o.customer_id = c.customer_id
WHERE  o.total_amount > 15000
  AND  c.city = 'Bangkok';

-- ✓ Filter ใน subquery ก่อน (ดึงข้อมูลน้อยลง)
SELECT c.first_name, big_orders.total_amount
FROM   customers c
JOIN (
    SELECT customer_id, total_amount
    FROM   orders
    WHERE  total_amount > 15000  -- ← filter ก่อน
) AS big_orders ON big_orders.customer_id = c.customer_id
WHERE c.city = 'Bangkok';
```

### ตัวอย่างที่ 14: Pattern - Avoid Redundant Subqueries

```sql
-- ❌ ช้า: คำนวณ subquery เดิมหลายครั้ง
SELECT product_name,
       price,
       price / (SELECT SUM(price) FROM products) * 100 AS pct_1,
       price - (SELECT AVG(price) FROM products)        AS diff,
       price / (SELECT MAX(price) FROM products) * 100  AS pct_max
FROM products;
-- Subquery รัน 3 ครั้งต่อแถว!

-- ✓ เร็ว: คำนวณครั้งเดียว
SELECT p.product_name,
       p.price,
       p.price / s.total_price * 100 AS pct_1,
       p.price - s.avg_price         AS diff,
       p.price / s.max_price * 100   AS pct_max
FROM products p
CROSS JOIN (
    SELECT SUM(price) AS total_price,
           AVG(price) AS avg_price,
           MAX(price) AS max_price
    FROM   products
) AS s;
```

### ตัวอย่างที่ 15: Pattern - CTE แทน Repeated Subquery

```sql
-- ❌ ช้า: subquery ซ้ำ
SELECT dept_id, avg_sal
FROM (
    SELECT department_id AS dept_id, AVG(salary) AS avg_sal
    FROM   employees GROUP BY department_id
) AS d1
WHERE avg_sal > (
    SELECT AVG(avg_sal)
    FROM (
        SELECT department_id, AVG(salary) AS avg_sal
        FROM   employees GROUP BY department_id
    ) AS d2  -- ← subquery เดิม ซ้ำ!
);

-- ✓ เร็ว: ใช้ CTE
WITH dept_avg AS (
    SELECT department_id, AVG(salary) AS avg_sal
    FROM   employees
    GROUP  BY department_id
)
SELECT department_id AS dept_id, avg_sal
FROM   dept_avg
WHERE  avg_sal > (SELECT AVG(avg_sal) FROM dept_avg);
```

---

## 49.8 EXPLAIN Output Analysis

### ตัวอย่างที่ 16: อ่าน EXPLAIN ผลลัพธ์

```sql
EXPLAIN SELECT c.first_name
FROM   customers c
WHERE  c.customer_id IN (
    SELECT o.customer_id
    FROM   orders o
    WHERE  o.total_amount > 20000
);
```

```
id | select_type  | table | type | key     | rows | filtered | Extra
1  | SIMPLE       | c     | ALL  | NULL    | 10   | 100.00   |
1  | SIMPLE       | o     | ALL  | NULL    | 15   | 33.33    | Using where; FirstMatch(c)
   ↑ SIMPLE หมายถึง optimizer แปลงเป็น semi-join แล้ว (ดี!)
   ↑ FirstMatch = semi-join strategy
```

### ตัวอย่างที่ 17: Bad EXPLAIN

```sql
EXPLAIN SELECT c.first_name,
               (SELECT COUNT(*) FROM orders WHERE customer_id = c.customer_id) AS cnt
FROM customers c;
```

```
id | select_type        | table     | type | rows | Extra
1  | PRIMARY            | c         | ALL  | 10   |
2  | DEPENDENT SUBQUERY | orders    | ALL  | 15   | Using where
   ↑ DEPENDENT SUBQUERY = รันซ้ำทุกแถว ⚠️
```

### ตัวอย่างที่ 18: Good EXPLAIN

```sql
EXPLAIN SELECT c.first_name, cnt_data.cnt
FROM customers c
LEFT JOIN (
    SELECT customer_id, COUNT(*) AS cnt
    FROM   orders
    GROUP  BY customer_id
) AS cnt_data ON cnt_data.customer_id = c.customer_id;
```

```
id | select_type | table     | type | rows | Extra
1  | PRIMARY     | c         | ALL  | 10   |
2  | DERIVED     | orders    | ALL  | 15   | Using temporary
1  | PRIMARY     | cnt_data  | ref  | 1    |
   ↑ DERIVED (ดี) + ref join (ดี)
```

---

## 49.9 Index Strategies

### ตัวอย่างที่ 19: Index บน Correlated Column

```sql
-- Correlated subquery ใช้ประโยชน์จาก index มาก

-- ก่อนสร้าง index:
EXPLAIN SELECT * FROM employees e1
WHERE salary > (SELECT AVG(salary) FROM employees WHERE department_id = e1.department_id);
-- rows = 10 (full scan)

-- สร้าง index:
CREATE INDEX idx_emp_dept_sal ON employees(department_id, salary);

-- หลังสร้าง index:
EXPLAIN SELECT * FROM employees e1
WHERE salary > (SELECT AVG(salary) FROM employees WHERE department_id = e1.department_id);
-- rows = 2-3 (index range scan)
```

### ตัวอย่างที่ 20: Covering Index

```sql
-- Covering index: index ที่ครอบคลุมทุก column ที่ subquery ต้องการ
-- ไม่ต้องกลับไปอ่าน table data

-- ตัวอย่าง: subquery ใช้แค่ customer_id และ total_amount
CREATE INDEX idx_orders_covering ON orders(customer_id, total_amount);

EXPLAIN SELECT c.first_name FROM customers c
WHERE c.customer_id IN (
    SELECT customer_id FROM orders WHERE total_amount > 10000
);
-- Extra: Using index (ไม่ต้องอ่าน table row)
```

---

## 49.10 Query Optimization Checklist

### ตัวอย่างที่ 21-25 (Checklist):

```sql
-- CHECKLIST ก่อน deploy query ที่มี subquery:

-- 1. EXPLAIN ดูว่ามี DEPENDENT SUBQUERY หรือไม่
EXPLAIN [your query];

-- 2. ถ้ามี DEPENDENT SUBQUERY → แปลงเป็น JOIN
-- BEFORE: correlated
SELECT * FROM customers c WHERE (SELECT COUNT(*) FROM orders WHERE customer_id = c.customer_id) > 0;
-- AFTER: semi-join
SELECT DISTINCT c.* FROM customers c JOIN orders o ON o.customer_id = c.customer_id;

-- 3. ตรวจ index บน column ที่ใช้ใน subquery
SHOW INDEX FROM orders;  -- ตรวจ customer_id, product_id

-- 4. ถ้า subquery ซ้ำ → ใช้ CTE หรือ CROSS JOIN
-- BEFORE: subquery ซ้ำ 3 ครั้ง
SELECT price / (SELECT MAX(price) FROM products) * 100 FROM products;
-- AFTER: CROSS JOIN
SELECT p.price / m.max_price * 100
FROM products p CROSS JOIN (SELECT MAX(price) AS max_price FROM products) m;

-- 5. Filter ใน subquery ก่อนเมื่อทำได้
-- BEFORE:
SELECT * FROM orders WHERE customer_id IN (SELECT customer_id FROM customers WHERE city = 'Bangkok');
-- AFTER: ดีกว่า (filter city ก่อน JOIN)
SELECT o.* FROM orders o
JOIN customers c ON c.customer_id = o.customer_id AND c.city = 'Bangkok';

-- 6. ใช้ EXISTS แทน IN เมื่อ subquery table ใหญ่
-- 7. ใช้ NOT EXISTS แทน NOT IN เสมอ (NULL safety + performance)
-- 8. ใช้ LIMIT 1 ใน subquery ที่ต้องการแค่แถวเดียว
```

---

## แบบฝึกหัดบทที่ 49

**ข้อ 1:** ใช้ EXPLAIN ดู execution plan ของ query ที่มี correlated subquery

```sql
-- เฉลย:
EXPLAIN SELECT c.first_name, c.last_name
FROM   customers c
WHERE  (
    SELECT SUM(total_amount)
    FROM   orders
    WHERE  customer_id = c.customer_id
) > 20000;
-- สังเกต select_type = DEPENDENT SUBQUERY
```

**ข้อ 2:** แปลง correlated subquery ต่อไปนี้เป็น JOIN และเปรียบเทียบ plan

```sql
-- Original:
SELECT e.first_name FROM employees e
WHERE (SELECT COUNT(*) FROM employees WHERE department_id = e.department_id) > 2;

-- เฉลย (แปลงเป็น JOIN):
EXPLAIN
SELECT e.first_name FROM employees e
JOIN (
    SELECT department_id, COUNT(*) AS emp_count
    FROM   employees
    GROUP  BY department_id
    HAVING COUNT(*) > 2
) AS big_depts ON big_depts.department_id = e.department_id;
```

**ข้อ 3:** เขียน optimized query แทน redundant subqueries ใน SELECT

```sql
-- Original (ช้า):
SELECT product_name,
       price,
       (SELECT MIN(price) FROM products) AS global_min,
       (SELECT MAX(price) FROM products) AS global_max,
       (SELECT AVG(price) FROM products) AS global_avg
FROM products;

-- เฉลย (เร็ว):
SELECT p.product_name, p.price,
       s.global_min, s.global_max, s.global_avg
FROM products p
CROSS JOIN (
    SELECT MIN(price) AS global_min,
           MAX(price) AS global_max,
           AVG(price) AS global_avg
    FROM   products
) AS s;
```

**ข้อ 4:** ระบุว่า EXPLAIN output ต่อไปนี้มีปัญหาอะไรและแก้ไขอย่างไร

```sql
-- Query ที่มีปัญหา:
SELECT c.first_name,
       (SELECT COUNT(*) FROM orders WHERE customer_id = c.customer_id) AS cnt,
       (SELECT SUM(total_amount) FROM orders WHERE customer_id = c.customer_id) AS total
FROM customers c;

-- เฉลย:
-- ปัญหา: DEPENDENT SUBQUERY 2 ชุด × จำนวนแถว customers
-- แก้ไข:
SELECT c.first_name, o_stats.cnt, o_stats.total
FROM customers c
LEFT JOIN (
    SELECT customer_id,
           COUNT(*) AS cnt,
           SUM(total_amount) AS total
    FROM   orders
    GROUP  BY customer_id
) AS o_stats ON o_stats.customer_id = c.customer_id;
```

**ข้อ 5:** สร้าง index ที่เหมาะสมสำหรับ query นี้และอธิบายทำไม

```sql
-- Query:
SELECT p.product_name
FROM   products p
WHERE  EXISTS (
    SELECT 1
    FROM   order_items oi
    WHERE  oi.product_id = p.product_id
      AND  oi.unit_price > 10000
);

-- เฉลย:
-- สร้าง composite index:
CREATE INDEX idx_oi_product_price ON order_items(product_id, unit_price);
-- เหตุผล: EXISTS subquery ค้นหาด้วย product_id + unit_price
-- Covering index: ไม่ต้องกลับไปอ่าน row data

EXPLAIN SELECT p.product_name
FROM   products p
WHERE  EXISTS (
    SELECT 1 FROM order_items oi
    WHERE oi.product_id = p.product_id AND oi.unit_price > 10000
);
```

**ข้อ 6:** เปรียบเทียบ EXISTS กับ IN สำหรับ large table โดยใช้ EXPLAIN ANALYZE

```sql
-- เฉลย:
-- Query A: EXISTS
EXPLAIN ANALYZE
SELECT p.product_name FROM products p
WHERE EXISTS (SELECT 1 FROM order_items WHERE product_id = p.product_id);

-- Query B: IN
EXPLAIN ANALYZE
SELECT p.product_name FROM products p
WHERE product_id IN (SELECT DISTINCT product_id FROM order_items);

-- สังเกต actual time และ actual rows ของทั้งสอง
```

**ข้อ 7:** แปลง query ที่มี subquery ซ้อน 2 ชั้นใน WHERE ให้มีประสิทธิภาพดีขึ้น

```sql
-- Original:
SELECT c.first_name FROM customers c
WHERE c.customer_id IN (
    SELECT o.customer_id FROM orders o
    WHERE o.order_id IN (
        SELECT oi.order_id FROM order_items oi
        WHERE oi.product_id IN (
            SELECT p.product_id FROM products p WHERE p.category = 'Electronics'
        )
    )
);

-- เฉลย (JOIN ดีกว่า):
SELECT DISTINCT c.first_name
FROM   customers c
JOIN   orders o      ON o.customer_id   = c.customer_id
JOIN   order_items oi ON oi.order_id    = o.order_id
JOIN   products p    ON p.product_id    = oi.product_id
WHERE  p.category = 'Electronics';
```

**ข้อ 8:** ใช้ CROSS JOIN แทน scalar subquery ที่ทำงานซ้ำ

```sql
-- Original (ช้า):
SELECT
    order_id,
    total_amount,
    total_amount / (SELECT SUM(total_amount) FROM orders) * 100 AS pct_of_total
FROM orders;

-- เฉลย:
SELECT
    o.order_id,
    o.total_amount,
    ROUND(o.total_amount / s.grand_total * 100, 2) AS pct_of_total
FROM orders o
CROSS JOIN (SELECT SUM(total_amount) AS grand_total FROM orders) s;
```

**ข้อ 9:** ระบุว่า query ต่อไปนี้ควรมี index อะไรและสร้างมัน

```sql
-- Query:
SELECT p.product_name, SUM(oi.quantity) AS total_qty
FROM   products p
JOIN   order_items oi ON oi.product_id = p.product_id
GROUP  BY p.product_id, p.product_name
HAVING SUM(oi.quantity) > (
    SELECT AVG(qty_sum)
    FROM (
        SELECT product_id, SUM(quantity) AS qty_sum
        FROM   order_items
        GROUP  BY product_id
    ) AS t
);

-- เฉลย:
-- Index ที่ควรมี:
CREATE INDEX idx_oi_product ON order_items(product_id);
-- เหตุผล: JOIN และ GROUP BY ใช้ product_id
-- Derived table ใน subquery ก็ใช้ product_id

EXPLAIN [the query above];
```

**ข้อ 10:** เขียน optimized version ของ query นี้และแสดง EXPLAIN ของทั้งคู่

```sql
-- Original:
SELECT e.first_name, e.last_name, e.salary
FROM   employees e
WHERE  salary > (
    SELECT AVG(salary)
    FROM   employees e2
    WHERE  e2.department_id = e.department_id
);

-- เฉลย Optimized:
EXPLAIN
SELECT e.first_name, e.last_name, e.salary
FROM   employees e
JOIN (
    SELECT department_id, AVG(salary) AS dept_avg
    FROM   employees
    GROUP  BY department_id
) AS dept_stats ON dept_stats.department_id = e.department_id
WHERE e.salary > dept_stats.dept_avg;

-- เปรียบเทียบ:
-- Original: DEPENDENT SUBQUERY → รัน 10 ครั้ง
-- Optimized: DERIVED + JOIN → รัน 2 ครั้ง
```

---

*จบบทที่ 49: Subquery Optimization*
*บทถัดไป: Part 50 - Subquery Projects - Real-world Complex Queries*
