# Part 003: Basic SELECT - Your First Queries

> **หลักสูตร SQL ครบวงจร | Part 3 of 120**

---

## 🎯 สิ่งที่จะได้เรียนรู้ในบทนี้

- SELECT syntax ทั้งหมด
- การเลือก columns แบบต่างๆ
- SELECT * vs. เลือก columns เฉพาะ
- Column expressions และการคำนวณ
- String literals ใน SELECT
- Mathematical expressions
- การใช้ DISTINCT
- Basic output formatting

**เวลาที่ใช้เรียน**: ประมาณ 2-3 ชั่วโมง

---

## ฐานข้อมูลที่ใช้ในบทนี้

ก่อนเริ่ม ให้สร้างตารางตัวอย่างนี้:

```sql
-- สร้างตารางที่ใช้ในบทนี้
CREATE TABLE departments (
    department_id   INTEGER PRIMARY KEY,
    department_name VARCHAR(50) NOT NULL,
    location        VARCHAR(100),
    budget          DECIMAL(15, 2)
);

CREATE TABLE employees (
    employee_id    INTEGER PRIMARY KEY,
    first_name     VARCHAR(50) NOT NULL,
    last_name      VARCHAR(50) NOT NULL,
    email          VARCHAR(100),
    hire_date      DATE,
    job_title      VARCHAR(100),
    salary         DECIMAL(10, 2),
    department_id  INTEGER,
    manager_id     INTEGER,
    is_active      BOOLEAN DEFAULT TRUE
);

CREATE TABLE products (
    product_id    INTEGER PRIMARY KEY,
    product_name  VARCHAR(100) NOT NULL,
    category      VARCHAR(50),
    unit_price    DECIMAL(10, 2),
    units_in_stock INTEGER,
    discontinued  BOOLEAN DEFAULT FALSE
);

-- ใส่ข้อมูล departments
INSERT INTO departments VALUES
(1, 'Information Technology', 'Building A, Floor 3', 5000000.00),
(2, 'Human Resources',        'Building B, Floor 1', 2000000.00),
(3, 'Finance',                'Building A, Floor 2', 3000000.00),
(4, 'Marketing',              'Building C, Floor 2', 4000000.00),
(5, 'Sales',                  'Building C, Floor 1', 8000000.00),
(6, 'Operations',             'Building D',          6000000.00),
(7, 'Research & Development', 'Building E',          7000000.00);

-- ใส่ข้อมูล employees
INSERT INTO employees VALUES
(1,  'สมชาย',   'นักเขียน',  'somchai.n@company.com',   '2018-03-15', 'Senior Developer',    75000.00, 1, NULL,  TRUE),
(2,  'สมหญิง',  'ดีงาม',     'somying.d@company.com',   '2019-07-01', 'HR Manager',          65000.00, 2, NULL,  TRUE),
(3,  'วิภา',    'รักงาน',    'wipa.r@company.com',      '2017-01-10', 'CFO',                 120000.00, 3, NULL,  TRUE),
(4,  'ประยูร',  'มีสุข',     'prayoon.m@company.com',   '2020-05-20', 'Marketing Manager',   70000.00, 4, NULL,  TRUE),
(5,  'กิตติ',   'เก่งมาก',   'kitti.k@company.com',     '2016-09-30', 'Sales Director',      95000.00, 5, NULL,  TRUE),
(6,  'มาลี',    'สวยงาม',    'malee.s@company.com',     '2021-02-14', 'Junior Developer',    45000.00, 1, 1,     TRUE),
(7,  'อนันต์',  'ใจดี',      'anan.j@company.com',      '2020-11-01', 'Developer',           58000.00, 1, 1,     TRUE),
(8,  'รัตนา',   'ขยันมาก',   'rattana.k@company.com',   '2019-03-22', 'HR Specialist',       42000.00, 2, 2,     TRUE),
(9,  'ชาญชัย', 'ฉลาดเฉลียว', 'chanchai.c@company.com',  '2018-08-15', 'Senior Accountant',   55000.00, 3, 3,     TRUE),
(10, 'นงนุช',   'น่ารักมาก', 'nongnuch.n@company.com',  '2022-01-03', 'Content Creator',     38000.00, 4, 4,     TRUE),
(11, 'ธนพล',   'รวยแน่',    'thanaphol.r@company.com', '2021-06-15', 'Sales Executive',     48000.00, 5, 5,     TRUE),
(12, 'พิมพ์ใจ', 'หวานใจ',    'pimjai.h@company.com',    '2020-09-01', 'Operations Manager',  68000.00, 6, NULL,  TRUE),
(13, 'สุรชัย',  'เด่นมาก',   'surachai.d@company.com',  '2019-12-20', 'R&D Lead',            85000.00, 7, NULL,  TRUE),
(14, 'จินตนา',  'คิดเก่ง',   'jintana.k@company.com',   '2023-03-01', 'Data Analyst',        52000.00, 1, 1,     TRUE),
(15, 'ไพโรจน์', 'กว้างขวาง', 'pairoj.k@company.com',    '2017-07-17', 'Senior Sales',        72000.00, 5, 5,     TRUE);

-- ใส่ข้อมูล products
INSERT INTO products VALUES
(1,  'Laptop Pro 15"',           'Electronics', 45000.00, 25, FALSE),
(2,  'Wireless Mouse',           'Electronics',   850.00, 150, FALSE),
(3,  'USB-C Hub',                'Electronics',  2500.00,  80, FALSE),
(4,  'Office Chair Premium',     'Furniture',    8900.00,  30, FALSE),
(5,  'Standing Desk',            'Furniture',   15000.00,  15, FALSE),
(6,  'SQL Programming Book',     'Books',          450.00, 200, FALSE),
(7,  'Python for Data Science',  'Books',          550.00, 180, FALSE),
(8,  'Noise Cancelling Headphone','Electronics',  7500.00,  45, FALSE),
(9,  'Monitor 27" 4K',           'Electronics', 18000.00,  20, FALSE),
(10, 'Keyboard Mechanical',      'Electronics',  3200.00,  60, FALSE),
(11, 'Webcam HD',                'Electronics',  2800.00,  40, FALSE),
(12, 'Desk Lamp LED',            'Furniture',    1200.00, 100, FALSE),
(13, 'Whiteboard A4',            'Stationery',    150.00, 500, FALSE),
(14, 'Notebook Premium',         'Stationery',    250.00, 300, FALSE),
(15, 'Pen Set',                  'Stationery',    120.00, 400, FALSE);
```

---

## 3.1 SELECT Syntax พื้นฐาน

### รูปแบบ SELECT Statement

```sql
-- รูปแบบเต็ม (ทุกส่วนเป็น optional ยกเว้น SELECT และ FROM)
SELECT [DISTINCT] column1, column2, ...
FROM table_name
[WHERE condition]
[GROUP BY columns]
[HAVING condition]
[ORDER BY columns [ASC|DESC]]
[LIMIT n]
[OFFSET m];

-- ลำดับที่ database ประมวลผล (ไม่เหมือนลำดับที่เขียน!):
-- 1. FROM - กำหนด table ที่ใช้
-- 2. WHERE - กรองแถว
-- 3. GROUP BY - จัดกลุ่ม
-- 4. HAVING - กรองกลุ่ม
-- 5. SELECT - เลือก columns
-- 6. DISTINCT - ลบ duplicates
-- 7. ORDER BY - จัดเรียง
-- 8. LIMIT/OFFSET - จำกัดผลลัพธ์
```

### คำอธิบายแต่ละส่วน

```sql
-- SELECT - ระบุ columns ที่ต้องการแสดง
SELECT first_name, last_name

-- FROM - ระบุตารางที่ดึงข้อมูล
FROM employees

-- WHERE - กรองข้อมูล (optional)
WHERE salary > 50000

-- ORDER BY - จัดเรียง (optional)
ORDER BY salary DESC

-- LIMIT - จำกัดจำนวนแถว (optional)
LIMIT 5;
```

---

## 3.2 SELECT * - ดูทุก Columns

```sql
-- ============================================================
-- EXAMPLE 1: SELECT * (ดูทุก columns)
-- ============================================================
SELECT * FROM employees;

-- ผลลัพธ์ (บางส่วน):
-- employee_id | first_name | last_name | email                    | ...
-- ------------|------------|-----------|--------------------------|----
-- 1           | สมชาย      | นักเขียน  | somchai.n@company.com   | ...
-- 2           | สมหญิง     | ดีงาม     | somying.d@company.com   | ...
-- ...

-- ============================================================
-- EXAMPLE 2: SELECT * จากตารางต่างๆ
-- ============================================================
SELECT * FROM departments;
SELECT * FROM products;

-- ============================================================
-- EXAMPLE 3: ข้อดี-ข้อเสียของ SELECT *
-- ============================================================
-- ข้อดี:
-- - เขียนง่าย เร็ว
-- - เห็นข้อมูลทั้งหมด
-- - ดีสำหรับ exploration

-- ข้อเสีย:
-- - ไม่รู้ว่าจะได้ columns อะไร (schema อาจเปลี่ยน)
-- - ดึงข้อมูลมากกว่าที่ต้องการ (performance)
-- - ใน JOIN อาจมี duplicate column names

-- แนะนำ: ใช้ SELECT * เฉพาะตอน explore data
-- ใน production code ควรระบุ columns เสมอ
```

---

## 3.3 SELECT Columns เฉพาะ

```sql
-- ============================================================
-- EXAMPLE 4: เลือก columns เฉพาะที่ต้องการ
-- ============================================================
SELECT first_name, last_name
FROM employees;

-- ผลลัพธ์:
-- first_name | last_name
-- -----------|----------
-- สมชาย      | นักเขียน
-- สมหญิง     | ดีงาม
-- วิภา       | รักงาน
-- ...

-- ============================================================
-- EXAMPLE 5: เลือกหลาย columns
-- ============================================================
SELECT employee_id, first_name, last_name, job_title, salary
FROM employees;

-- ============================================================
-- EXAMPLE 6: เลือก columns จาก products
-- ============================================================
SELECT product_name, category, unit_price
FROM products;

-- ผลลัพธ์:
-- product_name              | category    | unit_price
-- --------------------------|-------------|----------
-- Laptop Pro 15"            | Electronics | 45000.00
-- Wireless Mouse            | Electronics |   850.00
-- USB-C Hub                 | Electronics |  2500.00
-- Office Chair Premium      | Furniture   |  8900.00
-- ...

-- ============================================================
-- EXAMPLE 7: ลำดับ columns ในผลลัพธ์ตามที่เราระบุ
-- ============================================================
-- ลำดับใน SELECT จะเป็นลำดับในผลลัพธ์
SELECT salary, last_name, first_name, employee_id
FROM employees;

-- ============================================================
-- EXAMPLE 8: เลือก column เดียว
-- ============================================================
SELECT job_title
FROM employees;

-- ผลลัพธ์ (มี duplicates):
-- job_title
-- ------------------
-- Senior Developer
-- HR Manager
-- CFO
-- Marketing Manager
-- Sales Director
-- Junior Developer
-- Developer
-- HR Specialist
-- Senior Accountant
-- Content Creator
-- Sales Executive
-- Operations Manager
-- R&D Lead
-- Data Analyst
-- Senior Sales
```

---

## 3.4 Expressions ใน SELECT

คุณสามารถใช้ expressions ต่างๆ ได้โดยไม่ต้องมี table!

```sql
-- ============================================================
-- EXAMPLE 9: SELECT โดยไม่มี FROM (ทำงานบน PostgreSQL, MySQL)
-- ============================================================

-- ค่าตัวเลข
SELECT 42;
SELECT 3.14159;

-- การคำนวณ
SELECT 10 + 5;       -- 15
SELECT 100 - 37;     -- 63
SELECT 12 * 8;       -- 96
SELECT 100 / 4;      -- 25
SELECT 17 % 5;       -- 2 (modulo/เศษ)

-- String literals
SELECT 'Hello, World!';
SELECT 'สวัสดี ชาวโลก';

-- ใน SQLite ต้องใช้ FROM ด้วย:
-- SELECT 42;  ← ใช้ได้ใน SQLite เวอร์ชันใหม่
-- SELECT 42 FROM (SELECT 1);  ← ทำงานทุกเวอร์ชัน

-- ============================================================
-- EXAMPLE 10: Mathematical Expressions กับ column values
-- ============================================================

-- คำนวณจากข้อมูล
SELECT 
    first_name,
    salary,
    salary * 12 AS annual_salary    -- เงินเดือนต่อปี
FROM employees;

-- ผลลัพธ์:
-- first_name | salary    | annual_salary
-- -----------|-----------|---------------
-- สมชาย      | 75000.00  | 900000.00
-- สมหญิง     | 65000.00  | 780000.00
-- ...

-- ============================================================
-- EXAMPLE 11: การคำนวณหลายรูปแบบ
-- ============================================================
SELECT 
    product_name,
    unit_price,
    units_in_stock,
    unit_price * units_in_stock AS inventory_value,   -- มูลค่าสินค้าในคลัง
    unit_price * 0.07 AS vat_amount,                  -- ภาษี VAT 7%
    unit_price * 1.07 AS price_with_vat               -- ราคารวม VAT
FROM products;

-- ผลลัพธ์:
-- product_name        | unit_price | units_in_stock | inventory_value | vat_amount | price_with_vat
-- --------------------|------------|----------------|-----------------|------------|----------------
-- Laptop Pro 15"      | 45000.00   | 25             | 1125000.00      | 3150.00    | 48150.00
-- Wireless Mouse      | 850.00     | 150            | 127500.00       | 59.50      | 909.50
-- ...

-- ============================================================
-- EXAMPLE 12: Integer Division vs Float Division
-- ============================================================

-- Integer Division (ถ้า operands ทั้งคู่เป็น integer)
SELECT 10 / 3;        -- SQLite: 3 (integer division!)
                      -- PostgreSQL: 3 (integer division)
                      -- MySQL: 3.3333... (float)

-- ได้ผลเป็น float:
SELECT 10.0 / 3;      -- 3.3333...
SELECT CAST(10 AS FLOAT) / 3;  -- 3.3333...

-- ============================================================
-- EXAMPLE 13: String Concatenation
-- ============================================================

-- PostgreSQL / SQLite ใช้ ||
SELECT first_name || ' ' || last_name AS full_name
FROM employees;

-- MySQL ใช้ CONCAT
SELECT CONCAT(first_name, ' ', last_name) AS full_name
FROM employees;

-- ผลลัพธ์:
-- full_name
-- -----------------
-- สมชาย นักเขียน
-- สมหญิง ดีงาม
-- วิภา รักงาน
-- ...

-- ============================================================
-- EXAMPLE 14: String Expressions ซับซ้อนขึ้น
-- ============================================================
SELECT 
    first_name || ' ' || last_name AS full_name,
    'Mr./Ms. ' || first_name AS greeting,
    email || ' (Employee #' || employee_id || ')' AS email_with_id
FROM employees
LIMIT 5;
```

---

## 3.5 SELECT ค่าคงที่ (Literals)

```sql
-- ============================================================
-- EXAMPLE 15: String Literals
-- ============================================================
SELECT 'Hello' AS greeting;
SELECT 'SQL Course 2024' AS course_name;
SELECT 'ราคายังไม่รวม VAT' AS note;

-- ============================================================
-- EXAMPLE 16: Numeric Literals
-- ============================================================
SELECT 100 AS hundred;
SELECT 3.14159 AS pi;
SELECT -273.15 AS absolute_zero;
SELECT 1e6 AS one_million;    -- Scientific notation

-- ============================================================
-- EXAMPLE 17: Boolean Literals
-- ============================================================
SELECT TRUE AS is_active;
SELECT FALSE AS is_deleted;
SELECT 1 = 1 AS always_true;
SELECT 1 = 2 AS always_false;

-- ============================================================
-- EXAMPLE 18: NULL Literal
-- ============================================================
SELECT NULL AS nothing;
SELECT NULL AS missing_value;

-- ============================================================
-- EXAMPLE 19: Date Literals
-- ============================================================
SELECT DATE '2024-01-15' AS date_value;          -- PostgreSQL
SELECT '2024-01-15' AS date_string;               -- ทุก database

-- ============================================================
-- EXAMPLE 20: Mixing Literals กับ Column Values
-- ============================================================
SELECT 
    first_name,
    last_name,
    'Employee' AS type,
    2024 AS report_year,
    salary,
    salary * 12 AS annual_salary,
    'THB' AS currency
FROM employees
LIMIT 5;

-- ผลลัพธ์:
-- first_name | last_name | type     | report_year | salary   | annual_salary | currency
-- -----------|-----------|----------|-------------|----------|---------------|--------
-- สมชาย      | นักเขียน  | Employee | 2024        | 75000.00 | 900000.00     | THB
-- สมหญิง     | ดีงาม     | Employee | 2024        | 65000.00 | 780000.00     | THB
```

---

## 3.6 การทำงานกับ Numbers ใน SELECT

```sql
-- ============================================================
-- EXAMPLE 21: Arithmetic Operations ครบถ้วน
-- ============================================================
SELECT
    10 + 3 AS addition,        -- 13
    10 - 3 AS subtraction,     -- 7
    10 * 3 AS multiplication,  -- 30
    10 / 3 AS division,        -- 3 (integer)
    10.0 / 3 AS float_div,     -- 3.333...
    10 % 3 AS modulo,          -- 1
    10 ^ 2 AS power;           -- 100 (PostgreSQL)
    -- POWER(10, 2) ใช้ได้ทุก database

-- ============================================================
-- EXAMPLE 22: ROUND - การปัดเศษ
-- ============================================================
SELECT
    ROUND(3.14159, 2) AS rounded_2dp,    -- 3.14
    ROUND(3.14159, 4) AS rounded_4dp,    -- 3.1416
    ROUND(3.14159, 0) AS rounded_int,    -- 3.0
    ROUND(3.5)        AS round_half_up,  -- 4
    ROUND(2.5)        AS round_half_bank;-- 2 (banker's rounding บางระบบ)

-- ============================================================
-- EXAMPLE 23: ABS - ค่าสัมบูรณ์
-- ============================================================
SELECT
    ABS(-100) AS abs_negative,   -- 100
    ABS(100)  AS abs_positive,   -- 100
    ABS(-3.14) AS abs_float;     -- 3.14

-- ============================================================
-- EXAMPLE 24: ใช้ ROUND กับข้อมูลจริง
-- ============================================================
SELECT 
    product_name,
    unit_price,
    ROUND(unit_price * 1.07, 2) AS price_with_vat,
    ROUND(unit_price * 0.9, 2)  AS price_10pct_discount
FROM products
LIMIT 5;

-- ============================================================
-- EXAMPLE 25: การคำนวณ Commission
-- ============================================================
SELECT 
    first_name || ' ' || last_name AS sales_person,
    salary,
    ROUND(salary * 0.10, 2) AS commission_10pct,
    ROUND(salary * 0.15, 2) AS commission_15pct,
    salary + ROUND(salary * 0.10, 2) AS salary_with_commission
FROM employees
WHERE department_id = 5;  -- Sales department

-- ============================================================
-- EXAMPLE 26: เปรียบเทียบสิ่งที่ได้จาก Expression
-- ============================================================
SELECT 
    product_name,
    unit_price AS original_price,
    ROUND(unit_price * 0.85, 2) AS sale_price,        -- ลด 15%
    unit_price - ROUND(unit_price * 0.85, 2) AS discount_amount,
    '15% OFF' AS promotion_label
FROM products
WHERE category = 'Electronics';
```

---

## 3.7 String Operations ใน SELECT

```sql
-- ============================================================
-- EXAMPLE 27: String Concatenation หลายรูปแบบ
-- ============================================================

-- PostgreSQL / SQLite
SELECT first_name || ' ' || last_name AS full_name FROM employees;

-- MySQL / MariaDB
SELECT CONCAT(first_name, ' ', last_name) AS full_name FROM employees;

-- SQL Server
SELECT first_name + ' ' + last_name AS full_name FROM employees;

-- ANSI Standard (ใช้ได้บางตัว)
SELECT CONCAT(first_name, ' ', last_name) AS full_name FROM employees;

-- ============================================================
-- EXAMPLE 28: UPPER และ LOWER
-- ============================================================
SELECT 
    first_name,
    UPPER(first_name) AS upper_name,
    LOWER(last_name)  AS lower_name
FROM employees
LIMIT 5;

-- ============================================================
-- EXAMPLE 29: LENGTH / LEN
-- ============================================================
SELECT 
    product_name,
    LENGTH(product_name) AS name_length   -- PostgreSQL, MySQL, SQLite
    -- LEN(product_name) ← SQL Server
FROM products
ORDER BY LENGTH(product_name);

-- ============================================================
-- EXAMPLE 30: TRIM - ลบ whitespace
-- ============================================================
SELECT 
    TRIM('  Hello World  ') AS trimmed,       -- 'Hello World'
    LTRIM('  Hello World  ') AS left_trimmed, -- 'Hello World  '
    RTRIM('  Hello World  ') AS right_trimmed;-- '  Hello World'

-- ============================================================
-- EXAMPLE 31: SUBSTR / SUBSTRING - ดึงส่วนของ string
-- ============================================================

-- SUBSTR(string, start, length) - SQLite, MySQL
SELECT SUBSTR('Hello World', 1, 5);  -- 'Hello'
SELECT SUBSTR('Hello World', 7);     -- 'World'

-- SUBSTRING(string FROM start FOR length) - PostgreSQL
SELECT SUBSTRING('Hello World' FROM 1 FOR 5);  -- 'Hello'
SELECT SUBSTRING('Hello World' FROM 7);         -- 'World'

-- ตัวอย่างกับข้อมูลจริง
SELECT 
    email,
    SUBSTR(email, 1, INSTR(email, '@') - 1) AS username  -- SQLite
FROM employees;

-- ============================================================
-- EXAMPLE 32: REPLACE
-- ============================================================
SELECT 
    REPLACE('Hello World', 'World', 'SQL') AS replaced,  -- 'Hello SQL'
    REPLACE(email, '@company.com', '') AS email_username
FROM employees;
```

---

## 3.8 SELECT พร้อม Computed Columns

```sql
-- ============================================================
-- EXAMPLE 33: Business Calculations
-- ============================================================

-- มูลค่าสินค้าในคลัง
SELECT 
    product_name,
    category,
    unit_price,
    units_in_stock,
    unit_price * units_in_stock AS total_inventory_value
FROM products
ORDER BY total_inventory_value DESC;

-- ผลลัพธ์:
-- product_name         | category    | unit_price | units_in_stock | total_inventory_value
-- ---------------------|-------------|------------|----------------|----------------------
-- Monitor 27" 4K       | Electronics | 18000.00   | 20             | 360000.00
-- Laptop Pro 15"       | Electronics | 45000.00   | 25             | 1125000.00
-- ...

-- ============================================================
-- EXAMPLE 34: Employee Salary Calculations
-- ============================================================
SELECT 
    first_name || ' ' || last_name AS full_name,
    salary AS monthly_salary,
    salary * 12 AS annual_salary,
    salary / 30 AS daily_rate,
    salary / (30 * 8) AS hourly_rate,
    salary * 0.05 AS provident_fund_5pct,
    salary - (salary * 0.05) AS take_home_after_pf
FROM employees
ORDER BY salary DESC;

-- ============================================================
-- EXAMPLE 35: Age/Duration Calculations
-- ============================================================
-- คำนวณวันที่ทำงาน (ขึ้นกับ database)

-- SQLite:
SELECT 
    first_name,
    hire_date,
    DATE('now') AS today,
    -- จำนวนวันที่ทำงาน (approximate)
    (JULIANDAY(DATE('now')) - JULIANDAY(hire_date)) AS days_worked
FROM employees
LIMIT 5;

-- PostgreSQL:
-- SELECT first_name, hire_date, 
--        NOW()::date - hire_date AS days_worked
-- FROM employees;

-- MySQL:
-- SELECT first_name, hire_date,
--        DATEDIFF(CURDATE(), hire_date) AS days_worked
-- FROM employees;

-- ============================================================
-- EXAMPLE 36: Score / Percentage Calculations
-- ============================================================
SELECT 
    product_name,
    units_in_stock,
    -- สมมติว่า max stock = 200
    ROUND(units_in_stock * 100.0 / 200, 1) AS stock_percentage,
    CASE 
        WHEN units_in_stock > 100 THEN 'High Stock'
        WHEN units_in_stock > 50  THEN 'Medium Stock'
        ELSE 'Low Stock'
    END AS stock_status
FROM products
ORDER BY stock_percentage DESC;
```

---

## 3.9 SELECT พร้อม NULL Handling

```sql
-- ============================================================
-- EXAMPLE 37: NULL ใน SELECT
-- ============================================================
SELECT 
    employee_id,
    first_name,
    manager_id,              -- บางคนไม่มี manager (NULL)
    manager_id + 1 AS next_manager  -- NULL + 1 = NULL!
FROM employees;

-- ผลลัพธ์:
-- employee_id | first_name | manager_id | next_manager
-- ------------|------------|------------|-------------
-- 1           | สมชาย      | NULL       | NULL
-- 2           | สมหญิง     | NULL       | NULL
-- 3           | วิภา       | NULL       | NULL
-- 4           | ประยูร     | NULL       | NULL
-- 5           | กิตติ      | NULL       | NULL
-- 6           | มาลี       | 1          | 2
-- ...

-- ============================================================
-- EXAMPLE 38: COALESCE - จัดการ NULL
-- ============================================================
SELECT 
    first_name,
    manager_id,
    COALESCE(manager_id, 0) AS manager_or_zero,     -- ถ้า NULL ใช้ 0
    COALESCE(manager_id, -1) AS manager_or_minus_one
FROM employees;

-- ============================================================
-- EXAMPLE 39: NULL ใน String Operations
-- ============================================================
SELECT 
    first_name,
    last_name,
    -- ถ้า last_name เป็น NULL, CONCAT จะได้ NULL ใน PostgreSQL
    COALESCE(first_name || ' ' || last_name, first_name) AS full_name
FROM employees;
```

---

## 3.10 SELECT กับ DISTINCT

```sql
-- ============================================================
-- EXAMPLE 40: DISTINCT - ลบ duplicates
-- ============================================================

-- ไม่ใช้ DISTINCT: เห็น duplicates
SELECT job_title FROM employees;

-- ใช้ DISTINCT: เห็นแค่ unique values
SELECT DISTINCT job_title FROM employees;

-- ผลลัพธ์ SELECT DISTINCT:
-- job_title
-- ------------------
-- CFO
-- Content Creator
-- Data Analyst
-- Developer
-- HR Manager
-- HR Specialist
-- Junior Developer
-- Marketing Manager
-- Operations Manager
-- R&D Lead
-- Sales Director
-- Sales Executive
-- Senior Accountant
-- Senior Developer
-- Senior Sales

-- ============================================================
-- EXAMPLE 41: DISTINCT กับหลาย columns
-- ============================================================
SELECT DISTINCT department_id, job_title
FROM employees
ORDER BY department_id, job_title;

-- ผลลัพธ์: combination ที่ unique ของ department_id และ job_title

-- ============================================================
-- EXAMPLE 42: DISTINCT categories ของ products
-- ============================================================
SELECT DISTINCT category
FROM products
ORDER BY category;

-- ผลลัพธ์:
-- category
-- -----------
-- Books
-- Electronics
-- Furniture
-- Stationery
```

---

## 3.11 SELECT กับ CASE Expression

```sql
-- ============================================================
-- EXAMPLE 43: Simple CASE
-- ============================================================
SELECT 
    product_name,
    unit_price,
    CASE category
        WHEN 'Electronics' THEN 'อิเล็กทรอนิกส์'
        WHEN 'Furniture'   THEN 'เฟอร์นิเจอร์'
        WHEN 'Books'       THEN 'หนังสือ'
        WHEN 'Stationery'  THEN 'เครื่องเขียน'
        ELSE 'อื่นๆ'
    END AS category_thai
FROM products;

-- ============================================================
-- EXAMPLE 44: Searched CASE (ยืดหยุ่นกว่า)
-- ============================================================
SELECT 
    first_name,
    salary,
    CASE 
        WHEN salary >= 100000 THEN 'Executive'
        WHEN salary >= 70000  THEN 'Senior'
        WHEN salary >= 50000  THEN 'Mid-level'
        WHEN salary >= 35000  THEN 'Junior'
        ELSE 'Entry Level'
    END AS salary_grade
FROM employees
ORDER BY salary DESC;

-- ผลลัพธ์:
-- first_name | salary    | salary_grade
-- -----------|-----------|-------------
-- วิภา       | 120000.00 | Executive
-- กิตติ      | 95000.00  | Senior
-- สุรชัย     | 85000.00  | Senior
-- สมชาย      | 75000.00  | Senior
-- ไพโรจน์    | 72000.00  | Senior
-- ประยูร     | 70000.00  | Senior
-- พิมพ์ใจ    | 68000.00  | Mid-level
-- สมหญิง     | 65000.00  | Mid-level
-- อนันต์     | 58000.00  | Mid-level
-- ชาญชัย     | 55000.00  | Mid-level
-- จินตนา     | 52000.00  | Mid-level
-- ธนพล      | 48000.00  | Junior
-- มาลี       | 45000.00  | Junior
-- รัตนา      | 42000.00  | Junior
-- นงนุช      | 38000.00  | Mid-level

-- ============================================================
-- EXAMPLE 45: CASE สำหรับ stock status
-- ============================================================
SELECT 
    product_name,
    units_in_stock,
    CASE 
        WHEN units_in_stock = 0      THEN '🔴 Out of Stock'
        WHEN units_in_stock < 20     THEN '🟡 Low Stock'
        WHEN units_in_stock < 50     THEN '🟠 Medium Stock'
        ELSE                              '🟢 In Stock'
    END AS stock_status
FROM products
ORDER BY units_in_stock;
```

---

## 3.12 SELECT ที่ดีมาตรฐาน Professional

```sql
-- ============================================================
-- EXAMPLE 46: Professional Query Format
-- ============================================================

-- ❌ ไม่ดี
select first_name,last_name,salary,department_id from employees where salary>50000 order by salary desc;

-- ✓ ดี - อ่านง่าย มี comments
/*
 * Report: High Salary Employees
 * Purpose: List employees earning over 50,000 THB/month
 * Last updated: 2024-01-15
 */
SELECT 
    employee_id,
    first_name      AS "First Name",
    last_name       AS "Last Name",
    job_title       AS "Position",
    salary          AS "Monthly Salary",
    salary * 12     AS "Annual Salary",
    department_id   AS "Dept ID"
FROM employees
WHERE salary > 50000
ORDER BY salary DESC;

-- ============================================================
-- EXAMPLE 47: Employee Summary Report
-- ============================================================
SELECT 
    -- ข้อมูลพื้นฐาน
    employee_id                             AS id,
    first_name || ' ' || last_name          AS full_name,
    job_title,
    
    -- ข้อมูลการเงิน
    salary                                  AS monthly_salary,
    salary * 12                             AS annual_salary,
    ROUND(salary * 12 * 0.05, 2)           AS provident_fund_annual,
    
    -- Classification
    CASE 
        WHEN salary >= 80000 THEN 'Level 5'
        WHEN salary >= 60000 THEN 'Level 4'
        WHEN salary >= 50000 THEN 'Level 3'
        WHEN salary >= 40000 THEN 'Level 2'
        ELSE                      'Level 1'
    END                                     AS pay_level,
    
    -- Status
    CASE is_active 
        WHEN TRUE  THEN 'Active'
        WHEN FALSE THEN 'Inactive'
    END                                     AS status
    
FROM employees
ORDER BY salary DESC;

-- ============================================================
-- EXAMPLE 48: Product Catalog Report
-- ============================================================
SELECT 
    product_id                              AS "#",
    product_name                            AS "Product Name",
    category                               AS "Category",
    unit_price                             AS "Price (THB)",
    ROUND(unit_price * 1.07, 2)            AS "Price inc. VAT",
    units_in_stock                         AS "In Stock",
    unit_price * units_in_stock            AS "Stock Value",
    
    CASE 
        WHEN discontinued = TRUE THEN 'Discontinued'
        WHEN units_in_stock = 0  THEN 'Out of Stock'
        WHEN units_in_stock < 30 THEN 'Low Stock'
        ELSE                         'Available'
    END                                    AS "Status"
    
FROM products
ORDER BY category, product_name;
```

---

## 3.13 ตัวอย่าง Queries ครบถ้วน 30+ ตัวอย่าง

```sql
-- ============================================================
-- ชุดตัวอย่างครบถ้วน
-- ============================================================

-- 1. ดูพนักงานทั้งหมด
SELECT * FROM employees;

-- 2. ดูชื่อพนักงานทั้งหมด
SELECT first_name, last_name FROM employees;

-- 3. ดูชื่อพนักงานพร้อม full name
SELECT 
    first_name,
    last_name,
    first_name || ' ' || last_name AS full_name
FROM employees;

-- 4. ดูเงินเดือนพนักงานทั้งหมด
SELECT first_name, salary FROM employees;

-- 5. ดูสินค้าทั้งหมด
SELECT * FROM products;

-- 6. ดูสินค้าแค่บางคอลัมน์
SELECT product_name, unit_price FROM products;

-- 7. คำนวณราคารวม VAT ทุกสินค้า
SELECT 
    product_name,
    unit_price,
    unit_price * 1.07 AS price_with_vat
FROM products;

-- 8. ดูแผนกทั้งหมด
SELECT * FROM departments;

-- 9. ดูชื่อแผนกและงบประมาณ
SELECT department_name, budget FROM departments;

-- 10. คำนวณงบประมาณต่อเดือน
SELECT 
    department_name,
    budget,
    budget / 12 AS monthly_budget
FROM departments;

-- 11. ดู unique categories ของสินค้า
SELECT DISTINCT category FROM products;

-- 12. ดู unique job titles
SELECT DISTINCT job_title FROM employees;

-- 13. ดูสินค้ากับสถานะ stock
SELECT 
    product_name,
    units_in_stock,
    CASE 
        WHEN units_in_stock > 100 THEN 'High'
        WHEN units_in_stock > 50  THEN 'Medium'
        ELSE 'Low'
    END AS stock_level
FROM products;

-- 14. แสดงชื่อพนักงานเป็น uppercase
SELECT UPPER(first_name || ' ' || last_name) AS full_name_upper
FROM employees;

-- 15. คำนวณ annual salary
SELECT 
    first_name,
    salary AS monthly,
    salary * 12 AS annual,
    salary * 12 / 365 AS daily_rate
FROM employees;

-- 16. สร้าง greeting message
SELECT 
    'เรียน คุณ ' || first_name || ' ' || last_name AS greeting,
    'ตำแหน่ง: ' || job_title AS position
FROM employees;

-- 17. ดูสินค้าพร้อมมูลค่า inventory
SELECT 
    product_name,
    unit_price,
    units_in_stock,
    unit_price * units_in_stock AS total_value
FROM products;

-- 18. คำนวณ discount price
SELECT 
    product_name,
    unit_price AS original,
    ROUND(unit_price * 0.9, 2) AS after_10pct_discount,
    ROUND(unit_price * 0.8, 2) AS after_20pct_discount
FROM products;

-- 19. ดูพนักงานพร้อม department name แบบง่าย (ใช้ subquery)
SELECT 
    first_name,
    last_name,
    department_id,
    CASE department_id
        WHEN 1 THEN 'IT'
        WHEN 2 THEN 'HR'
        WHEN 3 THEN 'Finance'
        WHEN 4 THEN 'Marketing'
        WHEN 5 THEN 'Sales'
        WHEN 6 THEN 'Operations'
        WHEN 7 THEN 'R&D'
    END AS department_name
FROM employees;

-- 20. ดูข้อมูลสรุปจากค่าคงที่
SELECT 
    'Company Report' AS report_type,
    2024 AS report_year,
    'Q1' AS quarter,
    'January - March 2024' AS period;

-- 21. คำนวณ score percentage
SELECT 
    product_name,
    units_in_stock,
    500 AS max_stock,  -- สมมติ max = 500
    ROUND(units_in_stock * 100.0 / 500, 1) AS stock_pct
FROM products;

-- 22. ดูพนักงานกับ seniority
SELECT 
    first_name,
    hire_date,
    CASE 
        WHEN hire_date < '2018-01-01' THEN 'Senior (7+ years)'
        WHEN hire_date < '2020-01-01' THEN 'Mid-level (5-7 years)'
        WHEN hire_date < '2022-01-01' THEN 'Junior (2-5 years)'
        ELSE 'New hire (<2 years)'
    END AS seniority
FROM employees;

-- 23. Format phone numbers
SELECT 
    first_name,
    phone,
    '(' || SUBSTR(phone, 1, 3) || ') ' || SUBSTR(phone, 5) AS formatted_phone
FROM employees;

-- 24. ดูสินค้าในประเภท Electronics
SELECT product_name, unit_price
FROM products
WHERE category = 'Electronics';

-- 25. ดูพนักงานที่มีเงินเดือนสูงกว่า 60000
SELECT first_name, last_name, salary
FROM employees
WHERE salary > 60000;

-- 26. ดูสินค้าราคาต่ำกว่า 1000
SELECT product_name, unit_price
FROM products
WHERE unit_price < 1000;

-- 27. ดูแผนกที่มีงบประมาณสูงกว่า 5 ล้าน
SELECT department_name, budget
FROM departments
WHERE budget > 5000000;

-- 28. สร้าง email format ใหม่
SELECT 
    first_name,
    last_name,
    email,
    LOWER(first_name) || '.' || LOWER(last_name) || '@newdomain.com' AS new_email
FROM employees;

-- 29. ดู active employees
SELECT first_name, last_name, is_active
FROM employees
WHERE is_active = TRUE;

-- 30. ดู employees ที่มี manager
SELECT first_name, last_name, manager_id
FROM employees
WHERE manager_id IS NOT NULL;

-- 31. ดู employees ที่ไม่มี manager (top level)
SELECT first_name, last_name, job_title
FROM employees
WHERE manager_id IS NULL;

-- 32. คำนวณ total salary cost
SELECT 
    'Total Monthly Salary Cost:' AS label,
    SUM(salary) AS total
FROM employees;

-- 33. หาจำนวนพนักงานทั้งหมด
SELECT COUNT(*) AS total_employees FROM employees;

-- 34. หาเงินเดือนสูงสุดและต่ำสุด
SELECT 
    MAX(salary) AS max_salary,
    MIN(salary) AS min_salary,
    AVG(salary) AS avg_salary
FROM employees;

-- 35. ดูสินค้าที่มีในคลังน้อย (< 50)
SELECT product_name, units_in_stock
FROM products
WHERE units_in_stock < 50
ORDER BY units_in_stock;
```

---

## 3.14 Performance Tip สำหรับ SELECT

```sql
-- ============================================================
-- TIP 1: เลือก columns ที่จำเป็นเท่านั้น
-- ============================================================

-- ❌ ดึงข้อมูลมากเกิน
SELECT * FROM employees;

-- ✓ ดึงแค่ที่ต้องใช้
SELECT employee_id, first_name, last_name
FROM employees;

-- ============================================================
-- TIP 2: LIMIT สำหรับ exploration
-- ============================================================

-- ✓ ดูข้อมูลตัวอย่างก่อน
SELECT * FROM employees LIMIT 10;
SELECT * FROM orders LIMIT 5;

-- ============================================================
-- TIP 3: หลีกเลี่ยง SELECT DISTINCT ถ้าไม่จำเป็น
-- ============================================================

-- SELECT DISTINCT ต้อง sort ข้อมูลก่อน → ช้ากว่า
-- ถ้าต้องการ counts ให้ใช้ GROUP BY แทน

-- ❌ สำหรับ counting
SELECT DISTINCT department_id FROM employees;
-- แล้วนับ manually

-- ✓ ดีกว่า
SELECT department_id, COUNT(*) AS headcount
FROM employees
GROUP BY department_id;
```

---

## 3.15 สรุปบทที่ 3

```
✅ SELECT syntax: SELECT columns FROM table
✅ SELECT * ดึงทุก column
✅ ระบุ columns เฉพาะที่ต้องการ
✅ Expressions: คำนวณ, string concatenation
✅ Literals: ค่าคงที่ใน SELECT
✅ DISTINCT: ลบ duplicates
✅ CASE: conditional expressions
✅ NULL handling พื้นฐาน
✅ Professional formatting
```

---

## 📝 แบบฝึกหัดท้ายบท

**คำถาม 1:** เขียน query เพื่อแสดง `product_name`, `unit_price`, และ `unit_price * 1.07` (ราคา + VAT) จากตาราง `products`

**คำถาม 2:** เขียน query เพื่อแสดงชื่อเต็ม (first_name + space + last_name) ของพนักงานทุกคน ในคอลัมน์ชื่อ `full_name`

**คำถาม 3:** เขียน query เพื่อดู unique categories ของสินค้าทั้งหมด

**คำถาม 4:** เขียน query เพื่อแสดงพนักงานทุกคน พร้อมคอลัมน์ `pay_level` ที่:
- salary >= 80000 → 'Senior Executive'
- salary >= 60000 → 'Senior'
- salary >= 40000 → 'Mid-level'
- อื่นๆ → 'Junior'

**คำถาม 5:** เขียน query เพื่อคำนวณมูลค่า inventory ทั้งหมด (unit_price * units_in_stock) ของสินค้าแต่ละชิ้น

**คำถาม 6:** เขียน query ที่แสดง: `product_name`, `category` (ภาษาไทย), `unit_price` จากตาราง products

**คำถาม 7:** แสดงพนักงานพร้อม `annual_salary` (salary * 12) และ `provident_fund` (5% ของ annual)

**คำถาม 8:** แสดง `department_name` และ `budget_in_millions` (budget / 1000000) จาก departments

**คำถาม 9:** ใช้ SUBSTR เพื่อดึง username จาก email (ส่วนก่อน @)

**คำถาม 10:** สร้าง query ที่แสดง employee report สวยงาม มี: id, full_name, job_title, monthly_salary, annual_salary, pay_level

---

## ✅ เฉลยแบบฝึกหัด

### เฉลยที่ 1:
```sql
SELECT 
    product_name,
    unit_price,
    ROUND(unit_price * 1.07, 2) AS price_with_vat
FROM products;
```

### เฉลยที่ 2:
```sql
-- SQLite/PostgreSQL
SELECT first_name || ' ' || last_name AS full_name
FROM employees;

-- MySQL
SELECT CONCAT(first_name, ' ', last_name) AS full_name
FROM employees;
```

### เฉลยที่ 3:
```sql
SELECT DISTINCT category
FROM products
ORDER BY category;
```

### เฉลยที่ 4:
```sql
SELECT 
    first_name,
    last_name,
    salary,
    CASE 
        WHEN salary >= 80000 THEN 'Senior Executive'
        WHEN salary >= 60000 THEN 'Senior'
        WHEN salary >= 40000 THEN 'Mid-level'
        ELSE 'Junior'
    END AS pay_level
FROM employees
ORDER BY salary DESC;
```

### เฉลยที่ 5:
```sql
SELECT 
    product_name,
    unit_price,
    units_in_stock,
    unit_price * units_in_stock AS inventory_value
FROM products
ORDER BY inventory_value DESC;
```

### เฉลยที่ 6:
```sql
SELECT 
    product_name,
    CASE category
        WHEN 'Electronics' THEN 'อิเล็กทรอนิกส์'
        WHEN 'Furniture'   THEN 'เฟอร์นิเจอร์'
        WHEN 'Books'       THEN 'หนังสือ'
        WHEN 'Stationery'  THEN 'เครื่องเขียน'
        ELSE 'อื่นๆ'
    END AS category_thai,
    unit_price
FROM products;
```

### เฉลยที่ 7:
```sql
SELECT 
    first_name,
    last_name,
    salary AS monthly_salary,
    salary * 12 AS annual_salary,
    ROUND(salary * 12 * 0.05, 2) AS provident_fund
FROM employees
ORDER BY salary DESC;
```

### เฉลยที่ 8:
```sql
SELECT 
    department_name,
    budget,
    ROUND(budget / 1000000.0, 2) AS budget_in_millions
FROM departments
ORDER BY budget DESC;
```

### เฉลยที่ 9:
```sql
-- SQLite
SELECT 
    first_name,
    email,
    SUBSTR(email, 1, INSTR(email, '@') - 1) AS username
FROM employees;

-- PostgreSQL
SELECT 
    first_name,
    email,
    SPLIT_PART(email, '@', 1) AS username
FROM employees;
```

### เฉลยที่ 10:
```sql
SELECT 
    employee_id AS id,
    first_name || ' ' || last_name AS full_name,
    job_title,
    salary AS monthly_salary,
    salary * 12 AS annual_salary,
    CASE 
        WHEN salary >= 80000 THEN 'Level 5 - Senior Executive'
        WHEN salary >= 60000 THEN 'Level 4 - Senior'
        WHEN salary >= 50000 THEN 'Level 3 - Mid-level'
        WHEN salary >= 40000 THEN 'Level 2 - Junior'
        ELSE 'Level 1 - Entry'
    END AS pay_level
FROM employees
ORDER BY salary DESC;
```

---

## ➡️ บทถัดไป

**[Part 004: Filtering Data with WHERE Clause](part-004.md)**

ในบทถัดไปเราจะเรียนรู้การกรองข้อมูลด้วย WHERE clause และ operators ต่างๆ!

---

*Part 003 of 120 | หลักสูตร SQL ครบวงจร*
