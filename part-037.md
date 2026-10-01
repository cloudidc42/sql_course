# Part 037: GROUPING SETS - Custom Grouping Combinations

## บทนำ (Introduction)

`GROUPING SETS` เป็น extension ของ GROUP BY ที่ยืดหยุ่นที่สุด ช่วยให้เราเลือกเฉพาะ **grouping combinations ที่ต้องการ** โดยไม่ต้องสร้างทุก combination เหมือน CUBE หรือสร้างตามลำดับชั้นเหมือน ROLLUP

**สรุปความสัมพันธ์:**
```
ROLLUP(A, B, C) ≡ GROUPING SETS((A,B,C), (A,B), (A), ())
CUBE(A, B, C)   ≡ GROUPING SETS((A,B,C), (A,B), (A,C), (B,C), (A), (B), (C), ())
GROUPING SETS   = เลือกเฉพาะที่ต้องการ
```

---

## การตั้งค่าฐานข้อมูล

```sql
USE ecommerce_db;

-- ข้อมูลพร้อมใช้จาก Part 031-036
-- ตรวจสอบข้อมูล
SELECT 
    'orders' AS tbl, COUNT(*) AS cnt FROM orders
UNION ALL SELECT 'products', COUNT(*) FROM products
UNION ALL SELECT 'employees', COUNT(*) FROM employees
UNION ALL SELECT 'customers', COUNT(*) FROM customers;
```

---

## 1. GROUPING SETS Syntax

### PostgreSQL / SQL Server (Native Support)
```sql
SELECT col1, col2, aggregate_fn(col3)
FROM table
GROUP BY GROUPING SETS (
    (col1, col2),   -- group by both
    (col1),         -- group by col1 only
    (col2),         -- group by col2 only
    ()              -- grand total
);
```

### MySQL 8.0+ (Native Support)
```sql
-- MySQL 8.0 รองรับ GROUPING SETS
SELECT col1, col2, COUNT(*)
FROM table
GROUP BY GROUPING SETS ((col1, col2), (col1), (col2), ());
```

---

## 2. GROUPING SETS พื้นฐาน

```sql
-- ตัวอย่างที่ 1: GROUPING SETS พื้นฐาน (PostgreSQL/MySQL 8.0)
SELECT 
    p.category,
    p.brand,
    COUNT(*) AS products,
    SUM(p.price * p.stock_qty) AS inventory_value
FROM products p
GROUP BY GROUPING SETS (
    (category, brand),  -- detail: ต่อ category+brand
    (category),         -- subtotal ต่อ category
    (brand),            -- subtotal ต่อ brand ← จะไม่มีใน ROLLUP
    ()                  -- grand total
)
ORDER BY category NULLS LAST, brand NULLS LAST;
```

```sql
-- ตัวอย่างที่ 2: GROUPING SETS ที่เลือกเฉพาะที่ต้องการ
-- ต้องการแค่ (category+brand) และ (grand total) ไม่ต้องการ subtotals อื่น
SELECT 
    COALESCE(category, 'ALL') AS category,
    COALESCE(brand, 'ALL') AS brand,
    COUNT(*) AS products,
    SUM(price) AS total_price
FROM products
GROUP BY GROUPING SETS (
    (category, brand),
    ()
)
ORDER BY category NULLS LAST, brand NULLS LAST;
```

```sql
-- ตัวอย่างที่ 3: GROUPING SETS กับ Aggregate Functions
SELECT 
    GROUPING(category) AS g_cat,
    GROUPING(brand) AS g_brand,
    COALESCE(category, 'N/A') AS category,
    COALESCE(brand, 'N/A') AS brand,
    COUNT(*) AS count,
    SUM(price * stock_qty) AS value
FROM products
GROUP BY GROUPING SETS ((category, brand), (category), (brand), ())
ORDER BY category NULLS LAST, brand NULLS LAST;
```

---

## 3. GROUPING SETS vs UNION ALL

```sql
-- ตัวอย่างที่ 4: เปรียบเทียบ - UNION ALL (วิธีเก่า)
SELECT 'category+brand' AS grp_type, category, brand, 
       COUNT(*) AS products, SUM(price*stock_qty) AS value
FROM products GROUP BY category, brand
UNION ALL
SELECT 'category only', category, NULL,
       COUNT(*), SUM(price*stock_qty)
FROM products GROUP BY category
UNION ALL
SELECT 'brand only', NULL, brand,
       COUNT(*), SUM(price*stock_qty)
FROM products GROUP BY brand
UNION ALL
SELECT 'grand total', NULL, NULL,
       COUNT(*), SUM(price*stock_qty)
FROM products
ORDER BY category NULLS LAST, brand NULLS LAST;

-- ตัวอย่างที่ 5: เปรียบเทียบ - GROUPING SETS (วิธีใหม่ สั้นกว่า)
SELECT 
    CASE WHEN GROUPING(category)=0 AND GROUPING(brand)=0 THEN 'category+brand'
         WHEN GROUPING(category)=0 THEN 'category only'
         WHEN GROUPING(brand)=0 THEN 'brand only'
         ELSE 'grand total' END AS grp_type,
    category, brand,
    COUNT(*) AS products,
    SUM(price * stock_qty) AS value
FROM products
GROUP BY GROUPING SETS ((category, brand), (category), (brand), ())
ORDER BY category NULLS LAST, brand NULLS LAST;
```

---

## 4. Custom Grouping Combinations

```sql
-- ตัวอย่างที่ 6: เลือกเฉพาะ groupings ที่ business ต้องการ
-- Business requirement: ต้องการรายงาน 3 แบบในคำสั่งเดียว:
-- 1. ยอดขายต่อ Category
-- 2. ยอดขายต่อ Province
-- 3. Grand Total

SELECT 
    COALESCE(p.category, 'ALL CATEGORIES') AS category,
    COALESCE(c.province, 'ALL PROVINCES') AS province,
    COUNT(DISTINCT o.order_id) AS orders,
    SUM(oi.quantity * oi.unit_price) AS revenue
FROM orders o
JOIN order_items oi ON o.order_id = oi.order_id
JOIN products p ON oi.product_id = p.product_id
JOIN customers c ON o.customer_id = c.customer_id
WHERE o.status NOT IN ('cancelled')
GROUP BY GROUPING SETS (
    (p.category),           -- รายงาน 1: ต่อ Category
    (c.province),           -- รายงาน 2: ต่อ Province
    ()                      -- รายงาน 3: Grand Total
)
ORDER BY 
    GROUPING(p.category) + GROUPING(c.province),
    p.category NULLS LAST,
    c.province NULLS LAST;
```

```sql
-- ตัวอย่างที่ 7: GROUPING SETS กับ Multiple Columns ต่อ Set
SELECT 
    COALESCE(CAST(YEAR(o.order_date) AS CHAR), 'ALL') AS year,
    COALESCE(p.category, 'ALL') AS category,
    COALESCE(o.payment_method, 'ALL') AS payment_method,
    COUNT(*) AS orders,
    SUM(o.total_amount) AS revenue
FROM orders o
JOIN order_items oi ON o.order_id = oi.order_id
JOIN products p ON oi.product_id = p.product_id
WHERE o.status NOT IN ('cancelled')
GROUP BY GROUPING SETS (
    (YEAR(o.order_date), p.category),       -- ปี + Category
    (YEAR(o.order_date), o.payment_method), -- ปี + Payment
    (p.category, o.payment_method),          -- Category + Payment
    ()                                       -- Grand Total
)
ORDER BY year, category, payment_method;
```

---

## 5. Performance Benefits of GROUPING SETS

```sql
-- ตัวอย่างที่ 8: GROUPING SETS เร็วกว่า UNION ALL
-- เพราะผ่านข้อมูลแค่ครั้งเดียว

-- ✅ ดี: GROUPING SETS (ผ่านข้อมูลครั้งเดียว)
SELECT category, brand, COUNT(*), SUM(price)
FROM products
GROUP BY GROUPING SETS ((category, brand), (category), ());

-- ❌ แย่กว่า: UNION ALL (ผ่านข้อมูล 3 ครั้ง)
SELECT category, brand, COUNT(*), SUM(price) FROM products GROUP BY category, brand
UNION ALL SELECT category, NULL, COUNT(*), SUM(price) FROM products GROUP BY category
UNION ALL SELECT NULL, NULL, COUNT(*), SUM(price) FROM products;
```

---

## 6. Complex Reporting Scenarios

```sql
-- ตัวอย่างที่ 9: Multi-Report ในคำสั่งเดียว
-- สร้าง Sales Performance Report ที่มีหลาย View พร้อมกัน
SELECT 
    CASE 
        WHEN GROUPING(p.category)=0 AND GROUPING(o.status)=0 THEN 'Category × Status'
        WHEN GROUPING(p.category)=0 AND GROUPING(o.status)=1 THEN 'Category Total'
        WHEN GROUPING(p.category)=1 AND GROUPING(o.status)=0 THEN 'Status Total'
        ELSE 'Grand Total'
    END AS report_level,
    COALESCE(p.category, 'ALL') AS category,
    COALESCE(o.status, 'ALL') AS status,
    COUNT(*) AS orders,
    SUM(o.total_amount) AS revenue
FROM orders o
JOIN order_items oi ON o.order_id = oi.order_id
JOIN products p ON oi.product_id = p.product_id
GROUP BY GROUPING SETS (
    (p.category, o.status),
    (p.category),
    (o.status),
    ()
)
ORDER BY 
    GROUPING(p.category) + GROUPING(o.status),
    p.category NULLS LAST,
    o.status NULLS LAST;
```

```sql
-- ตัวอย่างที่ 10: Financial Report with Multiple Aggregation Levels
SELECT 
    CASE 
        WHEN GROUPING(d.department_name)=0 AND GROUPING(e.job_title)=0 THEN 'Detail'
        WHEN GROUPING(d.department_name)=0 THEN 'Dept Subtotal'
        WHEN GROUPING(e.job_title)=0 THEN 'Title Subtotal'
        ELSE 'Company Total'
    END AS level,
    COALESCE(d.department_name, 'ALL DEPARTMENTS') AS department,
    COALESCE(e.job_title, 'ALL TITLES') AS job_title,
    COUNT(*) AS headcount,
    MIN(e.salary) AS min_salary,
    MAX(e.salary) AS max_salary,
    ROUND(AVG(e.salary), 0) AS avg_salary,
    SUM(e.salary) AS total_salary
FROM employees e
JOIN departments d ON e.department_id = d.department_id
WHERE e.salary IS NOT NULL
GROUP BY GROUPING SETS (
    (d.department_name, e.job_title),
    (d.department_name),
    (e.job_title),
    ()
)
ORDER BY level DESC, department, job_title;
```

```sql
-- ตัวอย่างที่ 11: Regional Sales Dashboard
SELECT 
    COALESCE(c.province, 'ALL PROVINCES') AS province,
    COALESCE(c.city, 'ALL CITIES') AS city,
    COALESCE(p.category, 'ALL CATEGORIES') AS category,
    COUNT(DISTINCT o.order_id) AS orders,
    COUNT(DISTINCT o.customer_id) AS customers,
    SUM(o.total_amount) AS revenue
FROM orders o
JOIN customers c ON o.customer_id = c.customer_id
JOIN order_items oi ON o.order_id = oi.order_id
JOIN products p ON oi.product_id = p.product_id
WHERE o.status NOT IN ('cancelled')
GROUP BY GROUPING SETS (
    (c.province, c.city, p.category),  -- Full detail
    (c.province, c.city),              -- Geographic detail
    (c.province, p.category),          -- Province × Category
    (c.province),                       -- Province only
    (p.category),                       -- Category only
    ()                                  -- Grand Total
)
ORDER BY 
    COALESCE(c.province, 'ZZZ'),
    COALESCE(c.city, 'ZZZ'),
    COALESCE(p.category, 'ZZZ');
```

---

## 7. GROUPING SETS กับ ROLLUP และ CUBE รวมกัน

```sql
-- ตัวอย่างที่ 12: ผสม GROUPING SETS กับ ROLLUP (PostgreSQL/SQL Server)
-- GROUP BY GROUPING SETS ((a, b), ROLLUP(c, d))
-- เทียบเท่ากับ:
-- GROUP BY GROUPING SETS ((a,b), (a,b,c,d), (a,b,c), (a,b))

-- MySQL equivalent:
SELECT 
    COALESCE(CAST(YEAR(o.order_date) AS CHAR), 'ALL') AS year,
    COALESCE(CAST(QUARTER(o.order_date) AS CHAR), 'ALL Q') AS quarter,
    COALESCE(p.category, 'ALL') AS category,
    SUM(o.total_amount) AS revenue
FROM orders o
JOIN order_items oi ON o.order_id = oi.order_id
JOIN products p ON oi.product_id = p.product_id
WHERE o.status NOT IN ('cancelled')
GROUP BY YEAR(o.order_date), QUARTER(o.order_date), p.category WITH ROLLUP
HAVING GROUPING(YEAR(o.order_date)) = 1 
    OR GROUPING(p.category) = 1
    OR (GROUPING(YEAR(o.order_date)) = 0 AND GROUPING(QUARTER(o.order_date)) = 0 
        AND GROUPING(p.category) = 0)
ORDER BY year, quarter, category;
```

```sql
-- ตัวอย่างที่ 13: GROUPING SETS ที่ซับซ้อน - Monthly + Annual Report
SELECT 
    CASE 
        WHEN GROUPING(YEAR(order_date))=1 THEN 'GRAND TOTAL'
        WHEN GROUPING(MONTH(order_date))=1 THEN CONCAT(YEAR(order_date), ' TOTAL')
        ELSE CONCAT(YEAR(order_date), '-', LPAD(MONTH(order_date), 2, '0'))
    END AS period,
    COUNT(*) AS orders,
    SUM(total_amount) AS revenue,
    ROUND(AVG(total_amount), 0) AS avg_order
FROM orders
WHERE status NOT IN ('cancelled')
GROUP BY GROUPING SETS (
    (YEAR(order_date), MONTH(order_date)),  -- Monthly
    (YEAR(order_date)),                      -- Annual
    ()                                        -- Grand Total
)
ORDER BY YEAR(order_date) NULLS LAST, MONTH(order_date) NULLS LAST;
```

---

## 8. ตัวอย่างการใช้งาน Business Scenarios

```sql
-- ตัวอย่างที่ 14: E-commerce Revenue Report
SELECT 
    CASE 
        WHEN GROUPING(YEAR(o.order_date))=0 AND GROUPING(p.category)=0 THEN 'Year × Category'
        WHEN GROUPING(YEAR(o.order_date))=0 THEN 'Year Total'
        WHEN GROUPING(p.category)=0 THEN 'Category Total'
        ELSE 'Grand Total'
    END AS report_type,
    COALESCE(CAST(YEAR(o.order_date) AS CHAR), '—') AS year,
    COALESCE(p.category, '—') AS category,
    COUNT(DISTINCT o.order_id) AS orders,
    SUM(oi.quantity * oi.unit_price) AS revenue,
    ROUND(AVG(o.total_amount), 0) AS avg_order,
    SUM(oi.quantity) AS units_sold
FROM orders o
JOIN order_items oi ON o.order_id = oi.order_id
JOIN products p ON oi.product_id = p.product_id
WHERE o.status NOT IN ('cancelled')
GROUP BY GROUPING SETS (
    (YEAR(o.order_date), p.category),
    (YEAR(o.order_date)),
    (p.category),
    ()
)
ORDER BY 
    CASE WHEN GROUPING(YEAR(o.order_date))=1 AND GROUPING(p.category)=1 THEN 4
         WHEN GROUPING(YEAR(o.order_date))=0 AND GROUPING(p.category)=1 THEN 3
         WHEN GROUPING(YEAR(o.order_date))=1 AND GROUPING(p.category)=0 THEN 2
         ELSE 1 END,
    YEAR(o.order_date), p.category;
```

```sql
-- ตัวอย่างที่ 15: HR Analytics Report
SELECT 
    COALESCE(d.department_name, 'COMPANY') AS department,
    COALESCE(
        CASE WHEN e.salary >= 80000 THEN 'Senior'
             WHEN e.salary >= 50000 THEN 'Mid-Level'
             ELSE 'Junior' END,
        'ALL LEVELS'
    ) AS seniority_level,
    COUNT(*) AS headcount,
    ROUND(AVG(e.salary), 0) AS avg_salary,
    SUM(e.salary) AS payroll
FROM employees e
JOIN departments d ON e.department_id = d.department_id
WHERE e.salary IS NOT NULL
GROUP BY GROUPING SETS (
    (d.department_name, 
     CASE WHEN e.salary >= 80000 THEN 'Senior'
          WHEN e.salary >= 50000 THEN 'Mid-Level'
          ELSE 'Junior' END),
    (d.department_name),
    (CASE WHEN e.salary >= 80000 THEN 'Senior'
          WHEN e.salary >= 50000 THEN 'Mid-Level'
          ELSE 'Junior' END),
    ()
)
ORDER BY department, seniority_level;
```

```sql
-- ตัวอย่างที่ 16: Inventory Report ทุกระดับ
SELECT 
    COALESCE(category, 'ALL CATEGORIES') AS category,
    COALESCE(brand, 'ALL BRANDS') AS brand,
    COUNT(*) AS sku_count,
    SUM(stock_qty) AS total_units,
    SUM(cost * stock_qty) AS cost_value,
    SUM(price * stock_qty) AS retail_value,
    COUNT(CASE WHEN stock_qty < min_stock THEN 1 END) AS low_stock_count
FROM products
WHERE is_active = TRUE
GROUP BY GROUPING SETS (
    (category, brand),
    (category),
    (brand),
    ()
)
ORDER BY 
    GROUPING(category) + GROUPING(brand),
    category NULLS LAST,
    brand NULLS LAST;
```

---

## 9. GROUPING SETS กับ HAVING

```sql
-- ตัวอย่างที่ 17: HAVING กับ GROUPING SETS
SELECT 
    COALESCE(p.category, 'ALL') AS category,
    COALESCE(o.payment_method, 'ALL') AS payment_method,
    COUNT(*) AS orders,
    SUM(o.total_amount) AS revenue
FROM orders o
JOIN order_items oi ON o.order_id = oi.order_id
JOIN products p ON oi.product_id = p.product_id
WHERE o.status NOT IN ('cancelled')
GROUP BY GROUPING SETS (
    (p.category, o.payment_method),
    (p.category),
    ()
)
HAVING 
    GROUPING(p.category) = 1  -- แสดง Grand Total เสมอ
    OR SUM(o.total_amount) > 50000  -- หรือ กลุ่มที่มียอดมากกว่า 50,000
ORDER BY p.category NULLS LAST, o.payment_method NULLS LAST;
```

---

## 10. Advanced GROUPING SETS Patterns

```sql
-- ตัวอย่างที่ 18: Non-overlapping GROUPING SETS
-- ใช้เมื่อต้องการหลาย independent summaries
SELECT 
    'Time Report' AS report_section,
    COALESCE(CAST(YEAR(order_date) AS CHAR), 'TOTAL') AS dimension_value,
    COUNT(*) AS orders,
    SUM(total_amount) AS revenue
FROM orders WHERE status != 'cancelled'
GROUP BY YEAR(order_date) WITH ROLLUP

UNION ALL

SELECT 
    'Status Report',
    COALESCE(status, 'TOTAL'),
    COUNT(*),
    SUM(total_amount)
FROM orders
GROUP BY status WITH ROLLUP

UNION ALL

SELECT 
    'Payment Report',
    COALESCE(payment_method, 'TOTAL'),
    COUNT(*),
    SUM(total_amount)
FROM orders WHERE status != 'cancelled'
GROUP BY payment_method WITH ROLLUP;
```

```sql
-- ตัวอย่างที่ 19: Flexible Reporting Framework
-- สร้าง Report Template ที่ปรับเปลี่ยนได้

-- Configuration: กำหนดว่าต้องการ groupings อะไร
-- ในที่นี้เลือก: (Year), (Category), (Year, Category), ()

SELECT 
    GROUPING(yr) AS g_yr,
    GROUPING(cat) AS g_cat,
    COALESCE(yr, 'ALL YEARS') AS year,
    COALESCE(cat, 'ALL CATS') AS category,
    orders_count,
    revenue
FROM (
    SELECT 
        CAST(YEAR(o.order_date) AS CHAR) AS yr,
        p.category AS cat,
        COUNT(DISTINCT o.order_id) AS orders_count,
        SUM(oi.quantity * oi.unit_price) AS revenue
    FROM orders o
    JOIN order_items oi ON o.order_id = oi.order_id
    JOIN products p ON oi.product_id = p.product_id
    WHERE o.status NOT IN ('cancelled')
    GROUP BY YEAR(o.order_date), p.category
) AS base_data
-- สามารถเพิ่ม GROUPING SETS ที่นี่ใน PostgreSQL/SQL Server
ORDER BY year, category;
```

---

## 11. GROUPING SETS สำหรับ Complete Reporting Suite

```sql
-- ตัวอย่างที่ 20: Complete Business Intelligence Suite
-- รายงานที่ครบครัน สำหรับ Management Meeting

-- Part 1: Revenue by Time
SELECT '1. Revenue by Year' AS report, CAST(YEAR(order_date) AS CHAR) AS dimension, 
    COUNT(*) AS orders, SUM(total_amount) AS revenue
FROM orders WHERE status != 'cancelled' GROUP BY YEAR(order_date)
UNION ALL
-- Part 2: Revenue by Category
SELECT '2. Revenue by Category', p.category,
    COUNT(DISTINCT o.order_id), SUM(oi.quantity * oi.unit_price)
FROM orders o JOIN order_items oi ON o.order_id=oi.order_id
JOIN products p ON oi.product_id=p.product_id
WHERE o.status != 'cancelled' GROUP BY p.category
UNION ALL
-- Part 3: Revenue by Province
SELECT '3. Revenue by Province', c.province,
    COUNT(DISTINCT o.order_id), SUM(o.total_amount)
FROM orders o JOIN customers c ON o.customer_id=c.customer_id
WHERE o.status != 'cancelled' GROUP BY c.province
UNION ALL
-- Part 4: Revenue by Payment
SELECT '4. Revenue by Payment', payment_method,
    COUNT(*), SUM(total_amount)
FROM orders WHERE status != 'cancelled' GROUP BY payment_method
ORDER BY report, revenue DESC;
```

```sql
-- ตัวอย่างที่ 21: GROUPING SETS ที่มี Nested Groups
-- ต้องการ:
-- Level 1: ต่อปี (all categories)
-- Level 2: ต่อ category (all years)
-- Level 3: ต่อปี+category
-- Level 4: Grand Total

SELECT 
    CASE 
        WHEN GROUPING(YEAR(o.order_date)) = 0 AND GROUPING(p.category) = 0 
            THEN 'Year × Category'
        WHEN GROUPING(YEAR(o.order_date)) = 0 
            THEN 'Year Subtotal'
        WHEN GROUPING(p.category) = 0 
            THEN 'Category Subtotal'
        ELSE 'Grand Total'
    END AS report_level,
    COALESCE(CAST(YEAR(o.order_date) AS CHAR), 'ALL') AS year,
    COALESCE(p.category, 'ALL CATEGORIES') AS category,
    COUNT(DISTINCT o.order_id) AS order_count,
    SUM(oi.quantity * oi.unit_price) AS revenue,
    ROUND(AVG(o.total_amount), 0) AS avg_order_value
FROM orders o
JOIN order_items oi ON o.order_id = oi.order_id
JOIN products p ON oi.product_id = p.product_id
WHERE o.status NOT IN ('cancelled')
GROUP BY GROUPING SETS (
    (YEAR(o.order_date), p.category),
    (YEAR(o.order_date)),
    (p.category),
    ()
)
ORDER BY 
    COALESCE(CAST(YEAR(o.order_date) AS CHAR), 'ZZZ'),
    COALESCE(p.category, 'ZZZ');
```

```sql
-- ตัวอย่างที่ 22: Comparison Report - Current vs Previous Year
SELECT 
    COALESCE(p.category, 'ALL CATEGORIES') AS category,
    SUM(CASE WHEN YEAR(o.order_date) = 2024 THEN oi.quantity * oi.unit_price ELSE 0 END) AS revenue_2024,
    SUM(CASE WHEN YEAR(o.order_date) = 2023 THEN oi.quantity * oi.unit_price ELSE 0 END) AS revenue_2023,
    SUM(CASE WHEN YEAR(o.order_date) = 2024 THEN oi.quantity * oi.unit_price ELSE 0 END) -
    SUM(CASE WHEN YEAR(o.order_date) = 2023 THEN oi.quantity * oi.unit_price ELSE 0 END) AS growth,
    ROUND(
        (SUM(CASE WHEN YEAR(o.order_date) = 2024 THEN oi.quantity * oi.unit_price ELSE 0 END) -
         SUM(CASE WHEN YEAR(o.order_date) = 2023 THEN oi.quantity * oi.unit_price ELSE 0 END)) * 100.0 /
        NULLIF(SUM(CASE WHEN YEAR(o.order_date) = 2023 THEN oi.quantity * oi.unit_price ELSE 0 END), 0),
        2
    ) AS yoy_growth_pct
FROM orders o
JOIN order_items oi ON o.order_id = oi.order_id
JOIN products p ON oi.product_id = p.product_id
WHERE o.status NOT IN ('cancelled')
GROUP BY p.category WITH ROLLUP
ORDER BY GROUPING(p.category), revenue_2024 DESC;
```

```sql
-- ตัวอย่างที่ 23: GROUPING SETS ช่วยลด Code Complexity
-- แทนที่ Query ยาวๆ หลายตัว ใช้ GROUPING SETS ตัวเดียว

-- 1 Query แทน 4 Queries:
SELECT 
    COALESCE(CAST(YEAR(o.order_date) AS CHAR), 'ALL') AS year,
    COALESCE(p.category, 'ALL') AS category,
    COALESCE(c.province, 'ALL') AS province,
    SUM(o.total_amount) AS revenue
FROM orders o
JOIN customers c ON o.customer_id = c.customer_id
JOIN order_items oi ON o.order_id = oi.order_id
JOIN products p ON oi.product_id = p.product_id
WHERE o.status NOT IN ('cancelled')
GROUP BY GROUPING SETS (
    (YEAR(o.order_date), p.category, c.province),  -- Full Detail
    (YEAR(o.order_date), p.category),               -- Year × Category
    (p.category, c.province),                        -- Category × Province
    ()                                               -- Grand Total
)
ORDER BY year, category, province;
```

```sql
-- ตัวอย่างที่ 24: GROUPING SETS สำหรับ Customer Analytics
SELECT 
    CASE 
        WHEN GROUPING(loyalty_tier) = 0 AND GROUPING(city_group) = 0 
            THEN 'Tier × City'
        WHEN GROUPING(loyalty_tier) = 0 THEN 'Tier Subtotal'
        WHEN GROUPING(city_group) = 0 THEN 'City Subtotal'
        ELSE 'Total'
    END AS report_level,
    COALESCE(loyalty_tier, 'ALL TIERS') AS loyalty_tier,
    COALESCE(city_group, 'ALL CITIES') AS city_group,
    COUNT(*) AS customers,
    AVG(loyalty_points) AS avg_points,
    SUM(loyalty_points) AS total_points
FROM (
    SELECT 
        customer_id,
        loyalty_points,
        CASE 
            WHEN loyalty_points >= 5000 THEN 'Platinum'
            WHEN loyalty_points >= 2000 THEN 'Gold'
            WHEN loyalty_points >= 500  THEN 'Silver'
            ELSE 'Bronze'
        END AS loyalty_tier,
        CASE 
            WHEN city = 'กรุงเทพฯ' THEN 'Bangkok'
            WHEN province = 'เชียงใหม่' THEN 'Chiang Mai'
            ELSE 'Other Regions'
        END AS city_group
    FROM customers
) AS classified_customers
GROUP BY GROUPING SETS (
    (loyalty_tier, city_group),
    (loyalty_tier),
    (city_group),
    ()
)
ORDER BY loyalty_tier NULLS LAST, city_group NULLS LAST;
```

```sql
-- ตัวอย่างที่ 25: Executive Dashboard Query ด้วย GROUPING SETS
SELECT 
    'REVENUE SUMMARY' AS section,
    COALESCE(CAST(YEAR(order_date) AS CHAR), 'ALL YEARS') AS dimension,
    COUNT(*) AS orders,
    FORMAT(SUM(CASE WHEN status != 'cancelled' THEN total_amount ELSE 0 END), 0) AS revenue,
    FORMAT(AVG(CASE WHEN status != 'cancelled' THEN total_amount END), 0) AS avg_order,
    COUNT(CASE WHEN status = 'cancelled' THEN 1 END) AS cancellations,
    ROUND(COUNT(CASE WHEN status = 'cancelled' THEN 1 END) * 100.0 / COUNT(*), 1) AS cancel_rate
FROM orders
GROUP BY YEAR(order_date) WITH ROLLUP
ORDER BY YEAR(order_date);
```

---

## สรุปบทที่ 37

### เปรียบเทียบ GROUP BY Extensions

| Extension | Generates | Use Case |
|-----------|-----------|---------|
| `GROUP BY` | กลุ่มที่ระบุ | Basic aggregation |
| `GROUP BY ... WITH ROLLUP` | Hierarchical subtotals | Time series, Hierarchies |
| `GROUP BY CUBE(...)` | All combinations | Cross-dimensional analysis |
| `GROUP BY GROUPING SETS(...)` | Custom combinations | Custom reporting |

### เมื่อไหร่ควรใช้อะไร

- **ROLLUP**: เมื่อมี natural hierarchy (Year > Quarter > Month)
- **CUBE**: เมื่อต้องการวิเคราะห์ทุก dimension combination
- **GROUPING SETS**: เมื่อต้องการ custom groupings ที่ไม่ใช่ทั้ง ROLLUP หรือ CUBE

---

## แบบฝึกหัดท้ายบท

### แบบฝึกหัดที่ 1
เขียน GROUPING SETS ที่แสดง (category, brand), (category), และ Grand Total

```sql
-- เฉลย:
SELECT 
    COALESCE(category, 'ALL') AS category,
    COALESCE(brand, 'ALL') AS brand,
    COUNT(*) AS products,
    SUM(price * stock_qty) AS inventory_value
FROM products
GROUP BY GROUPING SETS ((category, brand), (category), ())
ORDER BY category NULLS LAST, brand NULLS LAST;
```

### แบบฝึกหัดที่ 2
ROLLUP(A, B, C) สร้าง groupings อะไรบ้าง? แปลงเป็น GROUPING SETS

```sql
-- เฉลย:
-- ROLLUP(A, B, C) ≡ GROUPING SETS((A,B,C), (A,B), (A), ())
-- GROUPING SETS equivalent:
SELECT col_a, col_b, col_c, COUNT(*)
FROM some_table
GROUP BY GROUPING SETS ((col_a, col_b, col_c), (col_a, col_b), (col_a), ());
```

### แบบฝึกหัดที่ 3
สร้าง Sales Report ที่มีทั้งยอดรวมต่อ Category, Payment Method และ Grand Total

```sql
-- เฉลย:
SELECT 
    COALESCE(p.category, 'ALL') AS category,
    COALESCE(o.payment_method, 'ALL') AS payment_method,
    COUNT(*) AS orders,
    SUM(o.total_amount) AS revenue
FROM orders o
JOIN order_items oi ON o.order_id = oi.order_id
JOIN products p ON oi.product_id = p.product_id
WHERE o.status != 'cancelled'
GROUP BY GROUPING SETS ((p.category), (o.payment_method), ())
ORDER BY GROUPING(p.category) + GROUPING(o.payment_method), p.category, o.payment_method;
```

### แบบฝึกหัดที่ 4
แสดงความแตกต่างระหว่าง CUBE และ GROUPING SETS ด้วยตัวอย่าง

```sql
-- เฉลย:
-- CUBE(category, brand) สร้าง GROUPING SETS เหมือนกับ:
-- GROUPING SETS((category, brand), (category), (brand), ())

-- ถ้าต้องการเฉพาะ (category+brand) และ () แต่ไม่ต้องการ subtotals:
-- GROUPING SETS((category, brand), ())  -- ใช้ GROUPING SETS
-- vs CUBE(category, brand) -- จะมี subtotals เพิ่มมา

SELECT 
    COALESCE(category, 'ALL') AS category,
    COALESCE(brand, 'ALL') AS brand,
    COUNT(*)
FROM products
GROUP BY GROUPING SETS ((category, brand), ())  -- ไม่มี subtotals
ORDER BY category NULLS LAST, brand NULLS LAST;
```

### แบบฝึกหัดที่ 5
สร้าง HR Report ด้วย GROUPING SETS แสดงยอดต่อแผนก, ต่อ job_title และ Grand Total

```sql
-- เฉลย:
SELECT 
    COALESCE(d.department_name, 'ALL DEPARTMENTS') AS department,
    COALESCE(e.job_title, 'ALL TITLES') AS job_title,
    COUNT(*) AS headcount,
    SUM(e.salary) AS total_salary,
    ROUND(AVG(e.salary), 0) AS avg_salary
FROM employees e
JOIN departments d ON e.department_id = d.department_id
WHERE e.salary IS NOT NULL
GROUP BY GROUPING SETS (
    (d.department_name),
    (e.job_title),
    ()
)
ORDER BY department NULLS LAST, job_title NULLS LAST;
```

### แบบฝึกหัดที่ 6
ใช้ GROUPING() ใน HAVING เพื่อกรองเฉพาะแถว Grand Total

```sql
-- เฉลย:
SELECT 
    COALESCE(category, 'ALL') AS category,
    COUNT(*) AS products,
    SUM(price * stock_qty) AS value
FROM products
GROUP BY GROUPING SETS ((category), ())
HAVING GROUPING(category) = 1;  -- เฉพาะ Grand Total แถวเดียว
```

### แบบฝึกหัดที่ 7
สร้าง Monthly Report ที่มีทั้งยอดรายเดือนและยอดรวมทั้งหมด

```sql
-- เฉลย:
SELECT 
    CASE WHEN GROUPING(YEAR(order_date)) = 1 THEN 'TOTAL'
         WHEN GROUPING(MONTH(order_date)) = 1 
              AND GROUPING(YEAR(order_date)) = 0 THEN CONCAT(YEAR(order_date), ' Total')
         ELSE DATE_FORMAT(order_date, '%Y-%m') END AS period,
    COUNT(*) AS orders,
    SUM(total_amount) AS revenue
FROM orders
WHERE status != 'cancelled'
GROUP BY GROUPING SETS (
    (YEAR(order_date), MONTH(order_date)),
    (YEAR(order_date)),
    ()
)
ORDER BY YEAR(order_date) NULLS LAST, MONTH(order_date) NULLS LAST;
```

### แบบฝึกหัดที่ 8
สร้าง Multi-dimensional Sales Analysis ด้วย GROUPING SETS

```sql
-- เฉลย:
SELECT 
    COALESCE(p.category, 'ALL') AS category,
    COALESCE(c.province, 'ALL') AS province,
    COUNT(DISTINCT o.order_id) AS orders,
    SUM(o.total_amount) AS revenue
FROM orders o
JOIN customers c ON o.customer_id = c.customer_id
JOIN order_items oi ON o.order_id = oi.order_id
JOIN products p ON oi.product_id = p.product_id
WHERE o.status NOT IN ('cancelled')
GROUP BY GROUPING SETS (
    (p.category, c.province),
    (p.category),
    (c.province),
    ()
)
ORDER BY p.category NULLS LAST, c.province NULLS LAST;
```

### แบบฝึกหัดที่ 9
เปรียบเทียบ GROUPING SETS กับ UNION ALL ในแง่ประสิทธิภาพ

```sql
-- เฉลย (UNION ALL - ช้ากว่าเพราะผ่านข้อมูล 4 รอบ):
SELECT 'detail', category, brand, COUNT(*), SUM(price) FROM products GROUP BY category, brand
UNION ALL SELECT 'cat_total', category, NULL, COUNT(*), SUM(price) FROM products GROUP BY category
UNION ALL SELECT 'brand_total', NULL, brand, COUNT(*), SUM(price) FROM products GROUP BY brand
UNION ALL SELECT 'grand_total', NULL, NULL, COUNT(*), SUM(price) FROM products;

-- เฉลย (GROUPING SETS - เร็วกว่าเพราะผ่านข้อมูลครั้งเดียว):
SELECT category, brand, COUNT(*), SUM(price)
FROM products
GROUP BY GROUPING SETS ((category, brand), (category), (brand), ());
```

### แบบฝึกหัดที่ 10
สร้าง Complete Executive Report ด้วย GROUPING SETS

```sql
-- เฉลย:
SELECT 
    CASE 
        WHEN GROUPING(YEAR(o.order_date))=0 AND GROUPING(p.category)=0 THEN 'Detail'
        WHEN GROUPING(YEAR(o.order_date))=0 THEN 'Year Summary'
        WHEN GROUPING(p.category)=0 THEN 'Category Summary'
        ELSE 'Grand Total'
    END AS report_level,
    COALESCE(CAST(YEAR(o.order_date) AS CHAR), 'ALL YEARS') AS year,
    COALESCE(p.category, 'ALL CATEGORIES') AS category,
    COUNT(DISTINCT o.order_id) AS orders,
    COUNT(DISTINCT o.customer_id) AS customers,
    SUM(oi.quantity * oi.unit_price) AS revenue,
    ROUND(AVG(o.total_amount), 0) AS avg_order
FROM orders o
JOIN order_items oi ON o.order_id = oi.order_id
JOIN products p ON oi.product_id = p.product_id
WHERE o.status NOT IN ('cancelled')
GROUP BY GROUPING SETS (
    (YEAR(o.order_date), p.category),
    (YEAR(o.order_date)),
    (p.category),
    ()
)
ORDER BY 
    GROUPING(YEAR(o.order_date)) + GROUPING(p.category),
    YEAR(o.order_date) NULLS LAST,
    p.category NULLS LAST;
```

---

*จบบทที่ 037 - GROUPING SETS: Custom Grouping Combinations*

*บทถัดไป: Part 038 - Window Functions Preview: Running Totals*
