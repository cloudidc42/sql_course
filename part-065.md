# Part 065: Reading Query Execution Plans

## การอ่าน Query Execution Plans

---

## บทนำ

Query Execution Plan (หรือ Execution Plan) คือแผนที่ database engine ใช้ในการ execute query โดย planner จะเลือกวิธีที่มีประสิทธิภาพสูงสุดตาม statistics ที่มีอยู่ การเข้าใจ execution plan ช่วยให้เราระบุปัญหา performance ได้อย่างแม่นยำ

---

## 1. EXPLAIN พื้นฐาน (PostgreSQL)

```sql
-- Syntax พื้นฐาน
EXPLAIN SELECT * FROM employees WHERE department_id = 5;

-- ผลลัพธ์:
/*
QUERY PLAN
-----------------------------------------------------------------
Seq Scan on employees  (cost=0.00..2500.00 rows=50 width=100)
  Filter: (department_id = 5)
(2 rows)
*/
```

### อ่านค่า cost อย่างไร?

```
(cost=0.00..2500.00 rows=50 width=100)
        │         │      │        │
        │         │      │        └── ความกว้างเฉลี่ยของแต่ละ row (bytes)
        │         │      └─────────── จำนวน rows ที่ planner ประมาณ
        │         └────────────────── total cost (ดึงข้อมูลทั้งหมด)
        └──────────────────────────── startup cost (cost ก่อนได้ row แรก)

หน่วย cost: arbitrary unit ไม่ใช่มิลลิวินาที
สูงกว่า = แพงกว่า (คาดว่า)
```

---

## 2. EXPLAIN ANALYZE (PostgreSQL)

EXPLAIN ANALYZE จะ **execute จริง** และแสดงเวลาจริง

```sql
-- ตัวอย่างที่ 1: EXPLAIN ANALYZE พื้นฐาน
EXPLAIN ANALYZE
SELECT * FROM employees WHERE department_id = 5;

/*
QUERY PLAN
-----------------------------------------------------------------
Seq Scan on employees  
  (cost=0.00..2500.00 rows=50 width=100)
  (actual time=0.012..45.234 rows=47 loops=1)
  Filter: (department_id = 5)
  Rows Removed by Filter: 99953
Planning Time: 0.234 ms
Execution Time: 45.456 ms
(5 rows)
*/
```

### อธิบายผลลัพธ์

```
Seq Scan on employees
  (cost=0.00..2500.00 rows=50 width=100)    ← ประมาณการณ์ของ planner
  (actual time=0.012..45.234 rows=47 loops=1) ← ผลจริงที่ได้

actual time=0.012..45.234:
  - 0.012: เวลา (ms) ก่อนได้ row แรก
  - 45.234: เวลา (ms) ทั้งหมด

rows=47: แถวจริงที่ได้ (vs rows=50 ที่ประมาณ)
loops=1: node นี้ถูกเรียกกี่ครั้ง

Rows Removed by Filter: 99953
→ scan 100,000 แถว ได้ 47 ที่ตรง filter
→ Selectivity = 47/100,000 = 0.047% → ควรมี index!
```

---

## 3. Plan Nodes ที่สำคัญ

### 3.1 Sequential Scan (Seq Scan)

```sql
-- ตัวอย่างที่ 2: Seq Scan
EXPLAIN ANALYZE
SELECT * FROM employees;

/*
Seq Scan on employees  
  (cost=0.00..1500.00 rows=100000 width=100)
  (actual time=0.015..235.678 rows=100000 loops=1)
Planning Time: 0.123 ms
Execution Time: 312.456 ms
*/

-- อ่านข้อมูล: Full table scan - อ่านทุกแถว
-- เมื่อไหร่จะเห็น: ไม่มี index, selectivity สูง, ตารางเล็ก
```

### 3.2 Index Scan

```sql
-- ตัวอย่างที่ 3: Index Scan
-- มี index บน department_id
EXPLAIN ANALYZE
SELECT * FROM employees WHERE department_id = 5;

/*
Index Scan using idx_emp_dept on employees  
  (cost=0.42..8.44 rows=5 width=100)
  (actual time=0.023..0.089 rows=5 loops=1)
  Index Cond: (department_id = 5)
Planning Time: 0.234 ms
Execution Time: 0.123 ms
*/

-- อ่านข้อมูล:
-- B-tree lookup เพื่อหา row pointers
-- แล้ว fetch rows จาก heap
-- เหมาะสำหรับ: few rows, high selectivity
```

### 3.3 Index Only Scan

```sql
-- ตัวอย่างที่ 4: Index Only Scan (covering index)
-- มี index: (department_id, salary)
EXPLAIN ANALYZE
SELECT department_id, salary FROM employees WHERE department_id = 5;

/*
Index Only Scan using idx_emp_dept_salary on employees  
  (cost=0.42..4.44 rows=5 width=12)
  (actual time=0.018..0.045 rows=5 loops=1)
  Index Cond: (department_id = 5)
  Heap Fetches: 0    ← ไม่ต้อง fetch heap เลย!
Planning Time: 0.189 ms
Execution Time: 0.067 ms
*/

-- อ่านข้อมูล:
-- ข้อมูลทั้งหมดอยู่ใน index
-- ไม่ต้อง access table เลย
-- Heap Fetches: 0 = perfect covering index
```

### 3.4 Bitmap Index Scan

```sql
-- ตัวอย่างที่ 5: Bitmap Index Scan (ดึงข้อมูลปริมาณปานกลาง)
EXPLAIN ANALYZE
SELECT * FROM orders WHERE customer_id = 1001 AND status = 'pending';

/*
Bitmap Heap Scan on orders  
  (cost=12.45..456.78 rows=100 width=150)
  (actual time=0.234..12.345 rows=87 loops=1)
  Recheck Cond: ((customer_id = 1001) AND (status = 'pending'))
  Heap Blocks: exact=45
  -> BitmapAnd  
       (cost=12.45..12.45 rows=100 width=0)
       (actual time=0.189..0.189 rows=0 loops=1)
     -> Bitmap Index Scan on idx_orders_customer  
          (cost=0.00..6.12 rows=150 width=0)
          (actual time=0.098..0.098 rows=150 loops=1)
          Index Cond: (customer_id = 1001)
     -> Bitmap Index Scan on idx_orders_status  
          (cost=0.00..6.23 rows=500 width=0)
          (actual time=0.067..0.067 rows=500 loops=1)
          Index Cond: (status = 'pending')
*/

-- อ่านข้อมูล:
-- สร้าง bitmap จาก 2 indexes แล้ว AND กัน
-- ดีกว่า Index Scan เมื่อ rows มาก + multiple conditions
-- Recheck Cond: ต้อง verify ใน heap (เพราะ bitmap อาจ lossy)
```

### 3.5 Hash Join

```sql
-- ตัวอย่างที่ 6: Hash Join
EXPLAIN ANALYZE
SELECT e.name, d.dept_name
FROM employees e
JOIN departments d ON e.department_id = d.department_id;

/*
Hash Join  
  (cost=5.00..2500.00 rows=1000 width=50)
  (actual time=1.234..234.567 rows=1000 loops=1)
  Hash Cond: (e.department_id = d.department_id)
  -> Seq Scan on employees e  
       (cost=0.00..1500.00 rows=100000 width=30)
       (actual time=0.012..123.456 rows=100000 loops=1)
  -> Hash  
       (cost=3.00..3.00 rows=50 width=20)
       (actual time=0.345..0.345 rows=50 loops=1)
       Buckets: 64  Batches: 1  Memory Usage: 9kB
       -> Seq Scan on departments d  
            (cost=0.00..3.00 rows=50 width=20)
            (actual time=0.012..0.234 rows=50 loops=1)
*/

-- อ่านข้อมูล:
-- Build phase: สร้าง hash table จาก departments (ตารางเล็ก)
-- Probe phase: scan employees แล้ว lookup ใน hash table
-- ดีสำหรับ: ตารางที่มีขนาดต่างกันมาก
```

### 3.6 Nested Loop Join

```sql
-- ตัวอย่างที่ 7: Nested Loop Join
EXPLAIN ANALYZE
SELECT e.name, d.dept_name
FROM employees e
JOIN departments d ON e.department_id = d.department_id
WHERE e.department_id = 5;

/*
Nested Loop  
  (cost=0.42..10.44 rows=5 width=50)
  (actual time=0.023..0.089 rows=5 loops=1)
  -> Index Scan using idx_emp_dept on employees e  
       (cost=0.42..4.44 rows=5 width=30)
       (actual time=0.018..0.056 rows=5 loops=1)
       Index Cond: (department_id = 5)
  -> Index Scan using departments_pkey on departments d  
       (cost=0.15..1.19 rows=1 width=20)
       (actual time=0.005..0.005 rows=1 loops=5)
       Index Cond: (department_id = 5)
*/

-- อ่านข้อมูล:
-- Loop ผ่าน outer table (employees)
-- สำหรับแต่ละ row ค้นหาใน inner table (departments)
-- loops=5: inner scan ถูกเรียก 5 ครั้ง (1 ครั้งต่อ employee)
-- ดีสำหรับ: outer set เล็ก, inner มี index
```

### 3.7 Merge Join

```sql
-- ตัวอย่างที่ 8: Merge Join
EXPLAIN ANALYZE
SELECT e.name, d.dept_name
FROM employees e
JOIN departments d ON e.department_id = d.department_id
ORDER BY e.department_id;

/*
Merge Join  
  (cost=0.85..5000.00 rows=100000 width=50)
  (actual time=0.023..456.789 rows=100000 loops=1)
  Merge Cond: (e.department_id = d.department_id)
  -> Index Scan using idx_emp_dept on employees e  
       (cost=0.42..3456.78 rows=100000 width=30)
  -> Index Scan using departments_pkey on departments d  
       (cost=0.15..4.50 rows=50 width=20)
*/

-- อ่านข้อมูล:
-- ทั้งสองตารางถูก sort ตาม join key
-- Scan ทั้งสองพร้อมกัน merge ตาม sort order
-- ดีสำหรับ: ทั้งสองตาราง sorted แล้ว (มี index)
```

---

## 4. EXPLAIN ใน MySQL

```sql
-- ตัวอย่างที่ 9: MySQL EXPLAIN
EXPLAIN SELECT * FROM employees WHERE department_id = 5;

/*
+----+-------------+-----------+------------+------+------------------+------------------+---------+-------+------+----------+-------+
| id | select_type | table     | partitions | type | possible_keys    | key              | key_len | ref   | rows | filtered | Extra |
+----+-------------+-----------+------------+------+------------------+------------------+---------+-------+------+----------+-------+
|  1 | SIMPLE      | employees | NULL       | ref  | idx_emp_dept_id  | idx_emp_dept_id  | 4       | const |   50 |   100.00 | NULL  |
+----+-------------+-----------+------------+------+------------------+------------------+---------+-------+------+----------+-------+
*/
```

### ความหมายของ columns ใน MySQL EXPLAIN

```
type (access method) - เรียงจากดีที่สุดไปแย่ที่สุด:
  const   = PK/Unique key lookup (เร็วที่สุด!)
  eq_ref  = PK/Unique key ใน JOIN
  ref     = Non-unique index lookup
  range   = Index range scan
  index   = Full index scan  
  ALL     = Full table scan (แย่ที่สุด!)

key: index ที่ถูกใช้จริง
key_len: ความยาวของ index key ที่ใช้
rows: จำนวนแถวที่ต้องอ่าน (ประมาณ)
filtered: % ของแถวที่ผ่าน WHERE
Extra: ข้อมูลเพิ่มเติมสำคัญ
  "Using index" = Index Only Scan (ดีมาก!)
  "Using where" = Filter หลัง read
  "Using filesort" = ต้อง sort เพิ่ม (อาจช้า)
  "Using temporary" = ใช้ temp table (ช้า)
  "Using join buffer" = Hash join (ปกติ)
```

```sql
-- ตัวอย่างที่ 10: MySQL EXPLAIN FORMAT=TREE (MySQL 8.0+)
EXPLAIN FORMAT=TREE
SELECT e.name, d.dept_name
FROM employees e
JOIN departments d ON e.dept_id = d.dept_id
WHERE e.salary > 50000;

/*
-> Nested loop inner join  (cost=1234.56 rows=456)
    -> Filter: (e.salary > 50000)  (cost=567.89 rows=456)
        -> Table scan on e  (cost=567.89 rows=100000)
    -> Single-row index lookup on d using PRIMARY (dept_id=e.dept_id)  
         (cost=0.25 rows=1)
*/
```

```sql
-- ตัวอย่างที่ 11: MySQL EXPLAIN ANALYZE (MySQL 8.0.18+)
EXPLAIN ANALYZE
SELECT * FROM employees WHERE salary > 50000;

/*
-> Filter: (employees.salary > 50000)  
     (cost=12345.67 rows=30000) 
     (actual time=0.034..234.567 rows=28456 loops=1)
    -> Table scan on employees  
         (cost=12345.67 rows=100000) 
         (actual time=0.023..189.345 rows=100000 loops=1)
*/
```

---

## 5. Execution Plan ใน SQL Server

```sql
-- ตัวอย่างที่ 12: SQL Server SET STATISTICS
-- Text plan
SET STATISTICS IO ON;
SET STATISTICS TIME ON;

SELECT * FROM employees WHERE department_id = 5;

/*
SQL Server Execution Times:
   CPU time = 15 ms,  elapsed time = 23 ms.
Table 'employees'. Scan count 1, logical reads 45, ...
*/

-- ดู Execution Plan แบบ graphical ใน SSMS: Ctrl+M
-- ดูแบบ text:
SET SHOWPLAN_TEXT ON;
GO
SELECT * FROM employees WHERE department_id = 5;
GO
SET SHOWPLAN_TEXT OFF;
```

---

## 6. EXPLAIN QUERY PLAN (SQLite)

```sql
-- ตัวอย่างที่ 13: SQLite EXPLAIN QUERY PLAN
EXPLAIN QUERY PLAN
SELECT * FROM employees WHERE department_id = 5;

/*
QUERY PLAN
--SCAN employees USING INDEX idx_emp_dept
*/

-- ตัวอย่างที่มี join:
EXPLAIN QUERY PLAN
SELECT e.name, d.name 
FROM employees e 
JOIN departments d ON e.dept_id = d.dept_id;

/*
QUERY PLAN
--SCAN employees e
--SEARCH d USING INTEGER PRIMARY KEY (rowid=?)
*/
```

---

## 7. ตัวอย่าง EXPLAIN ที่ซับซ้อน พร้อมคำอธิบาย

### ตัวอย่างที่ 14: Query ที่มีหลาย operations

```sql
EXPLAIN ANALYZE
SELECT 
    d.dept_name,
    COUNT(e.emp_id) AS emp_count,
    AVG(e.salary) AS avg_salary
FROM employees e
JOIN departments d ON e.department_id = d.department_id
WHERE e.hire_date >= '2020-01-01'
GROUP BY d.dept_name
ORDER BY avg_salary DESC;
```

**ผลลัพธ์ EXPLAIN ANALYZE:**
```
Sort  (cost=1234.56..1234.67 rows=50 width=36)
      (actual time=45.234..45.245 rows=50 loops=1)
  Sort Key: (avg(e.salary)) DESC
  Sort Method: quicksort  Memory: 28kB
  →  HashAggregate  (cost=1200.00..1231.25 rows=50 width=36)
       (actual time=44.456..44.567 rows=50 loops=1)
       Group Key: d.dept_name
       Batches: 1  Memory Usage: 40kB
       →  Hash Join  (cost=5.00..1150.00 rows=10000 width=30)
             (actual time=0.234..38.456 rows=9856 loops=1)
             Hash Cond: (e.department_id = d.department_id)
             →  Seq Scan on employees e  
                  (cost=0.00..1100.00 rows=10000 width=20)
                  (actual time=0.015..25.234 rows=9856 loops=1)
                  Filter: (hire_date >= '2020-01-01'::date)
                  Rows Removed by Filter: 90144
             →  Hash  
                  (cost=3.50..3.50 rows=50 width=20)
                  (actual time=0.098..0.098 rows=50 loops=1)
                  Buckets: 64  Batches: 1  Memory Usage: 11kB
                  →  Seq Scan on departments d  
                       (cost=0.00..3.50 rows=50 width=20)
                       (actual time=0.008..0.056 rows=50 loops=1)
Planning Time: 0.456 ms
Execution Time: 45.345 ms
```

**การวิเคราะห์:**
```
1. Seq Scan on employees (25ms): อ่าน 100,000 แถว ได้ 9,856
   → Rows Removed: 90,144 (90%!) → ควรมี index บน hire_date!

2. Seq Scan on departments: อ่าน 50 แถว → ตารางเล็ก, ปกติ

3. Hash Join: สร้าง hash table จาก departments (เล็ก)
   แล้ว probe ด้วย employees → ดีแล้ว

4. HashAggregate: GROUP BY dept_name → ใช้ memory ปกติ

5. Sort: ORDER BY avg_salary → quicksort ใน memory → ดีแล้ว

ปัญหาหลัก: hire_date ไม่มี index → scan 100K แถว ได้ 10K
แก้: CREATE INDEX idx_emp_hire_date ON employees(hire_date);
```

---

### ตัวอย่างที่ 15: Subquery Plan

```sql
EXPLAIN ANALYZE
SELECT * FROM products p
WHERE p.category_id IN (
    SELECT c.category_id FROM categories c
    WHERE c.parent_category = 'Electronics'
);
```

**ผลลัพธ์:**
```
Hash Semi Join  (cost=5.00..1500.00 rows=100 width=80)
                (actual time=0.345..45.234 rows=150 loops=1)
  Hash Cond: (p.category_id = c.category_id)
  →  Seq Scan on products p  
       (cost=0.00..1200.00 rows=50000 width=80)
       (actual time=0.012..23.456 rows=50000 loops=1)
  →  Hash  
       (cost=4.50..4.50 rows=30 width=4)
       (actual time=0.234..0.234 rows=30 loops=1)
       →  Seq Scan on categories c  
            (cost=0.00..4.50 rows=30 width=4)
            (actual time=0.008..0.123 rows=30 loops=1)
            Filter: (parent_category = 'Electronics')
            Rows Removed by Filter: 70
```

**การวิเคราะห์:**
```
PostgreSQL แปลง IN subquery → Hash Semi Join อัตโนมัติ
ไม่ต้อง rewrite เป็น JOIN เอง!

Hash Semi Join: ดีกว่า Nested Loop สำหรับ IN subquery
เพราะสร้าง hash table ครั้งเดียว

ปัญหา: Seq Scan บน products 50,000 แถว
แก้: ถ้ามี composite index (category_id, ...) ใน products จะดีกว่า
```

---

## 8. Plan Analysis Workflow

### กระบวนการวิเคราะห์ Plan

```
ขั้นตอนที่ 1: ดูที่ bottom-up (เริ่มจาก leaf nodes)
  → ดูว่าแต่ละ node อ่านข้อมูลอะไร

ขั้นตอนที่ 2: ตรวจสอบ estimated vs actual rows
  → ถ้าต่างกันมาก → statistics อาจล้าสมัย
  → รัน ANALYZE หรือ ANALYZE table

ขั้นตอนที่ 3: หา Seq Scan บน large tables
  → Rows Removed by Filter มาก? → ต้องการ index!

ขั้นตอนที่ 4: ดู Join types
  → Hash Join: ดีสำหรับ large tables
  → Nested Loop: ดีสำหรับ small result set + index
  → Merge Join: ดีสำหรับ sorted data

ขั้นตอนที่ 5: ดู Sort operations
  → Sort Method: quicksort ใน memory = ดี
  → Sort Method: external merge = disk sort = ช้า! เพิ่ม work_mem

ขั้นตอนที่ 6: ดูเวลารวม
  → Planning Time vs Execution Time
  → Node ไหนใช้เวลามากที่สุด?
```

---

## 9. Cost Estimates vs Actual

### เมื่อ Estimated ≠ Actual

```sql
-- ตัวอย่างที่ 16: estimated vs actual ต่างกันมาก
EXPLAIN ANALYZE
SELECT * FROM orders WHERE created_at = '2024-01-15';

/*
Seq Scan on orders (cost=0.00..25000.00 rows=1 width=150)
                   (actual time=0.012..234.567 rows=50000 loops=1)
  Filter: (created_at = '2024-01-15')
  Rows Removed by Filter: 950000
*/

-- estimated rows=1 แต่ actual rows=50000!
-- สาเหตุ: statistics ล้าสมัย หรือ column มี skewed distribution
-- แก้:
ANALYZE orders;  -- อัพเดต statistics
-- หรือเพิ่ม statistics_target:
ALTER TABLE orders ALTER COLUMN created_at SET STATISTICS 500;
ANALYZE orders;
```

---

## 10. ตัวอย่าง EXPLAIN ที่ควรรู้จัก (ทั้งหมด 15 ตัวอย่าง)

```sql
-- ตัวอย่างที่ 17: Query ที่ดีมาก (Index Only Scan)
EXPLAIN ANALYZE
SELECT customer_id, SUM(total_amount)
FROM orders
WHERE order_date >= '2024-01-01'
GROUP BY customer_id;

-- ถ้ามี covering index: (order_date, customer_id, total_amount)
/*
HashAggregate (cost=234.56..256.78 rows=1000 width=20)
  (actual time=45.234..46.789 rows=1000 loops=1)
  Group Key: customer_id
  -> Index Only Scan using idx_orders_cover on orders
       (cost=0.43..189.45 rows=15000 width=16)
       (actual time=0.023..23.456 rows=15234 loops=1)
       Index Cond: (order_date >= '2024-01-01')
       Heap Fetches: 0
*/
-- ดีมาก! Index Only Scan + Heap Fetches = 0
```

```sql
-- ตัวอย่างที่ 18: ปัญหา Function บน indexed column
EXPLAIN ANALYZE
SELECT * FROM employees WHERE UPPER(email) = 'SOMCHAI@EXAMPLE.COM';

/*
Seq Scan on employees (cost=0.00..2500.00 rows=1 width=100)
                      (actual time=0.012..234.567 rows=1 loops=1)
  Filter: (upper(email) = 'SOMCHAI@EXAMPLE.COM')
  Rows Removed by Filter: 99999
*/
-- Seq Scan! เพราะ UPPER() ครอบ column ที่มี index

-- แก้: สร้าง expression index
CREATE INDEX idx_emp_upper_email ON employees(UPPER(email));
-- หรือใช้ citext type
-- หรือ WHERE LOWER(email) = LOWER('...')
```

```sql
-- ตัวอย่างที่ 19: External Sort (ปัญหา!)
EXPLAIN ANALYZE
SELECT * FROM large_table ORDER BY created_at;

/*
Sort (cost=250000.00..252500.00 rows=1000000 width=100)
     (actual time=12345.678..15678.901 rows=1000000 loops=1)
  Sort Key: created_at
  Sort Method: external merge  Disk: 45678kB  ← ใช้ disk!
  -> Seq Scan on large_table (...)
*/
-- Sort Method: external merge = ช้า! เกิน work_mem
-- แก้: SET work_mem = '256MB'; หรือสร้าง index บน created_at
```

```sql
-- ตัวอย่างที่ 20: อ่าน plan แบบ JSON สำหรับ analysis ที่ละเอียด
EXPLAIN (FORMAT JSON, ANALYZE, BUFFERS)
SELECT * FROM orders WHERE customer_id = 1001;

-- ได้ output แบบ JSON ที่ใช้กับ tools เช่น:
-- https://explain.depesz.com/
-- https://explain.dalibo.com/
```

---

## แบบฝึกหัด (10 ข้อ)

**ข้อ 1:** อ่าน plan ต่อไปนี้และอธิบายว่าปัญหาคืออะไร:
```
Seq Scan on orders (cost=0.00..45000.00 rows=2 width=150)
  Filter: (status = 'cancelled')
  Rows Removed by Filter: 4999998
```

**เฉลยข้อ 1:**
- Seq Scan อ่าน 5,000,000 แถว ได้แค่ 2 แถว
- Selectivity = 2/5,000,000 = 0.00004% (ต่ำมาก)
- ควรมี index บน `status` หรือ partial index
- แก้: `CREATE INDEX idx_orders_cancelled ON orders(created_at) WHERE status = 'cancelled';`

---

**ข้อ 2:** Hash Join vs Nested Loop - ควรใช้อันไหนเมื่อไหร่?

**เฉลยข้อ 2:**
- **Hash Join**: ดีเมื่อ join 2 ตารางขนาดปานกลางถึงใหญ่, ไม่มี index บน join key
- **Nested Loop**: ดีเมื่อ outer table เล็กมาก (< 100 rows) หรือ inner table มี index บน join key
- PostgreSQL planner เลือกอัตโนมัติตาม statistics และ cost estimates

---

**ข้อ 3:** "Rows Removed by Filter: 95000" ใน Seq Scan หมายความว่าอะไร และควรทำอะไร?

**เฉลยข้อ 3:**
หมายความว่า Seq Scan อ่าน 95,000 แถว แต่ WHERE filter เอาออกหมดเหลือน้อยมาก แสดงว่า selectivity สูงมาก ควร:
1. สร้าง index บน column ที่ใช้ใน WHERE
2. ถ้า column นั้นมี low cardinality ให้พิจารณา partial index
3. ถ้าดึงข้อมูลจำนวนมาก (> 20%) Full Scan อาจดีกว่า

---

**ข้อ 4:** "Heap Fetches: 0" ใน Index Only Scan ดีอย่างไร?

**เฉลยข้อ 4:**
Heap Fetches: 0 หมายความว่า query ดึงข้อมูลทั้งหมดจาก index โดยตรง ไม่ต้อง fetch rows จาก heap table เลย ช่วยลด I/O อย่างมาก เพราะ:
- Index มักเล็กกว่าตาราง → fit in memory ได้ดีกว่า
- Random I/O บน heap table หายไป → ลด disk reads

---

**ข้อ 5:** Plan นี้บอกอะไร: `Sort Method: external merge Disk: 45678kB`

**เฉลยข้อ 5:**
บอกว่า Sort operation ไม่สามารถทำใน memory ได้ ต้องใช้ disk (45MB) ซึ่งช้ากว่า in-memory sort มาก แก้ไขโดย:
1. `SET work_mem = '64MB';` เพิ่ม memory สำหรับ sort
2. สร้าง index บน ORDER BY column เพื่อใช้ Index Scan แทน Sort
3. ลดจำนวนแถวก่อน sort ด้วย WHERE/LIMIT

---

**ข้อ 6:** รัน EXPLAIN ANALYZE บน query แล้วพบว่า estimated rows = 1 แต่ actual rows = 10,000 สาเหตุคืออะไร?

**เฉลยข้อ 6:**
สาเหตุที่เป็นไปได้:
1. Statistics ล้าสมัย: ANALYZE ไม่ได้รันมานาน
2. Data distribution skewed: column มีค่า outliers ที่ statistics ไม่ครอบคลุม
3. Statistics target ต่ำเกินไป: default = 100 samples/bucket
แก้: `ANALYZE table_name;` และ/หรือ `ALTER COLUMN col SET STATISTICS 500;`

---

**ข้อ 7:** MySQL EXPLAIN แสดง `type=ALL` หมายความว่าอะไร? แก้อย่างไร?

**เฉลยข้อ 7:**
`type=ALL` หมายถึง Full Table Scan ไม่มี index ที่ใช้ได้ ประสิทธิภาพแย่ที่สุด แก้โดย:
1. ตรวจดู `possible_keys` และ `key` column
2. ถ้า `possible_keys` ว่าง → สร้าง index บน WHERE columns
3. ถ้า `possible_keys` มีแต่ `key` ว่าง → planner ไม่เลือกใช้ index (อาจเพราะ selectivity ต่ำ)
4. ตรวจสอบ column ใน WHERE ว่ามี function ครอบหรือไม่

---

**ข้อ 8:** เขียน EXPLAIN ANALYZE สำหรับ query ที่มี GROUP BY และ ORDER BY เพื่อตรวจสอบว่ามีปัญหา performance หรือไม่

**เฉลยข้อ 8:**
```sql
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT 
    department_id,
    COUNT(*) as count,
    AVG(salary) as avg_salary
FROM employees
WHERE hire_date >= '2020-01-01'
GROUP BY department_id
ORDER BY avg_salary DESC;

-- สิ่งที่ต้องดู:
-- 1. Seq Scan หรือ Index Scan บน employees?
-- 2. Sort method: quicksort (memory) หรือ external merge (disk)?
-- 3. HashAggregate: batches > 1 = ใช้ disk → เพิ่ม work_mem
-- 4. Buffers: shared hit (cache) vs shared read (disk)
```

---

**ข้อ 9:** Plan แสดง "loops=1000" ใน Nested Loop inner scan หมายความว่าอะไร? ดีหรือไม่ดี?

**เฉลยข้อ 9:**
`loops=1000` หมายความว่า inner scan ถูกเรียก 1,000 ครั้ง (1 ครั้งต่อ row ใน outer table) ถ้า inner scan เป็น Index Scan ก็โอเค (แต่ละ loop เร็ว) แต่ถ้า inner scan เป็น Seq Scan หมายถึง 1,000 full scans → ช้ามาก! แก้โดยสร้าง index บน join column ของ inner table

---

**ข้อ 10:** EXPLAIN FORMAT=JSON มีประโยชน์อย่างไร เมื่อเทียบกับ FORMAT=TEXT?

**เฉลยข้อ 10:**
FORMAT=JSON มีประโยชน์สำหรับ:
1. Machine-readable: นำไปวิเคราะห์ด้วย script หรือ tool ได้
2. ข้อมูลครบกว่า: รวม buffer statistics, I/O timing
3. ใช้กับ online tools: https://explain.depesz.com, https://explain.dalibo.com
4. Nested structure: เห็น plan hierarchy ชัดกว่า

```sql
EXPLAIN (FORMAT JSON, ANALYZE, BUFFERS) SELECT ...;
-- output เป็น JSON object ที่ tools สามารถ visualize ได้
```

---

## สรุป

ใน Part 065 เราได้เรียนรู้:

1. **EXPLAIN/EXPLAIN ANALYZE** - อ่าน cost estimates และ actual execution
2. **Plan Nodes**: Seq Scan, Index Scan, Index Only Scan, Bitmap Index Scan
3. **Join Types**: Hash Join, Nested Loop, Merge Join
4. **MySQL EXPLAIN**: type column, Extra field
5. **SQL Server SET STATISTICS**: IO, TIME
6. **SQLite EXPLAIN QUERY PLAN**
7. **Plan Analysis Workflow**: bottom-up analysis
8. **15+ ตัวอย่าง** พร้อมคำอธิบายภาษาไทย
9. **Estimated vs Actual** - เมื่อต่างกันมาก → statistics issue

ใน Part 066 เราจะเรียนรู้เทคนิค Query Optimization เพื่อให้ database ใช้ index ได้อย่างมีประสิทธิภาพ
