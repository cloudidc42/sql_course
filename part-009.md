# Part 009: DISTINCT - Removing Duplicates

> **หลักสูตร SQL ครบวงจร | Part 9 of 120**

---

## 🎯 สิ่งที่จะได้เรียนรู้ในบทนี้

- SELECT DISTINCT syntax
- DISTINCT กับ column เดียว
- DISTINCT กับหลาย columns
- COUNT DISTINCT
- DISTINCT vs GROUP BY
- Performance ของ DISTINCT
- สถานการณ์จริงที่ใช้ DISTINCT

**เวลาที่ใช้เรียน**: ประมาณ 1.5 ชั่วโมง

---

## ฐานข้อมูลที่ใช้

```sql
-- ยืนยันข้อมูล
SELECT COUNT(*) FROM employees;  -- 15
SELECT COUNT(*) FROM products;   -- 15
SELECT COUNT(*) FROM orders;     -- 10
SELECT COUNT(*) FROM customers;  -- 10
```

---

## 9.1 SELECT DISTINCT พื้นฐาน

```sql
-- ============================================================
-- EXAMPLE 1: ปัญหาของการเห็น Duplicates
-- ============================================================

-- ดู categories ของสินค้าทั้งหมด
SELECT category FROM products;

-- ผลลัพธ์ (มี duplicates):
-- category
-- -----------
-- Electronics  ← ซ้ำ
-- Electronics  ← ซ้ำ
-- Electronics  ← ซ้ำ
-- Furniture
-- Furniture
-- Books
-- Books
-- Electronics
-- Electronics
-- Electronics
-- Electronics
-- Furniture
-- Stationery
-- Stationery
-- Stationery

-- ============================================================
-- EXAMPLE 2: DISTINCT ลบ Duplicates
-- ============================================================

SELECT DISTINCT category FROM products;

-- ผลลัพธ์ (unique values เท่านั้น):
-- category
-- -----------
-- Books
-- Electronics
-- Furniture
-- Stationery

-- ============================================================
-- EXAMPLE 3: DISTINCT กับ Text
-- ============================================================

-- ดู job titles ทั้งหมด (unique)
SELECT DISTINCT job_title
FROM employees
ORDER BY job_title;

-- ผลลัพธ์:
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
-- EXAMPLE 4: DISTINCT กับ Numbers
-- ============================================================

-- ดู department_id ที่มีพนักงาน
SELECT DISTINCT department_id
FROM employees
ORDER BY department_id;

-- ผลลัพธ์:
-- department_id
-- -----------
-- 1
-- 2
-- 3
-- 4
-- 5
-- 6
-- 7

-- ============================================================
-- EXAMPLE 5: DISTINCT กับ Status
-- ============================================================

-- ดู order statuses ที่มีใช้จริง
SELECT DISTINCT status
FROM orders;

-- ผลลัพธ์:
-- status
-- ----------
-- Cancelled
-- Delivered
-- Pending
-- Processing

-- ============================================================
-- EXAMPLE 6: DISTINCT กับ Cities
-- ============================================================

-- ดู cities ที่มีลูกค้า
SELECT DISTINCT city
FROM customers
ORDER BY city;

-- ผลลัพธ์:
-- city
-- -------------------
-- Bangkok
-- Chiang Mai
-- Chiang Rai
-- Khon Kaen
-- Nakhon Ratchasima
-- Phuket
```

---

## 9.2 DISTINCT กับหลาย Columns

```sql
-- ============================================================
-- EXAMPLE 7: DISTINCT กับ 2 columns
-- ============================================================

-- ดู combination ที่ unique ของ category + price range
SELECT DISTINCT 
    category,
    CASE 
        WHEN unit_price < 1000  THEN 'Budget'
        WHEN unit_price < 10000 THEN 'Mid-range'
        ELSE 'Premium'
    END AS price_range
FROM products
ORDER BY category, price_range;

-- ผลลัพธ์:
-- category    | price_range
-- ------------|----------
-- Books       | Budget
-- Electronics | Budget
-- Electronics | Mid-range
-- Electronics | Premium
-- Furniture   | Mid-range
-- Furniture   | Premium
-- Stationery  | Budget

-- ============================================================
-- EXAMPLE 8: DISTINCT กับ department + job_title
-- ============================================================

SELECT DISTINCT department_id, job_title
FROM employees
ORDER BY department_id, job_title;

-- ผลลัพธ์:
-- department_id | job_title
-- --------------|------------------
-- 1             | Data Analyst
-- 1             | Developer
-- 1             | Junior Developer
-- 1             | Senior Developer
-- 2             | HR Manager
-- 2             | HR Specialist
-- 3             | CFO
-- 3             | Senior Accountant
-- 4             | Content Creator
-- 4             | Marketing Manager
-- 5             | Sales Director
-- 5             | Sales Executive
-- 5             | Senior Sales
-- 6             | Operations Manager
-- 7             | R&D Lead

-- ============================================================
-- EXAMPLE 9: DISTINCT กับ NULL
-- ============================================================

-- DISTINCT นับ NULL เป็น 1 unique value
SELECT DISTINCT manager_id
FROM employees
ORDER BY manager_id NULLS LAST;

-- ผลลัพธ์:
-- manager_id
-- ----------
-- 1
-- 2
-- 3
-- 4
-- 5
-- NULL   ← NULL ถือเป็น 1 unique value

-- ============================================================
-- EXAMPLE 10: DISTINCT ORDER matters
-- ============================================================

-- DISTINCT ต้องอยู่ตรงหลัง SELECT เสมอ
-- ❌ ผิด:
-- SELECT first_name, DISTINCT department_id FROM employees;

-- ✓ ถูก: DISTINCT apply กับทุก columns ใน SELECT
SELECT DISTINCT first_name, department_id FROM employees;
```

---

## 9.3 COUNT DISTINCT

```sql
-- ============================================================
-- EXAMPLE 11: COUNT DISTINCT
-- ============================================================

-- นับจำนวน unique categories
SELECT COUNT(DISTINCT category) AS unique_categories
FROM products;
-- ผลลัพธ์: 4

-- นับจำนวน unique job titles
SELECT COUNT(DISTINCT job_title) AS unique_job_titles
FROM employees;
-- ผลลัพธ์: 15

-- นับ unique cities ที่มีลูกค้า
SELECT COUNT(DISTINCT city) AS unique_cities
FROM customers;
-- ผลลัพธ์: 6

-- ============================================================
-- EXAMPLE 12: COUNT DISTINCT กับ NULL
-- ============================================================

-- COUNT DISTINCT ไม่นับ NULL!
SELECT 
    COUNT(manager_id) AS count_non_null,          -- 8
    COUNT(DISTINCT manager_id) AS count_distinct_non_null  -- 5 (unique values: 1,2,3,4,5)
FROM employees;

-- ผลลัพธ์:
-- count_non_null | count_distinct_non_null
-- ---------------|------------------------
-- 8              | 5

-- ============================================================
-- EXAMPLE 13: COUNT DISTINCT ใน Business Context
-- ============================================================

-- กี่ customers ที่มี orders?
SELECT COUNT(DISTINCT customer_id) AS customers_with_orders
FROM orders;

-- ผลลัพธ์: 9 (จาก 10 customers, customer 10 ยังไม่มี order)

-- กี่ employees ที่ handle orders?
SELECT COUNT(DISTINCT employee_id) AS employees_with_orders
FROM orders;
-- ผลลัพธ์: 3

-- กี่ products ที่ถูกสั่งซื้อแล้ว?
SELECT COUNT(DISTINCT product_id) AS products_ordered
FROM order_items;
-- ผลลัพธ์: 11

-- ============================================================
-- EXAMPLE 14: Multiple COUNT DISTINCT
-- ============================================================

SELECT 
    COUNT(*) AS total_orders,
    COUNT(DISTINCT customer_id) AS unique_customers,
    COUNT(DISTINCT employee_id) AS unique_salespeople,
    COUNT(DISTINCT status) AS unique_statuses
FROM orders;

-- ผลลัพธ์:
-- total_orders | unique_customers | unique_salespeople | unique_statuses
-- -------------|------------------|--------------------|-----------------
-- 10           | 9                | 3                  | 4
```

---

## 9.4 DISTINCT vs GROUP BY

```sql
-- ============================================================
-- EXAMPLE 15: เปรียบเทียบ DISTINCT กับ GROUP BY
-- ============================================================

-- DISTINCT: ต้องการ unique values เท่านั้น
SELECT DISTINCT category
FROM products
ORDER BY category;

-- GROUP BY: เหมือนกัน (ถ้าไม่มี aggregate)
SELECT category
FROM products
GROUP BY category
ORDER BY category;

-- ผลลัพธ์เหมือนกัน!

-- ============================================================
-- EXAMPLE 16: เมื่อไหร่ควรใช้ GROUP BY แทน DISTINCT
-- ============================================================

-- ถ้าต้องการ aggregate → ต้องใช้ GROUP BY
SELECT category, COUNT(*) AS product_count
FROM products
GROUP BY category;

-- ผลลัพธ์:
-- category    | product_count
-- ------------|---------------
-- Books       | 2
-- Electronics | 7
-- Furniture   | 3
-- Stationery  | 3

-- ============================================================
-- EXAMPLE 17: DISTINCT + JOIN ต้องระวัง
-- ============================================================

-- ปัญหา: JOIN ทำให้มี duplicates
SELECT p.category
FROM order_items oi
JOIN products p ON oi.product_id = p.product_id;
-- อาจมี category ซ้ำเพราะสินค้าใน category เดียวกันถูกสั่งหลายครั้ง

-- แก้ด้วย DISTINCT
SELECT DISTINCT p.category
FROM order_items oi
JOIN products p ON oi.product_id = p.product_id
ORDER BY category;

-- ============================================================
-- EXAMPLE 18: Performance DISTINCT vs GROUP BY
-- ============================================================

/*
ทั่วไป:
- DISTINCT และ GROUP BY (ไม่มี aggregate) ให้ผลเหมือนกัน
- PostgreSQL มักใช้ HashAggregate algorithm สำหรับทั้งสอง
- Query planner อาจ optimize ต่างกัน

Best practice:
- DISTINCT สำหรับ unique values อย่างเดียว
- GROUP BY เมื่อต้องการ aggregate

Performance test (concept):
EXPLAIN SELECT DISTINCT category FROM products;
EXPLAIN SELECT category FROM products GROUP BY category;
*/
```

---

## 9.5 DISTINCT ในสถานการณ์จริง

```sql
-- ============================================================
-- EXAMPLE 19: Unique Values สำหรับ Dropdown Menu
-- ============================================================

-- ดึง categories สำหรับ dropdown filter ใน UI
SELECT DISTINCT category
FROM products
WHERE discontinued = FALSE
ORDER BY category;

-- ผลลัพธ์: ใช้เป็น options ใน dropdown

-- ============================================================
-- EXAMPLE 20: Data Quality Check
-- ============================================================

-- ตรวจสอบว่า statuses มีอะไรบ้าง (หา invalid values)
SELECT DISTINCT status, COUNT(*) AS count
FROM orders
GROUP BY status
ORDER BY status;

-- ถ้าเจอ status ที่ไม่คาดคิด เช่น 'Completed', 'DELIVERED' → data quality issue

-- ============================================================
-- EXAMPLE 21: Active Users
-- ============================================================

-- ดู customers ที่มี orders ในเดือน March 2024
SELECT DISTINCT customer_id
FROM orders
WHERE order_date >= '2024-03-01'
  AND order_date < '2024-04-01';

-- ============================================================
-- EXAMPLE 22: Product Inventory Snapshot
-- ============================================================

-- ดู suppliers/categories ที่ต้องการ restock
SELECT DISTINCT category
FROM products
WHERE units_in_stock < 30
  AND discontinued = FALSE
ORDER BY category;

-- ============================================================
-- EXAMPLE 23: Employee Skills/Roles
-- ============================================================

-- ดูว่ามี job titles อะไรบ้างในบริษัท
SELECT DISTINCT job_title
FROM employees
WHERE is_active = TRUE
ORDER BY job_title;

-- ============================================================
-- EXAMPLE 24: Reporting Regions
-- ============================================================

-- ดู cities ทั้งหมดที่มีลูกค้า
SELECT DISTINCT city, country
FROM customers
WHERE is_active = TRUE
ORDER BY country, city;

-- ============================================================
-- EXAMPLE 25: Find Duplicates (ตรงข้ามกับ DISTINCT)
-- ============================================================

-- หาค่าที่ซ้ำกัน (duplicate detection)
SELECT 
    department_id,
    job_title,
    COUNT(*) AS count
FROM employees
GROUP BY department_id, job_title
HAVING COUNT(*) > 1  -- มีมากกว่า 1 → ซ้ำ
ORDER BY count DESC;

-- ผลลัพธ์ (ตัวอย่าง):
-- department_id | job_title | count
-- --------------|-----------|------
-- 1             | Developer | 2  ← มีสองคนที่ title เหมือนกัน
-- ...
```

---

## 9.6 DISTINCT กับ Expressions

```sql
-- ============================================================
-- EXAMPLE 26: DISTINCT กับ Expressions
-- ============================================================

-- ดู salary ranges ที่มีอยู่ (หน่วยหมื่น)
SELECT DISTINCT 
    ROUND(salary / 10000) * 10000 AS salary_bracket
FROM employees
ORDER BY salary_bracket;

-- ผลลัพธ์:
-- salary_bracket
-- ---------------
-- 40000
-- 50000
-- 60000
-- 70000
-- 80000
-- 90000
-- 120000

-- ============================================================
-- EXAMPLE 27: DISTINCT กับ date parts
-- ============================================================

-- ดู years ที่มีการจ้างพนักงาน
-- SQLite:
SELECT DISTINCT strftime('%Y', hire_date) AS hire_year
FROM employees
ORDER BY hire_year;

-- PostgreSQL:
-- SELECT DISTINCT EXTRACT(YEAR FROM hire_date) AS hire_year
-- FROM employees ORDER BY hire_year;

-- ผลลัพธ์:
-- hire_year
-- ---------
-- 2016
-- 2017
-- 2018
-- 2019
-- 2020
-- 2021
-- 2022
-- 2023

-- ============================================================
-- EXAMPLE 28: DISTINCT กับ LOWER (case-insensitive)
-- ============================================================

-- ป้องกัน duplicate ที่เกิดจาก case ต่างกัน
SELECT DISTINCT LOWER(category) AS category_lower
FROM products
ORDER BY category_lower;
```

---

## 9.7 DISTINCT ALL ต่างกับ DISTINCT

```sql
-- ============================================================
-- EXAMPLE 29: SELECT ALL (default)
-- ============================================================

-- SELECT ALL เป็น default = SELECT (ไม่ใช้ DISTINCT)
SELECT ALL category FROM products;
-- เหมือนกับ
SELECT category FROM products;

-- ดังนั้น:
-- SELECT DISTINCT = ลบ duplicates
-- SELECT ALL = เก็บทั้งหมด (default)
-- SELECT = เหมือน SELECT ALL

-- ============================================================
-- EXAMPLE 30: เมื่อ DISTINCT ไม่จำเป็น
-- ============================================================

-- ❌ ไม่จำเป็น: SELECT DISTINCT บน PRIMARY KEY
SELECT DISTINCT employee_id FROM employees;
-- employee_id เป็น PK → unique เสมอ → DISTINCT ไม่มีผล

-- ❌ ไม่จำเป็น: SELECT DISTINCT บน UNIQUE column
SELECT DISTINCT email FROM employees;
-- email เป็น UNIQUE constraint → ไม่มี duplicate แน่นอน

-- ✓ จำเป็น: SELECT DISTINCT บน non-unique columns
SELECT DISTINCT department_id FROM employees;
-- department_id ซ้ำได้ → DISTINCT มีประโยชน์
```

---

## 9.8 20+ ตัวอย่างครบถ้วน

```sql
-- 1. ดู categories ทั้งหมด
SELECT DISTINCT category FROM products ORDER BY category;

-- 2. ดู job titles ทั้งหมด
SELECT DISTINCT job_title FROM employees ORDER BY job_title;

-- 3. ดู statuses ของ orders
SELECT DISTINCT status FROM orders;

-- 4. ดู cities ของ customers
SELECT DISTINCT city FROM customers ORDER BY city;

-- 5. ดู departments ที่มีพนักงาน
SELECT DISTINCT department_id FROM employees ORDER BY department_id;

-- 6. ดู managers (employees ที่มีคนรายงาน)
SELECT DISTINCT manager_id FROM employees WHERE manager_id IS NOT NULL ORDER BY manager_id;

-- 7. ดู years ที่มีคำสั่งซื้อ
SELECT DISTINCT strftime('%Y', order_date) AS year FROM orders;

-- 8. นับ unique categories
SELECT COUNT(DISTINCT category) FROM products;

-- 9. นับ unique customers ที่มี orders
SELECT COUNT(DISTINCT customer_id) FROM orders;

-- 10. ดู distinct (department, job_title) pairs
SELECT DISTINCT department_id, job_title FROM employees ORDER BY 1, 2;

-- 11. ดู unique salary ranges
SELECT DISTINCT 
    CASE WHEN salary < 50000 THEN 'Under 50K'
         WHEN salary < 80000 THEN '50K-80K'
         ELSE 'Over 80K' END AS range
FROM employees;

-- 12. DISTINCT กับ boolean
SELECT DISTINCT is_active FROM employees;

-- 13. DISTINCT กับ expression
SELECT DISTINCT LENGTH(first_name) AS name_len FROM employees ORDER BY 1;

-- 14. Unique employee names
SELECT DISTINCT first_name FROM employees ORDER BY first_name;

-- 15. Products with orders
SELECT DISTINCT product_id FROM order_items ORDER BY product_id;

-- 16. Products without orders (NOT IN)
SELECT DISTINCT product_id FROM products
WHERE product_id NOT IN (SELECT DISTINCT product_id FROM order_items);

-- 17. Customers without orders
SELECT customer_id FROM customers
WHERE customer_id NOT IN (SELECT DISTINCT customer_id FROM orders);

-- 18. Unique hire years and count
SELECT strftime('%Y', hire_date) AS year, COUNT(*) AS count
FROM employees
GROUP BY year ORDER BY year;

-- 19. DISTINCT vs GROUP BY comparison
-- Same result:
SELECT DISTINCT category FROM products;
SELECT category FROM products GROUP BY category;

-- 20. Unique product + category in orders
SELECT DISTINCT p.category
FROM order_items oi
JOIN products p ON oi.product_id = p.product_id;
```

---

## 9.9 สรุปบทที่ 9

```
✅ DISTINCT ลบ duplicate rows
✅ DISTINCT apply กับทุก columns ที่ SELECT
✅ NULL ถือเป็น 1 unique value
✅ COUNT DISTINCT = นับ unique values (ไม่นับ NULL)
✅ DISTINCT vs GROUP BY: ผลเหมือนกันถ้าไม่ aggregate
✅ เมื่อต้องการ aggregate → GROUP BY
✅ Performance: DISTINCT + ORDER BY ต้องการ sort
✅ ใช้ DISTINCT เมื่อ duplicates เป็น expected behavior (JOIN, etc.)
```

---

## 📝 แบบฝึกหัดท้ายบท

**คำถาม 1:** แสดง unique job titles ในแผนก IT

**คำถาม 2:** นับว่ามี unique categories กี่ประเภทในตาราง products

**คำถาม 3:** แสดง unique cities ที่มีลูกค้า เรียงตาม alphabet

**คำถาม 4:** หาว่า customer IDs ใดบ้างที่มี orders (ใช้ DISTINCT)

**คำถาม 5:** แสดง unique (category, status) pairs ของ products

**คำถาม 6:** นับว่ามีกี่ products ที่ถูกสั่งซื้อแล้ว (ใช้ COUNT DISTINCT บน order_items)

**คำถาม 7:** หา departments ที่มี salary range ต่างๆ (เงินเดือน: Under 50K, 50K-80K, Over 80K)

**คำถาม 8:** แสดงว่า DISTINCT กับ GROUP BY ให้ผลเหมือนกันหรือไม่

**คำถาม 9:** หา employee IDs ที่เป็น manager ด้วย SELECT DISTINCT

**คำถาม 10:** สร้าง summary report ด้วย COUNT DISTINCT ของ: unique customers, products, categories ใน orders

---

## ✅ เฉลยแบบฝึกหัด

### เฉลยที่ 1:
```sql
SELECT DISTINCT job_title
FROM employees
WHERE department_id = 1
ORDER BY job_title;
```

### เฉลยที่ 2:
```sql
SELECT COUNT(DISTINCT category) AS unique_categories
FROM products;
-- Result: 4
```

### เฉลยที่ 3:
```sql
SELECT DISTINCT city
FROM customers
ORDER BY city ASC;
```

### เฉลยที่ 4:
```sql
SELECT DISTINCT customer_id
FROM orders
ORDER BY customer_id;
```

### เฉลยที่ 5:
```sql
SELECT DISTINCT 
    category,
    CASE 
        WHEN discontinued = TRUE THEN 'Discontinued'
        WHEN units_in_stock = 0 THEN 'Out of Stock'
        ELSE 'Active'
    END AS status
FROM products
ORDER BY category, status;
```

### เฉลยที่ 6:
```sql
SELECT COUNT(DISTINCT product_id) AS products_ordered
FROM order_items;
-- Result: จำนวน unique products ที่มีในคำสั่งซื้อ
```

### เฉลยที่ 7:
```sql
SELECT DISTINCT
    department_id,
    CASE 
        WHEN salary < 50000 THEN 'Under 50K'
        WHEN salary < 80000 THEN '50K-80K'
        ELSE 'Over 80K'
    END AS salary_range
FROM employees
ORDER BY department_id, salary_range;
```

### เฉลยที่ 8:
```sql
-- ทั้งสองให้ผลเหมือนกัน:
SELECT DISTINCT category FROM products ORDER BY category;
SELECT category FROM products GROUP BY category ORDER BY category;
-- ผลลัพธ์: Books, Electronics, Furniture, Stationery (เหมือนกัน)
```

### เฉลยที่ 9:
```sql
SELECT DISTINCT manager_id AS manager_employee_id
FROM employees
WHERE manager_id IS NOT NULL
ORDER BY manager_id;
```

### เฉลยที่ 10:
```sql
SELECT 
    COUNT(DISTINCT o.customer_id) AS unique_customers,
    COUNT(DISTINCT oi.product_id) AS unique_products_ordered,
    COUNT(DISTINCT p.category) AS unique_categories_ordered,
    COUNT(DISTINCT o.order_id) AS total_orders
FROM orders o
JOIN order_items oi ON o.order_id = oi.order_id
JOIN products p ON oi.product_id = p.product_id
WHERE o.status <> 'Cancelled';
```

---

## ➡️ บทถัดไป

**[Part 010: String Functions Deep Dive](part-010.md)**

ในบทถัดไปเราจะเรียนรู้ String Functions อย่างครบถ้วน รวมถึง cross-database differences!

---

*Part 009 of 120 | หลักสูตร SQL ครบวงจร*
