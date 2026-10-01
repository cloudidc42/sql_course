# Part 036: CUBE - All Combinations

## บทนำ (Introduction)

`CUBE` เป็น extension ของ GROUP BY ที่ทรงพลังกว่า ROLLUP โดย CUBE จะสร้าง **ทุกการรวมกัน** (All possible combinations) ของคอลัมน์ที่ระบุ ไม่ใช่แค่ลำดับชั้น

**ความแตกต่างหลัก:**
- `ROLLUP(A, B, C)` → สร้าง (A,B,C), (A,B), (A), ()  ← ตามลำดับชั้น
- `CUBE(A, B, C)` → สร้างทุก combination รวม 2³ = 8 กลุ่ม

CUBE เหมาะสำหรับ **OLAP (Online Analytical Processing)** และการวิเคราะห์ข้อมูลหลายมิติ

---

## การตั้งค่าฐานข้อมูล

```sql
USE ecommerce_db;

-- ข้อมูลพร้อมใช้จาก Part 031-035
-- ตรวจสอบข้อมูลที่จะใช้
SELECT 
    COUNT(*) AS orders,
    MIN(order_date) AS first_order,
    MAX(order_date) AS last_order
FROM orders;
```

---

## 1. CUBE Syntax

### MySQL (ผ่าน GROUPING SETS)
```sql
-- MySQL ไม่มี CUBE โดยตรง ต้องใช้ GROUPING SETS แทน
-- แต่ MySQL 8.0 รองรับ GROUPING SETS

-- หากต้องการ CUBE(A, B) ใน MySQL:
SELECT A, B, aggregate_fn(C)
FROM table
GROUP BY A, B WITH ROLLUP;  -- นี่คือ ROLLUP ไม่ใช่ CUBE

-- CUBE ที่แท้จริงต้องใช้ UNION ALL หรือ GROUPING SETS
```

### PostgreSQL / SQL Server
```sql
-- PostgreSQL และ SQL Server รองรับ CUBE โดยตรง
SELECT col1, col2, aggregate_fn(col3)
FROM table
GROUP BY CUBE(col1, col2);
```

---

## 2. CUBE vs ROLLUP เปรียบเทียบ

```sql
-- ROLLUP(category, brand) สร้าง groupings:
-- (category, brand) → รายละเอียดต่อ category และ brand
-- (category)        → subtotal ต่อ category
-- ()                → grand total

-- CUBE(category, brand) สร้าง groupings:
-- (category, brand) → รายละเอียดต่อ category และ brand
-- (category)        → subtotal ต่อ category เท่านั้น
-- (brand)           → subtotal ต่อ brand เท่านั้น (ROLLUP ไม่มี!)
-- ()                → grand total

-- ตัวอย่างที่ 1: เปรียบเทียบ ROLLUP vs CUBE
-- ROLLUP:
SELECT 
    COALESCE(category, 'TOTAL') AS category,
    COALESCE(brand, 'ALL') AS brand,
    COUNT(*) AS products,
    SUM(price * stock_qty) AS value
FROM products
GROUP BY category, brand WITH ROLLUP;

-- CUBE (สำหรับ PostgreSQL/SQL Server):
-- SELECT 
--     COALESCE(category, 'TOTAL') AS category,
--     COALESCE(brand, 'ALL') AS brand,
--     COUNT(*) AS products,
--     SUM(price * stock_qty) AS value
-- FROM products
-- GROUP BY CUBE(category, brand);

-- MySQL equivalent ด้วย GROUPING SETS:
SELECT 
    COALESCE(category, 'TOTAL') AS category,
    COALESCE(brand, 'ALL') AS brand,
    COUNT(*) AS products,
    SUM(price * stock_qty) AS value
FROM products
GROUP BY category, brand
UNION ALL
SELECT category, NULL, COUNT(*), SUM(price * stock_qty)
FROM products GROUP BY category
UNION ALL
SELECT NULL, brand, COUNT(*), SUM(price * stock_qty)
FROM products GROUP BY brand
UNION ALL
SELECT NULL, NULL, COUNT(*), SUM(price * stock_qty)
FROM products;
```

---

## 3. CUBE กับ 2 มิติ

```sql
-- ตัวอย่างที่ 2: CUBE ยอดขายต่อ year และ status (PostgreSQL/SQL Server)
-- GROUP BY CUBE(YEAR(order_date), status)

-- MySQL equivalent:
SELECT 
    YEAR(order_date) AS year,
    status,
    COUNT(*) AS orders,
    SUM(total_amount) AS revenue
FROM orders
GROUP BY YEAR(order_date), status
UNION ALL
-- subtotal ต่อปี
SELECT YEAR(order_date), NULL, COUNT(*), SUM(total_amount)
FROM orders GROUP BY YEAR(order_date)
UNION ALL
-- subtotal ต่อ status
SELECT NULL, status, COUNT(*), SUM(total_amount)
FROM orders GROUP BY status
UNION ALL
-- grand total
SELECT NULL, NULL, COUNT(*), SUM(total_amount)
FROM orders
ORDER BY year NULLS LAST, status NULLS LAST;
```

```sql
-- ตัวอย่างที่ 3: CUBE สำหรับ Multi-dimensional Analysis
-- วิเคราะห์สินค้าตาม category และ is_active
-- PostgreSQL:
-- SELECT 
--     COALESCE(category, 'ALL CATEGORIES') AS category,
--     COALESCE(CAST(is_active AS VARCHAR), 'ALL STATUS') AS active_status,
--     COUNT(*) AS products,
--     SUM(stock_qty) AS total_stock
-- FROM products
-- GROUP BY CUBE(category, is_active);

-- MySQL equivalent:
SELECT 
    COALESCE(category, 'ALL CATEGORIES') AS category,
    CASE WHEN is_active IS NULL THEN 'ALL STATUS'
         WHEN is_active = 1 THEN 'Active'
         ELSE 'Inactive' END AS active_status,
    COUNT(*) AS products,
    SUM(stock_qty) AS total_stock
FROM products
GROUP BY category, is_active
UNION ALL
SELECT category, 'ALL STATUS', COUNT(*), SUM(stock_qty)
FROM products GROUP BY category
UNION ALL
SELECT 'ALL CATEGORIES', 
    CASE WHEN is_active = 1 THEN 'Active' ELSE 'Inactive' END,
    COUNT(*), SUM(stock_qty)
FROM products GROUP BY is_active
UNION ALL
SELECT 'ALL CATEGORIES', 'ALL STATUS', COUNT(*), SUM(stock_qty)
FROM products
ORDER BY category, active_status;
```

---

## 4. CUBE กับ 3 มิติ

```sql
-- ตัวอย่างที่ 4: CUBE 3 มิติ (PostgreSQL)
-- GROUP BY CUBE(year, quarter, category)
-- จะสร้าง 2³ = 8 combinations:
-- (year, quarter, category)
-- (year, quarter)
-- (year, category)
-- (quarter, category)
-- (year)
-- (quarter)
-- (category)
-- ()

-- PostgreSQL syntax:
-- SELECT 
--     COALESCE(CAST(YEAR(order_date) AS VARCHAR), 'ALL YEARS') AS year,
--     COALESCE(CAST(QUARTER(order_date) AS VARCHAR), 'ALL Q') AS quarter,
--     COALESCE(p.category, 'ALL CAT') AS category,
--     SUM(oi.quantity * oi.unit_price) AS revenue
-- FROM orders o
-- JOIN order_items oi ON o.order_id = oi.order_id
-- JOIN products p ON oi.product_id = p.product_id
-- WHERE o.status NOT IN ('cancelled')
-- GROUP BY CUBE(YEAR(order_date), QUARTER(order_date), p.category);
```

---

## 5. OLAP Concepts กับ CUBE

```sql
-- ตัวอย่างที่ 5: Data Cube for Business Intelligence
-- CUBE ช่วยสร้าง "Data Cube" ที่ใช้ใน OLAP

-- Multi-dimensional View:
-- Dimension 1: Time (Year, Quarter, Month)
-- Dimension 2: Geography (Province, City)
-- Dimension 3: Product (Category, Brand)
-- Measure: Revenue, Orders, Customers

-- Cross-dimensional Analysis:
SELECT 
    CASE WHEN GROUPING(YEAR(o.order_date)) = 1 THEN 'ALL YEARS'
         ELSE CAST(YEAR(o.order_date) AS CHAR) END AS year,
    CASE WHEN GROUPING(c.province) = 1 THEN 'ALL PROVINCES'
         ELSE c.province END AS province,
    COUNT(DISTINCT o.order_id) AS orders,
    COUNT(DISTINCT o.customer_id) AS customers,
    SUM(o.total_amount) AS revenue
FROM orders o
JOIN customers c ON o.customer_id = c.customer_id
WHERE o.status NOT IN ('cancelled')
GROUP BY YEAR(o.order_date), c.province WITH ROLLUP
ORDER BY YEAR(o.order_date), c.province;
```

```sql
-- ตัวอย่างที่ 6: Slice - ดู Dimension เดียว
-- Slice: ตัดแบ่งมิติข้อมูล เช่น ดูเฉพาะปี 2024
SELECT 
    p.category,
    p.brand,
    SUM(oi.quantity * oi.unit_price) AS revenue
FROM orders o
JOIN order_items oi ON o.order_id = oi.order_id
JOIN products p ON oi.product_id = p.product_id
WHERE o.status NOT IN ('cancelled')
    AND YEAR(o.order_date) = 2024   -- Slice: เฉพาะปี 2024
GROUP BY p.category, p.brand
ORDER BY p.category, revenue DESC;
```

```sql
-- ตัวอย่างที่ 7: Dice - กรองหลายมิติพร้อมกัน
-- Dice: กรองทั้ง Time และ Geography
SELECT 
    p.category,
    SUM(oi.quantity * oi.unit_price) AS revenue
FROM orders o
JOIN order_items oi ON o.order_id = oi.order_id
JOIN products p ON oi.product_id = p.product_id
JOIN customers c ON o.customer_id = c.customer_id
WHERE o.status NOT IN ('cancelled')
    AND YEAR(o.order_date) = 2024           -- Time dimension
    AND c.province IN ('กรุงเทพฯ', 'นนทบุรี')  -- Geography dimension
    AND p.category IN ('สมาร์ทโฟน', 'โน้ตบุ๊ค')  -- Product dimension
GROUP BY p.category
ORDER BY revenue DESC;
```

```sql
-- ตัวอย่างที่ 8: Drill-down - ขยายรายละเอียด
-- จาก Category → Brand → Product
SELECT 
    p.category,
    p.brand,
    p.product_name,
    SUM(oi.quantity) AS units,
    SUM(oi.quantity * oi.unit_price) AS revenue
FROM products p
JOIN order_items oi ON p.product_id = oi.product_id
JOIN orders o ON oi.order_id = o.order_id
WHERE o.status NOT IN ('cancelled')
    AND p.category = 'สมาร์ทโฟน'  -- เริ่มจาก Category
GROUP BY p.category, p.brand, p.product_id, p.product_name
ORDER BY p.brand, revenue DESC;
```

```sql
-- ตัวอย่างที่ 9: Roll-up - ลดระดับรายละเอียด
-- จาก Product → Brand → Category
SELECT 
    CASE WHEN GROUPING(p.category) = 1 THEN 'TOTAL'
         ELSE p.category END AS category,
    CASE WHEN GROUPING(p.brand) = 1 
              AND GROUPING(p.category) = 0 THEN 'Category Total'
         WHEN GROUPING(p.brand) = 1 THEN 'TOTAL'
         ELSE p.brand END AS brand,
    SUM(oi.quantity) AS units,
    SUM(oi.quantity * oi.unit_price) AS revenue
FROM products p
JOIN order_items oi ON p.product_id = oi.product_id
JOIN orders o ON oi.order_id = o.order_id
WHERE o.status NOT IN ('cancelled')
GROUP BY p.category, p.brand WITH ROLLUP
ORDER BY p.category, p.brand;
```

---

## 6. Generating Complete Pivot Tables with CUBE

```sql
-- ตัวอย่างที่ 10: Complete Pivot Table ด้วย CUBE
-- แสดงยอดขายแต่ละ Category × Year

-- Step 1: ข้อมูลดิบ
SELECT 
    p.category,
    YEAR(o.order_date) AS year,
    SUM(oi.quantity * oi.unit_price) AS revenue
FROM products p
JOIN order_items oi ON p.product_id = oi.product_id
JOIN orders o ON oi.order_id = o.order_id
WHERE o.status NOT IN ('cancelled')
GROUP BY p.category, YEAR(o.order_date);

-- Step 2: Pivot ด้วย Conditional Aggregation
SELECT 
    COALESCE(p.category, 'ALL CATEGORIES') AS category,
    SUM(CASE WHEN YEAR(o.order_date) = 2023 THEN oi.quantity * oi.unit_price ELSE 0 END) AS yr_2023,
    SUM(CASE WHEN YEAR(o.order_date) = 2024 THEN oi.quantity * oi.unit_price ELSE 0 END) AS yr_2024,
    SUM(oi.quantity * oi.unit_price) AS total
FROM products p
JOIN order_items oi ON p.product_id = oi.product_id
JOIN orders o ON oi.order_id = o.order_id
WHERE o.status NOT IN ('cancelled')
GROUP BY p.category WITH ROLLUP
ORDER BY GROUPING(p.category), total DESC;
```

---

## 7. CUBE สำหรับ Multi-dimensional Analysis

```sql
-- ตัวอย่างที่ 11: Revenue Analysis - Time × Product × Geography
SELECT 
    COALESCE(CAST(YEAR(o.order_date) AS CHAR), 'ALL') AS year,
    COALESCE(c.province, 'ALL') AS province,
    COALESCE(p.category, 'ALL') AS category,
    COUNT(DISTINCT o.order_id) AS orders,
    SUM(o.total_amount) AS revenue
FROM orders o
JOIN customers c ON o.customer_id = c.customer_id
JOIN order_items oi ON o.order_id = oi.order_id
JOIN products p ON oi.product_id = p.product_id
WHERE o.status NOT IN ('cancelled')
GROUP BY YEAR(o.order_date), c.province, p.category WITH ROLLUP
HAVING 
    -- แสดงเฉพาะ: Grand Total, Year Total, Province Total, Category Total
    (GROUPING(YEAR(o.order_date)) = 1)
    OR (GROUPING(c.province) = 1 AND GROUPING(p.category) = 1)
    OR (GROUPING(YEAR(o.order_date)) = 1 AND GROUPING(p.category) = 1)
ORDER BY year, province, category;
```

```sql
-- ตัวอย่างที่ 12: Salary Analysis ด้วย CUBE-like query
-- ต้องการ: (dept, job_title), (dept), (job_title), ()
SELECT 
    COALESCE(d.department_name, 'ALL DEPARTMENTS') AS department,
    COALESCE(e.job_title, 'ALL TITLES') AS job_title,
    COUNT(*) AS headcount,
    ROUND(AVG(e.salary), 0) AS avg_salary,
    SUM(e.salary) AS total_salary
FROM employees e
JOIN departments d ON e.department_id = d.department_id
WHERE e.salary IS NOT NULL
GROUP BY d.department_name, e.job_title
-- CUBE equivalent with UNION ALL:
UNION ALL
SELECT d.department_name, 'ALL TITLES', COUNT(*), ROUND(AVG(e.salary), 0), SUM(e.salary)
FROM employees e
JOIN departments d ON e.department_id = d.department_id
WHERE e.salary IS NOT NULL
GROUP BY d.department_name
UNION ALL
SELECT 'ALL DEPARTMENTS', e.job_title, COUNT(*), ROUND(AVG(e.salary), 0), SUM(e.salary)
FROM employees e
WHERE e.salary IS NOT NULL
GROUP BY e.job_title
UNION ALL
SELECT 'ALL DEPARTMENTS', 'ALL TITLES', COUNT(*), ROUND(AVG(e.salary), 0), SUM(e.salary)
FROM employees e
WHERE e.salary IS NOT NULL
ORDER BY department, job_title;
```

---

## 8. ความแตกต่าง CUBE vs ROLLUP - สรุปชัดเจน

```sql
-- ตัวอย่างที่ 13: แสดงความแตกต่างด้วยตัวอย่างเล็กๆ
-- สมมติ data: Brand=Apple, Category=สมาร์ทโฟน

-- ROLLUP(category, brand) สร้าง:
-- ('สมาร์ทโฟน', 'Apple')  ← Detail
-- ('สมาร์ทโฟน', NULL)     ← Category Subtotal
-- (NULL, NULL)             ← Grand Total
-- *** ไม่มี (NULL, 'Apple') ***

-- CUBE(category, brand) สร้าง:
-- ('สมาร์ทโฟน', 'Apple')  ← Detail
-- ('สมาร์ทโฟน', NULL)     ← Category Subtotal
-- (NULL, 'Apple')          ← Brand Subtotal ← มีเพิ่มมา!
-- (NULL, NULL)             ← Grand Total

-- ดังนั้น CUBE เหมาะกับการวิเคราะห์ที่ต้องการ Cross-dimensional Analysis
-- ROLLUP เหมาะกับข้อมูลที่มีลำดับชั้น (Hierarchy) เช่น ปี > เดือน > วัน
```

```sql
-- ตัวอย่างที่ 14: Use Case ที่เหมาะกับ CUBE
-- ต้องการดูยอดขายทั้ง:
-- 1. ต่อ brand และ category
-- 2. ต่อ brand ทุก category
-- 3. ต่อ category ทุก brand
-- 4. รวมทั้งหมด

-- ด้วย UNION ALL:
SELECT 'Detail' AS type, brand, category, COUNT(*), SUM(price)
FROM products GROUP BY brand, category
UNION ALL
SELECT 'Brand Total', brand, 'ALL', COUNT(*), SUM(price)
FROM products GROUP BY brand
UNION ALL
SELECT 'Category Total', 'ALL', category, COUNT(*), SUM(price)
FROM products GROUP BY category
UNION ALL
SELECT 'Grand Total', 'ALL', 'ALL', COUNT(*), SUM(price)
FROM products
ORDER BY brand, category;
```

---

## 9. CUBE กับ Business Intelligence

```sql
-- ตัวอย่างที่ 15: BI Dashboard - Multi-dimensional Sales View
SELECT 
    COALESCE(CAST(YEAR(o.order_date) AS CHAR), 'ALL YEARS') AS time_dim,
    COALESCE(p.category, 'ALL CATEGORIES') AS product_dim,
    COUNT(DISTINCT o.order_id) AS order_count,
    COUNT(DISTINCT o.customer_id) AS unique_customers,
    SUM(oi.quantity * oi.unit_price) AS revenue,
    AVG(o.total_amount) AS avg_order_value
FROM orders o
JOIN order_items oi ON o.order_id = oi.order_id
JOIN products p ON oi.product_id = p.product_id
WHERE o.status NOT IN ('cancelled')
GROUP BY YEAR(o.order_date), p.category WITH ROLLUP
ORDER BY time_dim, product_dim;
```

```sql
-- ตัวอย่างที่ 16: CUBE for Sales Attribution
-- ต้องการยอดขายแยกตาม: ปี × จังหวัด × หมวดสินค้า
-- ทุก combination

-- Compact version ด้วย 3-column ROLLUP + UNION:
-- (เนื่องจาก MySQL ไม่มี CUBE โดยตรง)
SELECT 
    COALESCE(CAST(YEAR(o.order_date) AS CHAR), 'ALL') AS year,
    COALESCE(c.province, 'ALL') AS province, 
    COALESCE(p.category, 'ALL') AS category,
    SUM(o.total_amount) AS revenue
FROM orders o
JOIN customers c ON o.customer_id = c.customer_id
JOIN order_items oi ON o.order_id = oi.order_id
JOIN products p ON oi.product_id = p.product_id
WHERE o.status NOT IN ('cancelled')
GROUP BY YEAR(o.order_date), c.province, p.category WITH ROLLUP
ORDER BY year, province, category;
```

```sql
-- ตัวอย่างที่ 17: Complete CUBE Analysis สำหรับ Executive Dashboard
-- สร้าง Multi-dimensional View ของธุรกิจ

-- View 1: ยอดขายต่อ Category และ Payment Method
SELECT 
    COALESCE(p.category, 'ALL') AS category,
    COALESCE(o.payment_method, 'ALL') AS payment_method,
    COUNT(*) AS orders,
    SUM(o.total_amount) AS revenue,
    ROUND(AVG(o.total_amount), 0) AS avg_order
FROM orders o
JOIN order_items oi ON o.order_id = oi.order_id
JOIN products p ON oi.product_id = p.product_id
WHERE o.status NOT IN ('cancelled')
GROUP BY p.category, o.payment_method WITH ROLLUP
HAVING COUNT(*) > 0
ORDER BY p.category, o.payment_method;
```

```sql
-- ตัวอย่างที่ 18: CUBE ช่วยสร้าง Comprehensive Report
-- แทนการเขียน Report หลายๆ ตัว

-- แทนที่จะเขียน:
-- Report 1: ยอดขายต่อ Category
-- Report 2: ยอดขายต่อ Year
-- Report 3: ยอดขายต่อ Category + Year
-- Report 4: Grand Total

-- ใช้ CUBE ครั้งเดียว:
SELECT 
    CASE WHEN GROUPING(p.category) = 1 THEN '★ ALL CATEGORIES ★' ELSE p.category END,
    CASE WHEN GROUPING(YEAR(o.order_date)) = 1 THEN '★ ALL YEARS ★' 
         ELSE CAST(YEAR(o.order_date) AS CHAR) END,
    COUNT(*) AS orders,
    SUM(oi.quantity * oi.unit_price) AS revenue
FROM orders o
JOIN order_items oi ON o.order_id = oi.order_id
JOIN products p ON oi.product_id = p.product_id
WHERE o.status NOT IN ('cancelled')
GROUP BY p.category, YEAR(o.order_date) WITH ROLLUP
ORDER BY 
    GROUPING(p.category), 
    GROUPING(YEAR(o.order_date)),
    p.category,
    YEAR(o.order_date);
```

---

## 10. Complete CUBE vs ROLLUP Comparison

```sql
-- ตัวอย่างที่ 19: Side-by-side Comparison
-- คำถาม: ต้องการ groupings อะไรบ้าง?

-- ROLLUP ให้:
-- 1. (category, brand) - detail
-- 2. (category) - category subtotal
-- 3. () - grand total
-- จำนวน: n+1 levels

-- CUBE ให้:
-- 1. (category, brand) - detail
-- 2. (category) - category subtotal
-- 3. (brand) - brand subtotal ← เพิ่มมา!
-- 4. () - grand total
-- จำนวน: 2^n levels

-- เลือกใช้:
-- ROLLUP: เมื่อข้อมูลมี natural hierarchy (Date: Year > Month > Day)
-- CUBE: เมื่อต้องการ cross-dimensional analysis (Category × Region × Time)
```

```sql
-- ตัวอย่างที่ 20: สรุป CUBE สำหรับ 3 มิติ (ด้วย UNION ALL สำหรับ MySQL)
-- CUBE(year, category, province) = 2^3 = 8 groupings

SELECT 'year+cat+prov' AS grouping_type,
    YEAR(o.order_date) AS yr, p.category AS cat, c.province AS prov,
    SUM(o.total_amount) AS rev
FROM orders o JOIN customers c ON o.customer_id=c.customer_id
JOIN order_items oi ON o.order_id=oi.order_id
JOIN products p ON oi.product_id=p.product_id
WHERE o.status!='cancelled'
GROUP BY YEAR(o.order_date), p.category, c.province

UNION ALL SELECT 'year+cat', YEAR(o.order_date), p.category, NULL, SUM(o.total_amount)
FROM orders o JOIN order_items oi ON o.order_id=oi.order_id
JOIN products p ON oi.product_id=p.product_id
WHERE o.status!='cancelled' GROUP BY YEAR(o.order_date), p.category

UNION ALL SELECT 'year+prov', YEAR(o.order_date), NULL, c.province, SUM(o.total_amount)
FROM orders o JOIN customers c ON o.customer_id=c.customer_id
WHERE o.status!='cancelled' GROUP BY YEAR(o.order_date), c.province

UNION ALL SELECT 'cat+prov', NULL, p.category, c.province, SUM(o.total_amount)
FROM orders o JOIN customers c ON o.customer_id=c.customer_id
JOIN order_items oi ON o.order_id=oi.order_id
JOIN products p ON oi.product_id=p.product_id
WHERE o.status!='cancelled' GROUP BY p.category, c.province

UNION ALL SELECT 'year only', YEAR(o.order_date), NULL, NULL, SUM(o.total_amount)
FROM orders o WHERE o.status!='cancelled' GROUP BY YEAR(o.order_date)

UNION ALL SELECT 'cat only', NULL, p.category, NULL, SUM(o.total_amount)
FROM orders o JOIN order_items oi ON o.order_id=oi.order_id
JOIN products p ON oi.product_id=p.product_id
WHERE o.status!='cancelled' GROUP BY p.category

UNION ALL SELECT 'prov only', NULL, NULL, c.province, SUM(o.total_amount)
FROM orders o JOIN customers c ON o.customer_id=c.customer_id
WHERE o.status!='cancelled' GROUP BY c.province

UNION ALL SELECT 'grand total', NULL, NULL, NULL, SUM(total_amount)
FROM orders WHERE status!='cancelled'

ORDER BY grouping_type;
```

---

## สรุปบทที่ 36

### เมื่อไหร่ควรใช้ CUBE?

| Scenario | ใช้ ROLLUP หรือ CUBE? |
|---------|---------------------|
| Year → Quarter → Month hierarchy | ROLLUP |
| Category × Region cross analysis | CUBE |
| Department → Team → Employee | ROLLUP |
| Time × Product × Geography | CUBE |
| Geographic hierarchy | ROLLUP |
| Multi-attribute comparison | CUBE |

### Performance Warning
- CUBE มี complexity สูง: n columns → 2^n groupings
- 3 columns = 8 groupings
- 4 columns = 16 groupings
- 5 columns = 32 groupings
- ระวังการใช้ CUBE กับหลาย columns มาก อาจช้ามาก!

---

## แบบฝึกหัดท้ายบท

### แบบฝึกหัดที่ 1
อธิบายความแตกต่างระหว่าง ROLLUP และ CUBE

```sql
-- เฉลย (คำอธิบาย):
-- ROLLUP สร้าง hierarchical subtotals: (A,B,C), (A,B), (A), ()
-- CUBE สร้างทุก combination: (A,B,C), (A,B), (A,C), (B,C), (A), (B), (C), ()
-- ROLLUP = n+1 groupings
-- CUBE = 2^n groupings
-- ROLLUP ใช้เมื่อมี hierarchy เช่น Year>Month>Day
-- CUBE ใช้เมื่อต้องการ cross-dimensional analysis
```

### แบบฝึกหัดที่ 2
สร้าง CUBE equivalent ด้วย UNION ALL สำหรับ category × brand

```sql
-- เฉลย:
SELECT category, brand, COUNT(*), SUM(price) FROM products GROUP BY category, brand
UNION ALL
SELECT category, NULL, COUNT(*), SUM(price) FROM products GROUP BY category
UNION ALL
SELECT NULL, brand, COUNT(*), SUM(price) FROM products GROUP BY brand
UNION ALL
SELECT NULL, NULL, COUNT(*), SUM(price) FROM products
ORDER BY category NULLS LAST, brand NULLS LAST;
```

### แบบฝึกหัดที่ 3
สร้าง Multi-dimensional Sales Analysis ด้วย ROLLUP

```sql
-- เฉลย:
SELECT 
    COALESCE(CAST(YEAR(o.order_date) AS CHAR), 'ALL YEARS') AS year,
    COALESCE(p.category, 'ALL CATEGORIES') AS category,
    COUNT(*) AS orders,
    SUM(oi.quantity * oi.unit_price) AS revenue
FROM orders o
JOIN order_items oi ON o.order_id = oi.order_id
JOIN products p ON oi.product_id = p.product_id
WHERE o.status NOT IN ('cancelled')
GROUP BY YEAR(o.order_date), p.category WITH ROLLUP
ORDER BY YEAR(o.order_date), p.category;
```

### แบบฝึกหัดที่ 4
คำนวณจำนวน groupings ที่ CUBE(A, B, C, D) จะสร้าง

```sql
-- เฉลย (คำอธิบาย):
-- CUBE(A,B,C,D) = 2^4 = 16 groupings:
-- (A,B,C,D), (A,B,C), (A,B,D), (A,C,D), (B,C,D),
-- (A,B), (A,C), (A,D), (B,C), (B,D), (C,D),
-- (A), (B), (C), (D), ()
```

### แบบฝึกหัดที่ 5
สร้าง Drill-down Query จาก Category → Brand → Product

```sql
-- เฉลย:
SELECT 
    CASE WHEN GROUPING(p.category) = 1 THEN 'ALL' ELSE p.category END AS category,
    CASE WHEN GROUPING(p.brand) = 1 
              AND GROUPING(p.category) = 0 THEN 'Category Total'
         WHEN GROUPING(p.brand) = 1 THEN 'ALL'
         ELSE p.brand END AS brand,
    COUNT(DISTINCT p.product_id) AS products,
    SUM(oi.quantity * oi.unit_price) AS revenue
FROM products p
JOIN order_items oi ON p.product_id = oi.product_id
GROUP BY p.category, p.brand WITH ROLLUP
ORDER BY p.category, p.brand;
```

### แบบฝึกหัดที่ 6
สร้าง Geographic Revenue Analysis (Province × City)

```sql
-- เฉลย:
SELECT 
    COALESCE(c.province, 'ALL PROVINCES') AS province,
    COALESCE(c.city, 'ALL CITIES') AS city,
    COUNT(DISTINCT c.customer_id) AS customers,
    SUM(o.total_amount) AS revenue
FROM customers c
JOIN orders o ON c.customer_id = o.customer_id
WHERE o.status NOT IN ('cancelled')
GROUP BY c.province, c.city WITH ROLLUP
ORDER BY c.province, c.city;
```

### แบบฝึกหัดที่ 7
เปรียบเทียบผลลัพธ์ของ ROLLUP(category, brand) vs CUBE equivalent

```sql
-- เฉลย ROLLUP:
SELECT COALESCE(category,'TOTAL'), COALESCE(brand,'ALL'), COUNT(*), SUM(price)
FROM products GROUP BY category, brand WITH ROLLUP ORDER BY category, brand;

-- เฉลย CUBE equivalent:
SELECT category, brand, COUNT(*), SUM(price) FROM products GROUP BY category, brand
UNION ALL SELECT category, NULL, COUNT(*), SUM(price) FROM products GROUP BY category
UNION ALL SELECT NULL, brand, COUNT(*), SUM(price) FROM products GROUP BY brand
UNION ALL SELECT NULL, NULL, COUNT(*), SUM(price) FROM products
ORDER BY category NULLS LAST, brand NULLS LAST;
-- CUBE มี (NULL, brand) เพิ่มมาจาก ROLLUP
```

### แบบฝึกหัดที่ 8
ใช้ Slice (กรองมิติเดียว) เพื่อดูยอดขายเฉพาะปี 2024

```sql
-- เฉลย:
SELECT 
    p.category,
    p.brand,
    SUM(oi.quantity * oi.unit_price) AS revenue_2024
FROM products p
JOIN order_items oi ON p.product_id = oi.product_id
JOIN orders o ON oi.order_id = o.order_id
WHERE o.status NOT IN ('cancelled')
    AND YEAR(o.order_date) = 2024
GROUP BY p.category, p.brand
ORDER BY p.category, revenue_2024 DESC;
```

### แบบฝึกหัดที่ 9
ใช้ Dice (กรองหลายมิติ) เพื่อดูสมาร์ทโฟนในกรุงเทพฯ ปี 2024

```sql
-- เฉลย:
SELECT 
    p.product_name,
    SUM(oi.quantity) AS units_sold,
    SUM(oi.quantity * oi.unit_price) AS revenue
FROM products p
JOIN order_items oi ON p.product_id = oi.product_id
JOIN orders o ON oi.order_id = o.order_id
JOIN customers c ON o.customer_id = c.customer_id
WHERE o.status NOT IN ('cancelled')
    AND YEAR(o.order_date) = 2024
    AND c.province = 'กรุงเทพฯ'
    AND p.category = 'สมาร์ทโฟน'
GROUP BY p.product_id, p.product_name
ORDER BY revenue DESC;
```

### แบบฝึกหัดที่ 10
สร้าง Complete Multi-dimensional Report

```sql
-- เฉลย:
SELECT 
    COALESCE(CAST(YEAR(o.order_date) AS CHAR), 'ALL YEARS') AS year,
    COALESCE(c.province, 'ALL PROVINCES') AS province,
    COUNT(DISTINCT o.order_id) AS orders,
    COUNT(DISTINCT o.customer_id) AS customers,
    SUM(o.total_amount) AS revenue,
    ROUND(AVG(o.total_amount), 0) AS avg_order
FROM orders o
JOIN customers c ON o.customer_id = c.customer_id
WHERE o.status NOT IN ('cancelled')
GROUP BY YEAR(o.order_date), c.province WITH ROLLUP
ORDER BY year, province;
```

---

*จบบทที่ 036 - CUBE: All Combinations*

*บทถัดไป: Part 037 - GROUPING SETS: Custom Grouping Combinations*
