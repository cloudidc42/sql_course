# Part 006: LIMIT, OFFSET, and Pagination

> **หลักสูตร SQL ครบวงจร | Part 6 of 120**

---

## 🎯 สิ่งที่จะได้เรียนรู้ในบทนี้

- LIMIT / TOP / FETCH FIRST syntax
- OFFSET สำหรับข้ามแถว
- Pagination patterns (เลข page)
- Cursor-based pagination
- Performance considerations
- Cross-database syntax differences

**เวลาที่ใช้เรียน**: ประมาณ 1.5 ชั่วโมง

---

## ฐานข้อมูลที่ใช้ในบทนี้

```sql
-- ตรวจสอบข้อมูล
SELECT COUNT(*) FROM employees;  -- 15
SELECT COUNT(*) FROM products;   -- 15
SELECT COUNT(*) FROM orders;     -- 10
```

---

## 6.1 LIMIT ใน SQLite / PostgreSQL / MySQL

```sql
-- ============================================================
-- EXAMPLE 1: LIMIT พื้นฐาน
-- ============================================================

-- ดู 5 สินค้าแรก
SELECT product_name, unit_price
FROM products
LIMIT 5;

-- ผลลัพธ์:
-- product_name     | unit_price
-- -----------------|----------
-- Laptop Pro 15"   | 45000.00
-- Wireless Mouse   | 850.00
-- USB-C Hub        | 2500.00
-- Office Chair Premium | 8900.00
-- Standing Desk    | 15000.00

-- ============================================================
-- EXAMPLE 2: LIMIT + ORDER BY (สำคัญมาก!)
-- ============================================================

-- ❌ LIMIT ไม่มี ORDER BY = ผลไม่แน่นอน
SELECT product_name FROM products LIMIT 5;

-- ✓ LIMIT + ORDER BY = ผลแน่นอน
SELECT product_name, unit_price
FROM products
ORDER BY unit_price ASC
LIMIT 5;

-- ผลลัพธ์ (5 สินค้าถูกที่สุด):
-- product_name              | unit_price
-- --------------------------|----------
-- Pen Set                   | 120.00
-- Whiteboard A4             | 150.00
-- Notebook Premium          | 250.00
-- SQL Programming Book      | 450.00
-- Python for Data Science   | 550.00

-- ============================================================
-- EXAMPLE 3: Top N Queries
-- ============================================================

-- Top 3 เงินเดือนสูงสุด
SELECT first_name, last_name, salary
FROM employees
ORDER BY salary DESC
LIMIT 3;

-- Top 5 orders ยอดสูงสุด
SELECT order_id, customer_id, total_amount
FROM orders
ORDER BY total_amount DESC
LIMIT 5;

-- สินค้าที่ stock น้อยที่สุด 3 อันดับ
SELECT product_name, units_in_stock
FROM products
ORDER BY units_in_stock ASC
LIMIT 3;
```

---

## 6.2 TOP ใน SQL Server

```sql
-- ============================================================
-- SQL Server ใช้ TOP แทน LIMIT
-- ============================================================

-- SQL Server
SELECT TOP 5 product_name, unit_price
FROM products
ORDER BY unit_price ASC;

-- TOP กับ PERCENT
SELECT TOP 10 PERCENT first_name, salary
FROM employees
ORDER BY salary DESC;
-- 10% ของ 15 พนักงาน = 1-2 คน

-- TOP WITH TIES
-- ถ้า rank ที่ N เท่ากันหลายคน จะดึงมาทั้งหมด
SELECT TOP 3 WITH TIES first_name, salary
FROM employees
ORDER BY salary DESC;
```

---

## 6.3 FETCH FIRST ใน SQL Standard (SQL:2008)

```sql
-- ============================================================
-- SQL Standard Syntax (รองรับ PostgreSQL, Oracle 12c+, DB2)
-- ============================================================

-- FETCH FIRST n ROWS ONLY
SELECT product_name, unit_price
FROM products
ORDER BY unit_price
FETCH FIRST 5 ROWS ONLY;

-- หรือ FETCH NEXT (เหมือนกัน)
SELECT product_name, unit_price
FROM products
ORDER BY unit_price
FETCH NEXT 5 ROWS ONLY;

-- FETCH FIRST n ROW ONLY (ถ้า n = 1)
SELECT product_name
FROM products
ORDER BY unit_price DESC
FETCH FIRST 1 ROW ONLY;  -- สินค้าแพงที่สุด

-- กับ OFFSET
SELECT product_name, unit_price
FROM products
ORDER BY unit_price
OFFSET 5 ROWS FETCH NEXT 5 ROWS ONLY;  -- แถวที่ 6-10
```

---

## 6.4 OFFSET - ข้ามแถว

```sql
-- ============================================================
-- EXAMPLE 4: OFFSET พื้นฐาน
-- ============================================================

-- OFFSET 0 = เริ่มต้น (ไม่มีผล)
SELECT product_name, unit_price
FROM products
ORDER BY unit_price
LIMIT 5 OFFSET 0;  -- แถวที่ 1-5

-- OFFSET 5 = ข้าม 5 แถวแรก
SELECT product_name, unit_price
FROM products
ORDER BY unit_price
LIMIT 5 OFFSET 5;  -- แถวที่ 6-10

-- ผลลัพธ์:
-- product_name        | unit_price
-- --------------------|----------
-- Wireless Mouse      | 850.00   ← แถวที่ 6
-- Desk Lamp LED       | 1200.00
-- USB-C Hub           | 2500.00
-- Webcam HD           | 2800.00
-- Keyboard Mechanical | 3200.00  ← แถวที่ 10

-- ============================================================
-- EXAMPLE 5: OFFSET syntax ต่างๆ
-- ============================================================

-- SQLite / MySQL / PostgreSQL
LIMIT 5 OFFSET 10;   -- ข้าม 10 แถว แล้วเอา 5

-- MySQL shortcut
LIMIT 10, 5;         -- LIMIT offset, count (ลำดับสลับกัน!)

-- PostgreSQL / SQLite (ใช้ได้ทั้งสองแบบ)
LIMIT 5 OFFSET 10;
OFFSET 10 LIMIT 5;   -- ลำดับสลับก็ได้

-- ============================================================
-- EXAMPLE 6: Pagination Pattern
-- ============================================================

-- Page 1 (แถวที่ 1-5)
SELECT product_name, unit_price, category
FROM products
ORDER BY product_name
LIMIT 5 OFFSET 0;

-- Page 2 (แถวที่ 6-10)
SELECT product_name, unit_price, category
FROM products
ORDER BY product_name
LIMIT 5 OFFSET 5;

-- Page 3 (แถวที่ 11-15)
SELECT product_name, unit_price, category
FROM products
ORDER BY product_name
LIMIT 5 OFFSET 10;

-- สูตร: OFFSET = (page_number - 1) * page_size
-- Page 1: OFFSET 0  (1-1) * 5 = 0
-- Page 2: OFFSET 5  (2-1) * 5 = 5
-- Page 3: OFFSET 10 (3-1) * 5 = 10
-- Page n: OFFSET (n-1) * page_size
```

---

## 6.5 Pagination Pattern ในการทำงานจริง

```sql
-- ============================================================
-- EXAMPLE 7: Complete Pagination Query
-- ============================================================

-- Variables (ใช้ application code ส่งมา):
-- :page_size = 5
-- :page_number = 2

-- PostgreSQL / MySQL / SQLite
SELECT 
    product_id,
    product_name,
    category,
    unit_price
FROM products
ORDER BY product_name ASC
LIMIT 5 OFFSET 5;   -- OFFSET = (2-1) * 5 = 5

-- ============================================================
-- EXAMPLE 8: Pagination พร้อม Total Count
-- ============================================================

-- Query 1: Total count (สำหรับแสดง "Page 2 of 3")
SELECT COUNT(*) AS total_count
FROM products
WHERE category = 'Electronics';

-- Query 2: Data สำหรับหน้า 2
SELECT product_id, product_name, unit_price
FROM products
WHERE category = 'Electronics'
ORDER BY product_name
LIMIT 3 OFFSET 3;

-- ============================================================
-- EXAMPLE 9: Pagination ใน Real Application (Pseudocode)
-- ============================================================

/*
-- ใน application code (Python/Node/PHP):

page_size = 10
page_number = 3  -- user อยู่หน้า 3
offset = (page_number - 1) * page_size  -- = 20

-- SQL query:
SELECT *
FROM products
ORDER BY product_name
LIMIT 10 OFFSET 20;

-- ใน Python:
import sqlite3
conn = sqlite3.connect('company.db')
cur = conn.cursor()

page_size = 10
page_number = 3
offset = (page_number - 1) * page_size

cur.execute("""
    SELECT product_id, product_name, unit_price
    FROM products
    ORDER BY product_name
    LIMIT ? OFFSET ?
""", (page_size, offset))

rows = cur.fetchall()
*/

-- ============================================================
-- EXAMPLE 10: Pagination Information
-- ============================================================

-- ดู metadata ของ pagination
SELECT 
    COUNT(*) AS total_rows,
    CEIL(COUNT(*) * 1.0 / 5) AS total_pages,
    5 AS page_size
FROM products
WHERE category = 'Electronics';

-- ผลลัพธ์:
-- total_rows | total_pages | page_size
-- -----------|-------------|----------
-- 7          | 2           | 5
-- (7 / 5 = 1.4, CEIL = 2 → มี 2 หน้า)
```

---

## 6.6 Cursor-Based Pagination

```sql
-- ============================================================
-- EXAMPLE 11: ปัญหาของ OFFSET Pagination
-- ============================================================

-- ปัญหา: ถ้าข้อมูลเพิ่ม/ลด ระหว่างที่ user paginate
-- Page 1: LIMIT 5 OFFSET 0 → items 1-5
-- [ลบ item 1 ออก]
-- Page 2: LIMIT 5 OFFSET 5 → items 7-11 (ข้าม item 6!)

-- ============================================================
-- EXAMPLE 12: Cursor-Based Pagination
-- ============================================================

-- แทนที่จะใช้ OFFSET ใช้ "cursor" (last seen id หรือ timestamp)

-- Page 1: ดูแถวแรก
SELECT product_id, product_name, unit_price
FROM products
ORDER BY product_id ASC
LIMIT 5;

-- ผลลัพธ์ last product_id = 5

-- Page 2: ดูต่อจาก product_id = 5
SELECT product_id, product_name, unit_price
FROM products
WHERE product_id > 5  -- cursor = 5
ORDER BY product_id ASC
LIMIT 5;

-- ผลลัพธ์ last product_id = 10

-- Page 3:
SELECT product_id, product_name, unit_price
FROM products
WHERE product_id > 10  -- cursor = 10
ORDER BY product_id ASC
LIMIT 5;

-- ============================================================
-- EXAMPLE 13: Cursor Pagination กับ Non-Integer Field
-- ============================================================

-- ใช้ created_at timestamp เป็น cursor
SELECT order_id, order_date, total_amount
FROM orders
ORDER BY order_date DESC, order_id DESC
LIMIT 3;

-- Last record: order_date='2024-02-14', order_id=7

-- Page 2:
SELECT order_id, order_date, total_amount
FROM orders
WHERE (order_date, order_id) < ('2024-02-14', 7)  -- PostgreSQL
ORDER BY order_date DESC, order_id DESC
LIMIT 3;

-- SQLite workaround:
SELECT order_id, order_date, total_amount
FROM orders
WHERE order_date < '2024-02-14'
   OR (order_date = '2024-02-14' AND order_id < 7)
ORDER BY order_date DESC, order_id DESC
LIMIT 3;

-- ============================================================
-- EXAMPLE 14: Infinite Scroll Pattern
-- ============================================================

-- สำหรับ social media feed, infinite scroll:
-- แทน page numbers ใช้ "load more"

-- ครั้งแรก
SELECT order_id, customer_id, order_date, total_amount
FROM orders
ORDER BY order_date DESC
LIMIT 3;

-- ผลลัพธ์: order_id 10, 9, 8
-- last_order_id = 8, last_order_date = '2024-03-01'

-- "Load More" (cursor = order_id < 8)
SELECT order_id, customer_id, order_date, total_amount
FROM orders
WHERE order_id < 8
ORDER BY order_date DESC
LIMIT 3;

-- ผลลัพธ์: order_id 7, 6, 5
```

---

## 6.7 LIMIT OFFSET Performance

```sql
-- ============================================================
-- EXAMPLE 15: Performance ปัญหาของ OFFSET ขนาดใหญ่
-- ============================================================

-- ❌ OFFSET ขนาดใหญ่ช้ามาก
SELECT product_name FROM products
ORDER BY product_name
LIMIT 10 OFFSET 1000000;  -- อ่านและข้าม 1 ล้านแถว!

-- Database ต้องอ่านทุกแถวตั้งแต่ต้นจนถึง offset

-- ✓ ใช้ cursor-based แทน
SELECT product_name FROM products
WHERE product_id > 1000000  -- ข้ามด้วย index
ORDER BY product_id
LIMIT 10;

-- ============================================================
-- EXAMPLE 16: Keyset Pagination (ดีที่สุดสำหรับ performance)
-- ============================================================

-- สร้าง index บน columns ที่ใช้ sort + filter
CREATE INDEX idx_orders_date_id ON orders(order_date DESC, order_id DESC);

-- Query:
SELECT order_id, order_date, total_amount
FROM orders
WHERE order_date <= '2024-03-01' 
  AND order_id < 8
ORDER BY order_date DESC, order_id DESC
LIMIT 5;

-- ใช้ index → เร็วมาก แม้ข้อมูลเป็นล้าน rows
```

---

## 6.8 Cross-Database Syntax Reference

```sql
-- ============================================================
-- EXAMPLE 17: Syntax เปรียบเทียบ
-- ============================================================

-- ========== SQLite ==========
SELECT * FROM employees
LIMIT 5;

SELECT * FROM employees
LIMIT 5 OFFSET 10;

-- หรือ (MySQL style ก็ใช้ได้)
SELECT * FROM employees
LIMIT 10, 5;  -- offset, count

-- ========== PostgreSQL ==========
SELECT * FROM employees
LIMIT 5;

SELECT * FROM employees
LIMIT 5 OFFSET 10;

-- หรือ SQL standard
SELECT * FROM employees
OFFSET 10 ROWS FETCH NEXT 5 ROWS ONLY;

-- ========== MySQL / MariaDB ==========
SELECT * FROM employees
LIMIT 5;

SELECT * FROM employees
LIMIT 5 OFFSET 10;

-- MySQL shorthand
SELECT * FROM employees
LIMIT 10, 5;  -- offset, count (สลับจาก standard!)

-- ========== SQL Server ==========
SELECT TOP 5 * FROM employees;

SELECT TOP 5 PERCENT * FROM employees;

-- SQL Server 2012+
SELECT * FROM employees
ORDER BY employee_id
OFFSET 10 ROWS FETCH NEXT 5 ROWS ONLY;
-- ต้องมี ORDER BY เสมอ!

-- ========== Oracle (ก่อน 12c) ==========
SELECT * FROM (
    SELECT * FROM employees
    WHERE ROWNUM <= 5
)
ORDER BY salary DESC;

-- Oracle 12c+
SELECT * FROM employees
ORDER BY salary DESC
FETCH FIRST 5 ROWS ONLY;

-- ============================================================
-- EXAMPLE 18: Portable Pagination (ใช้ได้หลาย databases)
-- ============================================================

-- วิธีที่ 1: ROW_NUMBER() - ทำงานได้บน PostgreSQL, SQL Server, Oracle, MySQL 8+
SELECT * FROM (
    SELECT 
        ROW_NUMBER() OVER (ORDER BY product_name) AS rn,
        product_id,
        product_name,
        unit_price
    FROM products
) AS numbered_products
WHERE rn BETWEEN 6 AND 10;  -- Page 2 (rows 6-10)

-- วิธีที่ 2: LIMIT/OFFSET - ใช้ได้บน MySQL, PostgreSQL, SQLite
SELECT product_id, product_name, unit_price
FROM products
ORDER BY product_name
LIMIT 5 OFFSET 5;  -- Page 2
```

---

## 6.9 LIMIT ใน Subqueries

```sql
-- ============================================================
-- EXAMPLE 19: LIMIT ใน Subquery
-- ============================================================

-- ดูรายละเอียดของ top 3 orders ยอดสูงสุด
SELECT 
    o.order_id,
    o.customer_id,
    o.order_date,
    o.total_amount
FROM orders o
WHERE o.order_id IN (
    SELECT order_id 
    FROM orders 
    ORDER BY total_amount DESC 
    LIMIT 3
);

-- ============================================================
-- EXAMPLE 20: LIMIT กับ Aggregate
-- ============================================================

-- หา category ที่มีสินค้ามากที่สุด (top 2)
SELECT category, COUNT(*) AS product_count
FROM products
GROUP BY category
ORDER BY product_count DESC
LIMIT 2;

-- ผลลัพธ์:
-- category    | product_count
-- ------------|---------------
-- Electronics | 7
-- Stationery  | 3
```

---

## 6.10 ตัวอย่างครบถ้วน

```sql
-- 1. ดู 10 รายการแรก
SELECT * FROM employees LIMIT 10;

-- 2. ดู 3 สินค้าแพงที่สุด
SELECT product_name, unit_price FROM products
ORDER BY unit_price DESC LIMIT 3;

-- 3. ดู 5 สินค้าถูกที่สุด
SELECT product_name, unit_price FROM products
ORDER BY unit_price ASC LIMIT 5;

-- 4. Page 1 (rows 1-5)
SELECT * FROM products ORDER BY product_id LIMIT 5 OFFSET 0;

-- 5. Page 2 (rows 6-10)
SELECT * FROM products ORDER BY product_id LIMIT 5 OFFSET 5;

-- 6. Page 3 (rows 11-15)
SELECT * FROM products ORDER BY product_id LIMIT 5 OFFSET 10;

-- 7. Latest 5 orders
SELECT order_id, order_date, total_amount FROM orders
ORDER BY order_date DESC LIMIT 5;

-- 8. Top 5 employees by salary
SELECT first_name, salary FROM employees
ORDER BY salary DESC LIMIT 5;

-- 9. Bottom 5 employees by salary
SELECT first_name, salary FROM employees
ORDER BY salary ASC LIMIT 5;

-- 10. ข้ามแถวแรก
SELECT product_name FROM products
ORDER BY product_name LIMIT 10 OFFSET 1;

-- 11. Random sample (ไม่แน่นอน แต่ใช้ได้สำหรับ sample)
SELECT first_name, salary FROM employees
ORDER BY RANDOM() LIMIT 5;  -- SQLite/PostgreSQL

-- 12. Top 3 per group (ต้องใช้ Window Functions - preview)
SELECT * FROM (
    SELECT 
        *,
        ROW_NUMBER() OVER (PARTITION BY department_id ORDER BY salary DESC) AS rn
    FROM employees
) WHERE rn <= 2;

-- 13. Cursor-based: after employee_id 5
SELECT employee_id, first_name, salary FROM employees
WHERE employee_id > 5
ORDER BY employee_id LIMIT 5;

-- 14. Total pages calculator
SELECT 
    COUNT(*) AS total,
    CEIL(COUNT(*) * 1.0 / 5) AS total_pages
FROM products;

-- 15. Check if more pages exist
SELECT 
    CASE WHEN COUNT(*) > 5 THEN 'Has more pages' 
    ELSE 'Last page' END AS pagination_status
FROM products
WHERE product_id > 5;  -- after page 1
```

---

## 6.11 สรุปบทที่ 6

```
✅ LIMIT n - ดึงข้อมูล n แถว
✅ LIMIT n OFFSET m - ข้าม m แถวก่อน
✅ TOP n (SQL Server), FETCH FIRST n ROWS ONLY (SQL Standard)
✅ Pagination formula: OFFSET = (page-1) * page_size
✅ Cursor-based pagination - ดีกว่าสำหรับ large datasets
✅ LIMIT ควรใช้ร่วมกับ ORDER BY เสมอ
✅ Performance: OFFSET ใหญ่ = ช้า, ใช้ cursor แทน
✅ Cross-database differences
```

---

## 📝 แบบฝึกหัดท้ายบท

**คำถาม 1:** แสดง 5 สินค้าแพงที่สุด

**คำถาม 2:** แสดงสินค้า page 2 (แถวที่ 6-10) เรียงตาม product_name

**คำถาม 3:** แสดง 3 orders ล่าสุด

**คำถาม 4:** สร้าง pagination query สำหรับ employees หน้า 3 (page_size = 3)

**คำถาม 5:** นับ total pages ของ products ถ้า page size = 4

**คำถาม 6:** หา top 5 employees เงินเดือนสูงสุดในแผนก IT

**คำถาม 7:** สร้าง cursor-based pagination สำหรับ orders เรียงตาม order_date

**คำถาม 8:** แสดงสินค้าราคาต่ำสุด 3 อันดับในหมวด Electronics

**คำถาม 9:** หน้าสุดท้ายของ products (แถวที่ 11-15) เรียงตาม unit_price

**คำถาม 10:** สร้าง query ที่แสดง: total records, page_size, current_page, total_pages สำหรับ employees

---

## ✅ เฉลยแบบฝึกหัด

### เฉลยที่ 1:
```sql
SELECT product_name, unit_price
FROM products
ORDER BY unit_price DESC
LIMIT 5;
```

### เฉลยที่ 2:
```sql
SELECT product_id, product_name, unit_price
FROM products
ORDER BY product_name
LIMIT 5 OFFSET 5;
```

### เฉลยที่ 3:
```sql
SELECT order_id, order_date, total_amount, status
FROM orders
ORDER BY order_date DESC
LIMIT 3;
```

### เฉลยที่ 4:
```sql
-- Page 3, page_size = 3
-- OFFSET = (3-1) * 3 = 6
SELECT employee_id, first_name, last_name, salary
FROM employees
ORDER BY employee_id
LIMIT 3 OFFSET 6;
```

### เฉลยที่ 5:
```sql
SELECT 
    COUNT(*) AS total_records,
    4 AS page_size,
    CEIL(COUNT(*) * 1.0 / 4) AS total_pages
FROM products;
-- Result: total=15, page_size=4, total_pages=4 (ceil(15/4)=4)
```

### เฉลยที่ 6:
```sql
SELECT first_name, last_name, salary
FROM employees
WHERE department_id = 1
ORDER BY salary DESC
LIMIT 5;
```

### เฉลยที่ 7:
```sql
-- First page
SELECT order_id, order_date, total_amount
FROM orders
ORDER BY order_date DESC, order_id DESC
LIMIT 3;

-- ผล: order_id 10,9,8 → cursor: order_date='2024-03-01', order_id=8

-- Second page (cursor-based)
SELECT order_id, order_date, total_amount
FROM orders
WHERE order_date < '2024-03-01'
   OR (order_date = '2024-03-01' AND order_id < 8)
ORDER BY order_date DESC, order_id DESC
LIMIT 3;
```

### เฉลยที่ 8:
```sql
SELECT product_name, unit_price
FROM products
WHERE category = 'Electronics'
ORDER BY unit_price ASC
LIMIT 3;
```

### เฉลยที่ 9:
```sql
SELECT product_name, unit_price
FROM products
ORDER BY unit_price ASC
LIMIT 5 OFFSET 10;
-- แถวที่ 11-15
```

### เฉลยที่ 10:
```sql
SELECT 
    (SELECT COUNT(*) FROM employees) AS total_records,
    5 AS page_size,
    1 AS current_page,
    CEIL((SELECT COUNT(*) FROM employees) * 1.0 / 5) AS total_pages;
```

---

## ➡️ บทถัดไป

**[Part 007: Working with Column Aliases](part-007.md)**

ในบทถัดไปเราจะเรียนรู้การตั้งชื่อ columns ด้วย AS keyword!

---

*Part 006 of 120 | หลักสูตร SQL ครบวงจร*
