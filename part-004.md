# Part 004: Filtering Data with WHERE Clause

> **หลักสูตร SQL ครบวงจร | Part 4 of 120**

---

## 🎯 สิ่งที่จะได้เรียนรู้ในบทนี้

- WHERE clause syntax และการทำงาน
- Comparison operators ทุกชนิด (=, <>, !=, <, >, <=, >=)
- BETWEEN operator
- IN operator
- LIKE operator และ wildcards (% และ _)
- IS NULL และ IS NOT NULL
- Logical operators: AND, OR, NOT
- การรวม conditions หลายอัน
- Operator precedence
- สถานการณ์จริงในการใช้ WHERE

**เวลาที่ใช้เรียน**: ประมาณ 3-4 ชั่วโมง

---

## ฐานข้อมูลที่ใช้ในบทนี้

```sql
-- สร้างและเติมข้อมูลตาราง (ถ้ายังไม่มี)
-- ดู script เต็มได้ที่ Part 001 หรือ Part 002

-- ตารางที่ใช้:
-- employees, departments, products, orders, customers

-- ยืนยันข้อมูล
SELECT COUNT(*) FROM employees;    -- 15
SELECT COUNT(*) FROM products;     -- 15
SELECT COUNT(*) FROM departments;  -- 7
SELECT COUNT(*) FROM orders;       -- 10
SELECT COUNT(*) FROM customers;    -- 10
```

---

## 4.1 WHERE Clause คืออะไร

WHERE clause ใช้กรอง (filter) แถวข้อมูลที่ต้องการ

```
ลำดับการประมวลผล:
1. FROM - เลือกตาราง
2. WHERE - กรองแถว ← ทำงานตรงนี้
3. SELECT - เลือก columns
4. ORDER BY - จัดเรียง

ไม่ใช้ WHERE → ได้ทุกแถว
ใช้ WHERE → ได้แถวที่ตรงตาม condition
```

### Syntax พื้นฐาน

```sql
SELECT column1, column2, ...
FROM table_name
WHERE condition;

-- condition ต้องเป็น TRUE, FALSE, หรือ UNKNOWN (NULL)
-- แถวที่ WHERE = TRUE จะถูกเลือก
-- แถวที่ WHERE = FALSE หรือ NULL จะไม่ถูกเลือก
```

---

## 4.2 Comparison Operators

### = (Equal To)

```sql
-- ============================================================
-- EXAMPLE 1: Equal (=)
-- ============================================================

-- หาพนักงานในแผนก IT (department_id = 1)
SELECT first_name, last_name, department_id
FROM employees
WHERE department_id = 1;

-- ผลลัพธ์:
-- first_name | last_name    | department_id
-- -----------|--------------|---------------
-- สมชาย      | นักเขียน     | 1
-- มาลี       | สวยงาม       | 1
-- อนันต์     | ใจดี         | 1
-- จินตนา     | คิดเก่ง      | 1

-- หาสินค้าในหมวด Electronics
SELECT product_name, unit_price
FROM products
WHERE category = 'Electronics';

-- ผลลัพธ์:
-- product_name                  | unit_price
-- ------------------------------|----------
-- Laptop Pro 15"                | 45000.00
-- Wireless Mouse                | 850.00
-- USB-C Hub                     | 2500.00
-- Noise Cancelling Headphone    | 7500.00
-- Monitor 27" 4K                | 18000.00
-- Keyboard Mechanical           | 3200.00
-- Webcam HD                     | 2800.00

-- หาคำสั่งซื้อที่ status เป็น 'Delivered'
SELECT order_id, customer_id, order_date, total_amount
FROM orders
WHERE status = 'Delivered';

-- ============================================================
-- EXAMPLE 2: = กับ NULL (อย่าทำ!)
-- ============================================================

-- ❌ ผิด! NULL = NULL ไม่เป็น TRUE ใน SQL
SELECT * FROM employees WHERE manager_id = NULL;
-- ผลลัพธ์: 0 rows (ไม่ถูกต้อง!)

-- ✓ ถูก! ต้องใช้ IS NULL
SELECT * FROM employees WHERE manager_id IS NULL;
```

### <> และ != (Not Equal)

```sql
-- ============================================================
-- EXAMPLE 3: Not Equal (<> หรือ !=)
-- ============================================================

-- หาสินค้าที่ไม่ใช่ Electronics
SELECT product_name, category
FROM products
WHERE category <> 'Electronics';

-- ผลลัพธ์:
-- product_name              | category
-- --------------------------|----------
-- Office Chair Premium      | Furniture
-- Standing Desk             | Furniture
-- SQL Programming Book      | Books
-- Python for Data Science   | Books
-- Desk Lamp LED             | Furniture
-- Whiteboard A4             | Stationery
-- Notebook Premium          | Stationery
-- Pen Set                   | Stationery

-- != เป็นแบบเดียวกัน (standard น้อยกว่า แต่ใช้ได้)
SELECT product_name, category
FROM products
WHERE category != 'Electronics';

-- หาพนักงานที่ไม่ได้อยู่แผนก Sales (5)
SELECT first_name, last_name, department_id
FROM employees
WHERE department_id <> 5;

-- หา orders ที่ยังไม่ cancelled
SELECT order_id, status, total_amount
FROM orders
WHERE status <> 'Cancelled';
```

### < , > , <= , >= (Comparison)

```sql
-- ============================================================
-- EXAMPLE 4: Less Than (<)
-- ============================================================

-- สินค้าราคาต่ำกว่า 1,000 บาท
SELECT product_name, unit_price
FROM products
WHERE unit_price < 1000;

-- ผลลัพธ์:
-- product_name            | unit_price
-- ------------------------|----------
-- Wireless Mouse          | 850.00
-- SQL Programming Book    | 450.00
-- Python for Data Science | 550.00
-- Whiteboard A4           | 150.00
-- Notebook Premium        | 250.00
-- Pen Set                 | 120.00

-- ============================================================
-- EXAMPLE 5: Greater Than (>)
-- ============================================================

-- พนักงานที่เงินเดือนมากกว่า 70,000
SELECT first_name, last_name, salary
FROM employees
WHERE salary > 70000;

-- ผลลัพธ์:
-- first_name | last_name  | salary
-- -----------|------------|----------
-- วิภา       | รักงาน     | 120000.00
-- กิตติ      | เก่งมาก    | 95000.00
-- สุรชัย     | เด่นมาก    | 85000.00
-- สมชาย      | นักเขียน   | 75000.00
-- ไพโรจน์    | กว้างขวาง  | 72000.00

-- ============================================================
-- EXAMPLE 6: Less Than or Equal (<=)
-- ============================================================

-- สินค้าที่ราคา 1,000 บาทหรือน้อยกว่า
SELECT product_name, unit_price
FROM products
WHERE unit_price <= 1000;

-- ผลลัพธ์ (รวม 1000 ด้วย):
-- product_name            | unit_price
-- ------------------------|----------
-- Wireless Mouse          | 850.00
-- SQL Programming Book    | 450.00
-- Python for Data Science | 550.00
-- Whiteboard A4           | 150.00
-- Notebook Premium        | 250.00
-- Pen Set                 | 120.00

-- ============================================================
-- EXAMPLE 7: Greater Than or Equal (>=)
-- ============================================================

-- สินค้าที่มีในคลัง 100 ชิ้นขึ้นไป
SELECT product_name, units_in_stock
FROM products
WHERE units_in_stock >= 100;

-- ผลลัพธ์:
-- product_name            | units_in_stock
-- ------------------------|---------------
-- Wireless Mouse          | 150
-- SQL Programming Book    | 200
-- Python for Data Science | 180
-- Desk Lamp LED           | 100
-- Whiteboard A4           | 500
-- Notebook Premium        | 300
-- Pen Set                 | 400

-- ============================================================
-- EXAMPLE 8: Comparison กับ Text (Alphabetical)
-- ============================================================

-- หาสินค้าที่ชื่อมาก่อน 'M' ตามตัวอักษร
SELECT product_name
FROM products
WHERE product_name < 'M'
ORDER BY product_name;

-- หาแผนกที่ชื่อมากกว่า 'Marketing' ตามตัวอักษร
SELECT department_name
FROM departments
WHERE department_name > 'Marketing'
ORDER BY department_name;

-- ============================================================
-- EXAMPLE 9: Comparison กับ Dates
-- ============================================================

-- พนักงานที่เข้าทำงานหลัง 2020
SELECT first_name, last_name, hire_date
FROM employees
WHERE hire_date > '2020-01-01'
ORDER BY hire_date;

-- ผลลัพธ์:
-- first_name | last_name | hire_date
-- -----------|-----------|----------
-- มาลี       | สวยงาม    | 2021-02-14
-- ธนพล      | รวยแน่    | 2021-06-15
-- พิมพ์ใจ    | หวานใจ    | 2020-09-01
-- นงนุช      | น่ารักมาก | 2022-01-03
-- จินตนา     | คิดเก่ง   | 2023-03-01

-- พนักงานที่เข้าทำงานก่อน 2018
SELECT first_name, last_name, hire_date
FROM employees
WHERE hire_date < '2018-01-01';
```

---

## 4.3 BETWEEN Operator

```sql
-- ============================================================
-- EXAMPLE 10: BETWEEN สำหรับ Numbers
-- ============================================================

-- สินค้าราคาระหว่าง 1,000 ถึง 5,000 บาท
SELECT product_name, unit_price
FROM products
WHERE unit_price BETWEEN 1000 AND 5000;

-- ผลลัพธ์:
-- product_name        | unit_price
-- --------------------|----------
-- USB-C Hub           | 2500.00
-- Keyboard Mechanical | 3200.00
-- Webcam HD           | 2800.00
-- Desk Lamp LED       | 1200.00

-- BETWEEN รวม boundary values (inclusive)
-- WHERE unit_price BETWEEN 1000 AND 5000
-- = WHERE unit_price >= 1000 AND unit_price <= 5000

-- พนักงานที่เงินเดือนระหว่าง 50,000 ถึง 80,000
SELECT first_name, last_name, salary
FROM employees
WHERE salary BETWEEN 50000 AND 80000
ORDER BY salary;

-- ============================================================
-- EXAMPLE 11: NOT BETWEEN
-- ============================================================

-- สินค้าที่ราคาไม่อยู่ระหว่าง 1,000 ถึง 10,000
SELECT product_name, unit_price
FROM products
WHERE unit_price NOT BETWEEN 1000 AND 10000;

-- ผลลัพธ์:
-- product_name              | unit_price
-- --------------------------|----------
-- Laptop Pro 15"            | 45000.00
-- Wireless Mouse            | 850.00
-- SQL Programming Book      | 450.00
-- Python for Data Science   | 550.00
-- Monitor 27" 4K            | 18000.00
-- Standing Desk             | 15000.00
-- Whiteboard A4             | 150.00
-- Notebook Premium          | 250.00
-- Pen Set                   | 120.00

-- ============================================================
-- EXAMPLE 12: BETWEEN สำหรับ Dates
-- ============================================================

-- พนักงานที่เข้าทำงานระหว่างปี 2018-2020
SELECT first_name, last_name, hire_date
FROM employees
WHERE hire_date BETWEEN '2018-01-01' AND '2020-12-31'
ORDER BY hire_date;

-- คำสั่งซื้อในเดือนมกราคม 2024
SELECT order_id, order_date, total_amount
FROM orders
WHERE order_date BETWEEN '2024-01-01' AND '2024-01-31';

-- ============================================================
-- EXAMPLE 13: BETWEEN กับ Text
-- ============================================================

-- สินค้าที่ชื่อขึ้นต้นด้วย A-M (ตามตัวอักษร)
SELECT product_name
FROM products
WHERE product_name BETWEEN 'A' AND 'N'
ORDER BY product_name;
```

---

## 4.4 IN Operator

```sql
-- ============================================================
-- EXAMPLE 14: IN กับ Numbers
-- ============================================================

-- พนักงานในแผนก IT (1), HR (2), หรือ Finance (3)
SELECT first_name, last_name, department_id
FROM employees
WHERE department_id IN (1, 2, 3)
ORDER BY department_id, first_name;

-- เทียบกับ OR:
-- WHERE department_id = 1 OR department_id = 2 OR department_id = 3
-- IN สั้นกว่าและอ่านง่ายกว่า!

-- ผลลัพธ์:
-- first_name | last_name    | department_id
-- -----------|--------------|---------------
-- จินตนา     | คิดเก่ง      | 1
-- มาลี       | สวยงาม       | 1
-- สมชาย      | นักเขียน     | 1
-- อนันต์     | ใจดี         | 1
-- รัตนา      | ขยันมาก      | 2
-- สมหญิง     | ดีงาม        | 2
-- ชาญชัย    | ฉลาดเฉลียว   | 3
-- วิภา       | รักงาน       | 3

-- ============================================================
-- EXAMPLE 15: IN กับ Text
-- ============================================================

-- สินค้าในหมวด Electronics และ Furniture
SELECT product_name, category, unit_price
FROM products
WHERE category IN ('Electronics', 'Furniture')
ORDER BY category, product_name;

-- ผลลัพธ์:
-- product_name                  | category    | unit_price
-- ------------------------------|-------------|----------
-- Keyboard Mechanical           | Electronics | 3200.00
-- Laptop Pro 15"                | Electronics | 45000.00
-- Monitor 27" 4K                | Electronics | 18000.00
-- Noise Cancelling Headphone    | Electronics | 7500.00
-- USB-C Hub                     | Electronics | 2500.00
-- Webcam HD                     | Electronics | 2800.00
-- Wireless Mouse                | Electronics | 850.00
-- Desk Lamp LED                 | Furniture   | 1200.00
-- Office Chair Premium          | Furniture   | 8900.00
-- Standing Desk                 | Furniture   | 15000.00

-- ============================================================
-- EXAMPLE 16: NOT IN
-- ============================================================

-- สินค้าที่ไม่ใช่ Electronics และ Furniture
SELECT product_name, category
FROM products
WHERE category NOT IN ('Electronics', 'Furniture');

-- ผลลัพธ์:
-- product_name              | category
-- --------------------------|----------
-- SQL Programming Book      | Books
-- Python for Data Science   | Books
-- Whiteboard A4             | Stationery
-- Notebook Premium          | Stationery
-- Pen Set                   | Stationery

-- ============================================================
-- EXAMPLE 17: IN กับ Status
-- ============================================================

-- คำสั่งซื้อที่ยังดำเนินการอยู่ (ไม่ cancelled)
SELECT order_id, status, total_amount
FROM orders
WHERE status IN ('Pending', 'Processing', 'Delivered');

-- คำสั่งซื้อที่ยังไม่เสร็จ
SELECT order_id, status, order_date
FROM orders
WHERE status IN ('Pending', 'Processing');

-- ============================================================
-- EXAMPLE 18: IN กับ Subquery (Preview - จะเรียนในบทหลัง)
-- ============================================================

-- หาพนักงานที่เป็นหัวหน้าแผนก
SELECT first_name, last_name, employee_id
FROM employees
WHERE employee_id IN (
    SELECT manager_id FROM departments
    WHERE manager_id IS NOT NULL
);

-- ============================================================
-- EXAMPLE 19: IN กับ NULL (ข้อควรระวัง!)
-- ============================================================

-- ถ้ามี NULL ใน IN list อาจให้ผลไม่คาดคิด
-- โดยเฉพาะ NOT IN

-- ตัวอย่างที่อาจพลาด:
-- WHERE manager_id NOT IN (1, 2, NULL)
-- ผลลัพธ์: 0 rows! เพราะ NULL ใน list ทำให้ทุกอย่างเป็น NULL

-- ✓ ถูก: ถ้าต้องการรวม NULL check
SELECT first_name, manager_id
FROM employees
WHERE manager_id NOT IN (1, 2)
  OR manager_id IS NULL;
```

---

## 4.5 LIKE Operator และ Wildcards

```sql
-- ============================================================
-- EXAMPLE 20: LIKE กับ % (matches any sequence)
-- ============================================================

-- % = matches ตัวอักษรกี่ตัวก็ได้ (รวมถึง 0 ตัว)

-- สินค้าที่ชื่อขึ้นต้นด้วย 'W'
SELECT product_name FROM products
WHERE product_name LIKE 'W%';
-- ผลลัพธ์: Wireless Mouse, Webcam HD, Whiteboard A4

-- สินค้าที่ชื่อลงท้ายด้วย 'k'
SELECT product_name FROM products
WHERE product_name LIKE '%k';
-- ผลลัพธ์: SQL Programming Book, Keyboard Mechanical, Notebook Premium, Desk Lamp

-- Wait... Desk Lamp ไม่ลงท้าย k... ลองเช็คอีกครั้ง:
SELECT product_name FROM products
WHERE product_name LIKE '%Book';
-- ผลลัพธ์: SQL Programming Book, Python for Data Science... wait

-- สินค้าที่มีคำว่า 'Pro' ในชื่อ
SELECT product_name FROM products
WHERE product_name LIKE '%Pro%';
-- ผลลัพธ์: Laptop Pro 15"

-- ============================================================
-- EXAMPLE 21: LIKE กับ _ (matches exactly one character)
-- ============================================================

-- _ = matches ตัวอักษรหนึ่งตัวพอดี

-- หา email ที่ username เป็น 4 ตัวอักษรก่อน .
SELECT email FROM employees
WHERE email LIKE '____%.%';  -- 4 ตัวอักษร แล้วตามด้วยอะไรก็ได้ จุด แล้วอะไรก็ได้

-- ============================================================
-- EXAMPLE 22: Pattern Matching ตัวอย่างจริง
-- ============================================================

-- หา email ทุกอัน
SELECT first_name, email FROM employees
WHERE email LIKE '%@company.com';

-- หาพนักงานที่ชื่อขึ้นต้นด้วย 'ส'
SELECT first_name, last_name FROM employees
WHERE first_name LIKE 'ส%';

-- ผลลัพธ์:
-- สมชาย | นักเขียน
-- สมหญิง | ดีงาม
-- สุรชัย | เด่นมาก

-- หาสินค้าที่มีตัวเลขในชื่อ
SELECT product_name FROM products
WHERE product_name LIKE '%[0-9]%';  -- SQL Server
-- หรือ
SELECT product_name FROM products
WHERE product_name GLOB '*[0-9]*';   -- SQLite

-- ============================================================
-- EXAMPLE 23: LIKE Case Sensitivity
-- ============================================================

-- PostgreSQL: LIKE เป็น case-sensitive
-- MySQL: LIKE ไม่ case-sensitive (โดย default)
-- SQLite: LIKE ไม่ case-sensitive สำหรับ ASCII

-- PostgreSQL case-insensitive ใช้ ILIKE
SELECT product_name FROM products
WHERE product_name ILIKE '%book%';  -- PostgreSQL only

-- ทุก database ใช้ UPPER/LOWER
SELECT product_name FROM products
WHERE UPPER(product_name) LIKE UPPER('%book%');
-- หรือ
WHERE LOWER(product_name) LIKE '%book%';

-- ============================================================
-- EXAMPLE 24: NOT LIKE
-- ============================================================

-- สินค้าที่ไม่ใช่ชื่อขึ้นต้น 'W' หรือ 'L'
SELECT product_name FROM products
WHERE product_name NOT LIKE 'W%'
  AND product_name NOT LIKE 'L%';

-- ============================================================
-- EXAMPLE 25: LIKE Escape Character
-- ============================================================

-- ถ้าต้องการหา literal % หรือ _ ต้องใช้ ESCAPE

-- หา string ที่มี % อยู่จริงๆ
SELECT * FROM some_table
WHERE column_name LIKE '100\%' ESCAPE '\';
-- หรือใน PostgreSQL
SELECT * FROM some_table
WHERE column_name LIKE '100!%' ESCAPE '!';

-- ============================================================
-- EXAMPLE 26: Pattern Matching ในสถานการณ์จริง
-- ============================================================

-- ค้นหาลูกค้าตามชื่อ (แบบ search)
SELECT first_name, last_name, email
FROM customers
WHERE first_name LIKE '%ภา%'
   OR last_name LIKE '%ภา%';

-- หา products ในหมวดหมู่ตาม keyword
SELECT product_name, category
FROM products
WHERE product_name LIKE '%Monitor%'
   OR product_name LIKE '%Screen%'
   OR product_name LIKE '%Display%';

-- หา email domains ต่างๆ
SELECT 
    email,
    CASE 
        WHEN email LIKE '%@gmail.com' THEN 'Gmail'
        WHEN email LIKE '%@company.com' THEN 'Company'
        ELSE 'Other'
    END AS email_provider
FROM employees;
```

---

## 4.6 IS NULL และ IS NOT NULL

```sql
-- ============================================================
-- EXAMPLE 27: IS NULL
-- ============================================================

-- พนักงานที่ไม่มี manager (top-level)
SELECT first_name, last_name, manager_id
FROM employees
WHERE manager_id IS NULL;

-- ผลลัพธ์:
-- first_name | last_name    | manager_id
-- -----------|--------------|----------
-- สมชาย      | นักเขียน     | NULL
-- สมหญิง     | ดีงาม        | NULL
-- วิภา       | รักงาน       | NULL
-- ประยูร     | มีสุข        | NULL
-- กิตติ      | เก่งมาก      | NULL
-- พิมพ์ใจ    | หวานใจ       | NULL
-- สุรชัย     | เด่นมาก      | NULL

-- ลูกค้าที่ไม่มีชื่อบริษัท (individual customers)
SELECT first_name, last_name, company_name
FROM customers
WHERE company_name IS NULL;

-- ผลลัพธ์:
-- สุภา | แก้วใส | NULL
-- กนกวรรณ | มีมาก | NULL

-- ============================================================
-- EXAMPLE 28: IS NOT NULL
-- ============================================================

-- พนักงานที่มี manager
SELECT first_name, last_name, manager_id
FROM employees
WHERE manager_id IS NOT NULL
ORDER BY manager_id;

-- ผลลัพธ์:
-- first_name | last_name    | manager_id
-- -----------|--------------|----------
-- มาลี       | สวยงาม       | 1
-- อนันต์     | ใจดี         | 1
-- จินตนา     | คิดเก่ง      | 1
-- รัตนา      | ขยันมาก      | 2
-- ชาญชัย    | ฉลาดเฉลียว   | 3
-- นงนุช      | น่ารักมาก    | 4
-- ธนพล      | รวยแน่       | 5
-- ไพโรจน์    | กว้างขวาง    | 5

-- คำสั่งซื้อที่ถูก ship แล้ว
SELECT order_id, order_date, shipped_date, status
FROM orders
WHERE shipped_date IS NOT NULL;

-- คำสั่งซื้อที่ยังไม่ถูก ship
SELECT order_id, order_date, status
FROM orders
WHERE shipped_date IS NULL;

-- ============================================================
-- EXAMPLE 29: NULL ใน Calculations
-- ============================================================

-- NULL ในการคำนวณ = NULL เสมอ
SELECT 
    employee_id,
    first_name,
    manager_id,
    manager_id + 100 AS manager_plus_100  -- NULL ถ้า manager_id เป็น NULL
FROM employees;

-- ตรวจสอบ: NULL = NULL?
SELECT NULL = NULL;    -- NULL (not TRUE!)
SELECT NULL <> NULL;   -- NULL (not TRUE!)
SELECT NULL IS NULL;   -- TRUE

-- ============================================================
-- EXAMPLE 30: Handling NULL ด้วย COALESCE
-- ============================================================

-- แทน NULL ด้วยค่า default
SELECT 
    first_name,
    COALESCE(manager_id, 0) AS manager_id_safe,
    CASE WHEN manager_id IS NULL 
         THEN 'No Manager' 
         ELSE 'Has Manager' 
    END AS manager_status
FROM employees;
```

---

## 4.7 Logical Operators: AND, OR, NOT

```sql
-- ============================================================
-- EXAMPLE 31: AND - ทั้งสอง condition ต้องจริง
-- ============================================================

-- พนักงานในแผนก IT ที่เงินเดือนมากกว่า 50,000
SELECT first_name, last_name, salary, department_id
FROM employees
WHERE department_id = 1
  AND salary > 50000;

-- ผลลัพธ์:
-- first_name | last_name | salary  | department_id
-- -----------|-----------|---------|---------------
-- สมชาย      | นักเขียน  | 75000   | 1
-- อนันต์     | ใจดี      | 58000   | 1

-- สินค้า Electronics ที่ราคาต่ำกว่า 5,000
SELECT product_name, category, unit_price
FROM products
WHERE category = 'Electronics'
  AND unit_price < 5000;

-- ผลลัพธ์:
-- product_name        | category    | unit_price
-- --------------------|-------------|----------
-- Wireless Mouse      | Electronics | 850.00
-- USB-C Hub           | Electronics | 2500.00
-- Keyboard Mechanical | Electronics | 3200.00
-- Webcam HD           | Electronics | 2800.00

-- ============================================================
-- EXAMPLE 32: OR - อย่างน้อยหนึ่ง condition จริง
-- ============================================================

-- พนักงานในแผนก IT หรือ Sales
SELECT first_name, last_name, department_id
FROM employees
WHERE department_id = 1
   OR department_id = 5;

-- ผลลัพธ์:
-- first_name | last_name    | department_id
-- -----------|--------------|---------------
-- สมชาย      | นักเขียน     | 1
-- กิตติ      | เก่งมาก      | 5
-- มาลี       | สวยงาม       | 1
-- อนันต์     | ใจดี         | 1
-- ธนพล      | รวยแน่       | 5
-- จินตนา     | คิดเก่ง      | 1
-- ไพโรจน์    | กว้างขวาง    | 5

-- สินค้าที่ราคาต่ำกว่า 500 หรือสูงกว่า 10,000
SELECT product_name, unit_price
FROM products
WHERE unit_price < 500
   OR unit_price > 10000
ORDER BY unit_price;

-- ============================================================
-- EXAMPLE 33: NOT - กลับ condition
-- ============================================================

-- พนักงานที่ไม่ได้อยู่แผนก IT (department_id != 1)
SELECT first_name, last_name, department_id
FROM employees
WHERE NOT department_id = 1;

-- เทียบกับ
SELECT first_name, last_name, department_id
FROM employees
WHERE department_id <> 1;
-- ผลลัพธ์เหมือนกัน

-- สินค้าที่ไม่ใช่ Electronics
SELECT product_name, category
FROM products
WHERE NOT category = 'Electronics';

-- ============================================================
-- EXAMPLE 34: รวม AND, OR, NOT
-- ============================================================

-- พนักงานในแผนก IT หรือ Sales ที่เงินเดือนมากกว่า 60,000
SELECT first_name, last_name, salary, department_id
FROM employees
WHERE (department_id = 1 OR department_id = 5)
  AND salary > 60000;

-- ผลลัพธ์:
-- first_name | last_name    | salary   | department_id
-- -----------|--------------|----------|---------------
-- สมชาย      | นักเขียน     | 75000.00 | 1
-- กิตติ      | เก่งมาก      | 95000.00 | 5
-- ไพโรจน์    | กว้างขวาง    | 72000.00 | 5
```

---

## 4.8 Operator Precedence (ลำดับความสำคัญ)

```sql
-- ============================================================
-- ลำดับความสำคัญ (จากสูงไปต่ำ)
-- ============================================================
-- 1. ()  - วงเล็บ (สูงสุด)
-- 2. NOT
-- 3. AND
-- 4. OR  (ต่ำสุด)

-- ============================================================
-- EXAMPLE 35: Precedence ปัญหาที่พบบ่อย
-- ============================================================

-- ❌ ผิด: AND ทำก่อน OR
-- ต้องการ: "พนักงานในแผนก IT หรือ Finance ที่เงินเดือนมากกว่า 60000"
SELECT first_name, salary, department_id
FROM employees
WHERE department_id = 1 OR department_id = 3 AND salary > 60000;
-- ได้: department_id = 1 (ทุกคน) OR (department_id = 3 AND salary > 60000)

-- ✓ ถูก: ใส่วงเล็บเพื่อความชัดเจน
SELECT first_name, salary, department_id
FROM employees
WHERE (department_id = 1 OR department_id = 3) AND salary > 60000;

-- ============================================================
-- EXAMPLE 36: ตัวอย่าง Precedence ที่ซับซ้อน
-- ============================================================

-- ต้องการ: สินค้า Electronics ราคา < 5000 หรือ สินค้า Furniture ราคา < 20000
-- ❌ ไม่มีวงเล็บ (อาจให้ผลผิด)
SELECT product_name, category, unit_price
FROM products
WHERE category = 'Electronics' AND unit_price < 5000
   OR category = 'Furniture' AND unit_price < 20000;
-- คำตอบนี้จริงๆ ถูก เพราะ AND ทำก่อน OR
-- แต่อ่านยาก

-- ✓ มีวงเล็บ (ชัดเจน)
SELECT product_name, category, unit_price
FROM products
WHERE (category = 'Electronics' AND unit_price < 5000)
   OR (category = 'Furniture'   AND unit_price < 20000)
ORDER BY category, unit_price;

-- ============================================================
-- EXAMPLE 37: NOT กับ AND/OR
-- ============================================================

-- ต้องการ: ไม่ใช่ Electronics และไม่ใช่ Furniture
-- ❌ อาจสับสน
SELECT product_name, category
FROM products
WHERE NOT category = 'Electronics' AND NOT category = 'Furniture';

-- ✓ ชัดเจนกว่า (De Morgan's Law)
SELECT product_name, category
FROM products
WHERE NOT (category = 'Electronics' OR category = 'Furniture');

-- ผลลัพธ์เหมือนกัน แต่อ่านง่ายกว่า:
-- product_name              | category
-- --------------------------|----------
-- SQL Programming Book      | Books
-- Python for Data Science   | Books
-- Whiteboard A4             | Stationery
-- Notebook Premium          | Stationery
-- Pen Set                   | Stationery
```

---

## 4.9 Complex WHERE Conditions

```sql
-- ============================================================
-- EXAMPLE 38: Complex Business Rules
-- ============================================================

-- หาสินค้า "Hot" ที่ควรโปรโมท:
-- Electronics ราคา 1000-10000 ที่มีในคลัง 50+ ชิ้น
SELECT product_name, category, unit_price, units_in_stock
FROM products
WHERE category = 'Electronics'
  AND unit_price BETWEEN 1000 AND 10000
  AND units_in_stock >= 50
ORDER BY unit_price;

-- ผลลัพธ์:
-- product_name        | category    | unit_price | units_in_stock
-- --------------------|-------------|------------|---------------
-- USB-C Hub           | Electronics | 2500.00    | 80
-- Keyboard Mechanical | Electronics | 3200.00    | 60

-- ============================================================
-- EXAMPLE 39: หา employees ที่ต้อง promote
-- ============================================================

-- พนักงานที่ทำงานมาก่อน 2020 และเงินเดือนยังต่ำกว่า 60000
SELECT 
    first_name || ' ' || last_name AS name,
    hire_date,
    salary,
    job_title
FROM employees
WHERE hire_date < '2020-01-01'
  AND salary < 60000
ORDER BY hire_date;

-- ============================================================
-- EXAMPLE 40: หา orders ที่ต้องติดตาม
-- ============================================================

-- Orders ที่ยังไม่ ship และเลยวันกำหนดส่งแล้ว
SELECT 
    order_id,
    customer_id,
    order_date,
    required_date,
    status
FROM orders
WHERE shipped_date IS NULL
  AND status NOT IN ('Cancelled')
  AND required_date < '2024-03-01'
ORDER BY required_date;

-- ============================================================
-- EXAMPLE 41: Multiple OR กับ LIKE
-- ============================================================

-- หาพนักงาน developer (ทุก seniority)
SELECT first_name, last_name, job_title
FROM employees
WHERE job_title LIKE '%Developer%'
   OR job_title LIKE '%Engineer%'
   OR job_title LIKE '%Programmer%';

-- ผลลัพธ์:
-- first_name | last_name | job_title
-- -----------|-----------|------------------
-- สมชาย      | นักเขียน  | Senior Developer
-- มาลี       | สวยงาม    | Junior Developer
-- อนันต์     | ใจดี      | Developer

-- ============================================================
-- EXAMPLE 42: Combined Conditions
-- ============================================================

-- พนักงาน senior (เงินเดือนสูง) ที่ทำงานใน technical roles
SELECT 
    first_name || ' ' || last_name AS name,
    job_title,
    salary,
    department_id
FROM employees
WHERE salary > 60000
  AND department_id IN (1, 7)  -- IT หรือ R&D
  AND (
      job_title LIKE '%Developer%'
   OR job_title LIKE '%Lead%'
   OR job_title LIKE '%Director%'
   OR job_title LIKE '%Manager%'
  )
ORDER BY salary DESC;

-- ============================================================
-- EXAMPLE 43: Filtering with Calculations
-- ============================================================

-- สินค้าที่มูลค่า inventory สูงกว่า 100,000 บาท
-- (ใน WHERE ต้องใช้ expression ตรงๆ ไม่ใช่ alias)
SELECT 
    product_name,
    unit_price,
    units_in_stock,
    unit_price * units_in_stock AS inventory_value
FROM products
WHERE unit_price * units_in_stock > 100000
ORDER BY unit_price * units_in_stock DESC;

-- ผลลัพธ์:
-- product_name        | unit_price | units_in_stock | inventory_value
-- --------------------|------------|----------------|----------------
-- Laptop Pro 15"      | 45000.00   | 25             | 1125000.00
-- Monitor 27" 4K      | 18000.00   | 20             | 360000.00
-- Standing Desk       | 15000.00   | 15             | 225000.00
-- Office Chair Premium| 8900.00    | 30             | 267000.00
-- Noise Cancelling    | 7500.00    | 45             | 337500.00
-- Whiteboard A4       | 150.00     | 500            | 75000.00  (ไม่เข้า)

-- หมายเหตุ: Whiteboard มูลค่า = 150 * 500 = 75,000 ไม่เกิน 100,000
-- Pen Set: 120 * 400 = 48,000 ไม่เข้า
```

---

## 4.10 สถานการณ์จริง

```sql
-- ============================================================
-- EXAMPLE 44: Customer Segmentation
-- ============================================================

-- ลูกค้าในกรุงเทพ ที่เป็นบริษัท
SELECT 
    customer_id,
    company_name,
    first_name || ' ' || last_name AS contact_name,
    city
FROM customers
WHERE city = 'Bangkok'
  AND company_name IS NOT NULL
ORDER BY company_name;

-- ============================================================
-- EXAMPLE 45: Product Search
-- ============================================================

-- ค้นหาสินค้าตาม keyword (เหมือน e-commerce search)
SELECT 
    product_id,
    product_name,
    category,
    unit_price
FROM products
WHERE 
    product_name LIKE '%Laptop%'
    OR product_name LIKE '%Computer%'
    OR product_name LIKE '%PC%'
    OR category = 'Electronics'
ORDER BY unit_price DESC;

-- ============================================================
-- EXAMPLE 46: Salary Range Filter
-- ============================================================

-- หาพนักงานเงินเดือนระดับกลาง ที่ยังไม่ใช่ manager
SELECT 
    first_name,
    last_name,
    salary,
    job_title,
    manager_id
FROM employees
WHERE salary BETWEEN 45000 AND 75000
  AND manager_id IS NOT NULL  -- ต้องมี manager (ไม่ใช่ top)
  AND job_title NOT LIKE '%Manager%'
  AND job_title NOT LIKE '%Director%'
ORDER BY salary DESC;

-- ============================================================
-- EXAMPLE 47: Active Orders Dashboard
-- ============================================================

-- คำสั่งซื้อที่ต้องดูแล:
-- 1. ยังไม่ ship
-- 2. ไม่ cancelled
-- 3. ยอดเงินสูงกว่า 5,000 บาท
SELECT 
    order_id,
    customer_id,
    order_date,
    required_date,
    status,
    total_amount
FROM orders
WHERE shipped_date IS NULL
  AND status <> 'Cancelled'
  AND total_amount > 5000
ORDER BY required_date ASC, total_amount DESC;

-- ============================================================
-- EXAMPLE 48: Employee Directory Search
-- ============================================================

-- ค้นหาพนักงานตาม keyword ในชื่อหรือ email
-- (แบบที่ทำจริงในระบบ HR)
SELECT 
    employee_id,
    first_name || ' ' || last_name AS full_name,
    email,
    job_title,
    department_id
FROM employees
WHERE first_name LIKE '%มาลี%'
   OR last_name LIKE '%มาลี%'
   OR email LIKE '%malee%';
```

---

## 4.11 WHERE กับ Boolean Values

```sql
-- ============================================================
-- EXAMPLE 49: Boolean filtering
-- ============================================================

-- พนักงาน active
SELECT first_name, last_name
FROM employees
WHERE is_active = TRUE;

-- หรือ (ขึ้นกับ database)
SELECT first_name, last_name
FROM employees
WHERE is_active;  -- PostgreSQL

SELECT first_name, last_name
FROM employees
WHERE is_active = 1;  -- MySQL, SQLite

-- พนักงาน inactive
SELECT first_name, last_name
FROM employees
WHERE is_active = FALSE;
-- หรือ
WHERE NOT is_active;

-- สินค้าที่ยังขายอยู่
SELECT product_name, unit_price
FROM products
WHERE discontinued = FALSE;

-- ============================================================
-- EXAMPLE 50: ตัวอย่างสมบูรณ์ - Business Report
-- ============================================================

-- Comprehensive Employee Filter
SELECT 
    e.employee_id,
    e.first_name || ' ' || e.last_name AS full_name,
    e.job_title,
    e.salary,
    e.department_id,
    CASE 
        WHEN e.salary >= 80000 THEN 'Senior Executive'
        WHEN e.salary >= 60000 THEN 'Senior Staff'
        WHEN e.salary >= 40000 THEN 'Regular Staff'
        ELSE 'Entry Level'
    END AS grade
FROM employees e
WHERE e.is_active = TRUE
  AND e.salary BETWEEN 40000 AND 100000
  AND e.department_id IN (1, 2, 3, 4, 5)
  AND e.hire_date >= '2018-01-01'
  AND e.job_title NOT LIKE '%Manager%'
ORDER BY e.salary DESC, e.hire_date ASC;
```

---

## 4.12 ตัวอย่าง 40+ Queries ครบถ้วน

```sql
-- 1. พนักงานเงินเดือนมากกว่า 50000
SELECT first_name, salary FROM employees WHERE salary > 50000;

-- 2. สินค้าในหมวด Books
SELECT product_name, unit_price FROM products WHERE category = 'Books';

-- 3. คำสั่งซื้อที่ delivered
SELECT order_id, total_amount FROM orders WHERE status = 'Delivered';

-- 4. พนักงานที่มี manager
SELECT first_name, manager_id FROM employees WHERE manager_id IS NOT NULL;

-- 5. สินค้าที่ discontinue
SELECT product_name FROM products WHERE discontinued = TRUE;

-- 6. ลูกค้าในกรุงเทพ
SELECT first_name, last_name, city FROM customers WHERE city = 'Bangkok';

-- 7. สินค้าราคาระหว่าง 1000-5000
SELECT product_name, unit_price FROM products WHERE unit_price BETWEEN 1000 AND 5000;

-- 8. พนักงานในแผนก IT, HR, Finance
SELECT first_name, department_id FROM employees WHERE department_id IN (1, 2, 3);

-- 9. สินค้าที่ชื่อมีคำว่า 'Pro'
SELECT product_name FROM products WHERE product_name LIKE '%Pro%';

-- 10. พนักงานที่ไม่มี manager
SELECT first_name, manager_id FROM employees WHERE manager_id IS NULL;

-- 11. Orders ที่ยังไม่ shipped
SELECT order_id, status FROM orders WHERE shipped_date IS NULL;

-- 12. สินค้า Electronics ราคาต่ำกว่า 3000
SELECT product_name, unit_price FROM products
WHERE category = 'Electronics' AND unit_price < 3000;

-- 13. พนักงานเงินเดือน 60000-90000
SELECT first_name, salary FROM employees WHERE salary BETWEEN 60000 AND 90000;

-- 14. ลูกค้าที่ไม่มีชื่อบริษัท
SELECT first_name, last_name FROM customers WHERE company_name IS NULL;

-- 15. Products ไม่ใช่ Electronics และ ไม่ใช่ Furniture
SELECT product_name, category FROM products
WHERE category NOT IN ('Electronics', 'Furniture');

-- 16. พนักงานที่เข้าทำงานปี 2020 เป็นต้นไป
SELECT first_name, hire_date FROM employees WHERE hire_date >= '2020-01-01';

-- 17. Orders ยอดเงินมากกว่า 20000
SELECT order_id, total_amount FROM orders WHERE total_amount > 20000;

-- 18. พนักงาน developer ทุกระดับ
SELECT first_name, job_title FROM employees WHERE job_title LIKE '%Developer%';

-- 19. สินค้าที่ stock ต่ำกว่า 30 ชิ้น
SELECT product_name, units_in_stock FROM products WHERE units_in_stock < 30;

-- 20. ลูกค้าที่มี email บน gmail
SELECT first_name, email FROM customers WHERE email LIKE '%@gmail.com';

-- 21. Orders ที่ cancelled
SELECT order_id, status FROM orders WHERE status = 'Cancelled';

-- 22. พนักงานในแผนก Sales เงินเดือนมากกว่า 70000
SELECT first_name, salary, department_id FROM employees
WHERE department_id = 5 AND salary > 70000;

-- 23. สินค้า Electronics หรือ Books
SELECT product_name, category FROM products
WHERE category IN ('Electronics', 'Books');

-- 24. พนักงานเงินเดือนไม่เกิน 50000 ที่ active
SELECT first_name, salary FROM employees
WHERE salary <= 50000 AND is_active = TRUE;

-- 25. ลูกค้านอกกรุงเทพ
SELECT first_name, city FROM customers WHERE city <> 'Bangkok';

-- 26. สินค้าที่ชื่อขึ้นต้นด้วย 'S'
SELECT product_name FROM products WHERE product_name LIKE 'S%';

-- 27. พนักงานที่ hire ระหว่าง 2019-2021
SELECT first_name, hire_date FROM employees
WHERE hire_date BETWEEN '2019-01-01' AND '2021-12-31';

-- 28. Orders ที่ยังไม่ cancelled และยังไม่ delivered
SELECT order_id, status FROM orders
WHERE status NOT IN ('Cancelled', 'Delivered');

-- 29. สินค้าที่ inventory value สูงกว่า 200000
SELECT product_name, unit_price * units_in_stock AS value
FROM products
WHERE unit_price * units_in_stock > 200000;

-- 30. พนักงานที่เป็น manager (ที่มีคนรายงาน)
SELECT first_name, employee_id FROM employees
WHERE employee_id IN (SELECT DISTINCT manager_id FROM employees WHERE manager_id IS NOT NULL);

-- 31. สินค้า Furniture ราคาต่ำกว่า 10000
SELECT product_name, unit_price FROM products
WHERE category = 'Furniture' AND unit_price < 10000;

-- 32. ลูกค้า active ในกรุงเทพและเชียงใหม่
SELECT first_name, city FROM customers
WHERE city IN ('Bangkok', 'Chiang Mai') AND is_active = TRUE;

-- 33. Orders ที่ required_date เลยวันนี้ไปแล้ว
-- (ใช้ DATE('now') สำหรับ SQLite หรือ CURRENT_DATE สำหรับ PostgreSQL)
SELECT order_id, required_date FROM orders
WHERE required_date < DATE('now') AND shipped_date IS NULL;

-- 34. พนักงานที่ job title ไม่ใช่ Manager และ Director
SELECT first_name, job_title FROM employees
WHERE job_title NOT LIKE '%Manager%'
  AND job_title NOT LIKE '%Director%';

-- 35. สินค้าที่มีในคลังเพียงพอ (stock >= 50) และยังขายได้
SELECT product_name, units_in_stock FROM products
WHERE units_in_stock >= 50 AND discontinued = FALSE;

-- 36. พนักงานในแผนก 1 หรือ 5 ที่มี manager
SELECT first_name, department_id, manager_id FROM employees
WHERE (department_id = 1 OR department_id = 5)
  AND manager_id IS NOT NULL;

-- 37. Orders ของลูกค้า ID 1 หรือ 2
SELECT order_id, customer_id, total_amount FROM orders
WHERE customer_id IN (1, 2);

-- 38. สินค้าราคาน้อยกว่า 500 หรือมากกว่า 20000
SELECT product_name, unit_price FROM products
WHERE unit_price < 500 OR unit_price > 20000;

-- 39. พนักงานที่เงินเดือนสูงกว่า 60000 แต่ไม่ใช่ CFO
SELECT first_name, salary, job_title FROM employees
WHERE salary > 60000 AND job_title <> 'CFO';

-- 40. ลูกค้าที่มีเบอร์โทรขึ้นต้นด้วย '02' (กรุงเทพ)
SELECT first_name, phone FROM customers WHERE phone LIKE '02-%';
```

---

## 4.13 สรุปบทที่ 4

```
✅ WHERE clause syntax
✅ Comparison operators: =, <>, !=, <, >, <=, >=
✅ BETWEEN (inclusive on both ends)
✅ IN (list of values)
✅ LIKE (pattern matching: % = any chars, _ = one char)
✅ IS NULL / IS NOT NULL (สำหรับ NULL)
✅ AND, OR, NOT (logical operators)
✅ Operator precedence: NOT > AND > OR
✅ วงเล็บเพื่อความชัดเจน
✅ สถานการณ์จริงในการกรองข้อมูล
```

---

## 📝 แบบฝึกหัดท้ายบท

**คำถาม 1:** หาพนักงานที่เงินเดือนระหว่าง 45,000 ถึง 70,000 บาท

**คำถาม 2:** หาสินค้าทุกชนิดที่ไม่ใช่ Electronics เรียงตามราคา

**คำถาม 3:** หาลูกค้าที่อยู่ใน 'Bangkok' หรือ 'Chiang Mai'

**คำถาม 4:** หาคำสั่งซื้อที่ยังไม่ shipped (shipped_date IS NULL) และ status ไม่ใช่ 'Cancelled'

**คำถาม 5:** หาสินค้าที่ชื่อมีคำว่า 'Book' (ใช้ LIKE) และราคาต่ำกว่า 600 บาท

**คำถาม 6:** หาพนักงานที่:
- อยู่ในแผนก IT (1) หรือ Sales (5) 
- AND เงินเดือนมากกว่า 50,000
- AND มี manager

**คำถาม 7:** หาสินค้าที่:
- ราคาระหว่าง 1,000 ถึง 10,000
- OR มีในคลังน้อยกว่า 30 ชิ้น

**คำถาม 8:** หาพนักงานที่ไม่ได้อยู่ในแผนก 1, 2, 3 และเงินเดือนมากกว่า 60,000

**คำถาม 9:** หาลูกค้าที่มีชื่อบริษัท (ไม่ NULL) และอยู่ใน 'Bangkok'

**คำถาม 10:** สร้าง complex query เพื่อหา "products ที่น่า feature":
- Electronics ราคา 2,000-15,000
- มีในคลัง 20+ ชิ้น
- ยังไม่ discontinued

---

## ✅ เฉลยแบบฝึกหัด

### เฉลยที่ 1:
```sql
SELECT first_name, last_name, salary
FROM employees
WHERE salary BETWEEN 45000 AND 70000
ORDER BY salary;
```

### เฉลยที่ 2:
```sql
SELECT product_name, category, unit_price
FROM products
WHERE category <> 'Electronics'
ORDER BY unit_price;
```

### เฉลยที่ 3:
```sql
SELECT first_name, last_name, city
FROM customers
WHERE city IN ('Bangkok', 'Chiang Mai');
```

### เฉลยที่ 4:
```sql
SELECT order_id, order_date, required_date, status
FROM orders
WHERE shipped_date IS NULL
  AND status <> 'Cancelled';
```

### เฉลยที่ 5:
```sql
SELECT product_name, unit_price
FROM products
WHERE product_name LIKE '%Book%'
  AND unit_price < 600;
```

### เฉลยที่ 6:
```sql
SELECT first_name, last_name, salary, department_id, manager_id
FROM employees
WHERE (department_id = 1 OR department_id = 5)
  AND salary > 50000
  AND manager_id IS NOT NULL;
```

### เฉลยที่ 7:
```sql
SELECT product_name, unit_price, units_in_stock
FROM products
WHERE (unit_price BETWEEN 1000 AND 10000)
   OR (units_in_stock < 30)
ORDER BY product_name;
```

### เฉลยที่ 8:
```sql
SELECT first_name, last_name, salary, department_id
FROM employees
WHERE department_id NOT IN (1, 2, 3)
  AND salary > 60000;
```

### เฉลยที่ 9:
```sql
SELECT company_name, first_name, last_name, city
FROM customers
WHERE company_name IS NOT NULL
  AND city = 'Bangkok';
```

### เฉลยที่ 10:
```sql
SELECT 
    product_name,
    category,
    unit_price,
    units_in_stock,
    unit_price * units_in_stock AS inventory_value
FROM products
WHERE category = 'Electronics'
  AND unit_price BETWEEN 2000 AND 15000
  AND units_in_stock >= 20
  AND discontinued = FALSE
ORDER BY unit_price;
```

---

## ➡️ บทถัดไป

**[Part 005: Sorting with ORDER BY](part-005.md)**

ในบทถัดไปเราจะเรียนรู้การจัดเรียงข้อมูลด้วย ORDER BY!

---

*Part 004 of 120 | หลักสูตร SQL ครบวงจร*
