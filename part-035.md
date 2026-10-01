# Part 035: ROLLUP - Multi-level Subtotals

## บทนำ (Introduction)

`ROLLUP` เป็น extension ของ GROUP BY ที่ช่วยสร้าง **Subtotals หลายระดับ** โดยอัตโนมัติ แทนที่เราจะต้องเขียน UNION ALL หลายๆ ครั้งเพื่อรวมยอดรวมย่อย ROLLUP จะทำให้เราโดยอัตโนมัติ

**ตัวอย่างการใช้งาน:**
- รายงานยอดขาย → ยอดรวมแต่ละเดือน → ยอดรวมแต่ละปี → ยอดรวมทั้งหมด
- รายงาน Department → ยอดรวมแต่ละแผนก → ยอดรวมทั้งบริษัท

---

## การตั้งค่าฐานข้อมูล

```sql
USE ecommerce_db;

-- ข้อมูลพร้อมใช้จาก Part 031-034
-- ตรวจสอบข้อมูล
SELECT COUNT(*) AS order_count FROM orders;
SELECT COUNT(*) AS product_count FROM products;
SELECT COUNT(*) AS employee_count FROM employees;
```

---

## 1. ROLLUP Syntax

### MySQL / MariaDB
```sql
SELECT col1, col2, aggregate_fn(col3)
FROM table
GROUP BY col1, col2 WITH ROLLUP;
```

### PostgreSQL / SQL Server
```sql
SELECT col1, col2, aggregate_fn(col3)
FROM table
GROUP BY ROLLUP(col1, col2);
```

---

## 2. ROLLUP พื้นฐาน - คอลัมน์เดียว

```sql
-- ตัวอย่างที่ 1: ROLLUP ง่ายๆ - ยอดขายต่อ category + Grand Total
SELECT 
    COALESCE(category, '=== GRAND TOTAL ===') AS category,
    COUNT(*) AS product_count,
    SUM(stock_qty * price) AS total_value
FROM products
WHERE is_active = TRUE
GROUP BY category WITH ROLLUP
ORDER BY category;

-- ผลลัพธ์:
-- category           | product_count | total_value
-- -------------------+---------------+------------
-- เกมคอนโซล          | 2             | 693000
-- เครื่องดูดฝุ่น       | 1             | 871500
-- แท็บเล็ต            | 2             | 1303500
-- ทีวี                | 2             | 1592000
-- ...
-- === GRAND TOTAL === | 19            | 8924510  ← แถว ROLLUP
```

```sql
-- ตัวอย่างที่ 2: ROLLUP สรุปพนักงานต่อแผนก
SELECT 
    COALESCE(CAST(department_id AS CHAR), 'รวมทั้งหมด') AS dept,
    COUNT(*) AS headcount,
    SUM(salary) AS total_salary,
    ROUND(AVG(salary), 0) AS avg_salary
FROM employees
WHERE salary IS NOT NULL
GROUP BY department_id WITH ROLLUP;
```

---

## 3. ROLLUP หลายคอลัมน์

ROLLUP ที่มีหลายคอลัมน์จะสร้าง Subtotal ที่ระดับต่างๆ

```sql
-- ตัวอย่างที่ 3: ROLLUP 2 ระดับ - ปีและเดือน
SELECT 
    COALESCE(CAST(YEAR(order_date) AS CHAR), 'GRAND TOTAL') AS year,
    COALESCE(CAST(MONTH(order_date) AS CHAR), 'Year Total') AS month,
    COUNT(*) AS orders,
    SUM(total_amount) AS revenue
FROM orders
WHERE status NOT IN ('cancelled')
GROUP BY YEAR(order_date), MONTH(order_date) WITH ROLLUP
ORDER BY YEAR(order_date), MONTH(order_date);

-- ผลลัพธ์จะมีแถวประเภทต่างๆ:
-- 2023  | 1          | 2  | 81900   ← รายเดือน
-- 2023  | 2          | 1  | 42450
-- ...
-- 2023  | Year Total | 12 | 453700  ← Subtotal รายปี
-- 2024  | 1          | 5  | 137050
-- ...
-- 2024  | Year Total | 20 | 658890  ← Subtotal รายปี
-- GRAND TOTAL | Year Total | 37 | 1112590 ← Grand Total
```

```sql
-- ตัวอย่างที่ 4: ROLLUP 3 ระดับ - ปี ไตรมาส เดือน
SELECT 
    COALESCE(CAST(YEAR(order_date) AS CHAR), 'GRAND TOTAL') AS year,
    COALESCE(CONCAT('Q', CAST(QUARTER(order_date) AS CHAR)), 'Year Total') AS quarter,
    COALESCE(CAST(MONTH(order_date) AS CHAR), 'Quarter Total') AS month,
    COUNT(*) AS orders,
    SUM(total_amount) AS revenue
FROM orders
WHERE status NOT IN ('cancelled')
GROUP BY YEAR(order_date), QUARTER(order_date), MONTH(order_date) WITH ROLLUP
ORDER BY YEAR(order_date), QUARTER(order_date), MONTH(order_date);
```

---

## 4. GROUPING() Function

`GROUPING()` ช่วยระบุว่าแถวนั้นเป็นแถว ROLLUP (Subtotal) หรือแถวข้อมูลปกติ
- คืนค่า `1` = แถว ROLLUP/Subtotal
- คืนค่า `0` = แถวข้อมูลปกติ

```sql
-- ตัวอย่างที่ 5: ใช้ GROUPING() เพื่อตรวจสอบแถว ROLLUP
SELECT 
    GROUPING(YEAR(order_date)) AS is_year_rollup,
    GROUPING(MONTH(order_date)) AS is_month_rollup,
    YEAR(order_date) AS year,
    MONTH(order_date) AS month,
    SUM(total_amount) AS revenue
FROM orders
WHERE status NOT IN ('cancelled')
GROUP BY YEAR(order_date), MONTH(order_date) WITH ROLLUP
ORDER BY YEAR(order_date), MONTH(order_date);
```

```sql
-- ตัวอย่างที่ 6: ใช้ GROUPING() กับ CASE เพื่อ Label สวยงาม
SELECT 
    CASE 
        WHEN GROUPING(YEAR(order_date)) = 1  THEN '*** GRAND TOTAL ***'
        ELSE CAST(YEAR(order_date) AS CHAR)
    END AS year,
    CASE 
        WHEN GROUPING(MONTH(order_date)) = 1 AND GROUPING(YEAR(order_date)) = 0 
             THEN '-- Year Subtotal --'
        WHEN GROUPING(MONTH(order_date)) = 1 
             THEN '*** GRAND TOTAL ***'
        ELSE CAST(MONTH(order_date) AS CHAR)
    END AS month,
    COUNT(*) AS orders,
    SUM(total_amount) AS revenue
FROM orders
WHERE status NOT IN ('cancelled')
GROUP BY YEAR(order_date), MONTH(order_date) WITH ROLLUP
ORDER BY YEAR(order_date), MONTH(order_date);
```

---

## 5. GROUPING_ID() Function

`GROUPING_ID()` รวม GROUPING() หลายคอลัมน์เป็น Bitmask เดียว

```sql
-- ตัวอย่างที่ 7: GROUPING_ID() ระบุระดับของ ROLLUP
SELECT 
    GROUPING_ID(category, brand) AS grouping_level,
    CASE GROUPING_ID(category, brand)
        WHEN 0 THEN 'Detail Row'
        WHEN 1 THEN 'Category Subtotal'
        WHEN 3 THEN 'Grand Total'
    END AS row_type,
    COALESCE(category, 'Grand Total') AS category,
    COALESCE(brand, 'All Brands') AS brand,
    COUNT(*) AS products,
    SUM(stock_qty * price) AS value
FROM products
WHERE is_active = TRUE
GROUP BY category, brand WITH ROLLUP
ORDER BY category, brand;
```

---

## 6. Formatting ROLLUP Output

```sql
-- ตัวอย่างที่ 8: รายงานการเงินที่มีการจัดรูปแบบสวยงาม
SELECT 
    CASE 
        WHEN GROUPING(d.department_name) = 1 THEN '★ COMPANY TOTAL ★'
        ELSE d.department_name
    END AS department,
    CASE 
        WHEN GROUPING(d.department_name) = 1 THEN '—'
        ELSE d.location
    END AS location,
    COUNT(e.employee_id) AS headcount,
    FORMAT(SUM(COALESCE(e.salary, 0)), 0) AS total_salary,
    FORMAT(SUM(COALESCE(e.salary, 0)) * 12, 0) AS annual_payroll,
    FORMAT(d.budget, 0) AS budget
FROM departments d
LEFT JOIN employees e ON d.department_id = e.department_id
GROUP BY d.department_id, d.department_name, d.location, d.budget WITH ROLLUP
HAVING GROUPING(d.department_name) = 1 
    OR d.department_name IS NOT NULL
ORDER BY d.department_name;
```

```sql
-- ตัวอย่างที่ 9: รายงานยอดขายแบบ Hierarchical
SELECT 
    CASE WHEN GROUPING(p.category) = 1 THEN '[ TOTAL ALL CATEGORIES ]'
         ELSE p.category END AS category,
    CASE WHEN GROUPING(p.brand) = 1 
              AND GROUPING(p.category) = 0 THEN '  → Category Subtotal'
         WHEN GROUPING(p.brand) = 1 THEN '[ TOTAL ALL CATEGORIES ]'
         ELSE '    ' || p.brand END AS brand,
    COUNT(DISTINCT p.product_id) AS products,
    SUM(oi.quantity) AS units_sold,
    FORMAT(SUM(oi.quantity * oi.unit_price), 0) AS revenue
FROM products p
JOIN order_items oi ON p.product_id = oi.product_id
JOIN orders o ON oi.order_id = o.order_id
WHERE o.status NOT IN ('cancelled')
GROUP BY p.category, p.brand WITH ROLLUP
ORDER BY p.category, p.brand;
```

---

## 7. รายงานการเงินจริง (Financial Reports)

```sql
-- ตัวอย่างที่ 10: รายงานยอดขายประจำปีแบบ Financial Statement
SELECT 
    CASE WHEN GROUPING(YEAR(order_date)) = 1 
         THEN '━━━ GRAND TOTAL ━━━'
         ELSE CONCAT('Year ', CAST(YEAR(order_date) AS CHAR))
    END AS period,
    CASE WHEN GROUPING(QUARTER(order_date)) = 1 
              AND GROUPING(YEAR(order_date)) = 0
         THEN '  Annual Total'
         WHEN GROUPING(QUARTER(order_date)) = 1
         THEN '━━━ GRAND TOTAL ━━━'
         ELSE CONCAT('  Q', CAST(QUARTER(order_date) AS CHAR))
    END AS quarter,
    COUNT(*) AS orders,
    SUM(total_amount) AS gross_revenue,
    SUM(discount_amt) AS discounts,
    SUM(total_amount) - SUM(discount_amt) AS net_revenue,
    SUM(shipping_fee) AS shipping_income
FROM orders
WHERE status NOT IN ('cancelled')
GROUP BY YEAR(order_date), QUARTER(order_date) WITH ROLLUP
ORDER BY YEAR(order_date), QUARTER(order_date);
```

```sql
-- ตัวอย่างที่ 11: Product Category P&L with ROLLUP
SELECT 
    CASE WHEN GROUPING(p.category) = 1 THEN 'TOTAL' ELSE p.category END AS category,
    SUM(oi.quantity * oi.unit_price) AS revenue,
    SUM(oi.quantity * p.cost) AS cogs,
    SUM(oi.quantity * oi.unit_price) - SUM(oi.quantity * p.cost) AS gross_profit,
    ROUND(
        (SUM(oi.quantity * oi.unit_price) - SUM(oi.quantity * p.cost)) * 100.0 /
        NULLIF(SUM(oi.quantity * oi.unit_price), 0), 2
    ) AS margin_pct
FROM products p
JOIN order_items oi ON p.product_id = oi.product_id
JOIN orders o ON oi.order_id = o.order_id
WHERE o.status NOT IN ('cancelled')
GROUP BY p.category WITH ROLLUP
ORDER BY 
    GROUPING(p.category),
    SUM(oi.quantity * oi.unit_price) DESC;
```

```sql
-- ตัวอย่างที่ 12: HR Budget Report with ROLLUP
SELECT 
    CASE WHEN GROUPING(d.department_name) = 1 
         THEN '★★★ TOTAL COMPANY ★★★'
         ELSE d.department_name END AS department,
    COUNT(e.employee_id) AS headcount,
    MIN(e.salary) AS min_salary,
    MAX(e.salary) AS max_salary,
    ROUND(AVG(e.salary), 0) AS avg_salary,
    SUM(e.salary) AS monthly_payroll,
    SUM(e.salary) * 12 AS annual_payroll,
    d.budget AS dept_budget,
    ROUND(SUM(e.salary) * 12 / NULLIF(d.budget, 0) * 100, 1) AS payroll_to_budget_pct
FROM departments d
LEFT JOIN employees e ON d.department_id = e.department_id
WHERE e.salary IS NOT NULL
GROUP BY d.department_id, d.department_name, d.budget WITH ROLLUP
HAVING GROUPING(d.department_name) = 1 
    OR COUNT(e.employee_id) > 0
ORDER BY 
    GROUPING(d.department_name),
    SUM(e.salary) DESC;
```

---

## 8. ROLLUP ใน Different Databases

### MySQL
```sql
-- MySQL 8.0 ขึ้นไป: สนับสนุนทั้ง WITH ROLLUP และ GROUPING()
SELECT category, brand, COUNT(*), SUM(price)
FROM products
GROUP BY category, brand WITH ROLLUP;

-- MySQL รองรับ GROUPING() ใน Version 8.0+
SELECT GROUPING(category), category, SUM(price)
FROM products
GROUP BY category WITH ROLLUP;
```

### PostgreSQL
```sql
-- PostgreSQL ใช้ ROLLUP() syntax
SELECT category, brand, COUNT(*), SUM(price)
FROM products
GROUP BY ROLLUP(category, brand);

-- หรือใช้ GROUPING SETS
SELECT category, brand, COUNT(*), SUM(price)
FROM products
GROUP BY GROUPING SETS ((category, brand), (category), ());
```

### SQL Server
```sql
-- SQL Server ใช้ ROLLUP() syntax เหมือน PostgreSQL
SELECT category, brand, COUNT(*), SUM(price)
FROM products
GROUP BY ROLLUP(category, brand);
```

---

## 9. Advanced ROLLUP Techniques

```sql
-- ตัวอย่างที่ 13: ROLLUP กับ Window Functions
SELECT 
    COALESCE(CAST(YEAR(order_date) AS CHAR), 'TOTAL') AS year,
    COALESCE(CAST(MONTH(order_date) AS CHAR), 'Annual Total') AS month,
    COUNT(*) AS orders,
    SUM(total_amount) AS revenue,
    -- เปอร์เซ็นต์ของยอดรวมทั้งหมด (ไม่รวมแถว ROLLUP)
    ROUND(
        SUM(total_amount) * 100.0 / 
        SUM(SUM(total_amount)) OVER (),
        2
    ) AS pct_of_total
FROM orders
WHERE status NOT IN ('cancelled')
GROUP BY YEAR(order_date), MONTH(order_date) WITH ROLLUP
ORDER BY YEAR(order_date), MONTH(order_date);
```

```sql
-- ตัวอย่างที่ 14: ROLLUP พร้อม HAVING
SELECT 
    COALESCE(category, 'TOTAL') AS category,
    COUNT(*) AS products,
    SUM(stock_qty * price) AS inventory_value
FROM products
GROUP BY category WITH ROLLUP
HAVING GROUPING(category) = 1   -- แสดงเฉพาะแถว ROLLUP (Grand Total)
    OR SUM(stock_qty * price) > 1000000;  -- หรือ category ที่มีค่ามาก
```

```sql
-- ตัวอย่างที่ 15: Partial ROLLUP (PostgreSQL)
-- ในบางกรณีต้องการ ROLLUP เฉพาะบางคอลัมน์
-- PostgreSQL: GROUP BY col1, ROLLUP(col2, col3)
-- = จัดกลุ่มตาม col1 เสมอ แต่ ROLLUP เฉพาะ col2, col3

-- MySQL ไม่รองรับ Partial ROLLUP โดยตรง
-- ต้องใช้ GROUPING SETS แทน (จะเรียนใน Part 037)
```

```sql
-- ตัวอย่างที่ 16: Sales Report พร้อม Percentage Subtotals
SELECT 
    CASE WHEN GROUPING(YEAR(o.order_date)) = 1 THEN 'ALL YEARS'
         ELSE CAST(YEAR(o.order_date) AS CHAR) END AS year,
    CASE WHEN GROUPING(p.category) = 1 
              AND GROUPING(YEAR(o.order_date)) = 0 THEN 'YEAR TOTAL'
         WHEN GROUPING(p.category) = 1 THEN 'GRAND TOTAL'
         ELSE p.category END AS category,
    SUM(oi.quantity * oi.unit_price) AS revenue,
    ROUND(
        SUM(oi.quantity * oi.unit_price) * 100.0 /
        SUM(SUM(oi.quantity * oi.unit_price)) OVER (),
        2
    ) AS pct_of_total
FROM orders o
JOIN order_items oi ON o.order_id = oi.order_id
JOIN products p ON oi.product_id = p.product_id
WHERE o.status NOT IN ('cancelled')
GROUP BY YEAR(o.order_date), p.category WITH ROLLUP
ORDER BY YEAR(o.order_date), p.category;
```

---

## 10. Complete Financial Report Example

```sql
-- ตัวอย่างที่ 17: Annual Sales Report with All Subtotals
SELECT 
    LPAD('', (2 - GROUPING(YEAR(order_date)) - GROUPING(QUARTER(order_date))) * 4, ' ')
    || CASE 
        WHEN GROUPING(YEAR(order_date)) = 1     THEN '▌GRAND TOTAL'
        WHEN GROUPING(QUARTER(order_date)) = 1  THEN CONCAT(YEAR(order_date), ' TOTAL')
        ELSE CONCAT('  ', YEAR(order_date), ' Q', QUARTER(order_date))
    END AS period,
    COUNT(*) AS orders,
    FORMAT(SUM(total_amount), 0) AS gross_revenue,
    FORMAT(SUM(discount_amt), 0) AS total_discounts,
    FORMAT(SUM(total_amount - discount_amt), 0) AS net_revenue,
    FORMAT(AVG(total_amount), 0) AS avg_order_value
FROM orders
WHERE status NOT IN ('cancelled')
GROUP BY YEAR(order_date), QUARTER(order_date) WITH ROLLUP
ORDER BY YEAR(order_date), QUARTER(order_date);
```

```sql
-- ตัวอย่างที่ 18: เปรียบเทียบ ROLLUP กับ UNION ALL (ทำงานเหมือนกันแต่ ROLLUP สั้นกว่า)

-- วิธีเก่า: ใช้ UNION ALL (ยาวมาก)
SELECT category, NULL AS brand, COUNT(*), SUM(price * stock_qty) FROM products GROUP BY category
UNION ALL
SELECT category, brand, COUNT(*), SUM(price * stock_qty) FROM products GROUP BY category, brand
UNION ALL
SELECT NULL, NULL, COUNT(*), SUM(price * stock_qty) FROM products
ORDER BY category, brand;

-- วิธีใหม่: ใช้ ROLLUP (สั้นกว่ามาก)
SELECT 
    COALESCE(category, 'TOTAL') AS category,
    COALESCE(brand, 'ALL BRANDS') AS brand,
    COUNT(*),
    SUM(price * stock_qty)
FROM products
GROUP BY category, brand WITH ROLLUP
ORDER BY category, brand;
```

```sql
-- ตัวอย่างที่ 19: Department Budget vs Actual Report
SELECT 
    CASE WHEN GROUPING(d.department_name) = 1 
         THEN '=== COMPANY TOTAL ===' 
         ELSE d.department_name END AS department,
    d.budget AS allocated_budget,
    SUM(e.salary) * 12 AS actual_annual_payroll,
    d.budget - SUM(e.salary) * 12 AS variance,
    ROUND((d.budget - SUM(e.salary) * 12) * 100.0 / NULLIF(d.budget, 0), 1) AS variance_pct,
    CASE 
        WHEN d.budget > SUM(e.salary) * 12 THEN 'Under Budget'
        WHEN d.budget < SUM(e.salary) * 12 THEN 'Over Budget'
        ELSE 'On Budget'
    END AS budget_status
FROM departments d
JOIN employees e ON d.department_id = e.department_id
WHERE e.salary IS NOT NULL
GROUP BY d.department_id, d.department_name, d.budget WITH ROLLUP
HAVING GROUPING(d.department_name) = 1 OR COUNT(e.employee_id) > 0;
```

```sql
-- ตัวอย่างที่ 20: Inventory Valuation Report with ROLLUP
SELECT 
    CASE WHEN GROUPING(category) = 1 THEN '▌ TOTAL INVENTORY' ELSE category END AS category,
    CASE WHEN GROUPING(brand) = 1 
              AND GROUPING(category) = 0 THEN '  Category Total'
         WHEN GROUPING(brand) = 1 THEN '▌ TOTAL INVENTORY'
         ELSE brand END AS brand,
    COUNT(*) AS sku_count,
    SUM(stock_qty) AS total_units,
    FORMAT(SUM(cost * stock_qty), 0) AS cost_value,
    FORMAT(SUM(price * stock_qty), 0) AS retail_value,
    FORMAT(SUM((price - cost) * stock_qty), 0) AS potential_profit
FROM products
WHERE is_active = TRUE
GROUP BY category, brand WITH ROLLUP
ORDER BY category, brand;
```

---

## 11. ROLLUP กับ NULL Values

```sql
-- ตัวอย่างที่ 21: ปัญหา: แยกแยะ NULL จากข้อมูลกับ NULL จาก ROLLUP
-- เมื่อข้อมูลจริงมี NULL ใน column จะทำให้สับสนกับ NULL จาก ROLLUP

-- สมมติว่ามีพนักงานที่ไม่มีแผนก (department_id = NULL)
-- ต้องใช้ GROUPING() เพื่อแยกแยะ

SELECT 
    CASE 
        WHEN GROUPING(department_id) = 1 THEN 'GRAND TOTAL'
        WHEN department_id IS NULL THEN 'ไม่มีแผนก'
        ELSE CAST(department_id AS CHAR)
    END AS department,
    COUNT(*) AS headcount,
    SUM(salary) AS total_salary
FROM employees
GROUP BY department_id WITH ROLLUP;
```

```sql
-- ตัวอย่างที่ 22: Best Practice - ใช้ GROUPING() เสมอเมื่อมี NULL ในข้อมูล
SELECT 
    CASE 
        WHEN GROUPING(p.category) = 1      THEN 'ALL CATEGORIES'
        WHEN p.category IS NULL             THEN 'Uncategorized'
        ELSE p.category
    END AS category,
    CASE 
        WHEN GROUPING(p.brand) = 1 
             AND GROUPING(p.category) = 0  THEN 'All Brands in Category'
        WHEN GROUPING(p.brand) = 1          THEN 'ALL BRANDS'
        WHEN p.brand IS NULL                THEN 'Unknown Brand'
        ELSE p.brand
    END AS brand,
    COUNT(*) AS products,
    SUM(stock_qty * price) AS inventory_value
FROM products
GROUP BY p.category, p.brand WITH ROLLUP
ORDER BY p.category, p.brand;
```

---

## 12. Performance Considerations

```sql
-- ตัวอย่างที่ 23: ROLLUP vs Multiple UNION ALL - Performance Test

-- EXPLAIN ANALYZE สำหรับ ROLLUP (PostgreSQL)
-- EXPLAIN ANALYZE
SELECT category, COUNT(*), SUM(price)
FROM products
GROUP BY ROLLUP(category);

-- ROLLUP มักจะเร็วกว่า UNION ALL เพราะ:
-- 1. ผ่านข้อมูลครั้งเดียว
-- 2. Database optimizer สามารถ optimize ได้ดีกว่า
-- 3. ไม่ต้อง sort/deduplicate หลายรอบ
```

---

## 13. ROLLUP ใน Reporting Queries

```sql
-- ตัวอย่างที่ 24: Monthly KPI Report with Annual Rollup
SELECT 
    CASE WHEN GROUPING(YEAR(order_date)) = 1 THEN 'TOTAL 2023-2024'
         ELSE CAST(YEAR(order_date) AS CHAR) END AS year,
    CASE WHEN GROUPING(MONTH(order_date)) = 1 
              AND GROUPING(YEAR(order_date)) = 0 THEN 'Annual Total'
         WHEN GROUPING(MONTH(order_date)) = 1 THEN 'GRAND TOTAL'
         ELSE DATE_FORMAT(
             MAKEDATE(YEAR(order_date), 1) + INTERVAL (MONTH(order_date)-1) MONTH,
             '%b'
         ) END AS month,
    -- Orders KPI
    COUNT(*) AS total_orders,
    COUNT(CASE WHEN status = 'delivered' THEN 1 END) AS completed,
    COUNT(CASE WHEN status = 'cancelled' THEN 1 END) AS cancelled,
    -- Revenue KPI
    FORMAT(SUM(CASE WHEN status != 'cancelled' THEN total_amount ELSE 0 END), 0) AS revenue,
    FORMAT(AVG(CASE WHEN status != 'cancelled' THEN total_amount END), 0) AS avg_order
FROM orders
GROUP BY YEAR(order_date), MONTH(order_date) WITH ROLLUP
ORDER BY YEAR(order_date), MONTH(order_date);
```

```sql
-- ตัวอย่างที่ 25: สรุปรายงานแบบ Top-Down Hierarchy
-- ใช้ ROLLUP สำหรับรายงานที่มีโครงสร้างแบบ Hierarchical
SELECT 
    CASE WHEN GROUPING(c.province) = 1 THEN '=== ทุกจังหวัด ==='
         ELSE c.province END AS province,
    CASE WHEN GROUPING(c.city) = 1 
              AND GROUPING(c.province) = 0 THEN '  (Province Total)'
         WHEN GROUPING(c.city) = 1 THEN '=== ทุกจังหวัด ==='
         ELSE CONCAT('  ', c.city) END AS city,
    COUNT(DISTINCT c.customer_id) AS customers,
    COUNT(o.order_id) AS orders,
    FORMAT(SUM(o.total_amount), 0) AS revenue
FROM customers c
LEFT JOIN orders o ON c.customer_id = o.customer_id 
    AND o.status NOT IN ('cancelled')
GROUP BY c.province, c.city WITH ROLLUP
ORDER BY c.province, c.city;
```

---

## สรุปบทที่ 35

### เมื่อไหร่ควรใช้ ROLLUP?

1. **รายงานการเงิน** ที่ต้องการ Subtotal หลายระดับ
2. **รายงาน HR** แสดงยอดรวมต่อแผนกและยอดรวมทั้งบริษัท
3. **Sales Report** แสดงยอดรายเดือน รายไตรมาส รายปี
4. **Inventory Report** แสดงมูลค่าต่อ Category และรวมทั้งหมด

### ความแตกต่างระหว่าง Database

| Feature | MySQL | PostgreSQL | SQL Server |
|---------|-------|-----------|------------|
| Syntax | `GROUP BY ... WITH ROLLUP` | `GROUP BY ROLLUP(...)` | `GROUP BY ROLLUP(...)` |
| GROUPING() | Version 8.0+ | Yes | Yes |
| GROUPING_ID() | Version 8.0+ | Yes | Yes |

---

## แบบฝึกหัดท้ายบท

### แบบฝึกหัดที่ 1
สร้างรายงาน ROLLUP แสดงจำนวนพนักงานและเงินเดือนรวมต่อแผนก พร้อม Grand Total

```sql
-- เฉลย:
SELECT 
    COALESCE(d.department_name, 'COMPANY TOTAL') AS department,
    COUNT(e.employee_id) AS headcount,
    SUM(e.salary) AS total_salary
FROM departments d
JOIN employees e ON d.department_id = e.department_id
WHERE e.salary IS NOT NULL
GROUP BY d.department_name WITH ROLLUP;
```

### แบบฝึกหัดที่ 2
สร้างรายงาน ROLLUP ยอดขายแยกตาม category และ brand

```sql
-- เฉลย:
SELECT 
    COALESCE(p.category, 'TOTAL') AS category,
    COALESCE(p.brand, 'All Brands') AS brand,
    COUNT(DISTINCT p.product_id) AS products,
    SUM(oi.quantity * oi.unit_price) AS revenue
FROM products p
JOIN order_items oi ON p.product_id = oi.product_id
JOIN orders o ON oi.order_id = o.order_id
WHERE o.status NOT IN ('cancelled')
GROUP BY p.category, p.brand WITH ROLLUP
ORDER BY p.category, p.brand;
```

### แบบฝึกหัดที่ 3
ใช้ GROUPING() เพื่อแยกแถว ROLLUP จากข้อมูลปกติ

```sql
-- เฉลย:
SELECT 
    GROUPING(category) AS is_rollup,
    CASE WHEN GROUPING(category) = 1 THEN 'ALL CATEGORIES'
         ELSE category END AS category,
    COUNT(*) AS products,
    SUM(stock_qty * price) AS inventory_value
FROM products
GROUP BY category WITH ROLLUP;
```

### แบบฝึกหัดที่ 4
สร้างรายงานยอดขายรายปีและรายเดือนด้วย ROLLUP

```sql
-- เฉลย:
SELECT 
    CASE WHEN GROUPING(YEAR(order_date)) = 1 THEN 'GRAND TOTAL'
         ELSE CAST(YEAR(order_date) AS CHAR) END AS year,
    CASE WHEN GROUPING(MONTH(order_date)) = 1 
              AND GROUPING(YEAR(order_date)) = 0 THEN 'Year Total'
         WHEN GROUPING(MONTH(order_date)) = 1 THEN 'GRAND TOTAL'
         ELSE CAST(MONTH(order_date) AS CHAR) END AS month,
    COUNT(*) AS orders,
    SUM(total_amount) AS revenue
FROM orders
WHERE status NOT IN ('cancelled')
GROUP BY YEAR(order_date), MONTH(order_date) WITH ROLLUP
ORDER BY YEAR(order_date), MONTH(order_date);
```

### แบบฝึกหัดที่ 5
สร้าง Inventory Report ด้วย ROLLUP แสดงมูลค่าสต็อกต่อ category + Grand Total

```sql
-- เฉลย:
SELECT 
    CASE WHEN GROUPING(category) = 1 THEN 'TOTAL INVENTORY'
         ELSE category END AS category,
    COUNT(*) AS sku_count,
    SUM(stock_qty) AS total_units,
    SUM(cost * stock_qty) AS cost_value,
    SUM(price * stock_qty) AS retail_value
FROM products
WHERE is_active = TRUE
GROUP BY category WITH ROLLUP
ORDER BY GROUPING(category), category;
```

### แบบฝึกหัดที่ 6
เปรียบเทียบ ROLLUP กับ UNION ALL สำหรับการนับพนักงานต่อแผนก

```sql
-- เฉลย: UNION ALL version
SELECT department_id, NULL AS subdept, COUNT(*) FROM employees GROUP BY department_id
UNION ALL
SELECT NULL, NULL, COUNT(*) FROM employees;

-- ROLLUP version
SELECT 
    COALESCE(CAST(department_id AS CHAR), 'TOTAL') AS department_id,
    COUNT(*) AS employee_count
FROM employees
GROUP BY department_id WITH ROLLUP;
```

### แบบฝึกหัดที่ 7
สร้าง ROLLUP Report แสดงยอดขายต่อจังหวัดและเมือง

```sql
-- เฉลย:
SELECT 
    COALESCE(c.province, 'ALL PROVINCES') AS province,
    COALESCE(c.city, COALESCE(c.province, 'ALL')) AS city,
    COUNT(DISTINCT c.customer_id) AS customers,
    SUM(o.total_amount) AS revenue
FROM customers c
JOIN orders o ON c.customer_id = o.customer_id
WHERE o.status != 'cancelled'
GROUP BY c.province, c.city WITH ROLLUP
ORDER BY c.province, c.city;
```

### แบบฝึกหัดที่ 8
ใช้ GROUPING_ID() เพื่อระบุระดับของแถว ROLLUP

```sql
-- เฉลย:
SELECT 
    GROUPING_ID(category, brand) AS level,
    COALESCE(category, 'TOTAL') AS category,
    COALESCE(brand, 'ALL') AS brand,
    COUNT(*) AS products,
    SUM(price * stock_qty) AS value
FROM products
GROUP BY category, brand WITH ROLLUP
ORDER BY category, brand;
```

### แบบฝึกหัดที่ 9
สร้างรายงาน Q1-Q4 Revenue ด้วย ROLLUP แสดงยอดรายปี

```sql
-- เฉลย:
SELECT 
    COALESCE(CAST(YEAR(order_date) AS CHAR), 'GRAND TOTAL') AS year,
    COALESCE(CONCAT('Q', CAST(QUARTER(order_date) AS CHAR)), 'Annual Total') AS quarter,
    COUNT(*) AS orders,
    SUM(total_amount) AS revenue,
    SUM(discount_amt) AS discounts
FROM orders
WHERE status NOT IN ('cancelled')
GROUP BY YEAR(order_date), QUARTER(order_date) WITH ROLLUP
ORDER BY YEAR(order_date), QUARTER(order_date);
```

### แบบฝึกหัดที่ 10
สร้าง Complete Financial Summary ด้วย ROLLUP สำหรับ Executive Meeting

```sql
-- เฉลย:
SELECT 
    CASE WHEN GROUPING(YEAR(o.order_date)) = 1 THEN '◆ GRAND TOTAL'
         ELSE CONCAT('Year ', YEAR(o.order_date)) END AS period,
    CASE WHEN GROUPING(p.category) = 1 
              AND GROUPING(YEAR(o.order_date)) = 0 THEN '  Annual Subtotal'
         WHEN GROUPING(p.category) = 1 THEN '◆ GRAND TOTAL'
         ELSE CONCAT('  ', p.category) END AS category,
    COUNT(DISTINCT o.order_id) AS orders,
    FORMAT(SUM(oi.quantity * oi.unit_price), 0) AS revenue,
    FORMAT(SUM(oi.quantity * p.cost), 0) AS cost,
    FORMAT(SUM(oi.quantity * (oi.unit_price - p.cost)), 0) AS gross_profit,
    ROUND(
        SUM(oi.quantity * (oi.unit_price - p.cost)) * 100.0 /
        NULLIF(SUM(oi.quantity * oi.unit_price), 0), 1
    ) AS margin_pct
FROM orders o
JOIN order_items oi ON o.order_id = oi.order_id
JOIN products p ON oi.product_id = p.product_id
WHERE o.status NOT IN ('cancelled')
GROUP BY YEAR(o.order_date), p.category WITH ROLLUP
ORDER BY YEAR(o.order_date), p.category;
```

---

*จบบทที่ 035 - ROLLUP: Multi-level Subtotals*

*บทถัดไป: Part 036 - CUBE: All Combinations*
