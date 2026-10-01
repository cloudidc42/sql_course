# ภาค 28: Advanced JOIN Techniques

## เทคนิค JOIN ขั้นสูง

ในบทนี้เราจะเรียนเทคนิค JOIN ที่ซับซ้อนกว่าปกติ:

1. **Non-Equi JOIN** — JOIN ด้วยเงื่อนไขที่ไม่ใช่ `=`
2. **JOIN กับ Subqueries** — ใช้ subquery เป็นตาราง
3. **JOIN กับ CTEs** — Common Table Expressions
4. **LATERAL JOIN** — JOIN แบบ correlated (PostgreSQL)
5. **JOIN กับ Window Functions**
6. **Range JOIN** — JOIN ด้วยช่วงค่า

---

## Non-Equi JOIN (Range JOIN)

**Non-equi JOIN** คือ JOIN ที่ใช้เงื่อนไขอื่นนอกจาก `=` เช่น `<`, `>`, `<=`, `>=`, `BETWEEN`, `LIKE`

### ตัวอย่างที่ 1: Salary Grade Classification

```sql
-- สร้าง salary grade table
CREATE TEMP TABLE salary_grades (
    grade VARCHAR(10),
    min_salary DECIMAL(10,2),
    max_salary DECIMAL(10,2),
    description VARCHAR(50)
);

INSERT INTO salary_grades VALUES
('Grade 1', 0, 80000, 'Entry Level'),
('Grade 2', 80001, 100000, 'Junior'),
('Grade 3', 100001, 130000, 'Mid-Level'),
('Grade 4', 130001, 160000, 'Senior'),
('Grade 5', 160001, 200000, 'Management'),
('Grade 6', 200001, 999999, 'Executive');

-- Non-Equi JOIN: BETWEEN สำหรับ range matching
SELECT 
    e.emp_id,
    CONCAT(e.first_name, ' ', e.last_name) AS employee,
    e.salary,
    sg.grade,
    sg.description
FROM employees e
JOIN salary_grades sg ON e.salary BETWEEN sg.min_salary AND sg.max_salary
ORDER BY e.salary DESC;
```

### ตัวอย่างที่ 2: Product Discount Tiers ด้วย Non-Equi

```sql
-- กำหนด discount ตามราคาสินค้า
CREATE TEMP TABLE discount_rules (
    price_min DECIMAL(10,2),
    price_max DECIMAL(10,2),
    discount_pct DECIMAL(5,2),
    tier_name VARCHAR(30)
);

INSERT INTO discount_rules VALUES
(0, 1000, 0, 'No Discount'),
(1001, 5000, 5, '5% Discount'),
(5001, 15000, 8, '8% Discount'),
(15001, 40000, 10, '10% Discount'),
(40001, 999999, 15, '15% Discount');

-- JOIN ด้วย BETWEEN (Non-equi)
SELECT 
    p.product_name,
    p.category,
    p.price,
    dr.tier_name,
    dr.discount_pct,
    ROUND(p.price * (1 - dr.discount_pct / 100), 2) AS final_price
FROM products p
JOIN discount_rules dr ON p.price BETWEEN dr.price_min AND dr.price_max
ORDER BY p.price DESC;
```

### ตัวอย่างที่ 3: Non-Equi JOIN สำหรับ Date Ranges

```sql
-- สมมติมีตาราง promotions ที่มีช่วงวันที่
CREATE TEMP TABLE promotions (
    promo_id INT PRIMARY KEY,
    promo_name VARCHAR(100),
    discount_pct DECIMAL(5,2),
    start_date DATE,
    end_date DATE,
    category VARCHAR(50)
);

INSERT INTO promotions VALUES
(1, 'New Year Sale', 20, '2024-01-01', '2024-01-07', 'Electronics'),
(2, 'Valentine Special', 15, '2024-02-10', '2024-02-14', NULL),  -- all categories
(3, 'Mid Year Sale', 10, '2024-06-01', '2024-06-30', NULL),
(4, 'Electronics Week', 12, '2024-03-01', '2024-03-07', 'Electronics');

-- หา orders ที่ได้รับ promotion ในวันที่สั่งซื้อ
SELECT 
    o.order_id,
    o.order_date,
    o.total_amount,
    p.promo_name,
    p.discount_pct,
    ROUND(o.total_amount * (1 - p.discount_pct / 100), 2) AS after_promo_amount
FROM orders o
JOIN promotions p 
    ON o.order_date BETWEEN p.start_date AND p.end_date  -- Non-equi!
    AND (p.category IS NULL OR p.category IN (
        SELECT DISTINCT prod.category 
        FROM order_items oi 
        JOIN products prod ON oi.product_id = prod.product_id
        WHERE oi.order_id = o.order_id
    ))
ORDER BY o.order_date, o.order_id;
```

### ตัวอย่างที่ 4: Finding Overlapping Time Ranges

```sql
-- หา employees ที่ทำงานอยู่ในช่วงเวลาเดียวกัน
-- (สมมติมี start_date และ end_date)
CREATE TEMP TABLE emp_assignments (
    emp_id INT,
    project_id INT,
    start_date DATE,
    end_date DATE
);

INSERT INTO emp_assignments VALUES
(5, 1, '2024-01-01', '2024-06-30'),
(5, 2, '2024-04-01', '2024-12-31'),  -- overlap กับ project 1
(6, 1, '2024-01-01', '2024-03-31'),
(6, 3, '2024-05-01', '2024-09-30'),
(7, 2, '2024-06-01', '2024-12-31');

-- หา employees ที่มี overlapping assignments
SELECT 
    a1.emp_id,
    a1.project_id AS project_1,
    a1.start_date AS start_1,
    a1.end_date AS end_1,
    a2.project_id AS project_2,
    a2.start_date AS start_2,
    a2.end_date AS end_2,
    GREATEST(a1.start_date, a2.start_date) AS overlap_start,
    LEAST(a1.end_date, a2.end_date) AS overlap_end
FROM emp_assignments a1
JOIN emp_assignments a2 
    ON a1.emp_id = a2.emp_id            -- Same employee
    AND a1.project_id < a2.project_id   -- Different projects, avoid duplicates
    AND a1.start_date <= a2.end_date     -- Non-equi: overlap condition
    AND a1.end_date >= a2.start_date     -- Non-equi: overlap condition
ORDER BY a1.emp_id, a1.project_id;
```

### ตัวอย่างที่ 5: Triangular JOIN (Self Non-equi)

```sql
-- เปรียบเทียบทุก employee pairs ในแผนกเดียวกัน
SELECT 
    e1.emp_id AS emp1_id,
    CONCAT(e1.first_name, ' ', e1.last_name) AS employee_1,
    e1.salary AS salary_1,
    e2.emp_id AS emp2_id,
    CONCAT(e2.first_name, ' ', e2.last_name) AS employee_2,
    e2.salary AS salary_2,
    e1.salary - e2.salary AS salary_gap,
    d.dept_name
FROM employees e1
JOIN employees e2 
    ON e1.dept_id = e2.dept_id          -- Same dept
    AND e1.emp_id < e2.emp_id           -- Avoid duplicates (Non-equi)
    AND e1.salary != e2.salary          -- Different salary (Non-equi)
JOIN departments d ON e1.dept_id = d.dept_id
ORDER BY ABS(e1.salary - e2.salary) DESC
LIMIT 20;
```

---

## JOIN กับ Subqueries

### ตัวอย่างที่ 6: Inline View (Subquery as Table)

```sql
-- Subquery ใน FROM clause (Inline View / Derived Table)
SELECT 
    e.emp_id,
    CONCAT(e.first_name, ' ', e.last_name) AS employee,
    e.salary,
    dept_stats.avg_salary AS dept_avg,
    e.salary - dept_stats.avg_salary AS vs_dept_avg
FROM employees e
JOIN (
    -- Subquery: คำนวณค่าเฉลี่ยต่อแผนก
    SELECT dept_id, 
           AVG(salary) AS avg_salary,
           MAX(salary) AS max_salary,
           COUNT(*) AS dept_size
    FROM employees
    GROUP BY dept_id
) dept_stats ON e.dept_id = dept_stats.dept_id
WHERE e.salary > dept_stats.avg_salary
ORDER BY (e.salary - dept_stats.avg_salary) DESC;
```

### ตัวอย่างที่ 7: Correlated Subquery vs JOIN

```sql
-- Correlated Subquery (ช้ากว่า)
SELECT 
    c.customer_id,
    c.first_name,
    (SELECT MAX(o.order_date) 
     FROM orders o 
     WHERE o.customer_id = c.customer_id) AS last_order_date
FROM customers c;

-- เทียบเท่าด้วย JOIN (เร็วกว่า)
SELECT 
    c.customer_id,
    c.first_name,
    MAX(o.order_date) AS last_order_date
FROM customers c
LEFT JOIN orders o ON c.customer_id = o.customer_id
GROUP BY c.customer_id, c.first_name;

-- หรือใช้ Window Function (เร็วกว่า + ไม่ต้อง GROUP BY)
SELECT DISTINCT
    c.customer_id,
    c.first_name,
    MAX(o.order_date) OVER (PARTITION BY c.customer_id) AS last_order_date
FROM customers c
LEFT JOIN orders o ON c.customer_id = o.customer_id;
```

### ตัวอย่างที่ 8: JOIN กับ Aggregated Subquery

```sql
-- หาพนักงานที่เงินเดือนสูงกว่าค่าเฉลี่ยของทุกแผนก
SELECT 
    e.emp_id,
    CONCAT(e.first_name, ' ', e.last_name) AS employee,
    e.salary,
    company_avg.avg_salary AS company_avg,
    e.salary - company_avg.avg_salary AS above_avg
FROM employees e
CROSS JOIN (
    SELECT AVG(salary) AS avg_salary FROM employees
) company_avg
WHERE e.salary > company_avg.avg_salary
ORDER BY above_avg DESC;
```

### ตัวอย่างที่ 9: Multi-Level Subquery JOIN

```sql
-- หา top customers ต่อ city พร้อม details
SELECT 
    city_top.city,
    city_top.customer_id,
    city_top.customer_name,
    city_top.total_spent,
    city_top.rank_in_city,
    -- Details of their favorite product
    fav.product_name AS favorite_product,
    fav.category AS favorite_category
FROM (
    -- Level 1: Top customers per city
    SELECT 
        c.city,
        c.customer_id,
        CONCAT(c.first_name, ' ', c.last_name) AS customer_name,
        SUM(o.total_amount) AS total_spent,
        RANK() OVER (PARTITION BY c.city ORDER BY SUM(o.total_amount) DESC) AS rank_in_city
    FROM customers c
    JOIN orders o ON c.customer_id = o.customer_id
    WHERE o.status = 'completed'
    GROUP BY c.city, c.customer_id, customer_name
) city_top
LEFT JOIN (
    -- Level 2: Favorite product per customer
    SELECT 
        o.customer_id,
        p.product_name,
        p.category,
        RANK() OVER (PARTITION BY o.customer_id ORDER BY SUM(oi.quantity) DESC) AS prod_rank
    FROM orders o
    JOIN order_items oi ON o.order_id = oi.order_id
    JOIN products p ON oi.product_id = p.product_id
    WHERE o.status = 'completed'
    GROUP BY o.customer_id, p.product_id, p.product_name, p.category
) fav ON city_top.customer_id = fav.customer_id AND fav.prod_rank = 1
WHERE city_top.rank_in_city = 1  -- เฉพาะ top customer ต่อ city
ORDER BY city_top.city;
```

---

## JOIN กับ CTEs (Common Table Expressions)

### ตัวอย่างที่ 10: CTE แทน Subquery

```sql
-- ใช้ CTE แทน nested subquery — อ่านง่ายกว่า
WITH dept_stats AS (
    SELECT 
        dept_id,
        AVG(salary) AS avg_salary,
        MAX(salary) AS max_salary,
        MIN(salary) AS min_salary,
        COUNT(*) AS headcount
    FROM employees
    GROUP BY dept_id
),
top_earners AS (
    SELECT emp_id, dept_id, salary
    FROM employees
    WHERE salary > 150000
)
SELECT 
    e.emp_id,
    CONCAT(e.first_name, ' ', e.last_name) AS employee,
    d.dept_name,
    e.salary,
    ds.avg_salary,
    ds.max_salary,
    e.salary - ds.avg_salary AS above_dept_avg,
    CASE WHEN te.emp_id IS NOT NULL THEN 'Top Earner' ELSE 'Regular' END AS earner_status
FROM employees e
JOIN departments d ON e.dept_id = d.dept_id
JOIN dept_stats ds ON e.dept_id = ds.dept_id
LEFT JOIN top_earners te ON e.emp_id = te.emp_id
ORDER BY e.salary DESC;
```

### ตัวอย่างที่ 11: Multiple CTEs

```sql
-- Multiple CTEs สำหรับ complex analysis
WITH monthly_revenue AS (
    SELECT 
        TO_CHAR(o.order_date, 'YYYY-MM') AS month,
        SUM(o.total_amount) AS revenue,
        COUNT(DISTINCT o.customer_id) AS unique_customers,
        COUNT(o.order_id) AS order_count
    FROM orders o
    WHERE o.status = 'completed'
    GROUP BY month
),
monthly_new_customers AS (
    SELECT 
        TO_CHAR(MIN(o.order_date), 'YYYY-MM') AS first_order_month,
        COUNT(DISTINCT o.customer_id) AS new_customers
    FROM orders o
    WHERE o.status = 'completed'
    GROUP BY o.customer_id
    -- Wait, this groups by customer to get their first month
),
-- Fixed version: new customers per month
new_cust_per_month AS (
    SELECT 
        first_month,
        COUNT(*) AS new_customers
    FROM (
        SELECT 
            o.customer_id,
            TO_CHAR(MIN(o.order_date), 'YYYY-MM') AS first_month
        FROM orders o
        WHERE o.status = 'completed'
        GROUP BY o.customer_id
    ) first_orders
    GROUP BY first_month
)
SELECT 
    mr.month,
    mr.revenue,
    mr.unique_customers,
    mr.order_count,
    COALESCE(nc.new_customers, 0) AS new_customers_this_month,
    ROUND(mr.revenue / mr.order_count, 2) AS avg_order_value
FROM monthly_revenue mr
LEFT JOIN new_cust_per_month nc ON mr.month = nc.first_month
ORDER BY mr.month;
```

### ตัวอย่างที่ 12: Recursive CTE สำหรับ Category Tree

```sql
-- Recursive CTE traverse category hierarchy
WITH RECURSIVE category_path AS (
    -- Anchor: root categories
    SELECT 
        cat_id,
        cat_name,
        parent_cat_id,
        cat_name::VARCHAR(500) AS full_path,
        0 AS depth
    FROM categories_hierarchy
    WHERE parent_cat_id IS NULL
    
    UNION ALL
    
    -- Recursive
    SELECT 
        c.cat_id,
        c.cat_name,
        c.parent_cat_id,
        CAST(cp.full_path || ' > ' || c.cat_name AS VARCHAR(500)),
        cp.depth + 1
    FROM categories_hierarchy c
    JOIN category_path cp ON c.parent_cat_id = cp.cat_id
)
SELECT 
    depth,
    REPEAT('  ', depth) || cat_name AS indented_name,
    full_path,
    cat_id
FROM category_path
ORDER BY full_path;
```

---

## LATERAL JOIN (PostgreSQL)

**LATERAL JOIN** อนุญาตให้ subquery อ้างอิง columns จาก tables ทางซ้าย (correlated subquery ใน FROM)

```
ปกติ: subquery ใน FROM ไม่รู้จัก rows ของ outer query
LATERAL: subquery สามารถ reference rows ของ outer query ได้

ใช้เมื่อ:
- ต้องการ TOP-N per group
- ต้องการ correlated aggregation
- หา first/last item per group
```

### ตัวอย่างที่ 13: Top N per Group ด้วย LATERAL

```sql
-- PostgreSQL: Top 2 expensive products ต่อ category ด้วย LATERAL
SELECT 
    cat.category,
    top_products.product_name,
    top_products.price,
    top_products.rank_in_cat
FROM (SELECT DISTINCT category FROM products) cat,
LATERAL (
    SELECT 
        p.product_name,
        p.price,
        RANK() OVER (ORDER BY p.price DESC) AS rank_in_cat
    FROM products p
    WHERE p.category = cat.category  -- references outer 'cat'
    ORDER BY p.price DESC
    LIMIT 2
) top_products
ORDER BY cat.category, top_products.rank_in_cat;
```

### ตัวอย่างที่ 14: LATERAL สำหรับ Latest Order per Customer

```sql
-- PostgreSQL: หา latest order ของแต่ละลูกค้า
SELECT 
    c.customer_id,
    c.first_name || ' ' || c.last_name AS customer,
    latest_order.order_id,
    latest_order.order_date,
    latest_order.total_amount,
    latest_order.status
FROM customers c,
LATERAL (
    SELECT o.order_id, o.order_date, o.total_amount, o.status
    FROM orders o
    WHERE o.customer_id = c.customer_id  -- Correlated reference!
    ORDER BY o.order_date DESC
    LIMIT 1
) latest_order
ORDER BY latest_order.order_date DESC;
```

### ตัวอย่างที่ 15: LEFT JOIN LATERAL

```sql
-- LEFT JOIN LATERAL: เก็บลูกค้าที่ไม่มี orders ด้วย
SELECT 
    c.customer_id,
    c.first_name || ' ' || c.last_name AS customer,
    COALESCE(latest.order_id::TEXT, 'No Orders') AS latest_order,
    latest.order_date,
    latest.total_amount
FROM customers c
LEFT JOIN LATERAL (
    SELECT o.order_id, o.order_date, o.total_amount
    FROM orders o
    WHERE o.customer_id = c.customer_id
    ORDER BY o.order_date DESC
    LIMIT 1
) latest ON TRUE  -- เสมอ true, LATERAL ทำ filtering เอง
ORDER BY c.customer_id;
```

### ตัวอย่างที่ 16: LATERAL สำหรับ Running Totals

```sql
-- Running total ของ orders ต่อลูกค้า
SELECT 
    c.customer_id,
    c.first_name,
    all_orders.order_id,
    all_orders.order_date,
    all_orders.order_amount,
    running_stats.cumulative_total,
    running_stats.order_sequence
FROM customers c
JOIN orders all_orders ON c.customer_id = all_orders.customer_id,
LATERAL (
    SELECT 
        SUM(o2.total_amount) AS cumulative_total,
        COUNT(*) AS order_sequence
    FROM orders o2
    WHERE o2.customer_id = c.customer_id
      AND o2.order_date <= all_orders.order_date
) running_stats
WHERE all_orders.status = 'completed'
ORDER BY c.customer_id, all_orders.order_date;
```

---

## JOIN กับ Window Functions

### ตัวอย่างที่ 17: Ranking กับ JOIN

```sql
-- Rank employees ตามเงินเดือนทั้งบริษัทและภายในแผนก
SELECT 
    e.emp_id,
    CONCAT(e.first_name, ' ', e.last_name) AS employee,
    d.dept_name,
    e.salary,
    RANK() OVER (ORDER BY e.salary DESC) AS company_rank,
    RANK() OVER (PARTITION BY e.dept_id ORDER BY e.salary DESC) AS dept_rank,
    ROUND(
        e.salary / SUM(e.salary) OVER (PARTITION BY e.dept_id) * 100, 1
    ) AS pct_of_dept_payroll
FROM employees e
JOIN departments d ON e.dept_id = d.dept_id
ORDER BY d.dept_name, dept_rank;
```

### ตัวอย่างที่ 18: Window Functions ใน Subquery แล้ว JOIN

```sql
-- หาพนักงานที่อยู่ใน top 25% ของเงินเดือนทั้งบริษัท
SELECT 
    ranked.emp_id,
    ranked.employee,
    ranked.dept_name,
    ranked.salary,
    ranked.salary_percentile,
    m.first_name || ' ' || m.last_name AS manager
FROM (
    SELECT 
        e.emp_id,
        CONCAT(e.first_name, ' ', e.last_name) AS employee,
        d.dept_name,
        e.salary,
        e.manager_id,
        NTILE(4) OVER (ORDER BY e.salary) AS salary_quartile,
        PERCENT_RANK() OVER (ORDER BY e.salary) AS salary_percentile
    FROM employees e
    JOIN departments d ON e.dept_id = d.dept_id
) ranked
LEFT JOIN employees m ON ranked.manager_id = m.emp_id
WHERE ranked.salary_quartile = 4  -- Top 25%
ORDER BY ranked.salary DESC;
```

---

## Advanced JOIN Patterns

### ตัวอย่างที่ 19: EXISTS กับ JOIN เปรียบเทียบ

```sql
-- Semi-join pattern: เทียบ EXISTS กับ JOIN

-- EXISTS (Semi-join): ลูกค้าที่มี completed order
SELECT c.customer_id, c.first_name
FROM customers c
WHERE EXISTS (
    SELECT 1 FROM orders o 
    WHERE o.customer_id = c.customer_id 
      AND o.status = 'completed'
);

-- JOIN equivalent (อาจได้ duplicates ถ้าไม่ใช้ DISTINCT)
SELECT DISTINCT c.customer_id, c.first_name
FROM customers c
JOIN orders o ON c.customer_id = o.customer_id
WHERE o.status = 'completed';

-- Performance: EXISTS มักเร็วกว่าสำหรับ semi-join
-- เพราะหยุดค้นหาเมื่อเจอ match แรก
```

### ตัวอย่างที่ 20: JOIN + DISTINCT vs GROUP BY

```sql
-- สองวิธีนี้ให้ผลเหมือนกัน แต่ performance ต่างกัน

-- วิธี 1: JOIN + DISTINCT
SELECT DISTINCT
    c.customer_id,
    c.first_name || ' ' || c.last_name AS customer
FROM customers c
JOIN orders o ON c.customer_id = o.customer_id
WHERE o.status = 'completed';

-- วิธี 2: JOIN + GROUP BY (มักเร็วกว่า)
SELECT 
    c.customer_id,
    c.first_name || ' ' || c.last_name AS customer
FROM customers c
JOIN orders o ON c.customer_id = o.customer_id
WHERE o.status = 'completed'
GROUP BY c.customer_id, customer;

-- วิธี 3: EXISTS (เร็วที่สุดสำหรับ semi-join)
SELECT 
    c.customer_id,
    c.first_name || ' ' || c.last_name AS customer
FROM customers c
WHERE EXISTS (
    SELECT 1 FROM orders o 
    WHERE o.customer_id = c.customer_id 
    AND o.status = 'completed'
);
```

### ตัวอย่างที่ 21: Bucket/Binning กับ Non-Equi JOIN

```sql
-- สร้าง order size buckets แล้ว JOIN
CREATE TEMP TABLE order_buckets (
    bucket_name VARCHAR(20),
    min_amount DECIMAL(12,2),
    max_amount DECIMAL(12,2)
);
INSERT INTO order_buckets VALUES
('Micro (<1000)', 0, 999.99),
('Small (1K-5K)', 1000, 4999.99),
('Medium (5K-15K)', 5000, 14999.99),
('Large (15K-50K)', 15000, 49999.99),
('XL (50K+)', 50000, 9999999);

SELECT 
    b.bucket_name,
    COUNT(o.order_id) AS order_count,
    ROUND(AVG(o.total_amount), 2) AS avg_order,
    SUM(o.total_amount) AS total_revenue,
    COUNT(DISTINCT o.customer_id) AS unique_customers
FROM orders o
JOIN order_buckets b ON o.total_amount BETWEEN b.min_amount AND b.max_amount
WHERE o.status = 'completed'
GROUP BY b.bucket_name, b.min_amount
ORDER BY b.min_amount;
```

### ตัวอย่างที่ 22: Fuzzy JOIN ด้วย LIKE

```sql
-- JOIN ด้วย LIKE (Non-equi): หา products ที่ category ตรงกับ keyword
CREATE TEMP TABLE search_keywords (
    keyword VARCHAR(50),
    category_map VARCHAR(50)
);
INSERT INTO search_keywords VALUES
('laptop', 'Electronics'),
('chair', 'Furniture'),
('pen', 'Stationery'),
('printer', 'Electronics'),
('desk', 'Furniture');

-- JOIN ด้วย LIKE
SELECT DISTINCT
    sk.keyword,
    p.product_name,
    p.category,
    p.price
FROM search_keywords sk
JOIN products p ON LOWER(p.product_name) LIKE '%' || sk.keyword || '%'
ORDER BY sk.keyword, p.price;
```

### ตัวอย่างที่ 23: Window Function JOIN Pattern

```sql
-- Lag/Lead ด้วย Window Function + JOIN
WITH order_with_lag AS (
    SELECT 
        o.order_id,
        o.customer_id,
        o.order_date,
        o.total_amount,
        LAG(o.total_amount) OVER (PARTITION BY o.customer_id ORDER BY o.order_date) AS prev_order_amount,
        LAG(o.order_date) OVER (PARTITION BY o.customer_id ORDER BY o.order_date) AS prev_order_date
    FROM orders o
    WHERE o.status = 'completed'
)
SELECT 
    ol.order_id,
    c.first_name || ' ' || c.last_name AS customer,
    ol.order_date,
    ol.total_amount,
    ol.prev_order_amount,
    ol.total_amount - ol.prev_order_amount AS order_growth,
    ol.order_date - ol.prev_order_date AS days_since_last_order
FROM order_with_lag ol
JOIN customers c ON ol.customer_id = c.customer_id
WHERE ol.prev_order_amount IS NOT NULL
ORDER BY c.customer_id, ol.order_date;
```

### ตัวอย่างที่ 24: Percentile-Based Segmentation

```sql
-- จัด segment ตาม percentile
WITH salary_percentiles AS (
    SELECT 
        e.emp_id,
        e.salary,
        NTILE(10) OVER (ORDER BY e.salary) AS decile,
        PERCENT_RANK() OVER (ORDER BY e.salary) AS percentile
    FROM employees e
)
SELECT 
    e.emp_id,
    CONCAT(e.first_name, ' ', e.last_name) AS employee,
    d.dept_name,
    e.salary,
    sp.decile,
    ROUND(sp.percentile * 100, 1) AS percentile_rank,
    CASE 
        WHEN sp.decile >= 9 THEN 'Top 20%'
        WHEN sp.decile >= 7 THEN 'Upper Middle'
        WHEN sp.decile >= 4 THEN 'Middle'
        WHEN sp.decile >= 2 THEN 'Lower Middle'
        ELSE 'Bottom 10%'
    END AS salary_band
FROM employees e
JOIN departments d ON e.dept_id = d.dept_id
JOIN salary_percentiles sp ON e.emp_id = sp.emp_id
ORDER BY e.salary DESC;
```

### ตัวอย่างที่ 25: Complex Business Rule JOIN

```sql
-- สร้าง recommendation engine แบบง่าย
-- ลูกค้าที่ซื้อ product_id=1 มักจะซื้อ product_id ใดด้วย?
WITH buyers_of_1 AS (
    SELECT DISTINCT o.customer_id
    FROM orders o
    JOIN order_items oi ON o.order_id = oi.order_id
    WHERE oi.product_id = 1
      AND o.status = 'completed'
),
also_bought AS (
    SELECT 
        oi.product_id,
        COUNT(DISTINCT o.customer_id) AS customer_count
    FROM buyers_of_1 b
    JOIN orders o ON b.customer_id = o.customer_id
    JOIN order_items oi ON o.order_id = oi.order_id
    WHERE oi.product_id != 1  -- ไม่เอา product 1 เอง
      AND o.status = 'completed'
    GROUP BY oi.product_id
)
SELECT 
    p.product_name AS recommended_product,
    p.category,
    p.price,
    ab.customer_count AS times_bought_together,
    ROUND(ab.customer_count::NUMERIC / (SELECT COUNT(*) FROM buyers_of_1) * 100, 1) AS recommendation_strength_pct
FROM also_bought ab
JOIN products p ON ab.product_id = p.product_id
ORDER BY recommendation_strength_pct DESC;
```

---

## แบบฝึกหัดภาค 28

**ข้อ 1:** สร้าง salary grades และใช้ Non-equi JOIN เพื่อจัด grade ให้พนักงาน

```sql
-- เฉลย
WITH salary_brackets AS (
    SELECT 
        grade_name,
        min_sal,
        max_sal
    FROM (VALUES
        ('Grade A', 0, 90000),
        ('Grade B', 90001, 120000),
        ('Grade C', 120001, 150000),
        ('Grade D', 150001, 999999)
    ) AS t(grade_name, min_sal, max_sal)
)
SELECT 
    e.emp_id,
    CONCAT(e.first_name, ' ', e.last_name) AS employee,
    e.salary,
    sb.grade_name,
    d.dept_name
FROM employees e
JOIN salary_brackets sb ON e.salary BETWEEN sb.min_sal AND sb.max_sal
JOIN departments d ON e.dept_id = d.dept_id
ORDER BY e.salary DESC;
```

**ข้อ 2:** ใช้ Inline View (subquery as table) เพื่อหาพนักงานเงินเดือนสูงกว่าค่าเฉลี่ย company

```sql
-- เฉลย
SELECT 
    e.emp_id,
    CONCAT(e.first_name, ' ', e.last_name) AS employee,
    e.salary,
    avg_data.company_avg,
    e.salary - avg_data.company_avg AS above_avg
FROM employees e
JOIN (SELECT AVG(salary) AS company_avg FROM employees) avg_data ON TRUE
WHERE e.salary > avg_data.company_avg
ORDER BY above_avg DESC;
```

**ข้อ 3:** เขียน CTE แทน nested subquery เพื่อหา top customers ต่อ city

```sql
-- เฉลย
WITH customer_spending AS (
    SELECT 
        c.customer_id,
        c.first_name || ' ' || c.last_name AS customer,
        c.city,
        SUM(o.total_amount) AS total_spent
    FROM customers c
    JOIN orders o ON c.customer_id = o.customer_id
    WHERE o.status = 'completed'
    GROUP BY c.customer_id, customer, c.city
),
city_rankings AS (
    SELECT *,
        RANK() OVER (PARTITION BY city ORDER BY total_spent DESC) AS city_rank
    FROM customer_spending
)
SELECT customer_id, customer, city, total_spent, city_rank
FROM city_rankings
WHERE city_rank = 1
ORDER BY city;
```

**ข้อ 4:** หา products ที่ถูกสั่งโดยลูกค้าจากหลายกว่า 2 cities

```sql
-- เฉลย
SELECT 
    p.product_id,
    p.product_name,
    p.category,
    COUNT(DISTINCT c.city) AS cities_ordered_from,
    COUNT(DISTINCT o.customer_id) AS unique_customers
FROM products p
JOIN order_items oi ON p.product_id = oi.product_id
JOIN orders o ON oi.order_id = o.order_id
JOIN customers c ON o.customer_id = c.customer_id
WHERE o.status = 'completed'
GROUP BY p.product_id, p.product_name, p.category
HAVING COUNT(DISTINCT c.city) > 2
ORDER BY cities_ordered_from DESC;
```

**ข้อ 5:** ใช้ Non-equi JOIN เพื่อ assign discount tier ให้แต่ละ order

```sql
-- เฉลย
WITH discount_tiers AS (
    SELECT tier, min_amount, max_amount, discount_pct
    FROM (VALUES
        ('Bronze', 0, 5000, 0),
        ('Silver', 5001, 15000, 5),
        ('Gold', 15001, 30000, 10),
        ('Platinum', 30001, 999999, 15)
    ) AS t(tier, min_amount, max_amount, discount_pct)
)
SELECT 
    o.order_id,
    c.first_name || ' ' || c.last_name AS customer,
    o.total_amount,
    dt.tier,
    dt.discount_pct,
    ROUND(o.total_amount * (1 - dt.discount_pct / 100.0), 2) AS after_tier_discount
FROM orders o
JOIN customers c ON o.customer_id = c.customer_id
JOIN discount_tiers dt ON o.total_amount BETWEEN dt.min_amount AND dt.max_amount
WHERE o.status = 'completed'
ORDER BY o.total_amount DESC;
```

**ข้อ 6:** เปรียบเทียบ performance ของ EXISTS vs JOIN สำหรับ semi-join

```sql
-- เฉลย: ทั้งสองได้ผลเหมือนกัน

-- วิธี 1: EXISTS
EXPLAIN ANALYZE
SELECT c.customer_id, c.first_name
FROM customers c
WHERE EXISTS (
    SELECT 1 FROM orders o 
    WHERE o.customer_id = c.customer_id 
    AND o.status = 'completed'
);

-- วิธี 2: JOIN
EXPLAIN ANALYZE
SELECT DISTINCT c.customer_id, c.first_name
FROM customers c
JOIN orders o ON c.customer_id = o.customer_id
WHERE o.status = 'completed';

-- ผล: EXISTS มักเร็วกว่าเพราะ:
-- 1. หยุดค้นหาเมื่อเจอ match แรก
-- 2. ไม่ต้อง dedup ด้วย DISTINCT
```

**ข้อ 7:** ใช้ Window Function + JOIN เพื่อแสดง running total ของ revenue ต่อเดือน

```sql
-- เฉลย
WITH monthly_rev AS (
    SELECT 
        TO_CHAR(o.order_date, 'YYYY-MM') AS month,
        COUNT(o.order_id) AS orders,
        SUM(o.total_amount) AS revenue
    FROM orders o
    WHERE o.status = 'completed'
    GROUP BY month
)
SELECT 
    month,
    orders,
    revenue,
    SUM(revenue) OVER (ORDER BY month) AS cumulative_revenue,
    LAG(revenue) OVER (ORDER BY month) AS prev_month_revenue,
    revenue - LAG(revenue) OVER (ORDER BY month) AS month_over_month_change
FROM monthly_rev
ORDER BY month;
```

**ข้อ 8:** ใช้ CTE + JOIN เพื่อสร้าง order aging report

```sql
-- เฉลย
WITH order_ages AS (
    SELECT 
        order_id,
        customer_id,
        order_date,
        status,
        total_amount,
        CURRENT_DATE - order_date AS age_days
    FROM orders
    WHERE status IN ('pending', 'processing')
)
SELECT 
    oa.order_id,
    c.first_name || ' ' || c.last_name AS customer,
    c.email,
    oa.order_date,
    oa.age_days,
    oa.total_amount,
    oa.status,
    CASE 
        WHEN oa.age_days <= 7 THEN 'Fresh'
        WHEN oa.age_days <= 14 THEN 'Normal'
        WHEN oa.age_days <= 30 THEN 'Aging'
        ELSE 'OVERDUE'
    END AS age_status,
    COUNT(oi.item_id) AS items
FROM order_ages oa
JOIN customers c ON oa.customer_id = c.customer_id
JOIN order_items oi ON oa.order_id = oi.order_id
GROUP BY oa.order_id, customer, c.email, oa.order_date, 
         oa.age_days, oa.total_amount, oa.status
ORDER BY oa.age_days DESC;
```

**ข้อ 9:** สร้าง product recommendation ด้วย Multi-table JOIN

```sql
-- เฉลย: "ลูกค้าที่ซื้อ X มักซื้อ Y ด้วย"
WITH product_pairs AS (
    SELECT 
        oi1.product_id AS product_a,
        oi2.product_id AS product_b,
        COUNT(DISTINCT oi1.order_id) AS times_together
    FROM order_items oi1
    JOIN order_items oi2 ON oi1.order_id = oi2.order_id
        AND oi1.product_id < oi2.product_id
    JOIN orders o ON oi1.order_id = o.order_id
    WHERE o.status = 'completed'
    GROUP BY product_a, product_b
    HAVING COUNT(DISTINCT oi1.order_id) >= 2
)
SELECT 
    pa.product_name AS product_a,
    pb.product_name AS product_b,
    pp.times_together,
    pa.category AS cat_a,
    pb.category AS cat_b
FROM product_pairs pp
JOIN products pa ON pp.product_a = pa.product_id
JOIN products pb ON pp.product_b = pb.product_id
ORDER BY pp.times_together DESC;
```

**ข้อ 10:** Overlap Detection: หา orders ที่ใช้ promotion เดียวกัน

```sql
-- เฉลย: Non-equi JOIN สำหรับ date overlap detection
WITH order_promos AS (
    SELECT 
        o.order_id,
        o.customer_id,
        o.order_date,
        o.total_amount,
        p.promo_name,
        p.discount_pct
    FROM orders o
    JOIN promotions p ON o.order_date BETWEEN p.start_date AND p.end_date
    WHERE o.status = 'completed'
)
SELECT 
    op.order_id,
    c.first_name || ' ' || c.last_name AS customer,
    op.order_date,
    op.total_amount,
    op.promo_name,
    op.discount_pct,
    ROUND(op.total_amount * (1 - op.discount_pct / 100), 2) AS discounted_total
FROM order_promos op
JOIN customers c ON op.customer_id = c.customer_id
ORDER BY op.promo_name, op.order_date;
```

---

## สรุปภาค 28

1. **Non-Equi JOIN** — JOIN ด้วย `<`, `>`, `BETWEEN`, `LIKE` สำหรับ range matching
2. **Inline Views** — Subquery ใน FROM clause เป็น derived table
3. **CTEs** — ทำให้ query อ่านง่าย แยกตรรกะเป็นชั้นๆ
4. **LATERAL JOIN** — Correlated subquery ใน FROM, ดีสำหรับ top-N per group
5. **Window Functions** — ใช้ร่วมกับ JOIN เพื่อ ranking, percentile
6. **EXISTS vs JOIN** — EXISTS ดีกว่าสำหรับ semi-join pattern

**ในภาคถัดไป** จะเรียน JOIN Performance Optimization!
