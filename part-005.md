# Part 005: Sorting with ORDER BY

> **หลักสูตร SQL ครบวงจร | Part 5 of 120**

---

## 🎯 สิ่งที่จะได้เรียนรู้ในบทนี้

- ORDER BY syntax และการทำงาน
- ASC (Ascending) และ DESC (Descending)
- Sorting หลาย columns
- ORDER BY กับ expressions
- ORDER BY กับ column aliases
- NULLS FIRST / NULLS LAST
- ORDER BY กับ CASE expression
- Performance considerations
- สถานการณ์จริง

**เวลาที่ใช้เรียน**: ประมาณ 2 ชั่วโมง

---

## ฐานข้อมูลที่ใช้ในบทนี้

```sql
-- ใช้ตารางจาก Part 001-004
-- ยืนยันข้อมูล:
SELECT COUNT(*) AS employees FROM employees;  -- 15
SELECT COUNT(*) AS products  FROM products;   -- 15
SELECT COUNT(*) AS orders    FROM orders;     -- 10
```

---

## 5.1 ORDER BY Syntax

```sql
-- Syntax พื้นฐาน
SELECT column1, column2
FROM table_name
[WHERE condition]
ORDER BY column1 [ASC|DESC], column2 [ASC|DESC], ...;

-- ORDER BY ทำงานหลังจาก:
-- FROM → WHERE → SELECT → ORDER BY → LIMIT

-- Default = ASC (ascending = เรียงจากน้อยไปมาก)
-- Numbers: 1, 2, 3, 10, 100...
-- Text: A, B, C... (alphabetical)
-- Dates: เก่าไปใหม่
```

---

## 5.2 ASC - Ascending Order

```sql
-- ============================================================
-- EXAMPLE 1: เรียงตัวเลขจากน้อยไปมาก (default)
-- ============================================================

SELECT product_name, unit_price
FROM products
ORDER BY unit_price;  -- ASC เป็น default

-- ผลลัพธ์:
-- product_name              | unit_price
-- --------------------------|----------
-- Pen Set                   | 120.00
-- Whiteboard A4             | 150.00
-- Notebook Premium          | 250.00
-- SQL Programming Book      | 450.00
-- Python for Data Science   | 550.00
-- Wireless Mouse            | 850.00
-- Desk Lamp LED             | 1200.00
-- USB-C Hub                 | 2500.00
-- Webcam HD                 | 2800.00
-- Keyboard Mechanical       | 3200.00
-- Noise Cancelling Headphone| 7500.00
-- Office Chair Premium      | 8900.00
-- Monitor 27" 4K            | 18000.00
-- Standing Desk             | 15000.00
-- Laptop Pro 15"            | 45000.00

-- ============================================================
-- EXAMPLE 2: เรียงข้อความตาม alphabet
-- ============================================================

SELECT first_name, last_name
FROM employees
ORDER BY first_name ASC;

-- ============================================================
-- EXAMPLE 3: เรียงตามวันที่ (เก่าไปใหม่)
-- ============================================================

SELECT first_name, hire_date
FROM employees
ORDER BY hire_date ASC;

-- ผลลัพธ์:
-- first_name | hire_date
-- -----------|----------
-- กิตติ      | 2016-09-30
-- วิภา       | 2017-01-10
-- ไพโรจน์    | 2017-07-17
-- สมชาย      | 2018-03-15
-- ...
```

---

## 5.3 DESC - Descending Order

```sql
-- ============================================================
-- EXAMPLE 4: เรียงตัวเลขจากมากไปน้อย
-- ============================================================

SELECT first_name, last_name, salary
FROM employees
ORDER BY salary DESC;

-- ผลลัพธ์:
-- first_name | last_name    | salary
-- -----------|--------------|----------
-- วิภา       | รักงาน       | 120000.00
-- กิตติ      | เก่งมาก      | 95000.00
-- สุรชัย     | เด่นมาก      | 85000.00
-- สมชาย      | นักเขียน     | 75000.00
-- ไพโรจน์    | กว้างขวาง    | 72000.00
-- ประยูร     | มีสุข        | 70000.00
-- พิมพ์ใจ    | หวานใจ       | 68000.00
-- สมหญิง     | ดีงาม        | 65000.00
-- อนันต์     | ใจดี         | 58000.00
-- ชาญชัย    | ฉลาดเฉลียว   | 55000.00
-- จินตนา     | คิดเก่ง      | 52000.00
-- ธนพล      | รวยแน่       | 48000.00
-- มาลี       | สวยงาม       | 45000.00
-- รัตนา      | ขยันมาก      | 42000.00
-- นงนุช      | น่ารักมาก    | 38000.00

-- ============================================================
-- EXAMPLE 5: เรียงวันที่จากใหม่ไปเก่า
-- ============================================================

SELECT order_id, order_date, total_amount
FROM orders
ORDER BY order_date DESC;

-- ผลลัพธ์:
-- order_id | order_date  | total_amount
-- ---------|-------------|-------------
-- 10       | 2024-03-10  | 5500.00
-- 9        | 2024-03-05  | 27800.00
-- 8        | 2024-03-01  | 3200.00
-- 7        | 2024-02-14  | 12500.00
-- 6        | 2024-02-10  | 18000.00
-- 5        | 2024-02-01  | 46350.00
-- 4        | 2024-01-15  | 850.00
-- 3        | 2024-01-12  | 9200.00
-- 2        | 2024-01-08  | 22500.00
-- 1        | 2024-01-05  | 48700.00

-- ============================================================
-- EXAMPLE 6: เรียง text DESC (Z ไป A)
-- ============================================================

SELECT product_name, category
FROM products
ORDER BY category DESC, product_name DESC;
```

---

## 5.4 Multiple Column Sorting

```sql
-- ============================================================
-- EXAMPLE 7: เรียง 2 columns
-- ============================================================

-- เรียงตาม department ก่อน แล้วตามเงินเดือน
SELECT first_name, last_name, department_id, salary
FROM employees
ORDER BY department_id ASC, salary DESC;

-- ผลลัพธ์:
-- first_name | last_name    | dept_id | salary
-- -----------|--------------|---------|----------
-- สมชาย      | นักเขียน     | 1       | 75000.00
-- อนันต์     | ใจดี         | 1       | 58000.00
-- จินตนา     | คิดเก่ง      | 1       | 52000.00
-- มาลี       | สวยงาม       | 1       | 45000.00
-- สมหญิง     | ดีงาม        | 2       | 65000.00
-- รัตนา      | ขยันมาก      | 2       | 42000.00
-- วิภา       | รักงาน       | 3       | 120000.00
-- ชาญชัย    | ฉลาดเฉลียว   | 3       | 55000.00
-- ประยูร     | มีสุข        | 4       | 70000.00
-- นงนุช      | น่ารักมาก    | 4       | 38000.00
-- กิตติ      | เก่งมาก      | 5       | 95000.00
-- ไพโรจน์    | กว้างขวาง    | 5       | 72000.00
-- ธนพล      | รวยแน่       | 5       | 48000.00
-- พิมพ์ใจ    | หวานใจ       | 6       | 68000.00
-- สุรชัย     | เด่นมาก      | 7       | 85000.00

-- ============================================================
-- EXAMPLE 8: เรียง 3 columns
-- ============================================================

SELECT 
    category,
    product_name,
    unit_price
FROM products
ORDER BY 
    category ASC,       -- 1st: จัดกลุ่มตาม category
    unit_price DESC,    -- 2nd: ใน category เรียงจากแพงไปถูก
    product_name ASC;   -- 3rd: ชื่อเดิมเรียงตาม alphabet

-- ผลลัพธ์:
-- category    | product_name              | unit_price
-- ------------|---------------------------|----------
-- Books       | Python for Data Science   | 550.00
-- Books       | SQL Programming Book      | 450.00
-- Electronics | Laptop Pro 15"            | 45000.00
-- Electronics | Monitor 27" 4K            | 18000.00
-- Electronics | Noise Cancelling Headphone| 7500.00
-- Electronics | Keyboard Mechanical       | 3200.00
-- Electronics | Webcam HD                 | 2800.00
-- Electronics | USB-C Hub                 | 2500.00
-- Electronics | Wireless Mouse            | 850.00
-- Furniture   | Standing Desk             | 15000.00
-- Furniture   | Office Chair Premium      | 8900.00
-- Furniture   | Desk Lamp LED             | 1200.00
-- Stationery  | Notebook Premium          | 250.00
-- Stationery  | Whiteboard A4             | 150.00
-- Stationery  | Pen Set                   | 120.00

-- ============================================================
-- EXAMPLE 9: Mixed ASC/DESC
-- ============================================================

-- เรียงตาม status (A-Z) แล้วตาม total amount (มากไปน้อย)
SELECT order_id, status, total_amount, order_date
FROM orders
ORDER BY status ASC, total_amount DESC;
```

---

## 5.5 ORDER BY กับ Expressions

```sql
-- ============================================================
-- EXAMPLE 10: ORDER BY calculated value
-- ============================================================

-- เรียงตาม inventory value (ราคา * จำนวน)
SELECT 
    product_name,
    unit_price,
    units_in_stock,
    unit_price * units_in_stock AS inventory_value
FROM products
ORDER BY unit_price * units_in_stock DESC;

-- ผลลัพธ์:
-- product_name             | unit_price | units_in_stock | inventory_value
-- -------------------------|------------|----------------|----------------
-- Laptop Pro 15"           | 45000.00   | 25             | 1125000.00
-- Monitor 27" 4K           | 18000.00   | 20             | 360000.00
-- Noise Cancelling Headphone| 7500.00  | 45             | 337500.00
-- Standing Desk            | 15000.00   | 15             | 225000.00
-- ...

-- ============================================================
-- EXAMPLE 11: ORDER BY column alias
-- ============================================================

-- สามารถใช้ alias ที่ตั้งใน SELECT ได้ใน ORDER BY
SELECT 
    product_name,
    unit_price * units_in_stock AS inventory_value
FROM products
ORDER BY inventory_value DESC;  -- ใช้ alias ได้!

-- หมายเหตุ: ORDER BY ประมวลผลหลัง SELECT จึงเห็น alias
-- แต่ WHERE ประมวลผลก่อน SELECT จึงใช้ alias ใน WHERE ไม่ได้

-- ============================================================
-- EXAMPLE 12: ORDER BY string functions
-- ============================================================

-- เรียงตามความยาวชื่อสินค้า (สั้นไปยาว)
SELECT product_name, LENGTH(product_name) AS name_len
FROM products
ORDER BY LENGTH(product_name) ASC;

-- เรียงตาม uppercase ของชื่อ (case-insensitive sort)
SELECT product_name
FROM products
ORDER BY UPPER(product_name);

-- ============================================================
-- EXAMPLE 13: ORDER BY date functions
-- ============================================================

-- เรียงตามเดือนที่เข้าทำงาน (ไม่สนใจปี)
-- SQLite:
SELECT first_name, hire_date
FROM employees
ORDER BY CAST(strftime('%m', hire_date) AS INTEGER);

-- PostgreSQL:
-- ORDER BY EXTRACT(MONTH FROM hire_date);

-- MySQL:
-- ORDER BY MONTH(hire_date);
```

---

## 5.6 ORDER BY กับ CASE

```sql
-- ============================================================
-- EXAMPLE 14: Custom sort order ด้วย CASE
-- ============================================================

-- เรียง status ตามลำดับความสำคัญ (ไม่ใช่ alphabet)
-- ต้องการ: Processing > Pending > Delivered > Cancelled
SELECT 
    order_id,
    status,
    total_amount
FROM orders
ORDER BY 
    CASE status
        WHEN 'Processing' THEN 1
        WHEN 'Pending'    THEN 2
        WHEN 'Delivered'  THEN 3
        WHEN 'Cancelled'  THEN 4
        ELSE 5
    END ASC,
    total_amount DESC;

-- ผลลัพธ์:
-- order_id | status     | total_amount
-- ---------|------------|-------------
-- 3        | Processing | 9200.00   ← Processing ก่อน
-- 10       | Processing | 5500.00
-- 6        | Pending    | 18000.00  ← Pending ถัดมา
-- 1        | Delivered  | 48700.00  ← Delivered
-- ...
-- 8        | Cancelled  | 3200.00   ← Cancelled สุดท้าย

-- ============================================================
-- EXAMPLE 15: เรียง products ตาม category priority
-- ============================================================

SELECT 
    product_name,
    category,
    unit_price
FROM products
ORDER BY 
    CASE category
        WHEN 'Electronics' THEN 1  -- Electronics ก่อน
        WHEN 'Furniture'   THEN 2
        WHEN 'Books'       THEN 3
        WHEN 'Stationery'  THEN 4
        ELSE 5
    END,
    unit_price DESC;

-- ============================================================
-- EXAMPLE 16: เรียง employees ตาม seniority
-- ============================================================

SELECT 
    first_name,
    job_title,
    salary
FROM employees
ORDER BY 
    CASE 
        WHEN job_title LIKE '%Director%' OR job_title LIKE '%CFO%' THEN 1
        WHEN job_title LIKE '%Manager%' OR job_title LIKE '%Lead%' THEN 2
        WHEN job_title LIKE '%Senior%'                              THEN 3
        WHEN job_title LIKE '%Junior%'                              THEN 5
        ELSE 4
    END,
    salary DESC;
```

---

## 5.7 NULLS FIRST / NULLS LAST

```sql
-- ============================================================
-- EXAMPLE 17: NULLS พฤติกรรม default
-- ============================================================

-- เรียง employees ตาม manager_id (มี NULL)
SELECT first_name, manager_id
FROM employees
ORDER BY manager_id;

-- PostgreSQL/SQLite default: NULL ไปท้าย (ASC)
-- MySQL default: NULL ไปหน้า (ASC)
-- SQL Server: NULL ไปหน้า (ASC)

-- ============================================================
-- EXAMPLE 18: NULLS LAST (PostgreSQL, SQLite 3.30+)
-- ============================================================

-- เรียง manager_id ASC แต่ NULL ไปท้าย
SELECT first_name, manager_id
FROM employees
ORDER BY manager_id ASC NULLS LAST;

-- ผลลัพธ์:
-- first_name | manager_id
-- -----------|-----------
-- มาลี       | 1
-- อนันต์     | 1
-- จินตนา     | 1
-- รัตนา      | 2
-- ชาญชัย    | 3
-- นงนุช      | 4
-- ธนพล      | 5
-- ไพโรจน์    | 5
-- สมชาย      | NULL  ← NULL ท้ายสุด
-- สมหญิง     | NULL
-- วิภา       | NULL
-- ...

-- ============================================================
-- EXAMPLE 19: NULLS FIRST
-- ============================================================

-- เรียง manager_id DESC แต่ NULL ไปหน้า
SELECT first_name, manager_id
FROM employees
ORDER BY manager_id DESC NULLS FIRST;

-- ผลลัพธ์:
-- สมชาย      | NULL  ← NULL หน้าสุด
-- สมหญิง     | NULL
-- ...
-- ไพโรจน์    | 5
-- ธนพล      | 5
-- นงนุช      | 4
-- ชาญชัย    | 3
-- รัตนา      | 2
-- มาลี       | 1
-- อนันต์     | 1
-- จินตนา     | 1

-- ============================================================
-- EXAMPLE 20: Workaround สำหรับ MySQL/SQL Server
-- ============================================================

-- MySQL/SQL Server ไม่รองรับ NULLS FIRST/LAST syntax
-- ใช้ CASE แทน:

-- NULL ไปท้าย (ASC):
SELECT first_name, manager_id
FROM employees
ORDER BY 
    CASE WHEN manager_id IS NULL THEN 1 ELSE 0 END,
    manager_id ASC;

-- NULL ไปหน้า (DESC):
SELECT first_name, manager_id
FROM employees
ORDER BY 
    CASE WHEN manager_id IS NULL THEN 0 ELSE 1 END,
    manager_id DESC;
```

---

## 5.8 ORDER BY กับ Column Position

```sql
-- ============================================================
-- EXAMPLE 21: ORDER BY column number (ไม่แนะนำ)
-- ============================================================

-- สามารถใช้เลข position แทนชื่อ column ได้
SELECT product_name, category, unit_price
FROM products
ORDER BY 3 DESC;  -- 3 = unit_price (column ที่ 3)

-- ❌ ไม่แนะนำเพราะ:
-- - อ่านยาก ไม่รู้ว่า 3 คือ column อะไร
-- - ถ้าเปลี่ยนลำดับ column ผลลัพธ์จะผิด
-- - แต่ใช้ได้ใน GROUP BY บางครั้ง

-- ✓ แนะนำ: ใช้ชื่อ column
SELECT product_name, category, unit_price
FROM products
ORDER BY unit_price DESC;
```

---

## 5.9 ORDER BY และ Performance

```sql
-- ============================================================
-- EXAMPLE 22: INDEX ช่วย ORDER BY
-- ============================================================

-- ถ้ามี INDEX บน column ที่ใช้ ORDER BY → เร็วมาก
-- ถ้าไม่มี INDEX → database ต้อง sort ทุกครั้ง

-- ดู query plan (PostgreSQL)
EXPLAIN SELECT * FROM employees ORDER BY salary DESC;
-- มักเห็น "Sort" operation ถ้าไม่มี index

-- สร้าง index (จะเรียนละเอียดใน Part 056)
CREATE INDEX idx_employees_salary ON employees(salary);

-- ============================================================
-- EXAMPLE 23: LIMIT + ORDER BY สำคัญ
-- ============================================================

-- ดู top 5 เงินเดือนสูงสุด
SELECT first_name, salary
FROM employees
ORDER BY salary DESC
LIMIT 5;

-- ผลลัพธ์:
-- first_name | salary
-- -----------|----------
-- วิภา       | 120000.00
-- กิตติ      | 95000.00
-- สุรชัย     | 85000.00
-- สมชาย      | 75000.00
-- ไพโรจน์    | 72000.00

-- หมายเหตุ: ORDER BY LIMIT มักได้ optimization พิเศษ
-- database ไม่ต้อง sort ทั้งหมด แค่หา top N

-- ============================================================
-- EXAMPLE 24: ORDER BY กับ index หลาย columns
-- ============================================================

-- ถ้า ORDER BY ใช้หลาย columns และมี composite index
-- ลำดับต้องตรงกับ index definition
CREATE INDEX idx_emp_dept_salary ON employees(department_id, salary DESC);

-- ใช้ประโยชน์จาก index:
SELECT first_name, department_id, salary
FROM employees
ORDER BY department_id, salary DESC;
```

---

## 5.10 Practical Examples

```sql
-- ============================================================
-- EXAMPLE 25: Top Earners Report
-- ============================================================

SELECT 
    RANK() OVER (ORDER BY salary DESC) AS rank_position,
    first_name || ' ' || last_name AS name,
    job_title,
    salary,
    salary * 12 AS annual_salary
FROM employees
ORDER BY salary DESC
LIMIT 10;

-- ============================================================
-- EXAMPLE 26: Product Price List (sorted)
-- ============================================================

SELECT 
    ROW_NUMBER() OVER (ORDER BY unit_price) AS no,
    product_name,
    category,
    unit_price,
    ROUND(unit_price * 1.07, 2) AS price_with_vat
FROM products
ORDER BY unit_price;

-- ============================================================
-- EXAMPLE 27: Recent Orders Report
-- ============================================================

SELECT 
    order_id,
    customer_id,
    order_date,
    status,
    total_amount
FROM orders
ORDER BY 
    order_date DESC,
    total_amount DESC
LIMIT 5;

-- ผลลัพธ์:
-- order_id | customer_id | order_date  | status     | total_amount
-- ---------|-------------|-------------|------------|-------------
-- 10       | 9           | 2024-03-10  | Processing | 5500.00
-- 9        | 8           | 2024-03-05  | Delivered  | 27800.00
-- 8        | 7           | 2024-03-01  | Cancelled  | 3200.00
-- 7        | 6           | 2024-02-14  | Delivered  | 12500.00
-- 6        | 1           | 2024-02-10  | Pending    | 18000.00

-- ============================================================
-- EXAMPLE 28: Alphabetical Employee Directory
-- ============================================================

SELECT 
    last_name || ', ' || first_name AS name,
    job_title,
    email,
    department_id
FROM employees
ORDER BY 
    last_name ASC,
    first_name ASC;

-- ============================================================
-- EXAMPLE 29: Sales by Amount (descending)
-- ============================================================

SELECT 
    order_id,
    customer_id,
    order_date,
    total_amount,
    ROUND(total_amount * 100.0 / (SELECT SUM(total_amount) FROM orders), 2) AS pct_of_total
FROM orders
WHERE status = 'Delivered'
ORDER BY total_amount DESC;

-- ============================================================
-- EXAMPLE 30: Inventory Alert (sorted by urgency)
-- ============================================================

SELECT 
    product_name,
    units_in_stock,
    CASE 
        WHEN units_in_stock = 0  THEN 1  -- Critical
        WHEN units_in_stock < 20 THEN 2  -- Warning
        WHEN units_in_stock < 50 THEN 3  -- Low
        ELSE 4                           -- OK
    END AS alert_level,
    CASE 
        WHEN units_in_stock = 0  THEN 'CRITICAL - Out of Stock'
        WHEN units_in_stock < 20 THEN 'WARNING - Very Low'
        WHEN units_in_stock < 50 THEN 'LOW - Reorder Soon'
        ELSE 'OK'
    END AS alert_message
FROM products
WHERE discontinued = FALSE
ORDER BY 
    CASE 
        WHEN units_in_stock = 0  THEN 1
        WHEN units_in_stock < 20 THEN 2
        WHEN units_in_stock < 50 THEN 3
        ELSE 4
    END ASC,
    units_in_stock ASC;
```

---

## 5.11 25+ ตัวอย่างครบถ้วน

```sql
-- 1. เรียงตามราคาจากถูกไปแพง
SELECT product_name, unit_price FROM products ORDER BY unit_price ASC;

-- 2. เรียงตามเงินเดือนจากมากไปน้อย
SELECT first_name, salary FROM employees ORDER BY salary DESC;

-- 3. เรียงตาม hire_date ล่าสุดก่อน
SELECT first_name, hire_date FROM employees ORDER BY hire_date DESC;

-- 4. เรียงตาม last_name แล้ว first_name
SELECT last_name, first_name FROM employees ORDER BY last_name, first_name;

-- 5. เรียง departments ตาม budget มากไปน้อย
SELECT department_name, budget FROM departments ORDER BY budget DESC;

-- 6. เรียง products ตาม category แล้วราคา
SELECT category, product_name, unit_price FROM products 
ORDER BY category ASC, unit_price DESC;

-- 7. เรียง orders ตาม total_amount มากไปน้อย
SELECT order_id, total_amount FROM orders ORDER BY total_amount DESC;

-- 8. เรียง customers ตาม city
SELECT first_name, city FROM customers ORDER BY city ASC;

-- 9. เรียงตาม calculated value
SELECT product_name, unit_price * units_in_stock AS value
FROM products ORDER BY value DESC;

-- 10. ORDER BY กับ WHERE
SELECT first_name, salary FROM employees
WHERE department_id = 1
ORDER BY salary DESC;

-- 11. เรียงตาม LENGTH ของชื่อ
SELECT product_name, LENGTH(product_name) AS len
FROM products ORDER BY LENGTH(product_name);

-- 12. เรียงตาม alias
SELECT product_name, unit_price * 1.07 AS vat_price
FROM products ORDER BY vat_price;

-- 13. เรียง orders ตาม status แล้ว date
SELECT order_id, status, order_date FROM orders
ORDER BY status, order_date DESC;

-- 14. NULLS LAST (PostgreSQL)
SELECT first_name, manager_id FROM employees
ORDER BY manager_id ASC NULLS LAST;

-- 15. Custom sort ด้วย CASE
SELECT order_id, status FROM orders
ORDER BY CASE status 
    WHEN 'Processing' THEN 1 
    WHEN 'Pending' THEN 2 
    WHEN 'Delivered' THEN 3 
    ELSE 4 END;

-- 16. เรียง products ในหมวด Electronics ตามราคา
SELECT product_name, unit_price FROM products
WHERE category = 'Electronics'
ORDER BY unit_price;

-- 17. Top 3 เงินเดือนสูงสุด
SELECT first_name, salary FROM employees
ORDER BY salary DESC LIMIT 3;

-- 18. เรียง alphabetically case-insensitive
SELECT product_name FROM products
ORDER BY LOWER(product_name);

-- 19. เรียงตาม year เข้าทำงาน
SELECT first_name, hire_date FROM employees
-- SQLite:
ORDER BY CAST(strftime('%Y', hire_date) AS INTEGER) DESC;
-- PostgreSQL: ORDER BY EXTRACT(YEAR FROM hire_date) DESC;

-- 20. เรียงตาม stock percentage
SELECT product_name, units_in_stock, 
       units_in_stock * 100.0 / 500 AS pct
FROM products
ORDER BY pct DESC;

-- 21. เรียง employees ตาม department แล้ว seniority
SELECT first_name, department_id, hire_date FROM employees
ORDER BY department_id ASC, hire_date ASC;

-- 22. เรียง orders ตาม required_date ที่ใกล้จะมาถึง
SELECT order_id, required_date, status FROM orders
WHERE status NOT IN ('Delivered', 'Cancelled')
ORDER BY required_date ASC;

-- 23. เรียง customers ตาม company แล้วชื่อ
SELECT company_name, first_name, last_name FROM customers
ORDER BY 
    CASE WHEN company_name IS NULL THEN 1 ELSE 0 END,
    company_name, last_name;

-- 24. เรียง products ตาม price range
SELECT product_name, unit_price,
    CASE 
        WHEN unit_price < 500   THEN '1-Under 500'
        WHEN unit_price < 2000  THEN '2-Under 2000'
        WHEN unit_price < 10000 THEN '3-Under 10000'
        ELSE '4-Premium'
    END AS price_range
FROM products
ORDER BY price_range, unit_price;

-- 25. เรียงตาม salary กลุ่ม แล้วชื่อ
SELECT first_name, salary,
    CASE 
        WHEN salary >= 80000 THEN 'A-Senior Executive'
        WHEN salary >= 60000 THEN 'B-Senior'
        WHEN salary >= 40000 THEN 'C-Mid-level'
        ELSE 'D-Junior'
    END AS pay_group
FROM employees
ORDER BY pay_group, salary DESC;
```

---

## 5.12 สรุปบทที่ 5

```
✅ ORDER BY syntax
✅ ASC = จากน้อยไปมาก (default)
✅ DESC = จากมากไปน้อย
✅ Multiple columns: เรียง column แรกก่อน
✅ ORDER BY expressions, functions, aliases
✅ CASE ใน ORDER BY สำหรับ custom sort order
✅ NULLS FIRST / NULLS LAST (PostgreSQL, SQLite 3.30+)
✅ Workaround สำหรับ MySQL ด้วย CASE
✅ Performance: INDEX ช่วย ORDER BY
✅ LIMIT + ORDER BY สำหรับ top N queries
```

---

## 📝 แบบฝึกหัดท้ายบท

**คำถาม 1:** เรียงสินค้าจากราคาถูกไปแพง

**คำถาม 2:** เรียงพนักงานตาม hire_date จากใหม่ไปเก่า

**คำถาม 3:** เรียงสินค้าตาม category ก่อน แล้วในแต่ละ category เรียงตาม unit_price จากแพงไปถูก

**คำถาม 4:** หา top 5 สินค้าที่มี inventory value สูงสุด

**คำถาม 5:** เรียงคำสั่งซื้อโดยที่ Processing มาก่อน Pending มาก่อน Delivered มาก่อน Cancelled

**คำถาม 6:** เรียงพนักงานตาม department_id และ salary DESC แต่ให้ NULL manager_id ไปท้ายสุด

**คำถาม 7:** สร้าง report พนักงาน 10 คนที่มีเงินเดือนสูงสุด มีคอลัมน์: rank (ลำดับ), name, job_title, salary

**คำถาม 8:** เรียงสินค้าตาม "urgency" โดย Out of Stock → Low Stock → Medium → High

**คำถาม 9:** เรียง customers โดย company customers มาก่อน individual customers แต่ละกลุ่มเรียงตาม last_name

**คำถาม 10:** เรียงพนักงานตาม department เรียง A-Z แล้วตาม salary DESC

---

## ✅ เฉลยแบบฝึกหัด

### เฉลยที่ 1:
```sql
SELECT product_name, unit_price
FROM products
ORDER BY unit_price ASC;
```

### เฉลยที่ 2:
```sql
SELECT first_name, last_name, hire_date
FROM employees
ORDER BY hire_date DESC;
```

### เฉลยที่ 3:
```sql
SELECT category, product_name, unit_price
FROM products
ORDER BY category ASC, unit_price DESC;
```

### เฉลยที่ 4:
```sql
SELECT product_name, unit_price, units_in_stock,
       unit_price * units_in_stock AS inventory_value
FROM products
ORDER BY inventory_value DESC
LIMIT 5;
```

### เฉลยที่ 5:
```sql
SELECT order_id, status, total_amount
FROM orders
ORDER BY 
    CASE status
        WHEN 'Processing' THEN 1
        WHEN 'Pending'    THEN 2
        WHEN 'Delivered'  THEN 3
        WHEN 'Cancelled'  THEN 4
        ELSE 5
    END,
    total_amount DESC;
```

### เฉลยที่ 6:
```sql
-- PostgreSQL/SQLite
SELECT first_name, department_id, salary, manager_id
FROM employees
ORDER BY department_id ASC, salary DESC, manager_id ASC NULLS LAST;

-- MySQL (workaround)
SELECT first_name, department_id, salary, manager_id
FROM employees
ORDER BY 
    department_id ASC,
    salary DESC,
    CASE WHEN manager_id IS NULL THEN 1 ELSE 0 END,
    manager_id ASC;
```

### เฉลยที่ 7:
```sql
SELECT 
    ROW_NUMBER() OVER (ORDER BY salary DESC) AS rank_no,
    first_name || ' ' || last_name AS full_name,
    job_title,
    salary
FROM employees
ORDER BY salary DESC
LIMIT 10;
```

### เฉลยที่ 8:
```sql
SELECT product_name, units_in_stock,
    CASE 
        WHEN units_in_stock = 0  THEN 'Out of Stock'
        WHEN units_in_stock < 30 THEN 'Low Stock'
        WHEN units_in_stock < 100 THEN 'Medium Stock'
        ELSE 'High Stock'
    END AS stock_status
FROM products
ORDER BY
    CASE 
        WHEN units_in_stock = 0  THEN 1
        WHEN units_in_stock < 30 THEN 2
        WHEN units_in_stock < 100 THEN 3
        ELSE 4
    END ASC,
    units_in_stock ASC;
```

### เฉลยที่ 9:
```sql
SELECT 
    company_name,
    first_name,
    last_name,
    city
FROM customers
ORDER BY 
    CASE WHEN company_name IS NULL THEN 1 ELSE 0 END ASC,
    last_name ASC,
    first_name ASC;
```

### เฉลยที่ 10:
```sql
SELECT first_name, last_name, department_id, salary
FROM employees
WHERE department_id IN (
    SELECT department_id FROM departments ORDER BY department_name
)
ORDER BY 
    (SELECT department_name FROM departments d WHERE d.department_id = employees.department_id) ASC,
    salary DESC;

-- หรือง่ายกว่า (ถ้ายังไม่รู้ JOIN):
SELECT first_name, last_name, department_id, salary
FROM employees
ORDER BY department_id ASC, salary DESC;
```

---

## ➡️ บทถัดไป

**[Part 006: LIMIT, OFFSET, and Pagination](part-006.md)**

ในบทถัดไปเราจะเรียนรู้การจำกัดผลลัพธ์และการสร้างระบบ pagination!

---

*Part 005 of 120 | หลักสูตร SQL ครบวงจร*
