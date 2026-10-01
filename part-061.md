# Part 061: Understanding Database Indexes

## ทำความเข้าใจ Database Indexes

---

## บทนำ: ทำไมต้องมี Index?

ลองนึกภาพว่าคุณต้องหาข้อมูลพนักงานชื่อ "สมชาย" จากตารางที่มีพนักงาน 10 ล้านคน หากไม่มี Index ระบบฐานข้อมูลจะต้องอ่านข้อมูลทุกแถวจากตาราง (Full Table Scan) ซึ่งอาจใช้เวลาเป็นนาที

แต่ถ้ามี Index ที่คอลัมน์ชื่อ ระบบจะสามารถค้นหาข้อมูลได้ในเวลาไม่กี่มิลลิวินาที

**Index คืออะไร?**

Index คือโครงสร้างข้อมูลพิเศษที่ช่วยเพิ่มความเร็วในการค้นหาข้อมูลในตาราง โดยการสร้าง "สารบัญ" ให้กับคอลัมน์ที่ต้องการค้นหาบ่อยๆ คล้ายกับสารบัญหนังสือที่ช่วยให้เราหาหน้าที่ต้องการได้เร็วขึ้น โดยไม่ต้องพลิกอ่านทุกหน้า

---

## 1. โครงสร้างข้อมูล B-tree (Balanced Tree)

Index ส่วนใหญ่ใช้โครงสร้างข้อมูลที่เรียกว่า **B-tree** (Balanced Tree) ซึ่งเป็นต้นไม้ที่มีการจัดเรียงข้อมูลให้สมดุล

### B-tree คืออะไร?

B-tree เป็น tree data structure ที่:
- ทุก node มีข้อมูลที่เรียงลำดับแล้ว
- Leaf nodes ทั้งหมดอยู่ที่ความลึกเท่ากัน (balanced)
- แต่ละ node มีลูกหลายตัวได้ (branching factor สูง)
- ค้นหา, แทรก, ลบข้อมูลใช้เวลา O(log n)

### แผนภาพ B-tree Structure

```
ตัวอย่าง B-tree Index บนคอลัมน์ employee_id:

                        [  50  |  100  ]
                       /        |        \
              [25|37]         [75|88]         [125|150]
             /  |   \        /  |   \        /    |    \
          [10] [30] [40]  [60] [80] [95]  [110] [130] [175]
           |    |    |     |    |    |      |     |     |
          Row  Row  Row   Row  Row  Row    Row   Row   Row
         Ptrs Ptrs Ptrs  Ptrs Ptrs Ptrs  Ptrs  Ptrs  Ptrs

Root Node: [50 | 100]
  - ถ้าค้นหาค่า < 50 → ไปทางซ้าย
  - ถ้าค้นหาค่า 50-100 → ไปตรงกลาง  
  - ถ้าค้นหาค่า > 100 → ไปทางขวา

Internal Nodes: [25|37], [75|88], [125|150]
  - แต่ละ node มี key values สำหรับนำทาง

Leaf Nodes: [10], [30], [40], [60], ...
  - เก็บ key values จริงๆ พร้อม pointers ไปยัง row จริงในตาราง
  - Leaf nodes เชื่อมต่อกันเป็น linked list (สำหรับ range queries)
```

### การค้นหาใน B-tree

```
ตัวอย่าง: ค้นหา employee_id = 80

ขั้นตอนที่ 1: เริ่มที่ Root Node [50 | 100]
              80 > 50 และ 80 < 100 → ไปตรงกลาง

ขั้นตอนที่ 2: เข้า Internal Node [75 | 88]
              80 > 75 และ 80 < 88 → ไปตรงกลาง

ขั้นตอนที่ 3: เข้า Leaf Node [80]
              พบ key = 80 → ดึง row pointer

ขั้นตอนที่ 4: ใช้ row pointer เข้าถึงข้อมูลจริงในตาราง

จำนวน I/O operations: 3 (ขนาดต้นไม้ = 3 ระดับ)
เทียบกับ Full Table Scan: 10,000,000 row reads
```

### ข้อดีของ B-tree

```
สมมติมีข้อมูล 1,000,000 แถว:

Full Table Scan:
- อ่านทุกแถว: 1,000,000 reads
- เวลาที่ใช้: ~1 วินาที

B-tree Index Search:
- ความสูงของต้นไม้: log₃(1,000,000) ≈ 13 levels
- จำนวน I/O: ~13 reads
- เวลาที่ใช้: ~1 มิลลิวินาที

ประสิทธิภาพดีกว่า: ~1000 เท่า!
```

---

## 2. Index Trade-offs: ความเร็ว vs พื้นที่จัดเก็บ vs Write Overhead

### ข้อดีของ Index
- **เพิ่มความเร็ว SELECT** อย่างมาก
- **ช่วย ORDER BY** ที่ใช้คอลัมน์เดียวกับ index
- **ช่วย GROUP BY** ในบางกรณี
- **ช่วย JOIN** conditions
- **Enforce UNIQUE** constraints

### ข้อเสียของ Index

```sql
-- ทุกครั้งที่ INSERT ข้อมูลใหม่:
INSERT INTO employees (emp_id, name, dept_id, salary) 
VALUES (10001, 'สมชาย', 5, 50000);

-- ระบบต้องทำ:
-- 1. เขียนข้อมูลลงตาราง (1 write)
-- 2. อัพเดต index บน emp_id (1 write)
-- 3. อัพเดต index บน name (1 write)  
-- 4. อัพเดต index บน dept_id (1 write)
-- รวม: 4 writes แทนที่จะเป็น 1!
```

### ตาราง Trade-offs

```
┌─────────────────┬───────────────────┬───────────────────┐
│ การกระทำ        │ ไม่มี Index       │ มี Index          │
├─────────────────┼───────────────────┼───────────────────┤
│ SELECT (ค้นหา)  │ ช้า (Full Scan)   │ เร็ว (Index Scan) │
│ INSERT          │ เร็ว              │ ช้าลงเล็กน้อย     │
│ UPDATE          │ ปานกลาง           │ ช้าลง (2 updates) │
│ DELETE          │ ปานกลาง           │ ช้าลงเล็กน้อย     │
│ พื้นที่จัดเก็บ  │ ต่ำ               │ สูงขึ้น           │
│ VACUUM/ANALYZE  │ เร็ว              │ ช้าลง             │
└─────────────────┴───────────────────┴───────────────────┘
```

---

## 3. เมื่อใด Index ช่วยได้ และเมื่อใดไม่ช่วย

### Index ช่วยได้เมื่อ

**3.1 ค้นหาด้วย equality condition**
```sql
-- ดี: ใช้ index ได้
SELECT * FROM employees WHERE emp_id = 1001;
SELECT * FROM orders WHERE order_date = '2024-01-15';
```

**3.2 ค้นหาด้วย range condition**
```sql
-- ดี: ใช้ index ได้ (B-tree supports range)
SELECT * FROM orders WHERE order_date BETWEEN '2024-01-01' AND '2024-12-31';
SELECT * FROM products WHERE price > 1000;
SELECT * FROM employees WHERE salary >= 50000;
```

**3.3 ค้นหาด้วย prefix LIKE**
```sql
-- ดี: ใช้ index ได้ (prefix search)
SELECT * FROM customers WHERE last_name LIKE 'สมช%';
-- ไม่ดี: ไม่ใช้ index (leading wildcard)
SELECT * FROM customers WHERE last_name LIKE '%ชาย';
```

**3.4 JOIN conditions**
```sql
-- ดี: index บน foreign key ช่วย JOIN
SELECT e.name, d.dept_name
FROM employees e
JOIN departments d ON e.dept_id = d.dept_id;
-- ควรมี index บน employees.dept_id
```

**3.5 ORDER BY**
```sql
-- ดี: index ช่วย sort
SELECT * FROM employees ORDER BY last_name;
-- ควรมี index บน last_name
```

### Index ไม่ช่วยเมื่อ

**3.6 ตาราง selectivity ต่ำ (ค่าซ้ำกันมาก)**
```sql
-- ไม่ดี: คอลัมน์มีแค่ 2 ค่า (M/F) - index ไม่คุ้ม
SELECT * FROM employees WHERE gender = 'M';
-- ถ้า 50% เป็นผู้ชาย ยังต้องอ่านครึ่งตาราง
-- Full scan อาจเร็วกว่า!
```

**3.7 ตารางเล็กมาก**
```sql
-- ไม่ดี: ตาราง 100 แถว - Full scan เร็วกว่า
SELECT * FROM small_config_table WHERE config_key = 'timeout';
-- Index overhead > benefit
```

**3.8 Query ดึงข้อมูลส่วนใหญ่ของตาราง**
```sql
-- ไม่ดี: ดึงข้อมูล 80% ของตาราง
SELECT * FROM employees WHERE hire_date > '2000-01-01';
-- Planner จะเลือก Full Scan แทน
```

**3.9 Function ครอบคอลัมน์ที่มี index**
```sql
-- ไม่ดี: index บน hire_date ถูก bypass
SELECT * FROM employees WHERE YEAR(hire_date) = 2024;
-- ควรเขียนว่า:
SELECT * FROM employees WHERE hire_date >= '2024-01-01' AND hire_date < '2025-01-01';
```

---

## 4. Clustered vs Non-Clustered Indexes

### Clustered Index

Clustered Index คือ index ที่กำหนดลำดับการเก็บข้อมูลจริงในตาราง ข้อมูลถูกเก็บในลำดับเดียวกับ index key

```
ตาราง employees ที่มี Clustered Index บน emp_id:

Disk Layout:
┌──────┬─────────┬──────────┬────────┐
│  1   │ สมชาย  │ IT       │ 50,000 │  ← แถวแรก
├──────┼─────────┼──────────┼────────┤
│  2   │ สมหญิง │ HR       │ 45,000 │
├──────┼─────────┼──────────┼────────┤
│  3   │ วิชาย  │ Finance  │ 55,000 │
├──────┼─────────┼──────────┼────────┤
│ ...  │ ...     │ ...      │ ...    │
└──────┴─────────┴──────────┴────────┘

ข้อมูลเรียงตาม emp_id บน disk จริงๆ
```

**ข้อดี Clustered Index:**
- ค้นหา range ได้เร็วมาก (ข้อมูลอยู่ติดกัน)
- ไม่ต้องมี additional lookup (ข้อมูลอยู่ใน leaf node เลย)

**ข้อจำกัด:**
- มีได้แค่ 1 clustered index ต่อตาราง
- INSERT อาจทำให้เกิด page splits

### Non-Clustered Index

Non-Clustered Index เก็บ index แยกต่างหากจากตาราง โดย leaf nodes มีเพียง pointer กลับไปยังข้อมูลจริง

```
ตาราง employees ที่มี Non-Clustered Index บน last_name:

Index Structure:                    ตารางจริง (เรียงตาม emp_id):
                                    ┌──────┬─────────┬─────────┐
Non-Clustered Index:                │  1   │ สมชาย  │ 50,000  │
┌──────────┬──────────┐             ├──────┼─────────┼─────────┤
│ กมล      │ → Row 7  │─────────→  │  2   │ สมหญิง │ 45,000  │
├──────────┼──────────┤             ├──────┼─────────┼─────────┤
│ กานดา    │ → Row 15 │             │  3   │ วิชาย  │ 55,000  │
├──────────┼──────────┤             ├──────┼─────────┼─────────┤
│ จินดา    │ → Row 3  │─────────→  │ ...  │ ...     │ ...     │
├──────────┼──────────┤             └──────┴─────────┴─────────┘
│ สมชาย   │ → Row 1  │─────────→
└──────────┴──────────┘
    Index                              Table Data
    (sorted)                         (heap/clustered)
```

**ข้อดี Non-Clustered Index:**
- มีได้หลายตัวต่อตาราง
- ไม่กระทบ physical order ของตาราง

**ข้อเสีย:**
- ต้องมี "bookmark lookup" หรือ "heap fetch" เพิ่มเติม

---

## 5. Index Selectivity และ Cardinality

### Selectivity คืออะไร?

Selectivity คือสัดส่วนของ rows ที่ query จะดึงออกมา เมื่อเทียบกับทั้งหมด

```
Selectivity = จำนวน rows ที่ตรงเงื่อนไข / จำนวน rows ทั้งหมด

ตัวอย่าง:
- ตาราง employees มี 1,000,000 แถว
- ค้นหา emp_id = 12345 → ได้ 1 แถว
  Selectivity = 1/1,000,000 = 0.000001 (ต่ำมาก = ดีมากสำหรับ index)

- ค้นหา dept_id = 5 → ได้ 50,000 แถว (5%)
  Selectivity = 50,000/1,000,000 = 0.05 (ต่ำ = ดีสำหรับ index)

- ค้นหา gender = 'M' → ได้ 500,000 แถว (50%)
  Selectivity = 500,000/1,000,000 = 0.5 (สูง = ไม่ดีสำหรับ index)
```

**กฎทั่วไป:**
- Selectivity < 15-20%: Index มักช่วยได้
- Selectivity > 20-30%: Full scan อาจเร็วกว่า (ขึ้นอยู่กับ workload)

### Cardinality คืออะไร?

Cardinality คือจำนวนค่าที่ไม่ซ้ำกัน (unique values) ในคอลัมน์

```sql
-- ตรวจสอบ cardinality
SELECT 
    column_name,
    COUNT(DISTINCT column_value) as cardinality,
    COUNT(*) as total_rows,
    ROUND(COUNT(DISTINCT column_value)::numeric / COUNT(*) * 100, 2) as selectivity_pct
FROM your_table;

-- ตัวอย่าง:
-- emp_id: cardinality = 1,000,000 (100%) → High cardinality = ดีสำหรับ index
-- dept_id: cardinality = 20 (0.002%) → Low cardinality = อาจไม่คุ้ม
-- gender: cardinality = 2 (0.0002%) → Very low cardinality = ไม่ควร index เดี่ยว
```

```sql
-- ตรวจสอบ cardinality ใน PostgreSQL
SELECT 
    attname AS column_name,
    n_distinct,
    CASE 
        WHEN n_distinct < 0 THEN ABS(n_distinct) * reltuples  -- negative = fraction of rows
        ELSE n_distinct 
    END AS estimated_distinct_values,
    reltuples AS total_rows
FROM pg_stats s
JOIN pg_class c ON c.relname = s.tablename
WHERE tablename = 'employees'
ORDER BY n_distinct DESC;
```

```sql
-- ตรวจสอบ cardinality ใน MySQL
SELECT 
    INDEX_NAME,
    COLUMN_NAME,
    CARDINALITY,
    TABLE_ROWS,
    ROUND(CARDINALITY / TABLE_ROWS * 100, 2) AS selectivity_pct
FROM information_schema.STATISTICS
WHERE TABLE_SCHEMA = 'your_db'
AND TABLE_NAME = 'employees';
```

---

## 6. Index Statistics

ระบบฐานข้อมูลเก็บ **statistics** เกี่ยวกับข้อมูลในตาราง เพื่อให้ Query Planner สามารถตัดสินใจได้ว่าควรใช้ index หรือไม่

### Statistics ใน PostgreSQL

```sql
-- ดู statistics ของตาราง
SELECT 
    schemaname,
    relname AS table_name,
    n_live_tup AS live_rows,
    n_dead_tup AS dead_rows,
    last_analyze,
    last_autoanalyze
FROM pg_stat_user_tables
WHERE relname = 'employees';

-- ดู column statistics
SELECT 
    attname AS column_name,
    n_distinct,
    null_frac,
    avg_width,
    correlation  -- 1.0 = perfectly correlated, 0 = random
FROM pg_stats
WHERE tablename = 'employees';
```

```sql
-- อัพเดต statistics ด้วยตนเอง
ANALYZE employees;
ANALYZE VERBOSE employees;  -- แสดงผลรายละเอียด

-- อัพเดต statistics ทั้ง database
ANALYZE;
```

### Statistics ใน MySQL

```sql
-- ดู table statistics
SHOW TABLE STATUS LIKE 'employees';

-- ดู index statistics  
SHOW INDEX FROM employees;

-- อัพเดต statistics
ANALYZE TABLE employees;
```

---

## 7. Auto-Created Indexes

ฐานข้อมูลจะสร้าง index อัตโนมัติในบางกรณี:

### Primary Key Index

```sql
-- PostgreSQL/MySQL: สร้าง index อัตโนมัติสำหรับ PRIMARY KEY
CREATE TABLE employees (
    emp_id INT PRIMARY KEY,  -- ← index ถูกสร้างอัตโนมัติ
    name VARCHAR(100),
    salary DECIMAL(10,2)
);
```

### Unique Constraint Index

```sql
-- PostgreSQL/MySQL: สร้าง index อัตโนมัติสำหรับ UNIQUE constraint
CREATE TABLE employees (
    emp_id INT PRIMARY KEY,
    email VARCHAR(200) UNIQUE,  -- ← index ถูกสร้างอัตโนมัติ
    name VARCHAR(100)
);
```

### Foreign Key Index (MySQL)

```sql
-- MySQL สร้าง index สำหรับ FOREIGN KEY อัตโนมัติ
-- PostgreSQL ไม่สร้างอัตโนมัติ! ต้องสร้างเอง

-- PostgreSQL: ต้องสร้าง index เองสำหรับ FK
CREATE TABLE orders (
    order_id INT PRIMARY KEY,
    customer_id INT REFERENCES customers(customer_id),  -- FK
    order_date DATE
);

-- สร้าง index เองสำหรับ FK ใน PostgreSQL
CREATE INDEX idx_orders_customer_id ON orders(customer_id);
```

---

## 8. ตัวอย่างประสิทธิภาพ Index ในทางปฏิบัติ

### ตัวอย่างที่ 1: การค้นหาพนักงาน

```sql
-- สร้างตาราง sample
CREATE TABLE employees (
    emp_id SERIAL PRIMARY KEY,
    first_name VARCHAR(50),
    last_name VARCHAR(50),
    department VARCHAR(50),
    salary DECIMAL(10,2),
    hire_date DATE,
    email VARCHAR(100)
);

-- สร้างข้อมูล sample (สมมติ 1 ล้านแถว)
-- ไม่มี index บน last_name

-- Query 1: ไม่มี index
EXPLAIN ANALYZE
SELECT * FROM employees WHERE last_name = 'สมชาย';
-- ผล: Seq Scan on employees (cost=0.00..25000.00 rows=50 width=150)
--     Execution Time: 1200 ms

-- สร้าง index
CREATE INDEX idx_employees_last_name ON employees(last_name);

-- Query 2: มี index
EXPLAIN ANALYZE
SELECT * FROM employees WHERE last_name = 'สมชาย';
-- ผล: Index Scan using idx_employees_last_name on employees
--     (cost=0.56..8.58 rows=50 width=150)
--     Execution Time: 2 ms
```

### ตัวอย่างที่ 2: Range Query

```sql
-- ไม่มี index บน hire_date
EXPLAIN ANALYZE
SELECT COUNT(*) FROM employees 
WHERE hire_date BETWEEN '2023-01-01' AND '2023-12-31';
-- ผล: Seq Scan, Execution Time: 800 ms

-- สร้าง index
CREATE INDEX idx_employees_hire_date ON employees(hire_date);

-- มี index
EXPLAIN ANALYZE
SELECT COUNT(*) FROM employees 
WHERE hire_date BETWEEN '2023-01-01' AND '2023-12-31';
-- ผล: Index Scan, Execution Time: 5 ms
```

---

## 9. ประเภทของ Index Scans

เมื่อใช้ index ระบบสามารถทำการค้นหาได้หลายรูปแบบ:

### Sequential Scan (ไม่ใช้ index)
```
Sequential Scan (Full Table Scan):
┌────┬────┬────┬────┬────┬────┬────┬────┬────┬────┐
│  1 │  2 │  3 │  4 │  5 │  6 │  7 │  8 │  9 │ 10 │
└────┴────┴────┴────┴────┴────┴────┴────┴────┴────┘
  ↑    ↑    ↑    ↑    ↑    ↑    ↑    ↑    ↑    ↑
  อ่านทุกแถว
```

### Index Scan
```
Index Scan:
Index → Row Pointer → Table Row
[B-tree lookup] → [pointer] → [heap fetch]

เร็วสำหรับ: ค้นหาน้อยแถว, high selectivity
```

### Index Only Scan (Covering Index)
```
Index Only Scan:
Index → ข้อมูลอยู่ใน index เลย (ไม่ต้อง fetch table)

เร็วที่สุด! ไม่ต้อง I/O เพิ่มเติม
```

### Bitmap Index Scan
```
Bitmap Index Scan:
1. สร้าง bitmap จาก index (ว่า row ไหนตรงเงื่อนไข)
2. ใช้ bitmap fetch rows จาก table ครั้งเดียว

เร็วสำหรับ: ค้นหาหลายแถว, multiple indexes
```

---

## 10. Best Practices สำหรับ Index

### ควรสร้าง Index เมื่อ:
1. **คอลัมน์ที่ใช้ใน WHERE บ่อย** และมี selectivity สูง
2. **คอลัมน์ที่ใช้ JOIN** (โดยเฉพาะ foreign keys)
3. **คอลัมน์ที่ใช้ ORDER BY หรือ GROUP BY**
4. **คอลัมน์ที่ต้องการ UNIQUE** values

### ไม่ควรสร้าง Index เมื่อ:
1. **ตารางเล็กมาก** (< 1000 แถว)
2. **Cardinality ต่ำมาก** (เช่น boolean, status code)
3. **ตารางมี write-heavy workload** มาก
4. **Query ดึงข้อมูลส่วนใหญ่ของตาราง**

### หลักการทั่วไป:
```sql
-- ตรวจสอบ indexes ที่ไม่ถูกใช้ใน PostgreSQL
SELECT 
    schemaname,
    tablename,
    indexname,
    idx_scan as times_used,
    pg_size_pretty(pg_relation_size(indexrelid)) as index_size
FROM pg_stat_user_indexes
WHERE idx_scan = 0  -- ไม่เคยถูกใช้
AND indexrelname NOT LIKE 'pk_%'  -- ไม่ใช่ PK
ORDER BY pg_relation_size(indexrelid) DESC;
```

```sql
-- ตรวจสอบ indexes ที่ซ้อนทับกัน (redundant indexes) ใน PostgreSQL  
SELECT 
    a.indexrelid::regclass AS index1,
    b.indexrelid::regclass AS index2,
    a.indrelid::regclass AS table_name
FROM pg_index a
JOIN pg_index b ON a.indrelid = b.indrelid
    AND a.indexrelid != b.indexrelid
WHERE (a.indkey::int[])[0:1] = (b.indkey::int[])[0:1]  -- same first column
ORDER BY table_name;
```

---

## 11. การตรวจสอบ Index ที่มีอยู่

```sql
-- PostgreSQL: ดู indexes ทั้งหมด
SELECT 
    t.relname AS table_name,
    i.relname AS index_name,
    ix.indisprimary AS is_primary,
    ix.indisunique AS is_unique,
    array_agg(a.attname ORDER BY array_position(ix.indkey, a.attnum)) AS columns
FROM pg_class t
JOIN pg_index ix ON t.oid = ix.indrelid
JOIN pg_class i ON i.oid = ix.indexrelid
JOIN pg_attribute a ON a.attrelid = t.oid AND a.attnum = ANY(ix.indkey)
WHERE t.relkind = 'r'
AND t.relname = 'employees'
GROUP BY t.relname, i.relname, ix.indisprimary, ix.indisunique
ORDER BY t.relname, i.relname;
```

```sql
-- MySQL: ดู indexes
SHOW INDEXES FROM employees;

-- หรือ
SELECT 
    TABLE_NAME,
    INDEX_NAME,
    COLUMN_NAME,
    NON_UNIQUE,
    SEQ_IN_INDEX
FROM information_schema.STATISTICS  
WHERE TABLE_SCHEMA = DATABASE()
AND TABLE_NAME = 'employees'
ORDER BY INDEX_NAME, SEQ_IN_INDEX;
```

```sql
-- SQLite: ดู indexes
SELECT name, tbl_name, sql 
FROM sqlite_master 
WHERE type = 'index';
```

---

## แบบฝึกหัด (10 ข้อ)

**ข้อ 1:** ตาราง `orders` มี 5 ล้านแถว มี index บน `customer_id` และ `order_date` จงอธิบายว่า query ต่อไปนี้จะใช้ index ไหน:
```sql
SELECT * FROM orders WHERE customer_id = 1001 AND order_date > '2024-01-01';
```

**เฉลยข้อ 1:**
Query นี้มีเงื่อนไข 2 อย่าง:
- `customer_id = 1001` - ค่าเดียว (equality) selectivity สูง → ใช้ index บน customer_id
- `order_date > '2024-01-01'` - range query → ใช้ index บน order_date
Planner จะเลือก index ที่ selectivity สูงกว่า หรืออาจใช้ Bitmap Index Scan รวมทั้งสอง index

---

**ข้อ 2:** คอลัมน์ `status` ในตาราง `orders` มีค่าเป็น 'pending', 'completed', 'cancelled' โดยมี `completed` 70%, `pending` 25%, `cancelled` 5% ควรสร้าง index บน `status` หรือไม่? เพราะอะไร?

**เฉลยข้อ 2:**
- Index ทั่วไปบน `status` ไม่คุ้มค่า เพราะ selectivity ต่ำ (ค่าซ้ำกันมาก)
- ยกเว้น query ค้นหา `status = 'cancelled'` ซึ่งมีแค่ 5% - อาจคุ้มค่าถ้าค้นบ่อย
- แนะนำ: ใช้ Partial Index สำหรับ 'cancelled' เท่านั้น:
```sql
CREATE INDEX idx_orders_cancelled ON orders(status) WHERE status = 'cancelled';
```

---

**ข้อ 3:** B-tree index สูง 4 ระดับ (4 levels) สามารถเก็บข้อมูลได้กี่ rows ถ้า branching factor = 100?

**เฉลยข้อ 3:**
จำนวน leaf nodes = 100^3 = 1,000,000 (level 1→2→3→leaf)
ถ้าแต่ละ leaf node เก็บ 100 entries = 100,000,000 rows
B-tree สูง 4 ระดับรองรับข้อมูลได้ถึง 100 ล้านแถว!

---

**ข้อ 4:** เขียน query เพื่อตรวจสอบว่า index ใดบนตาราง `products` ใน PostgreSQL ไม่เคยถูกใช้งานเลย

**เฉลยข้อ 4:**
```sql
SELECT 
    indexrelname AS index_name,
    idx_scan AS times_used,
    pg_size_pretty(pg_relation_size(indexrelid)) AS index_size
FROM pg_stat_user_indexes
WHERE relname = 'products'
AND idx_scan = 0
ORDER BY pg_relation_size(indexrelid) DESC;
```

---

**ข้อ 5:** ตาราง `users` มี 10,000 แถว มี index บน `email` คุณคิดว่าการค้นหา `WHERE email LIKE '%@gmail.com'` จะใช้ index หรือไม่? เพราะอะไร?

**เฉลยข้อ 5:**
ไม่ใช้ index เพราะ leading wildcard `%` ทำให้ระบบไม่สามารถค้นหาจาก prefix ใน B-tree ได้ ต้องทำ full scan แทน นอกจากนี้ตาราง 10,000 แถวก็เล็กพอที่ full scan จะเร็วพอแล้ว

---

**ข้อ 6:** อธิบายความแตกต่างระหว่าง Clustered Index และ Non-Clustered Index พร้อมตัวอย่าง use case

**เฉลยข้อ 6:**
- **Clustered Index**: ข้อมูลในตารางเรียงตาม index key จริงๆ บน disk มีได้แค่ 1 ตัว ดีสำหรับ range queries บน primary key เช่น `ORDER BY emp_id`
- **Non-Clustered Index**: โครงสร้างแยกต่างหาก มี pointer กลับไปยังตาราง มีได้หลายตัว ดีสำหรับ lookup บน secondary columns เช่น `WHERE email = '...'`

---

**ข้อ 7:** เขียน SQL query เพื่อแสดง cardinality ของทุกคอลัมน์ในตาราง `employees` ใน MySQL

**เฉลยข้อ 7:**
```sql
SELECT 
    COLUMN_NAME,
    COUNT(DISTINCT column_value_expression) AS cardinality,
    COUNT(*) AS total_rows
FROM employees;

-- หรือใช้ INFORMATION_SCHEMA สำหรับ indexed columns:
SELECT 
    COLUMN_NAME,
    CARDINALITY,
    TABLE_ROWS
FROM information_schema.STATISTICS
WHERE TABLE_SCHEMA = DATABASE()
AND TABLE_NAME = 'employees'
ORDER BY CARDINALITY DESC;
```

---

**ข้อ 8:** ตาราง `transactions` มี 100 ล้านแถว มี columns: `trans_id`, `account_id`, `trans_date`, `amount`, `status` คุณจะสร้าง index อะไรบ้าง? จงอธิบายเหตุผล

**เฉลยข้อ 8:**
```sql
-- 1. Primary Key (auto-created)
-- trans_id PRIMARY KEY → index อัตโนมัติ

-- 2. Index สำหรับค้นหาตาม account
CREATE INDEX idx_transactions_account_id ON transactions(account_id);
-- เหตุผล: ค้นหา transactions ของ account ใดๆ บ่อยมาก

-- 3. Composite index สำหรับ account + date range
CREATE INDEX idx_transactions_account_date ON transactions(account_id, trans_date);
-- เหตุผล: query "ดู transactions ของ account X ในช่วงวันที่ Y-Z"

-- 4. Partial index สำหรับ pending transactions
CREATE INDEX idx_transactions_pending ON transactions(account_id, trans_date) 
WHERE status = 'pending';
-- เหตุผล: query pending transactions บ่อย แต่มีน้อย
```

---

**ข้อ 9:** ทำไม PostgreSQL ถึงไม่สร้าง index บน Foreign Key อัตโนมัติ ต่างจาก MySQL อย่างไร?

**เฉลยข้อ 9:**
PostgreSQL ไม่สร้าง index บน FK อัตโนมัติเพราะไม่ใช่ทุก FK จะถูก query บ่อย การสร้าง index มี overhead ทั้ง storage และ write performance ดังนั้น PostgreSQL ให้ DBA ตัดสินใจเอง MySQL สร้าง FK index อัตโนมัติเพื่อ enforce referential integrity โดยใช้ index ตรวจสอบ parent row ก่อน delete/update

---

**ข้อ 10:** จงเขียน query เพื่อหา tables ที่ไม่มี index เลย (นอกจาก PK) ใน PostgreSQL schema ชื่อ 'public'

**เฉลยข้อ 10:**
```sql
SELECT 
    t.table_name,
    t.table_rows
FROM information_schema.tables t
LEFT JOIN (
    SELECT 
        tablename,
        COUNT(*) as index_count
    FROM pg_indexes
    WHERE schemaname = 'public'
    AND indexname NOT LIKE '%_pkey'  -- ไม่นับ PK
    GROUP BY tablename
) i ON t.table_name = i.tablename
WHERE t.table_schema = 'public'
AND t.table_type = 'BASE TABLE'
AND i.index_count IS NULL  -- ไม่มี index นอกจาก PK
ORDER BY t.table_name;
```

---

## สรุป

ใน Part 061 นี้เราได้เรียนรู้:

1. **Index คืออะไร** - โครงสร้างข้อมูลที่ช่วยเพิ่มความเร็วการค้นหา
2. **B-tree Structure** - โครงสร้างต้นไม้สมดุลที่ใช้เป็น index หลัก
3. **Trade-offs** - Index เพิ่มความเร็ว SELECT แต่ลดความเร็ว INSERT/UPDATE/DELETE
4. **เมื่อใด index ช่วย** - Selectivity สูง, WHERE/JOIN/ORDER BY บ่อย
5. **Clustered vs Non-Clustered** - ความแตกต่างและ use cases
6. **Selectivity และ Cardinality** - การวัดประสิทธิภาพ index
7. **Statistics** - ข้อมูลที่ช่วย planner ตัดสินใจ
8. **Auto-created indexes** - PK, UNIQUE constraints

ใน Part 062 เราจะเรียนรู้วิธีสร้างและจัดการ index อย่างละเอียด
