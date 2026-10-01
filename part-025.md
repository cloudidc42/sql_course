# ภาค 25: CROSS JOIN — Cartesian Product

## CROSS JOIN คืออะไร?

**CROSS JOIN** คือการรวมตารางที่ทุกแถวของตาราง A จับคู่กับทุกแถวของตาราง B โดยไม่มีเงื่อนไข (ไม่มี ON clause)

ผลลัพธ์ = จำนวนแถวของ A × จำนวนแถวของ B = **Cartesian Product**

```
ตัวอย่าง:
Table A (3 rows):     Table B (4 rows):
┌────┬──────┐         ┌────┬─────────┐
│ id │ name │         │ id │ color   │
├────┼──────┤         ├────┼─────────┤
│ 1  │ Cat  │         │ 1  │ Red     │
│ 2  │ Dog  │         │ 2  │ Blue    │
│ 3  │ Bird │         │ 3  │ Green   │
└────┴──────┘         │ 4  │ Yellow  │
                      └────┴─────────┘

CROSS JOIN Result: 3 × 4 = 12 rows
┌──────┬─────────┐
│ name │ color   │
├──────┼─────────┤
│ Cat  │ Red     │
│ Cat  │ Blue    │
│ Cat  │ Green   │
│ Cat  │ Yellow  │
│ Dog  │ Red     │
│ Dog  │ Blue    │
│ Dog  │ Green   │
│ Dog  │ Yellow  │
│ Bird │ Red     │
│ Bird │ Blue    │
│ Bird │ Green   │
│ Bird │ Yellow  │
└──────┴─────────┘
```

---

## Syntax

```sql
-- แบบ EXPLICIT
SELECT * FROM table_a CROSS JOIN table_b;

-- แบบ IMPLICIT (comma)
SELECT * FROM table_a, table_b;  -- ไม่มี WHERE = CROSS JOIN

-- แบบ IMPLICIT with WHERE (INNER JOIN)
SELECT * FROM table_a, table_b
WHERE table_a.id = table_b.id;  -- กลายเป็น INNER JOIN
```

---

## ตัวอย่าง CROSS JOIN (1-15)

### ตัวอย่างที่ 1: Basic CROSS JOIN

```sql
-- สร้าง combinations ของแผนกกับพนักงาน (ไม่สมเหตุสมผล แต่แสดง concept)
SELECT 
    d.dept_name,
    e.first_name || ' ' || e.last_name AS employee
FROM departments d
CROSS JOIN employees e
ORDER BY d.dept_name, employee;
-- ผลลัพธ์: 10 แผนก × 50 พนักงาน = 500 แถว!
```

### ตัวอย่างที่ 2: CROSS JOIN สำหรับ Size × Color Matrix

```sql
-- สร้าง product variations (Size × Color)
WITH sizes AS (
    SELECT unnest(ARRAY['S', 'M', 'L', 'XL', 'XXL']) AS size
),
colors AS (
    SELECT unnest(ARRAY['Red', 'Blue', 'Green', 'Black', 'White']) AS color
)
SELECT 
    s.size,
    c.color,
    CONCAT(s.size, '-', c.color) AS variant_code
FROM sizes s
CROSS JOIN colors c
ORDER BY s.size, c.color;
-- 5 sizes × 5 colors = 25 combinations
```

### ตัวอย่างที่ 3: Date Range Generation

```sql
-- สร้าง calendar ของทุกวันในเดือน (PostgreSQL)
WITH months AS (
    SELECT generate_series(1, 12) AS month
),
days AS (
    SELECT generate_series(1, 31) AS day
)
SELECT 
    m.month,
    d.day,
    CASE 
        WHEN m.month IN (4,6,9,11) AND d.day = 31 THEN 'Invalid'
        WHEN m.month = 2 AND d.day > 29 THEN 'Invalid'
        ELSE CONCAT('2024-', LPAD(m.month::TEXT, 2, '0'), '-', LPAD(d.day::TEXT, 2, '0'))
    END AS date_value
FROM months m
CROSS JOIN days d
WHERE NOT (m.month IN (4,6,9,11) AND d.day = 31)
  AND NOT (m.month = 2 AND d.day > 29)
ORDER BY m.month, d.day;
```

### ตัวอย่างที่ 4: สร้าง Test Data ด้วย CROSS JOIN

```sql
-- Generate test data: customers × products combinations
SELECT 
    c.customer_id,
    c.first_name || ' ' || c.last_name AS customer,
    p.product_id,
    p.product_name,
    p.price,
    -- สมมติ recommended_qty ตาม category
    CASE p.category
        WHEN 'Electronics' THEN 1
        WHEN 'Stationery' THEN 5
        WHEN 'Office' THEN 2
        ELSE 1
    END AS recommended_qty
FROM customers c
CROSS JOIN products p
WHERE c.city = 'Bangkok'  -- เฉพาะลูกค้า Bangkok
  AND p.category = 'Stationery'  -- เฉพาะ Stationery
ORDER BY c.customer_id, p.product_id;
```

### ตัวอย่างที่ 5: CROSS JOIN สำหรับ Pivot Table Skeleton

```sql
-- สร้าง skeleton สำหรับ pivot: categories × months
WITH categories AS (
    SELECT DISTINCT category FROM products ORDER BY category
),
months_2024 AS (
    SELECT DISTINCT TO_CHAR(order_date, 'YYYY-MM') AS month
    FROM orders
    WHERE EXTRACT(YEAR FROM order_date) = 2024
    ORDER BY month
)
SELECT 
    c.category,
    m.month,
    0 AS placeholder_revenue
FROM categories c
CROSS JOIN months_2024 m
ORDER BY c.category, m.month;
```

### ตัวอย่างที่ 6: CROSS JOIN + LEFT JOIN (Pivot with actual data)

```sql
-- Pivot: category revenue per month (ใช้ CROSS JOIN เป็น skeleton)
WITH categories AS (
    SELECT DISTINCT category FROM products
),
months AS (
    SELECT DISTINCT TO_CHAR(order_date, 'YYYY-MM') AS month
    FROM orders
    WHERE EXTRACT(YEAR FROM order_date) = 2024
    ORDER BY month
),
actual_sales AS (
    SELECT 
        p.category,
        TO_CHAR(o.order_date, 'YYYY-MM') AS month,
        SUM(oi.quantity * oi.unit_price - oi.discount) AS revenue
    FROM products p
    JOIN order_items oi ON p.product_id = oi.product_id
    JOIN orders o ON oi.order_id = o.order_id
    WHERE EXTRACT(YEAR FROM o.order_date) = 2024
    GROUP BY p.category, month
)
SELECT 
    c.category,
    m.month,
    COALESCE(asales.revenue, 0) AS revenue
FROM categories c
CROSS JOIN months m
LEFT JOIN actual_sales asales ON c.category = asales.category 
                              AND m.month = asales.month
ORDER BY c.category, m.month;
```

### ตัวอย่างที่ 7: Multiplication Table

```sql
-- ตารางสูตรคูณ
WITH nums AS (
    SELECT generate_series(1, 10) AS n
)
SELECT 
    a.n AS multiplier,
    b.n AS multiplicand,
    a.n * b.n AS product
FROM nums a
CROSS JOIN nums b
ORDER BY a.n, b.n;
```

### ตัวอย่างที่ 8: All Possible Routes

```sql
-- สร้าง combinations เส้นทางระหว่างเมือง
WITH cities AS (
    SELECT DISTINCT city FROM customers ORDER BY city
)
SELECT 
    a.city AS from_city,
    b.city AS to_city,
    'Route: ' || a.city || ' → ' || b.city AS route
FROM cities a
CROSS JOIN cities b
WHERE a.city != b.city  -- ไม่เอา city เดียวกัน
ORDER BY from_city, to_city;
```

### ตัวอย่างที่ 9: Time Slot Generation

```sql
-- สร้าง time slots สำหรับตาราง schedule
WITH hours AS (
    SELECT generate_series(9, 17) AS hour  -- 9am to 5pm
),
days_of_week AS (
    SELECT unnest(ARRAY['Monday','Tuesday','Wednesday','Thursday','Friday']) AS day
)
SELECT 
    d.day,
    h.hour || ':00' AS time_slot,
    h.hour || ':00 - ' || (h.hour + 1) || ':00' AS slot_label,
    FALSE AS is_booked  -- default
FROM days_of_week d
CROSS JOIN hours h
ORDER BY 
    CASE d.day 
        WHEN 'Monday' THEN 1 WHEN 'Tuesday' THEN 2 WHEN 'Wednesday' THEN 3
        WHEN 'Thursday' THEN 4 WHEN 'Friday' THEN 5 
    END,
    h.hour;
```

### ตัวอย่างที่ 10: Pricing Matrix

```sql
-- Matrix ราคาตาม quantity breaks
WITH quantity_breaks AS (
    SELECT unnest(ARRAY[1, 5, 10, 25, 50, 100]) AS qty_break
),
discount_levels AS (
    SELECT unnest(ARRAY[0, 0.05, 0.10, 0.15, 0.20]) AS discount_pct,
           unnest(ARRAY['No Discount', '5%', '10%', '15%', '20%']) AS discount_label
)
SELECT 
    qb.qty_break AS quantity,
    dl.discount_label,
    dl.discount_pct,
    p.product_name,
    p.price AS list_price,
    ROUND(p.price * (1 - dl.discount_pct), 2) AS discounted_price,
    ROUND(p.price * (1 - dl.discount_pct) * qb.qty_break, 2) AS total_price
FROM quantity_breaks qb
CROSS JOIN discount_levels dl
CROSS JOIN (SELECT * FROM products WHERE product_id = 1) p  -- Laptop Pro 15
ORDER BY qb.qty_break, dl.discount_pct;
```

### ตัวอย่างที่ 11: Generating Number Series

```sql
-- MySQL workaround สำหรับ generate_series
-- สร้าง number series 1-100 ด้วย CROSS JOIN
CREATE TEMP TABLE tens AS 
    SELECT 0 n UNION SELECT 1 UNION SELECT 2 UNION SELECT 3 UNION SELECT 4
    UNION SELECT 5 UNION SELECT 6 UNION SELECT 7 UNION SELECT 8 UNION SELECT 9;

SELECT t1.n * 10 + t2.n + 1 AS num
FROM tens t1
CROSS JOIN tens t2
ORDER BY num;
-- ได้เลข 1-100
```

### ตัวอย่างที่ 12: Combination Product Bundles

```sql
-- สร้าง product bundles (Electronic + Accessory combinations)
SELECT 
    e.product_name AS electronics,
    e.price AS electronics_price,
    a.product_name AS accessory,
    a.price AS accessory_price,
    e.price + a.price AS bundle_price,
    ROUND((e.price + a.price) * 0.9, 2) AS discounted_bundle  -- 10% bundle discount
FROM products e
CROSS JOIN products a
WHERE e.category = 'Electronics'
  AND a.category = 'Accessories'
  AND e.price > 10000  -- เฉพาะสินค้าหลักราคาสูง
  AND a.price < 5000   -- เฉพาะ accessories ราคาไม่แพง
ORDER BY bundle_price DESC
LIMIT 20;
```

### ตัวอย่างที่ 13: CROSS JOIN กับ VALUES

```sql
-- ใช้ VALUES เป็น inline table แล้ว CROSS JOIN
SELECT 
    e.emp_id,
    e.first_name,
    d.dept_name,
    perf.rating
FROM employees e
JOIN departments d ON e.dept_id = d.dept_id
CROSS JOIN (VALUES 
    ('Excellent'), 
    ('Good'), 
    ('Average'), 
    ('Needs Improvement')
) AS perf(rating)
WHERE e.dept_id = 1  -- Engineering only
ORDER BY e.emp_id, rating;
-- แต่ละพนักงานจะมี 4 แถวสำหรับแต่ละ rating level
```

### ตัวอย่างที่ 14: Report Template Generation

```sql
-- สร้าง empty report template: departments × quarters
WITH quarters AS (
    SELECT 1 AS q, 'Q1' AS label UNION ALL
    SELECT 2, 'Q2' UNION ALL
    SELECT 3, 'Q3' UNION ALL
    SELECT 4, 'Q4'
)
SELECT 
    d.dept_name,
    q.label AS quarter,
    0.00 AS budget_allocated,
    0.00 AS actual_spent,
    0 AS headcount_start,
    0 AS headcount_end
FROM departments d
CROSS JOIN quarters q
ORDER BY d.dept_name, q.q;
```

### ตัวอย่างที่ 15: Performance Test — CROSS JOIN ระวัง!

```sql
-- ระวัง! CROSS JOIN ของตารางใหญ่ = ช้ามาก
-- employees × employees = 50 × 50 = 2,500 แถว (OK)
-- customers × products = 60 × 50 = 3,000 แถว (OK)
-- customers × orders = 60 × 70 = 4,200 แถว (OK แต่ไม่มีประโยชน์)
-- employees × customers × products = 50 × 60 × 50 = 150,000 แถว (อันตราย!)

-- ตัวอย่างที่ดี: CROSS JOIN พร้อม WHERE filter
SELECT 
    c.customer_id,
    c.first_name,
    p.product_id,
    p.product_name
FROM customers c
CROSS JOIN products p
WHERE p.price BETWEEN 1000 AND 5000  -- filter ลด rows ก่อน
  AND c.city = 'Bangkok'
ORDER BY c.customer_id, p.price;

-- ตัวอย่างที่ไม่ดี (อย่าทำ!):
-- SELECT * FROM employees CROSS JOIN customers CROSS JOIN products;
-- = 50 × 60 × 50 = 150,000 rows!
```

---

## CROSS JOIN สำหรับ Use Cases จริง

### ตัวอย่างพิเศษ: Filling Missing Data with CROSS JOIN

```sql
-- ปัญหา: report แสดง sales ต่อ category ต่อเดือน
-- แต่บาง category ไม่มียอดในบางเดือน → ช่องว่าง!
-- แก้: CROSS JOIN categories × months แล้ว LEFT JOIN ยอดขายจริง

WITH all_categories AS (
    SELECT DISTINCT category FROM products
),
all_months AS (
    SELECT 
        generate_series(
            DATE '2024-01-01',
            DATE '2024-09-01',
            INTERVAL '1 month'
        )::DATE AS month_start
),
actual_sales AS (
    SELECT 
        p.category,
        DATE_TRUNC('month', o.order_date)::DATE AS month_start,
        SUM(oi.quantity * oi.unit_price - oi.discount) AS revenue
    FROM products p
    JOIN order_items oi ON p.product_id = oi.product_id
    JOIN orders o ON oi.order_id = o.order_id
    WHERE o.status IN ('completed', 'shipped')
    GROUP BY p.category, month_start
)
SELECT 
    ac.category,
    am.month_start,
    TO_CHAR(am.month_start, 'Mon YYYY') AS month_label,
    COALESCE(asales.revenue, 0) AS revenue,
    CASE WHEN asales.revenue IS NULL THEN 'No Sales' ELSE 'Has Sales' END AS status
FROM all_categories ac
CROSS JOIN all_months am
LEFT JOIN actual_sales asales 
    ON ac.category = asales.category 
    AND am.month_start = asales.month_start
ORDER BY ac.category, am.month_start;
```

---

## แบบฝึกหัดภาค 25

**ข้อ 1:** สร้าง combinations ของ customers ใน Bangkok กับ products ราคาต่ำกว่า 5000

```sql
-- เฉลย
SELECT 
    c.customer_id,
    c.first_name || ' ' || c.last_name AS customer,
    p.product_id,
    p.product_name,
    p.price
FROM customers c
CROSS JOIN products p
WHERE c.city = 'Bangkok'
  AND p.price < 5000
ORDER BY c.customer_id, p.price;
```

**ข้อ 2:** สร้าง schedule slots สำหรับทุกพนักงานกับทุกวันในสัปดาห์

```sql
-- เฉลย
WITH weekdays AS (
    SELECT unnest(ARRAY['Monday','Tuesday','Wednesday','Thursday','Friday']) AS day,
           generate_series(1, 5) AS day_num
)
SELECT 
    e.emp_id,
    e.first_name || ' ' || e.last_name AS employee,
    d.dept_name,
    w.day AS work_day,
    'Available' AS status
FROM employees e
JOIN departments d ON e.dept_id = d.dept_id
CROSS JOIN weekdays w
WHERE e.dept_id IN (1, 2)  -- Engineering, Marketing
ORDER BY e.emp_id, w.day_num;
```

**ข้อ 3:** สร้าง product bundle recommendations (Electronics + Software)

```sql
-- เฉลย
SELECT 
    e.product_name AS main_product,
    e.price AS main_price,
    s.product_name AS software_bundle,
    s.price AS software_price,
    e.price + s.price AS combined_price,
    ROUND((e.price + s.price) * 0.95, 2) AS bundle_deal  -- 5% bundle discount
FROM products e
CROSS JOIN products s
WHERE e.category = 'Electronics'
  AND s.category = 'Software'
  AND e.price > 15000
ORDER BY combined_price DESC;
```

**ข้อ 4:** สร้าง date series สำหรับปี 2024 แล้วนับ orders ต่อวัน

```sql
-- เฉลย (PostgreSQL)
WITH dates_2024 AS (
    SELECT generate_series(
        '2024-01-01'::DATE,
        '2024-12-31'::DATE,
        '1 day'::INTERVAL
    )::DATE AS date
)
SELECT 
    d.date,
    TO_CHAR(d.date, 'Day') AS day_of_week,
    COUNT(o.order_id) AS order_count,
    COALESCE(SUM(o.total_amount), 0) AS daily_revenue
FROM dates_2024 d
LEFT JOIN orders o ON d.date = o.order_date
GROUP BY d.date
ORDER BY d.date;
```

**ข้อ 5:** อธิบาย Cartesian Product และผลกระทบ performance

```sql
-- เฉลย: คำอธิบาย + ตัวอย่าง
-- Cartesian Product = แถวทุกแถวของตาราง A จับคู่กับทุกแถวของตาราง B
-- ขนาด = rows_A × rows_B

-- ตัวอย่าง performance impact:
SELECT 
    table_name,
    row_count,
    row_count * row_count AS self_cross_join_rows
FROM (
    VALUES 
    ('employees', 50),
    ('customers', 60),
    ('products', 50),
    ('orders', 70),
    ('order_items', 106)
) AS t(table_name, row_count)
ORDER BY self_cross_join_rows DESC;

-- สรุป:
-- employees CROSS JOIN customers = 50 × 60 = 3,000 rows (OK)
-- orders CROSS JOIN order_items = 70 × 106 = 7,420 rows (เริ่มใหญ่)
-- ทุกตาราง CROSS JOIN กัน = มากกว่า 1 billion rows! (อันตราย)
```

**ข้อ 6:** ใช้ CROSS JOIN สร้าง monthly report template สำหรับทุก category

```sql
-- เฉลย
WITH categories AS (
    SELECT DISTINCT category FROM products ORDER BY category
),
months AS (
    SELECT 
        generate_series(1, 12) AS month_num,
        TO_CHAR(DATE '2024-01-01' + (generate_series(1, 12) - 1) * INTERVAL '1 month', 'Mon') AS month_name
)
SELECT 
    c.category,
    m.month_num,
    m.month_name,
    0::DECIMAL AS planned_revenue,
    0::DECIMAL AS actual_revenue,
    0::INT AS units_sold
FROM categories c
CROSS JOIN months m
ORDER BY c.category, m.month_num;
```

**ข้อ 7:** สร้าง all employee pairs สำหรับ mentorship matching

```sql
-- เฉลย: จับคู่ Senior กับ Junior
SELECT 
    s.emp_id AS senior_id,
    s.first_name || ' ' || s.last_name AS senior_name,
    s.job_title AS senior_title,
    j.emp_id AS junior_id,
    j.first_name || ' ' || j.last_name AS junior_name,
    j.job_title AS junior_title,
    s.dept_id = j.dept_id AS same_department
FROM employees s  -- Senior
CROSS JOIN employees j  -- Junior
WHERE (s.job_title LIKE '%Senior%' OR s.job_title LIKE '%Director%' OR s.job_title LIKE '%VP%')
  AND (j.job_title LIKE '%Junior%' OR j.job_title LIKE '%Intern%' OR j.job_title LIKE '%Analyst%')
  AND s.emp_id != j.emp_id
ORDER BY s.emp_id, j.emp_id;
```

**ข้อ 8:** สร้าง pricing tiers สำหรับสินค้าทุกชิ้น

```sql
-- เฉลย
WITH price_tiers AS (
    SELECT 
        tier_name,
        discount_pct
    FROM (VALUES 
        ('Retail', 0.00),
        ('Partner', 0.10),
        ('Distributor', 0.20),
        ('Wholesale', 0.30),
        ('VIP', 0.40)
    ) AS t(tier_name, discount_pct)
)
SELECT 
    p.product_name,
    p.category,
    p.price AS list_price,
    pt.tier_name,
    pt.discount_pct,
    ROUND(p.price * (1 - pt.discount_pct), 2) AS tier_price
FROM products p
CROSS JOIN price_tiers pt
ORDER BY p.category, p.product_name, pt.discount_pct;
```

**ข้อ 9:** เมื่อใดที่ CROSS JOIN ไม่เหมาะสม?

```sql
-- เฉลย: อธิบายกรณีที่ไม่ควรใช้ CROSS JOIN
-- 1. เมื่อไม่ต้องการ ALL combinations (ใช้ INNER JOIN แทน)
-- 2. เมื่อตารางมีขนาดใหญ่ (rows > 1000 แต่ละตาราง)
-- 3. เมื่อ result set ใหญ่เกินความจำเป็น

-- ตัวอย่างที่ไม่ดี: เจตนาจะหา matching records แต่ลืม WHERE
-- BAD:
SELECT e.first_name, d.dept_name
FROM employees e
CROSS JOIN departments d;
-- ได้ 50 × 10 = 500 rows (ผิด!)

-- GOOD: ควรใช้ JOIN
SELECT e.first_name, d.dept_name
FROM employees e
JOIN departments d ON e.dept_id = d.dept_id;
-- ได้ 50 rows (ถูกต้อง)

-- CROSS JOIN เหมาะกับ:
SELECT 'Use CROSS JOIN when:' AS cases
UNION ALL SELECT 'Generating all combinations intentionally'
UNION ALL SELECT 'Creating test data'
UNION ALL SELECT 'Building pivot skeletons'
UNION ALL SELECT 'Calendar/time series generation';
```

**ข้อ 10:** สร้าง Recommendation Matrix: ลูกค้าในแต่ละเมือง × สินค้ายอดนิยมในเมืองนั้น

```sql
-- เฉลย
WITH city_top_products AS (
    SELECT 
        c.city,
        p.product_id,
        p.product_name,
        p.category,
        COUNT(o.order_id) AS city_order_count,
        RANK() OVER (PARTITION BY c.city ORDER BY COUNT(o.order_id) DESC) AS rank_in_city
    FROM customers c
    JOIN orders o ON c.customer_id = o.customer_id
    JOIN order_items oi ON o.order_id = oi.order_id
    JOIN products p ON oi.product_id = p.product_id
    WHERE o.status = 'completed'
    GROUP BY c.city, p.product_id, p.product_name, p.category
)
SELECT 
    cust.customer_id,
    cust.first_name || ' ' || cust.last_name AS customer,
    cust.city,
    ctp.product_name AS recommended_product,
    ctp.category,
    ctp.city_order_count AS popularity_score
FROM customers cust
JOIN city_top_products ctp ON cust.city = ctp.city
WHERE ctp.rank_in_city <= 3  -- Top 3 products per city
ORDER BY cust.city, cust.customer_id, ctp.rank_in_city;
```

---

## สรุปภาค 25

1. **CROSS JOIN** — ทุกแถวของ A จับคู่กับทุกแถวของ B → rows_A × rows_B
2. **ใช้เมื่อ** — สร้าง combinations, test data, calendar, pivot skeleton
3. **ระวัง performance** — ขนาดใหญ่ขึ้น exponential
4. **ใช้ร่วมกับ WHERE** เพื่อ filter combinations ที่ต้องการ
5. **ใช้ร่วมกับ LEFT JOIN** เพื่อ fill missing data

**ในภาคถัดไป** จะเรียน Self JOIN ซึ่งเป็นการ JOIN ตารางกับตัวเอง!
