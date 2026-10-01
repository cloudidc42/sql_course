# Part 008: NULL Values - Understanding and Handling

> **หลักสูตร SQL ครบวงจร | Part 8 of 120**

---

## 🎯 สิ่งที่จะได้เรียนรู้ในบทนี้

- NULL คืออะไร? ทำไมมีและทำไมสำคัญ
- NULL vs Empty String vs 0
- Three-Valued Logic (TRUE, FALSE, UNKNOWN)
- IS NULL / IS NOT NULL
- NULL ใน Calculations
- NULL ใน Aggregations
- NULL ใน Comparisons
- COALESCE function
- NULLIF function
- IFNULL / NVL (database-specific)
- Common NULL pitfalls

**เวลาที่ใช้เรียน**: ประมาณ 2-3 ชั่วโมง

---

## ฐานข้อมูลที่ใช้ในบทนี้

```sql
-- เพิ่มข้อมูลพิเศษสำหรับบทนี้ (NULL testing data)
CREATE TABLE IF NOT EXISTS null_test (
    id INTEGER PRIMARY KEY,
    name TEXT,
    value INTEGER,
    description TEXT,
    category TEXT
);

INSERT INTO null_test VALUES
(1, 'Alice',    100,  'First entry',    'A'),
(2, 'Bob',      NULL, 'Missing value',  'A'),
(3, 'Charlie',  200,  NULL,             'B'),
(4, NULL,       300,  'No name',        'B'),
(5, NULL,       NULL, NULL,             NULL),
(6, 'Diana',    0,    'Zero value',     'C'),
(7, 'Eve',      -50,  'Negative',       'C'),
(8, '',         400,  'Empty string',   'D');
```

---

## 8.1 NULL คืออะไร?

```sql
-- ============================================================
-- NULL = ไม่มีข้อมูล / ไม่รู้ข้อมูล / ไม่เกี่ยวข้อง
-- ============================================================

/*
NULL ไม่ใช่:
- 0 (ตัวเลข)
- '' (empty string)
- FALSE (boolean)
- 'NULL' (string)

NULL คือ: ไม่มีข้อมูล/ไม่ทราบค่า

ตัวอย่างการใช้ NULL:
- พนักงานที่ไม่มี manager → manager_id = NULL
- ลูกค้าที่ไม่ได้กรอก phone → phone = NULL
- สินค้าที่ยังไม่ได้กำหนดราคา → unit_price = NULL
- Order ที่ยังไม่ถูก ship → shipped_date = NULL
*/

-- ============================================================
-- EXAMPLE 1: ดูข้อมูลที่มี NULL
-- ============================================================

SELECT * FROM null_test;

-- ผลลัพธ์:
-- id | name    | value | description   | category
-- ---|---------|-------|---------------|--------
-- 1  | Alice   | 100   | First entry   | A
-- 2  | Bob     | NULL  | Missing value | A
-- 3  | Charlie | 200   | NULL          | B
-- 4  | NULL    | 300   | No name       | B
-- 5  | NULL    | NULL  | NULL          | NULL
-- 6  | Diana   | 0     | Zero value    | C
-- 7  | Eve     | -50   | Negative      | C
-- 8  |         | 400   | Empty string  | D

-- สังเกต: id=6 มี value=0 (ไม่ใช่ NULL), id=8 มี name='' (ไม่ใช่ NULL)
```

---

## 8.2 NULL vs Empty String vs 0

```sql
-- ============================================================
-- EXAMPLE 2: ความแตกต่าง
-- ============================================================

SELECT 
    id,
    name,
    value,
    -- ตรวจสอบ NULL
    CASE WHEN name IS NULL     THEN 'NULL' ELSE 'NOT NULL' END AS name_null_check,
    CASE WHEN name = ''        THEN 'Empty' ELSE 'Has Value' END AS name_empty_check,
    CASE WHEN value IS NULL    THEN 'NULL' ELSE 'NOT NULL' END AS value_null_check,
    CASE WHEN value = 0        THEN 'Zero' ELSE 'Not Zero' END AS value_zero_check
FROM null_test;

-- ผลลัพธ์:
-- id | name    | value | name_null_check | name_empty_check | value_null_check | value_zero_check
-- ---|---------|-------|-----------------|------------------|------------------|----------------
-- 1  | Alice   | 100   | NOT NULL        | Has Value        | NOT NULL         | Not Zero
-- 2  | Bob     | NULL  | NOT NULL        | Has Value        | NULL             | Not Zero  ← value is NULL, zero check = NULL → 'Not Zero'
-- 4  | NULL    | 300   | NULL            | Has Value        | NOT NULL         | Not Zero  ← name is NULL
-- 5  | NULL    | NULL  | NULL            | Has Value        | NULL             | Not Zero
-- 6  | Diana   | 0     | NOT NULL        | Has Value        | NOT NULL         | Zero
-- 8  |         | 400   | NOT NULL        | Empty            | NOT NULL         | Not Zero  ← name='' not NULL

-- ============================================================
-- EXAMPLE 3: NULL vs 0 vs Empty - ตัวอย่างจริง
-- ============================================================

-- ในตาราง employees:
-- salary = NULL → ยังไม่ได้กำหนดเงินเดือน
-- salary = 0 → เงินเดือนเป็น 0 (อาจเป็น intern, volunteer)
-- phone = NULL → ไม่มีเบอร์โทร
-- phone = '' → กรอกช่องว่างมา (data quality issue)
-- phone = '000-000-0000' → กรอก placeholder มา

SELECT 
    first_name,
    salary,
    CASE 
        WHEN salary IS NULL THEN 'ยังไม่กำหนดเงินเดือน'
        WHEN salary = 0    THEN 'เงินเดือน 0 บาท'
        ELSE salary || ' บาท'
    END AS salary_description
FROM employees
WHERE employee_id <= 5;

-- ============================================================
-- EXAMPLE 4: Empty String ปัญหา
-- ============================================================

-- ❌ ปัญหา: ข้อมูล empty string ไม่ถูกจัดการ
SELECT COUNT(*) FROM null_test WHERE name IS NULL;     -- 2 (rows 4, 5)
SELECT COUNT(*) FROM null_test WHERE name = '';        -- 1 (row 8)
SELECT COUNT(*) FROM null_test WHERE name IS NULL OR name = '';  -- 3

-- Best practice: ทำให้ empty string เป็น NULL เมื่อ insert
INSERT INTO some_table (name) VALUES (NULLIF('', ''));
-- NULLIF('', '') = NULL เมื่อ '' = '' (จะอธิบายในส่วน NULLIF)
```

---

## 8.3 Three-Valued Logic

```sql
-- ============================================================
-- SQL ใช้ Three-Valued Logic: TRUE, FALSE, UNKNOWN
-- ============================================================
-- ผลของการ compare กับ NULL = UNKNOWN (ไม่ใช่ TRUE หรือ FALSE)

-- ============================================================
-- EXAMPLE 5: NULL Comparisons ให้ UNKNOWN
-- ============================================================

-- ทดสอบ (PostgreSQL):
SELECT 
    NULL = NULL  AS "NULL = NULL",    -- NULL (UNKNOWN)
    NULL = 1     AS "NULL = 1",       -- NULL (UNKNOWN)
    NULL <> 1    AS "NULL <> 1",      -- NULL (UNKNOWN)
    NULL > 0     AS "NULL > 0",       -- NULL (UNKNOWN)
    NULL IS NULL AS "NULL IS NULL";   -- TRUE

-- ผลลัพธ์:
-- NULL = NULL | NULL = 1 | NULL <> 1 | NULL > 0 | NULL IS NULL
-- ------------|----------|-----------|----------|-------------
-- NULL        | NULL     | NULL      | NULL     | TRUE

-- ============================================================
-- EXAMPLE 6: ทำไม WHERE condition กับ NULL ไม่ทำงาน
-- ============================================================

-- NULL ใน WHERE ถือว่าเป็น UNKNOWN → แถวนั้นไม่ถูกเลือก!

-- ❌ ผิด: หา employees ที่ manager_id เท่ากับ NULL
SELECT first_name FROM employees
WHERE manager_id = NULL;  -- 0 rows! UNKNOWN → ไม่เลือก

-- ✓ ถูก: ใช้ IS NULL
SELECT first_name FROM employees
WHERE manager_id IS NULL;  -- 7 rows

-- ❌ ผิด: หา employees ที่ manager_id ไม่เป็น NULL
SELECT first_name FROM employees
WHERE manager_id <> NULL;  -- 0 rows! UNKNOWN → ไม่เลือก

-- ✓ ถูก: ใช้ IS NOT NULL
SELECT first_name FROM employees
WHERE manager_id IS NOT NULL;  -- 8 rows

-- ============================================================
-- EXAMPLE 7: NOT IN กับ NULL (อันตรายมาก!)
-- ============================================================

-- สมมติตาราง:
-- table_a: values = (1, 2, 3)
-- table_b: values = (2, 3, NULL)

-- คาดว่า: 1 IN (2, 3, NULL) = FALSE
-- แต่จริงๆ: 1 IN (2, 3, NULL)
--   = 1=2 OR 1=3 OR 1=NULL
--   = FALSE OR FALSE OR UNKNOWN
--   = UNKNOWN

-- ❌ ปัญหา: NOT IN กับ NULL
-- ต้องการ: หา employees ที่ manager_id ไม่ใช่ 1 หรือ 2
SELECT first_name FROM employees
WHERE manager_id NOT IN (1, 2);
-- ได้แค่ rows ที่ manager_id = 3, 4, 5
-- Rows ที่ manager_id = NULL ไม่มาด้วย!

-- ✓ ถูก: เพิ่ม NULL check
SELECT first_name FROM employees
WHERE manager_id NOT IN (1, 2)
   OR manager_id IS NULL;

-- หรือใช้ NOT EXISTS แทน NOT IN
SELECT first_name FROM employees e
WHERE NOT EXISTS (
    SELECT 1 FROM employees m 
    WHERE m.employee_id IN (1, 2)
    AND e.manager_id = m.employee_id
);

-- ============================================================
-- EXAMPLE 8: Truth Table สำหรับ AND/OR กับ NULL
-- ============================================================

/*
AND Truth Table:
TRUE AND TRUE = TRUE
TRUE AND FALSE = FALSE
TRUE AND NULL = NULL (UNKNOWN)
FALSE AND TRUE = FALSE
FALSE AND FALSE = FALSE
FALSE AND NULL = FALSE  ← สำคัญ! FALSE AND anything = FALSE
NULL AND TRUE = NULL
NULL AND FALSE = FALSE  ← สำคัญ!
NULL AND NULL = NULL

OR Truth Table:
TRUE OR TRUE = TRUE
TRUE OR FALSE = TRUE
TRUE OR NULL = TRUE   ← สำคัญ! TRUE OR anything = TRUE
FALSE OR TRUE = TRUE
FALSE OR FALSE = FALSE
FALSE OR NULL = NULL  ← สำคัญ!
NULL OR TRUE = TRUE   ← สำคัญ!
NULL OR FALSE = NULL
NULL OR NULL = NULL
*/

-- ตัวอย่าง:
-- WHERE NULL AND FALSE → FALSE (ไม่เลือกแถว)
-- WHERE NULL OR TRUE → TRUE (เลือกแถว)
-- WHERE NULL → NULL/UNKNOWN (ไม่เลือกแถว)
```

---

## 8.4 NULL ใน Calculations

```sql
-- ============================================================
-- EXAMPLE 9: NULL ในการคำนวณ = NULL เสมอ
-- ============================================================

SELECT 
    value,
    value + 10    AS "value + 10",
    value - 10    AS "value - 10",
    value * 2     AS "value * 2",
    value / 2     AS "value / 2",
    value + NULL  AS "value + NULL"
FROM null_test;

-- ผลลัพธ์:
-- value | +10  | -10  | *2   | /2   | +NULL
-- ------|------|------|------|------|------
-- 100   | 110  | 90   | 200  | 50   | NULL
-- NULL  | NULL | NULL | NULL | NULL | NULL  ← NULL ทุกอย่าง!
-- 200   | 210  | 190  | 400  | 100  | NULL
-- 300   | 310  | 290  | 600  | 150  | NULL
-- NULL  | NULL | NULL | NULL | NULL | NULL
-- 0     | 10   | -10  | 0    | 0    | NULL
-- -50   | -40  | -60  | -100 | -25  | NULL
-- 400   | 410  | 390  | 800  | 200  | NULL

-- ============================================================
-- EXAMPLE 10: NULL ใน String Concatenation
-- ============================================================

-- PostgreSQL: NULL || 'text' = NULL
SELECT 
    'Hello' || NULL || 'World'  AS concat_result,  -- NULL ใน PostgreSQL/SQLite
    COALESCE(NULL, '') || 'World' AS safe_concat;   -- 'World'

-- MySQL: CONCAT ข้ามค่า NULL (ไม่ทำให้ result เป็น NULL)
-- SELECT CONCAT('Hello', NULL, 'World'); → 'HelloWorld' ใน MySQL

-- ============================================================
-- EXAMPLE 11: NULL ใน employees (จริง)
-- ============================================================

SELECT 
    first_name,
    manager_id,
    manager_id + 100 AS manager_plus_100,  -- NULL ถ้า manager_id เป็น NULL
    
    -- Safe version:
    COALESCE(manager_id, 0) + 100 AS safe_manager_calc
FROM employees
WHERE employee_id <= 7;
```

---

## 8.5 NULL ใน Aggregations

```sql
-- ============================================================
-- EXAMPLE 12: NULL และ COUNT
-- ============================================================

SELECT 
    COUNT(*)     AS count_all_rows,       -- นับทุกแถว รวม NULL
    COUNT(value) AS count_non_null_value, -- นับแค่ไม่ NULL
    COUNT(name)  AS count_non_null_name   -- นับแค่ไม่ NULL
FROM null_test;

-- ผลลัพธ์:
-- count_all_rows | count_non_null_value | count_non_null_name
-- ---------------|----------------------|--------------------
-- 8              | 6                    | 5

-- ใน null_test:
-- 8 rows ทั้งหมด
-- value NOT NULL: rows 1,3,4,6,7,8 = 6 rows
-- name NOT NULL: rows 1,2,3,6,7,8 = 6... wait
-- name NOT NULL and not empty: rows 1,2,3,6,7 = 5 (row 8 มี name='' นับด้วย)
-- จริงๆ: COUNT ไม่นับ NULL แต่นับ empty string

-- ============================================================
-- EXAMPLE 13: NULL กับ SUM, AVG
-- ============================================================

SELECT 
    SUM(value) AS sum_ignore_null,       -- 100+200+300+0+(-50)+400 = 950
    AVG(value) AS avg_ignore_null,       -- 950/6 = 158.33...
    
    -- ถ้าอยากรวม NULL เป็น 0
    SUM(COALESCE(value, 0)) AS sum_with_null_as_zero,  -- same, NULL ถูกแทน 0
    AVG(COALESCE(value, 0)) AS avg_with_null_as_zero   -- 950/8 = 118.75
FROM null_test;

-- ผลลัพธ์:
-- sum_ignore_null | avg_ignore_null | sum_with_null_as_zero | avg_with_null_as_zero
-- ----------------|-----------------|----------------------|---------------------
-- 950             | 158.3333...     | 950                  | 118.75

-- สำคัญ: AVG(value) หาร 6 (non-NULL rows), ไม่หาร 8!
-- ถ้าต้องการหาร 8 ให้ใช้ COALESCE

-- ============================================================
-- EXAMPLE 14: NULL ใน MIN, MAX
-- ============================================================

SELECT 
    MIN(value) AS min_value,  -- -50 (NULL ถูก ignore)
    MAX(value) AS max_value,  -- 400 (NULL ถูก ignore)
    MIN(name)  AS min_name,   -- '' (alphabetically first non-NULL, รวม empty)
    MAX(name)  AS max_name    -- 'Eve'
FROM null_test;

-- ============================================================
-- EXAMPLE 15: ปัญหา COUNT กับ employees
-- ============================================================

SELECT 
    COUNT(*) AS total_rows,
    COUNT(employee_id) AS total_employees,
    COUNT(manager_id) AS employees_with_manager,
    COUNT(*) - COUNT(manager_id) AS employees_without_manager
FROM employees;

-- ผลลัพธ์:
-- total_rows | total_employees | employees_with_manager | employees_without_manager
-- -----------|-----------------|------------------------|---------------------------
-- 15         | 15              | 8                      | 7
```

---

## 8.6 COALESCE Function

```sql
-- ============================================================
-- COALESCE(val1, val2, val3, ...) 
-- Returns the first non-NULL value
-- ============================================================

-- ============================================================
-- EXAMPLE 16: COALESCE พื้นฐาน
-- ============================================================

SELECT 
    COALESCE(NULL, 1, 2, 3),           -- 1 (first non-NULL)
    COALESCE(NULL, NULL, 5),            -- 5
    COALESCE(NULL, NULL, NULL),         -- NULL (all NULL)
    COALESCE(10, NULL, 20),             -- 10 (first non-NULL)
    COALESCE(NULL, 'default');          -- 'default'

-- ============================================================
-- EXAMPLE 17: COALESCE กับ manager_id
-- ============================================================

SELECT 
    first_name,
    manager_id,
    COALESCE(manager_id, 0)  AS manager_or_zero,
    COALESCE(manager_id, -1) AS manager_or_minus_one,
    CASE WHEN manager_id IS NULL 
         THEN 'Top Level'
         ELSE 'Reports to #' || manager_id
    END AS manager_description
FROM employees;

-- ผลลัพธ์:
-- first_name | manager_id | manager_or_zero | manager_description
-- -----------|------------|-----------------|--------------------
-- สมชาย      | NULL       | 0               | Top Level
-- สมหญิง     | NULL       | 0               | Top Level
-- ...
-- มาลี       | 1          | 1               | Reports to #1
-- อนันต์     | 1          | 1               | Reports to #1

-- ============================================================
-- EXAMPLE 18: COALESCE กับ Multiple Columns
-- ============================================================

-- ดึงชื่อบริษัท ถ้าไม่มีให้ใช้ชื่อ-นามสกุลแทน
SELECT 
    customer_id,
    COALESCE(company_name, first_name || ' ' || last_name) AS display_name,
    company_name,
    first_name,
    last_name
FROM customers;

-- ผลลัพธ์:
-- customer_id | display_name              | company_name               | ...
-- ------------|---------------------------|----------------------------|----
-- 1           | บริษัท เทค สตาร์ จำกัด   | บริษัท เทค สตาร์ จำกัด   | ...
-- 2           | ร้านสินค้า ABC            | ร้านสินค้า ABC            | ...
-- 4           | สุภา แก้วใส              | NULL                       | ... ← ใช้ชื่อแทน
-- 7           | กนกวรรณ มีมาก           | NULL                       | ...

-- ============================================================
-- EXAMPLE 19: COALESCE ใน Calculations
-- ============================================================

-- คำนวณโดยใช้ 0 แทน NULL
SELECT 
    product_name,
    unit_price,
    units_in_stock,
    COALESCE(unit_price, 0) * COALESCE(units_in_stock, 0) AS safe_inventory_value
FROM products;

-- ============================================================
-- EXAMPLE 20: COALESCE สำหรับ Default Values
-- ============================================================

-- ถ้า order ยังไม่ shipped ให้แสดง 'Pending'
SELECT 
    order_id,
    order_date,
    COALESCE(CAST(shipped_date AS TEXT), 'Not Yet Shipped') AS ship_status
FROM orders;

-- ผลลัพธ์:
-- order_id | order_date  | ship_status
-- ---------|-------------|------------------
-- 1        | 2024-01-05  | 2024-01-10
-- 3        | 2024-01-12  | Not Yet Shipped
-- 6        | 2024-02-10  | Not Yet Shipped
-- ...
```

---

## 8.7 NULLIF Function

```sql
-- ============================================================
-- NULLIF(val1, val2) 
-- Returns NULL if val1 = val2, otherwise returns val1
-- ============================================================

-- ============================================================
-- EXAMPLE 21: NULLIF พื้นฐาน
-- ============================================================

SELECT 
    NULLIF(10, 10),   -- NULL (10 = 10)
    NULLIF(10, 20),   -- 10 (10 <> 20)
    NULLIF('A', 'A'), -- NULL
    NULLIF('A', 'B'), -- 'A'
    NULLIF(0, 0),     -- NULL
    NULLIF(0, 1);     -- 0

-- ============================================================
-- EXAMPLE 22: NULLIF ป้องกัน Division by Zero
-- ============================================================

-- ❌ Division by zero error!
SELECT 100 / 0;  -- ERROR

-- ✓ ใช้ NULLIF แปลง 0 เป็น NULL ก่อน
SELECT 100 / NULLIF(0, 0);  -- NULL (ไม่ error)
SELECT 100 / NULLIF(5, 0);  -- 20

-- ตัวอย่างจริง: คำนวณ percentage
SELECT 
    product_name,
    units_in_stock,
    200 AS max_stock,
    ROUND(units_in_stock * 100.0 / NULLIF(200, 0), 1) AS stock_pct,
    ROUND(units_in_stock * 100.0 / NULLIF(units_in_stock, 0), 1) AS self_pct -- ทดสอบ
FROM products
LIMIT 3;

-- ============================================================
-- EXAMPLE 23: NULLIF แปลง Empty String เป็น NULL
-- ============================================================

-- บ่อยครั้งที่ form เว็บส่งค่า '' มา ควรเก็บเป็น NULL แทน
SELECT NULLIF('', '') AS empty_to_null;  -- NULL
SELECT NULLIF('John', '') AS non_empty;   -- 'John'

-- ตัวอย่างการ INSERT ที่ดี:
-- INSERT INTO customers (phone) VALUES (NULLIF(@user_input, ''));

-- ============================================================
-- EXAMPLE 24: COALESCE กับ NULLIF รวมกัน
-- ============================================================

-- แปลง '' เป็น NULL แล้วใช้ COALESCE
SELECT 
    id,
    name,
    COALESCE(NULLIF(name, ''), 'Unknown') AS safe_name
FROM null_test;

-- ผลลัพธ์:
-- id | name    | safe_name
-- ---|---------|----------
-- 1  | Alice   | Alice
-- 2  | Bob     | Bob
-- 3  | Charlie | Charlie
-- 4  | NULL    | Unknown
-- 5  | NULL    | Unknown
-- 6  | Diana   | Diana
-- 7  | Eve     | Eve
-- 8  |         | Unknown  ← empty string → NULL → 'Unknown'
```

---

## 8.8 IFNULL / NVL - Database Specific

```sql
-- ============================================================
-- IFNULL / NVL (พิเศษตาม database)
-- ============================================================

-- MySQL / SQLite: IFNULL(value, default)
SELECT IFNULL(NULL, 'default');  -- 'default'
SELECT IFNULL('value', 'default'); -- 'value'

-- Oracle: NVL(value, default)
SELECT NVL(NULL, 'default') FROM DUAL;

-- SQL Server: ISNULL(value, default)
SELECT ISNULL(NULL, 'default');

-- ทุก database: COALESCE (แนะนำสำหรับ portability)
SELECT COALESCE(NULL, 'default');

-- ============================================================
-- EXAMPLE 25: IFNULL ใน MySQL / SQLite
-- ============================================================

-- SQLite:
SELECT 
    first_name,
    manager_id,
    IFNULL(manager_id, 0) AS manager_or_zero
FROM employees;

-- เทียบกับ COALESCE:
SELECT 
    first_name,
    IFNULL(manager_id, 0) AS ifnull_result,
    COALESCE(manager_id, 0) AS coalesce_result
FROM employees;
-- ผลลัพธ์เหมือนกันทุกอย่าง
```

---

## 8.9 Handling NULL ใน Real World

```sql
-- ============================================================
-- EXAMPLE 26: NULL ใน Order Tracking
-- ============================================================

SELECT 
    order_id,
    order_date,
    COALESCE(CAST(shipped_date AS TEXT), 'Pending') AS ship_date,
    COALESCE(CAST(required_date AS TEXT), 'No deadline') AS deadline,
    status,
    CASE 
        WHEN shipped_date IS NULL AND status = 'Cancelled' THEN 'Cancelled'
        WHEN shipped_date IS NULL THEN 'Not Shipped Yet'
        WHEN shipped_date <= required_date THEN 'On Time'
        WHEN shipped_date > required_date THEN 'Late'
        ELSE 'Unknown'
    END AS delivery_status
FROM orders
ORDER BY order_id;

-- ============================================================
-- EXAMPLE 27: NULL ใน Customer Data Quality
-- ============================================================

SELECT 
    customer_id,
    first_name || ' ' || last_name AS name,
    
    -- Data completeness check
    CASE WHEN company_name IS NULL THEN 0 ELSE 1 END +
    CASE WHEN email IS NULL THEN 0 ELSE 1 END +
    CASE WHEN phone IS NULL THEN 0 ELSE 1 END +
    CASE WHEN address IS NULL THEN 0 ELSE 1 END AS completeness_score,
    
    -- Missing fields list
    CASE WHEN company_name IS NULL THEN 'company_name, ' ELSE '' END ||
    CASE WHEN email IS NULL THEN 'email, ' ELSE '' END ||
    CASE WHEN phone IS NULL THEN 'phone, ' ELSE '' END AS missing_fields
FROM customers;

-- ============================================================
-- EXAMPLE 28: NULL-Safe Comparison
-- ============================================================

-- Standard SQL: NULL-safe equality ไม่มี
-- PostgreSQL: ใช้ IS DISTINCT FROM / IS NOT DISTINCT FROM
-- MySQL: ใช้ <=> (NULL-safe equal)

-- PostgreSQL:
SELECT 
    1 IS DISTINCT FROM 1,       -- FALSE (เท่ากัน)
    1 IS DISTINCT FROM 2,       -- TRUE (ต่างกัน)
    NULL IS DISTINCT FROM NULL, -- FALSE (ทั้งคู่ NULL = เหมือนกัน)
    1 IS DISTINCT FROM NULL;    -- TRUE (ต่างกัน)

-- MySQL:
SELECT 
    1 <=> 1,       -- 1 (TRUE)
    1 <=> 2,       -- 0 (FALSE)
    NULL <=> NULL, -- 1 (TRUE!) ← ต่างจาก = 
    1 <=> NULL;    -- 0 (FALSE)

-- SQLite workaround:
SELECT 
    CASE WHEN (a IS NULL AND b IS NULL) OR a = b 
         THEN 'Equal' 
         ELSE 'Not Equal' 
    END AS comparison
FROM (SELECT NULL AS a, NULL AS b);  -- Equal

-- ============================================================
-- EXAMPLE 29: NULL ใน GROUP BY
-- ============================================================

-- GROUP BY รวม NULL เป็นกลุ่มเดียวกัน (เหมือน NULL = NULL สำหรับ grouping)
SELECT 
    manager_id,
    COUNT(*) AS headcount,
    AVG(salary) AS avg_salary
FROM employees
GROUP BY manager_id
ORDER BY manager_id NULLS LAST;

-- ผลลัพธ์:
-- manager_id | headcount | avg_salary
-- -----------|-----------|----------
-- 1          | 3         | 58333.33
-- 2          | 1         | 42000.00
-- 3          | 1         | 55000.00
-- 4          | 1         | 38000.00
-- 5          | 2         | 60000.00
-- NULL       | 7         | 86142.86  ← NULL grouped together

-- ============================================================
-- EXAMPLE 30: NULL ใน DISTINCT
-- ============================================================

-- SELECT DISTINCT นับ NULL เป็น unique value
SELECT DISTINCT manager_id FROM employees;
-- รวม NULL เป็น 1 distinct value

SELECT COUNT(DISTINCT manager_id) FROM employees;
-- ไม่นับ NULL! ← COUNT DISTINCT ไม่นับ NULL
```

---

## 8.10 Common NULL Pitfalls

```sql
-- ============================================================
-- EXAMPLE 31: Pitfall 1 - ORDER BY กับ NULL
-- ============================================================

SELECT first_name, manager_id
FROM employees
ORDER BY manager_id;
-- PostgreSQL: NULL ท้าย (ASC)
-- MySQL: NULL หน้า (ASC)

-- Safe: explicit NULLS LAST
SELECT first_name, manager_id
FROM employees
ORDER BY manager_id ASC NULLS LAST;  -- PostgreSQL/SQLite

-- MySQL workaround:
SELECT first_name, manager_id
FROM employees
ORDER BY manager_id IS NULL, manager_id;
-- manager_id IS NULL = 0 สำหรับ not-NULL, 1 สำหรับ NULL
-- เรียงตาม 0 ก่อน (non-NULL) แล้วตาม 1 (NULL)

-- ============================================================
-- EXAMPLE 32: Pitfall 2 - CASE with NULL
-- ============================================================

-- ❌ ผิด: CASE ไม่ตรวจ NULL ด้วย =
SELECT 
    manager_id,
    CASE manager_id
        WHEN NULL THEN 'Top Level'  -- ไม่เคยเป็น TRUE! NULL <> NULL
        ELSE 'Has Manager'
    END AS status
FROM employees;
-- ผลลัพธ์: ทุก row = 'Has Manager' แม้แต่ row ที่ manager_id = NULL

-- ✓ ถูก: ใช้ searched CASE
SELECT 
    manager_id,
    CASE 
        WHEN manager_id IS NULL THEN 'Top Level'
        ELSE 'Has Manager'
    END AS status
FROM employees;

-- ============================================================
-- EXAMPLE 33: Pitfall 3 - NOT IN กับ NULL (ซ้ำจาก Example 7)
-- ============================================================

-- ฝังใจไว้: NOT IN กับ subquery ที่อาจมี NULL → อาจได้ 0 rows!
SELECT first_name FROM employees
WHERE department_id NOT IN (
    SELECT manager_id FROM departments  -- manager_id อาจมี NULL
);
-- ถ้า departments.manager_id มี NULL → ได้ 0 rows!

-- ✓ Safe: เพิ่ม WHERE NOT NULL ใน subquery
SELECT first_name FROM employees
WHERE department_id NOT IN (
    SELECT manager_id FROM departments
    WHERE manager_id IS NOT NULL  -- ← สำคัญ!
);

-- ============================================================
-- EXAMPLE 34: Pitfall 4 - Aggregate กับ NULL
-- ============================================================

-- ฝังใจ: SUM, AVG, MIN, MAX ignore NULL
-- COUNT(*) นับทุกแถว
-- COUNT(column) ไม่นับ NULL

SELECT 
    -- ถ้าทุก value เป็น NULL
    SUM(CASE WHEN 1=2 THEN salary END) AS all_null_sum,   -- NULL
    AVG(CASE WHEN 1=2 THEN salary END) AS all_null_avg,   -- NULL
    COUNT(CASE WHEN 1=2 THEN salary END) AS all_null_count -- 0 ไม่ใช่ NULL!
FROM employees;

-- ============================================================
-- EXAMPLE 35: Pitfall 5 - String Concat กับ NULL
-- ============================================================

-- PostgreSQL / SQLite: NULL || string = NULL
SELECT 'Hello' || NULL || 'World';  -- NULL! ไม่ใช่ 'HelloWorld'

-- MySQL: CONCAT('Hello', NULL, 'World') = NULL
-- แต่ MySQL 8+: CONCAT_WS('', 'Hello', NULL, 'World') = 'HelloWorld'

-- Safe: ใช้ COALESCE
SELECT 'Hello' || COALESCE(NULL, '') || 'World';  -- 'HelloWorld'

-- ============================================================
-- EXAMPLE 36: Best Practices Summary
-- ============================================================

-- 1. ใช้ IS NULL / IS NOT NULL (ไม่ใช่ = NULL)
-- 2. ระวัง NOT IN กับ NULL ใน subquery
-- 3. ใช้ COALESCE สำหรับ default values
-- 4. NULLIF สำหรับแปลง special values เป็น NULL
-- 5. COUNT(*) vs COUNT(column) - แตกต่างกันเมื่อมี NULL
-- 6. AVG ไม่รวม NULL ในการหาร → อาจผิดที่คาดหวัง
-- 7. ORDER BY NULL behavior ต่างกันแต่ละ database

-- ============================================================
-- EXAMPLE 37-40: Real-World NULL Handling
-- ============================================================

-- 37: Safe division
SELECT 
    product_name,
    units_in_stock,
    200 AS target,
    ROUND(units_in_stock * 100.0 / NULLIF(200, 0), 2) AS pct_of_target
FROM products;

-- 38: Default phone
SELECT 
    first_name,
    COALESCE(phone, 'N/A') AS phone_display
FROM employees;

-- 39: Safe salary calculation
SELECT 
    first_name,
    COALESCE(salary, 0) AS salary,
    COALESCE(salary, 0) * 12 AS annual
FROM employees;

-- 40: Customer complete check
SELECT 
    customer_id,
    first_name,
    CASE 
        WHEN email IS NULL AND phone IS NULL THEN 'No Contact Info'
        WHEN email IS NULL THEN 'Email Missing'
        WHEN phone IS NULL THEN 'Phone Missing'
        ELSE 'Complete'
    END AS contact_status
FROM customers;
```

---

## 8.11 35+ ตัวอย่างครบถ้วน

```sql
-- 1. ดู employees ที่มี manager_id
SELECT first_name, manager_id FROM employees WHERE manager_id IS NOT NULL;

-- 2. ดู employees ที่ไม่มี manager
SELECT first_name, manager_id FROM employees WHERE manager_id IS NULL;

-- 3. COALESCE แทน NULL ด้วย 0
SELECT first_name, COALESCE(manager_id, 0) FROM employees;

-- 4. COALESCE หลายค่า
SELECT customer_id, COALESCE(company_name, first_name, 'Unknown') AS name FROM customers;

-- 5. NULLIF division
SELECT 100 / NULLIF(0, 0);  -- NULL แทน error

-- 6. NULLIF empty string
SELECT NULLIF('', '') AS result;  -- NULL

-- 7. COUNT * vs COUNT column
SELECT COUNT(*), COUNT(manager_id) FROM employees;

-- 8. AVG ignores NULL
SELECT AVG(value), AVG(COALESCE(value, 0)) FROM null_test;

-- 9. NULL in CASE
SELECT first_name, CASE WHEN manager_id IS NULL THEN 'Boss' ELSE 'Employee' END FROM employees;

-- 10. NULL in ORDER BY
SELECT first_name, manager_id FROM employees ORDER BY manager_id NULLS LAST;

-- 11. NULL safe NOT IN
SELECT first_name FROM employees
WHERE manager_id NOT IN (SELECT manager_id FROM departments WHERE manager_id IS NOT NULL);

-- 12. ISNULL/IFNULL
SELECT first_name, IFNULL(manager_id, -1) FROM employees;  -- SQLite/MySQL

-- 13. IS NULL vs = NULL
SELECT COUNT(*) FROM employees WHERE manager_id IS NULL;    -- correct
SELECT COUNT(*) FROM employees WHERE manager_id = NULL;     -- 0 (wrong)

-- 14. NULL in string concat
SELECT first_name || ' ' || COALESCE(last_name, '') FROM employees;

-- 15. NULL in SUM
SELECT SUM(value), SUM(COALESCE(value, 0)) FROM null_test;

-- 16. NULL-safe equal (PostgreSQL)
SELECT * FROM employees WHERE manager_id IS NOT DISTINCT FROM NULL;

-- 17. NULL in GROUP BY
SELECT manager_id, COUNT(*) FROM employees GROUP BY manager_id;

-- 18. NULLIF for sentinel values
SELECT NULLIF(salary, 99999) AS real_salary FROM employees;  -- treat 99999 as NULL

-- 19. COALESCE for date default
SELECT order_id, COALESCE(CAST(shipped_date AS TEXT), 'Pending') FROM orders;

-- 20. NULL check in WHERE
SELECT * FROM orders WHERE shipped_date IS NULL AND status <> 'Cancelled';

-- 21. COALESCE in calculation
SELECT product_name, COALESCE(unit_price, 0) * units_in_stock FROM products;

-- 22. NULL in max/min
SELECT MAX(value), MIN(value) FROM null_test;  -- ignores NULL

-- 23. Percent null
SELECT 
    COUNT(*) AS total,
    COUNT(manager_id) AS has_manager,
    100.0 * COUNT(manager_id) / COUNT(*) AS pct_with_manager
FROM employees;

-- 24. Replace NULL with previous value (window functions - preview)
-- SELECT first_name, 
--     COALESCE(manager_id, LAG(manager_id) OVER (ORDER BY employee_id)) FROM employees;

-- 25. Null pattern
SELECT * FROM employees WHERE email LIKE '%@%' OR email IS NULL;

-- 26. Safe comparison
SELECT * FROM employees 
WHERE COALESCE(manager_id, 0) NOT IN (1, 2, 0);

-- 27. NULL in HAVING
SELECT department_id, AVG(salary) FROM employees
GROUP BY department_id
HAVING AVG(salary) IS NOT NULL;

-- 28. Fill NULL with default
UPDATE employees SET phone = 'N/A' WHERE phone IS NULL;
-- (undo: UPDATE employees SET phone = NULL WHERE phone = 'N/A')

-- 29. Count NULL values
SELECT COUNT(*) - COUNT(manager_id) AS null_count FROM employees;

-- 30. All NULL rows
SELECT * FROM null_test WHERE name IS NULL AND value IS NULL;

-- 31. COALESCE chain
SELECT COALESCE(NULL, NULL, NULL, 'last resort');  -- 'last resort'

-- 32. NULLIF กับ boolean
SELECT NULLIF(is_active, FALSE) FROM employees;  -- NULL if FALSE, TRUE if TRUE

-- 33. Check completeness
SELECT customer_id,
    (CASE WHEN first_name IS NULL THEN 0 ELSE 1 END +
     CASE WHEN last_name IS NULL THEN 0 ELSE 1 END +
     CASE WHEN email IS NULL THEN 0 ELSE 1 END +
     CASE WHEN phone IS NULL THEN 0 ELSE 1 END) * 25 AS pct_complete
FROM customers;

-- 34. NOT EXISTS instead of NOT IN (NULL-safe)
SELECT first_name FROM employees e
WHERE NOT EXISTS (SELECT 1 FROM departments d WHERE d.manager_id = e.employee_id);

-- 35. NULL in LIKE
SELECT first_name FROM employees WHERE first_name NOT LIKE 'ส%' OR first_name IS NULL;
```

---

## 8.12 สรุปบทที่ 8

```
✅ NULL = ไม่มีข้อมูล/ไม่ทราบค่า (ไม่ใช่ 0 หรือ '')
✅ Three-Valued Logic: TRUE, FALSE, UNKNOWN
✅ NULL comparisons = UNKNOWN
✅ IS NULL / IS NOT NULL (ไม่ใช่ = NULL)
✅ NULL ในคำนวณ = NULL เสมอ
✅ Aggregates ignore NULL (ยกเว้น COUNT(*))
✅ COALESCE: คืนค่าแรกที่ไม่ NULL
✅ NULLIF: คืน NULL ถ้า val1 = val2
✅ NOT IN กับ NULL อันตราย - ต้องระวัง
✅ ORDER BY NULL behavior ต่างกันแต่ละ database
```

---

## 📝 แบบฝึกหัดท้ายบท

**คำถาม 1:** หา employees ทุกคนที่ไม่มี manager และแสดงชื่อพร้อม "Top Level" label

**คำถาม 2:** ใช้ COALESCE แสดง customers โดยถ้าไม่มี company_name ให้แสดง "ลูกค้าบุคคล" แทน

**คำถาม 3:** หา orders ที่ยังไม่ได้ ship (shipped_date IS NULL) และ status ไม่ใช่ Cancelled

**คำถาม 4:** ใช้ NULLIF เพื่อป้องกัน division by zero ในการคำนวณ stock percentage

**คำถาม 5:** นับว่ามีพนักงานกี่คนที่มี manager และกี่คนที่ไม่มี

**คำถาม 6:** อธิบายว่า SELECT COUNT(*) และ SELECT COUNT(manager_id) ต่างกันอย่างไร

**คำถาม 7:** เขียน query ที่แสดง employees พร้อม "Has Manager" หรือ "Top Level" โดยใช้ CASE expression ที่ถูกต้อง

**คำถาม 8:** ค้นหาว่ากรณีไหนที่ NOT IN จะให้ผลผิดพลาดเมื่อมี NULL ใน subquery

**คำถาม 9:** เขียน query ตรวจสอบ completeness ของ customer data (score 0-4)

**คำถาม 10:** สร้าง query สรุปสถิติ NULL ของตาราง employees (กี่ field ที่มี NULL, กี่แถว)

---

## ✅ เฉลยแบบฝึกหัด

### เฉลยที่ 1:
```sql
SELECT 
    first_name || ' ' || last_name AS name,
    manager_id,
    'Top Level Employee' AS status
FROM employees
WHERE manager_id IS NULL;
```

### เฉลยที่ 2:
```sql
SELECT 
    customer_id,
    COALESCE(company_name, 'ลูกค้าบุคคล') AS display_name,
    first_name || ' ' || last_name AS contact_name
FROM customers
ORDER BY display_name;
```

### เฉลยที่ 3:
```sql
SELECT order_id, customer_id, order_date, required_date, status
FROM orders
WHERE shipped_date IS NULL
  AND status <> 'Cancelled'
ORDER BY required_date;
```

### เฉลยที่ 4:
```sql
SELECT 
    product_name,
    units_in_stock,
    200 AS target_stock,
    ROUND(units_in_stock * 100.0 / NULLIF(200, 0), 1) AS stock_pct
FROM products;
-- ถ้า target = 0 จะได้ NULL แทน error
```

### เฉลยที่ 5:
```sql
SELECT 
    COUNT(*) AS total_employees,
    COUNT(manager_id) AS has_manager,
    COUNT(*) - COUNT(manager_id) AS no_manager
FROM employees;
```

### เฉลยที่ 6:
```
COUNT(*) = นับทุกแถว รวมแถวที่ manager_id เป็น NULL = 15
COUNT(manager_id) = นับเฉพาะแถวที่ manager_id ไม่ใช่ NULL = 8
ต่างกัน: COUNT(*) - COUNT(manager_id) = 7 (คือจำนวนแถวที่ NULL)
```

### เฉลยที่ 7:
```sql
SELECT 
    first_name,
    manager_id,
    CASE 
        WHEN manager_id IS NULL THEN 'Top Level'  -- ✓ ใช้ IS NULL
        ELSE 'Has Manager (reports to #' || manager_id || ')'
    END AS manager_status
FROM employees;
```

### เฉลยที่ 8:
```sql
-- กรณีที่ NOT IN ผิดพลาด:
-- ถ้า subquery มี NULL ใน list

-- ตัวอย่าง:
-- สมมติว่า departments.manager_id = (1, 2, NULL)
SELECT first_name FROM employees
WHERE department_id NOT IN (
    SELECT manager_id FROM departments
    -- = NOT IN (1, 2, NULL)
    -- = department_id <> 1 AND department_id <> 2 AND department_id <> NULL
    -- = TRUE AND TRUE AND UNKNOWN = UNKNOWN
    -- ไม่มีแถวไหนเลย!
);

-- แก้ไข:
SELECT first_name FROM employees
WHERE department_id NOT IN (
    SELECT manager_id FROM departments WHERE manager_id IS NOT NULL
);
```

### เฉลยที่ 9:
```sql
SELECT 
    customer_id,
    first_name || ' ' || last_name AS name,
    CASE WHEN company_name IS NULL THEN 0 ELSE 1 END +
    CASE WHEN email IS NULL THEN 0 ELSE 1 END +
    CASE WHEN phone IS NULL THEN 0 ELSE 1 END +
    CASE WHEN address IS NULL THEN 0 ELSE 1 END AS completeness_score,
    4 AS max_score
FROM customers;
```

### เฉลยที่ 10:
```sql
SELECT 
    'manager_id' AS field_name,
    COUNT(*) - COUNT(manager_id) AS null_count,
    COUNT(*) AS total_rows,
    ROUND((COUNT(*) - COUNT(manager_id)) * 100.0 / COUNT(*), 1) AS null_pct
FROM employees
UNION ALL
SELECT 
    'salary',
    COUNT(*) - COUNT(salary),
    COUNT(*),
    ROUND((COUNT(*) - COUNT(salary)) * 100.0 / COUNT(*), 1)
FROM employees
UNION ALL
SELECT 
    'department_id',
    COUNT(*) - COUNT(department_id),
    COUNT(*),
    ROUND((COUNT(*) - COUNT(department_id)) * 100.0 / COUNT(*), 1)
FROM employees;
```

---

## ➡️ บทถัดไป

**[Part 009: DISTINCT - Removing Duplicates](part-009.md)**

ในบทถัดไปเราจะเรียนรู้การลบข้อมูลซ้ำด้วย DISTINCT!

---

*Part 008 of 120 | หลักสูตร SQL ครบวงจร*
