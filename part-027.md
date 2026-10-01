# ภาค 27: Multiple Table JOINs (4+ ตาราง)

## การ JOIN หลายตาราง

ในการทำงานจริง เราต้องดึงข้อมูลจาก 3, 4, 5 หรือมากกว่าตารางพร้อมกัน บทนี้จะสอนวิธีเขียน query ที่ JOIN หลายตารางอย่างมีประสิทธิภาพและอ่านง่าย

### หลักการเขียน Multi-Table JOIN

```
กฎของ Multi-Table JOIN:

1. เริ่มจากตารางหลัก (main table)
2. JOIN ทีละตาราง โดยระบุ condition ชัดเจน
3. ใช้ aliases ที่มีความหมาย (e=employees, d=departments, o=orders)
4. เรียงลำดับ JOIN ให้อ่านง่าย (ไม่จำเป็นต้องตรงกับ query plan)
5. แต่ละ JOIN ต้องมี ON condition

Performance ของ Multi-Table JOIN:
- SQL engine จะหาลำดับ JOIN ที่ดีที่สุดเอง (query optimizer)
- เราเขียนเพื่อ readability ไม่ใช่ performance
- แต่ถ้ามี index ที่ถูกต้อง ก็ช่วยได้มาก
```

---

## ตัวอย่าง 3-Table JOIN

### ตัวอย่างที่ 1: employees + departments + managers

```sql
-- ข้อมูลพนักงานครบถ้วน: แผนก + ผู้จัดการ
SELECT 
    e.emp_id,
    CONCAT(e.first_name, ' ', e.last_name) AS employee,
    e.job_title,
    e.salary,
    e.hire_date,
    d.dept_name,
    d.location,
    CONCAT(m.first_name, ' ', m.last_name) AS manager
FROM employees e                              -- ตาราง 1
JOIN departments d ON e.dept_id = d.dept_id  -- ตาราง 2
LEFT JOIN employees m ON e.manager_id = m.emp_id  -- ตาราง 3 (self)
ORDER BY d.dept_name, e.salary DESC;
```

### ตัวอย่างที่ 2: orders + customers + order_items

```sql
-- รายละเอียด order พร้อม items ทั้งหมด
SELECT 
    o.order_id,
    o.order_date,
    o.status,
    CONCAT(c.first_name, ' ', c.last_name) AS customer,
    c.city,
    COUNT(oi.item_id) AS item_count,
    SUM(oi.quantity) AS total_qty,
    SUM(oi.quantity * oi.unit_price - oi.discount) AS calculated_total,
    o.total_amount AS stored_total
FROM orders o                                    -- ตาราง 1
JOIN customers c ON o.customer_id = c.customer_id  -- ตาราง 2
LEFT JOIN order_items oi ON o.order_id = oi.order_id  -- ตาราง 3
GROUP BY o.order_id, o.order_date, o.status, customer, c.city, o.total_amount
ORDER BY o.order_date;
```

### ตัวอย่างที่ 3: order_items + orders + products

```sql
-- รายการสินค้าทั้งหมดพร้อม order info
SELECT 
    oi.item_id,
    o.order_id,
    o.order_date,
    o.status,
    p.product_name,
    p.category,
    oi.quantity,
    oi.unit_price,
    oi.discount,
    oi.quantity * oi.unit_price - oi.discount AS line_total
FROM order_items oi                            -- ตาราง 1
JOIN orders o ON oi.order_id = o.order_id     -- ตาราง 2
JOIN products p ON oi.product_id = p.product_id  -- ตาราง 3
ORDER BY o.order_date, oi.item_id;
```

---

## ตัวอย่าง 4-Table JOIN

### ตัวอย่างที่ 4: orders + customers + order_items + products

```sql
-- Complete Sales Report (4 ตาราง)
SELECT 
    o.order_id,
    o.order_date,
    o.status,
    CONCAT(c.first_name, ' ', c.last_name) AS customer_name,
    c.city AS customer_city,
    p.product_name,
    p.category,
    oi.quantity,
    oi.unit_price,
    oi.discount,
    (oi.quantity * oi.unit_price - oi.discount) AS line_amount
FROM orders o                                           -- ตาราง 1
JOIN customers c ON o.customer_id = c.customer_id      -- ตาราง 2
JOIN order_items oi ON o.order_id = oi.order_id       -- ตาราง 3
JOIN products p ON oi.product_id = p.product_id        -- ตาราง 4
ORDER BY o.order_id, p.product_name;
```

### ตัวอย่างที่ 5: employees + departments + managers + department info

```sql
-- Full employee directory (4 ตาราง: 2× employees + 2× departments)
SELECT 
    e.emp_id,
    CONCAT(e.first_name, ' ', e.last_name) AS employee,
    e.job_title,
    e.salary,
    ed.dept_name AS employee_dept,
    ed.location AS dept_location,
    CONCAT(m.first_name, ' ', m.last_name) AS manager,
    md.dept_name AS manager_dept  -- แผนกของ manager อาจต่างกัน
FROM employees e                                        -- ตาราง 1: employee
JOIN departments ed ON e.dept_id = ed.dept_id          -- ตาราง 2: emp's dept
LEFT JOIN employees m ON e.manager_id = m.emp_id       -- ตาราง 3: manager
LEFT JOIN departments md ON m.dept_id = md.dept_id     -- ตาราง 4: mgr's dept
ORDER BY ed.dept_name, e.emp_id;
```

### ตัวอย่างที่ 6: 4-Table สำหรับ Sales Analysis

```sql
-- Sales ต่อ category ต่อ customer city
SELECT 
    c.city,
    c.country,
    p.category,
    COUNT(DISTINCT o.order_id) AS orders,
    SUM(oi.quantity) AS units_sold,
    SUM(oi.quantity * oi.unit_price - oi.discount) AS revenue,
    RANK() OVER (PARTITION BY p.category ORDER BY SUM(oi.quantity * oi.unit_price - oi.discount) DESC) AS rank_by_category
FROM customers c                                           -- ตาราง 1
JOIN orders o ON c.customer_id = o.customer_id            -- ตาราง 2
JOIN order_items oi ON o.order_id = oi.order_id          -- ตาราง 3
JOIN products p ON oi.product_id = p.product_id           -- ตาราง 4
WHERE o.status IN ('completed', 'shipped')
GROUP BY c.city, c.country, p.category
ORDER BY p.category, revenue DESC;
```

### ตัวอย่างที่ 7: อ่าน Order ด้วย 4 ตาราง + Aggregation

```sql
-- Customer Purchase Summary: customer + order + items + product
SELECT 
    c.customer_id,
    CONCAT(c.first_name, ' ', c.last_name) AS customer,
    c.city,
    COUNT(DISTINCT o.order_id) AS total_orders,
    COUNT(oi.item_id) AS total_line_items,
    SUM(oi.quantity) AS total_units,
    SUM(oi.quantity * oi.unit_price - oi.discount) AS total_revenue,
    -- Most bought category
    MODE() WITHIN GROUP (ORDER BY p.category) AS favorite_category,
    MIN(o.order_date) AS first_order,
    MAX(o.order_date) AS last_order
FROM customers c                                           -- 1
JOIN orders o ON c.customer_id = o.customer_id            -- 2
JOIN order_items oi ON o.order_id = oi.order_id          -- 3
JOIN products p ON oi.product_id = p.product_id           -- 4
WHERE o.status = 'completed'
GROUP BY c.customer_id, customer, c.city
ORDER BY total_revenue DESC;
```

---

## ตัวอย่าง 5-Table JOIN

### ตัวอย่างที่ 8: 5 ตาราง — Full Order Details

```sql
-- Complete order report: customer + order + items + product + dept (of who manages product)
SELECT 
    o.order_id,
    o.order_date,
    CONCAT(c.first_name, ' ', c.last_name) AS customer,
    c.city,
    p.product_name,
    p.category,
    oi.quantity,
    oi.unit_price,
    (oi.quantity * oi.unit_price - oi.discount) AS line_total,
    o.status,
    o.shipped_date,
    CASE 
        WHEN o.shipped_date IS NOT NULL 
        THEN o.shipped_date - o.order_date 
        ELSE NULL
    END AS days_to_ship
FROM orders o                                              -- 1
JOIN customers c ON o.customer_id = c.customer_id         -- 2
JOIN order_items oi ON o.order_id = oi.order_id          -- 3
JOIN products p ON oi.product_id = p.product_id           -- 4
LEFT JOIN (
    -- Subquery เป็นตาราง "5" (สมมติ product_managers)
    SELECT emp_id, job_title, dept_id
    FROM employees
    WHERE job_title LIKE '%Manager%'
) pm ON pm.dept_id = 3  -- Sales dept manages products
ORDER BY o.order_date, o.order_id;
```

### ตัวอย่างที่ 9: 5 ตาราง — Employee Full Profile

```sql
-- Full employee profile with all relationships
SELECT 
    e.emp_id,
    CONCAT(e.first_name, ' ', e.last_name) AS employee,
    e.email,
    e.job_title,
    e.salary,
    e.hire_date,
    -- Department info
    d.dept_name,
    d.location,
    d.budget AS dept_budget,
    -- Manager info
    CONCAT(m.first_name, ' ', m.last_name) AS direct_manager,
    m.job_title AS manager_title,
    -- Manager's manager
    CONCAT(gm.first_name, ' ', gm.last_name) AS grand_manager,
    gm.job_title AS grand_manager_title
FROM employees e                                           -- 1
JOIN departments d ON e.dept_id = d.dept_id               -- 2
LEFT JOIN employees m ON e.manager_id = m.emp_id          -- 3
LEFT JOIN departments dm ON m.dept_id = dm.dept_id        -- 4
LEFT JOIN employees gm ON m.manager_id = gm.emp_id        -- 5
ORDER BY d.dept_name, e.emp_id;
```

### ตัวอย่างที่ 10: 5 ตาราง — Revenue by Customer City and Product Category

```sql
-- Revenue matrix: customer location × product category
SELECT 
    c.country,
    c.city,
    p.category,
    -- Monthly breakdown
    SUM(CASE WHEN EXTRACT(MONTH FROM o.order_date) = 1 THEN oi.quantity * oi.unit_price - oi.discount ELSE 0 END) AS jan,
    SUM(CASE WHEN EXTRACT(MONTH FROM o.order_date) = 2 THEN oi.quantity * oi.unit_price - oi.discount ELSE 0 END) AS feb,
    SUM(CASE WHEN EXTRACT(MONTH FROM o.order_date) = 3 THEN oi.quantity * oi.unit_price - oi.discount ELSE 0 END) AS mar,
    SUM(CASE WHEN EXTRACT(MONTH FROM o.order_date) = 4 THEN oi.quantity * oi.unit_price - oi.discount ELSE 0 END) AS apr,
    SUM(CASE WHEN EXTRACT(MONTH FROM o.order_date) = 5 THEN oi.quantity * oi.unit_price - oi.discount ELSE 0 END) AS may,
    SUM(CASE WHEN EXTRACT(MONTH FROM o.order_date) = 6 THEN oi.quantity * oi.unit_price - oi.discount ELSE 0 END) AS jun,
    SUM(oi.quantity * oi.unit_price - oi.discount) AS total_revenue
FROM customers c                                           -- 1
JOIN orders o ON c.customer_id = o.customer_id            -- 2
JOIN order_items oi ON o.order_id = oi.order_id          -- 3
JOIN products p ON oi.product_id = p.product_id           -- 4
JOIN (SELECT DISTINCT category FROM products) cats ON p.category = cats.category  -- 5
WHERE o.status IN ('completed', 'shipped')
  AND EXTRACT(YEAR FROM o.order_date) = 2024
GROUP BY c.country, c.city, p.category
ORDER BY c.city, p.category;
```

---

## ตัวอย่าง 6-Table JOIN

### ตัวอย่างที่ 11: 6 ตาราง — Complete Business Report

```sql
-- Ultimate business report: ทุกตารางรวมกัน
SELECT 
    -- Customer info
    c.customer_id,
    c.first_name || ' ' || c.last_name AS customer,
    c.city,
    
    -- Order info
    o.order_id,
    o.order_date,
    o.status,
    
    -- Product info
    p.product_id,
    p.product_name,
    p.category,
    
    -- Order item details
    oi.quantity,
    oi.unit_price,
    oi.discount,
    oi.quantity * oi.unit_price - oi.discount AS line_total,
    
    -- Department responsible for product category
    d.dept_name AS responsible_dept
FROM orders o                                                -- 1
JOIN customers c ON o.customer_id = c.customer_id           -- 2
JOIN order_items oi ON o.order_id = oi.order_id            -- 3
JOIN products p ON oi.product_id = p.product_id             -- 4
-- แผนก Sales รับผิดชอบ Electronics, Marketing รับผิดชอบ rest
JOIN departments d ON (
    (p.category = 'Electronics' AND d.dept_name = 'Sales') OR
    (p.category != 'Electronics' AND d.dept_name = 'Marketing')
)                                                            -- 5 (conditional)
LEFT JOIN (
    SELECT dept_id, COUNT(*) AS active_staff
    FROM employees
    GROUP BY dept_id
) staff ON staff.dept_id = d.dept_id                       -- 6 (subquery as table)
WHERE o.status IN ('completed', 'shipped')
ORDER BY o.order_id, p.category;
```

### ตัวอย่างที่ 12: 6 ตาราง — Employee Order Impact Analysis

```sql
-- วิเคราะห์ว่าพนักงานในแต่ละแผนกส่งผลต่อ sales อย่างไร
WITH dept_sales AS (
    SELECT 
        p.category,
        SUM(oi.quantity * oi.unit_price - oi.discount) AS category_revenue
    FROM order_items oi
    JOIN orders o ON oi.order_id = o.order_id
    JOIN products p ON oi.product_id = p.product_id
    WHERE o.status IN ('completed', 'shipped')
    GROUP BY p.category
)
SELECT 
    d.dept_name,
    d.location,
    d.budget,
    COUNT(DISTINCT e.emp_id) AS headcount,
    AVG(e.salary) AS avg_salary,
    SUM(e.salary) * 12 AS annual_payroll,
    COALESCE(ds.category_revenue, 0) AS related_revenue,
    COALESCE(ds.category_revenue, 0) / NULLIF(d.budget, 0) AS revenue_to_budget_ratio
FROM departments d                                    -- 1
JOIN employees e ON d.dept_id = e.dept_id            -- 2
LEFT JOIN (
    SELECT 
        CASE 
            WHEN p.category = 'Electronics' THEN 3   -- Sales dept_id
            WHEN p.category IN ('Software') THEN 8   -- R&D dept_id
            ELSE 2                                    -- Marketing dept_id
        END AS dept_id,
        SUM(oi.quantity * oi.unit_price - oi.discount) AS revenue
    FROM order_items oi                               -- 3 (in subquery)
    JOIN products p ON oi.product_id = p.product_id  -- 4 (in subquery)
    JOIN orders o ON oi.order_id = o.order_id        -- 5 (in subquery)
    WHERE o.status = 'completed'
    GROUP BY dept_id
) ds ON d.dept_id = ds.dept_id                       -- 6
GROUP BY d.dept_id, d.dept_name, d.location, d.budget, ds.category_revenue
ORDER BY related_revenue DESC NULLS LAST;
```

---

## Aliasing Strategies สำหรับ Multi-Table JOIN

### ตัวอย่างที่ 13: Meaningful Aliases

```sql
-- BAD: aliases ไม่ชัดเจน
SELECT a.emp_id, b.dept_name, c.first_name
FROM employees a
JOIN departments b ON a.dept_id = b.dept_id
LEFT JOIN employees c ON a.manager_id = c.emp_id;

-- GOOD: aliases มีความหมาย
SELECT emp.emp_id, dept.dept_name, mgr.first_name AS manager_name
FROM employees emp
JOIN departments dept ON emp.dept_id = dept.dept_id
LEFT JOIN employees mgr ON emp.manager_id = mgr.emp_id;

-- BETTER: ชัดเจนยิ่งขึ้น
SELECT 
    e.emp_id,
    CONCAT(e.first_name, ' ', e.last_name) AS employee_full_name,
    d.dept_name AS department,
    CONCAT(manager.first_name, ' ', manager.last_name) AS manager_full_name
FROM employees e           -- e = employee (main)
JOIN departments d          -- d = department  
    ON e.dept_id = d.dept_id
LEFT JOIN employees manager -- manager = employee acting as manager
    ON e.manager_id = manager.emp_id
ORDER BY d.dept_name, e.emp_id;
```

### ตัวอย่างที่ 14: Avoiding Column Ambiguity

```sql
-- เมื่อ column ชื่อเหมือนกันในหลายตาราง ต้องระบุ table alias เสมอ
SELECT 
    o.order_id,              -- ระบุ! 'order_id' อาจ ambiguous ถ้าหลายตาราง
    c.customer_id,           -- ระบุ!
    c.first_name,            -- first_name อยู่แค่ใน customers → OK ไม่ระบุก็ได้
    o.order_date,
    oi.quantity,
    p.product_id,
    p.product_name
FROM orders o
JOIN customers c ON o.customer_id = c.customer_id
JOIN order_items oi ON o.order_id = oi.order_id
JOIN products p ON oi.product_id = p.product_id
ORDER BY o.order_id;
-- ถ้าไม่ระบุ table: "ERROR: column reference 'customer_id' is ambiguous"
-- (ทั้ง orders.customer_id และ customers.customer_id มีค่าเหมือนกัน แต่ต้องระบุ)
```

---

## Complex Business Queries ด้วย Multi-Table JOIN

### ตัวอย่างที่ 15: Product Performance by Department

```sql
-- รายงาน: แผนกไหนรับผิดชอบ category ไหน และ performance เป็นอย่างไร
SELECT 
    d.dept_name,
    d.budget,
    p.category,
    COUNT(DISTINCT p.product_id) AS products_in_category,
    COUNT(DISTINCT o.order_id) AS orders,
    SUM(oi.quantity) AS units_sold,
    SUM(oi.quantity * oi.unit_price - oi.discount) AS revenue,
    ROUND(SUM(oi.quantity * oi.unit_price - oi.discount) / d.budget * 100, 2) AS revenue_to_budget_pct
FROM departments d
CROSS JOIN (SELECT DISTINCT category FROM products) cats
JOIN employees e ON d.dept_id = e.dept_id
JOIN products p ON p.category = cats.category
JOIN order_items oi ON p.product_id = oi.product_id
JOIN orders o ON oi.order_id = o.order_id
WHERE o.status IN ('completed', 'shipped')
  AND d.dept_name = 'Sales'
GROUP BY d.dept_id, d.dept_name, d.budget, p.category
ORDER BY revenue DESC;
```

### ตัวอย่างที่ 16: Customer Segmentation with Purchase Details

```sql
-- จัด segment ลูกค้าพร้อม detail สินค้าที่ชื่นชอบ
WITH customer_stats AS (
    SELECT 
        c.customer_id,
        CONCAT(c.first_name, ' ', c.last_name) AS customer,
        c.city,
        COUNT(DISTINCT o.order_id) AS order_count,
        SUM(oi.quantity * oi.unit_price - oi.discount) AS total_spent,
        MAX(o.order_date) AS last_order_date
    FROM customers c
    JOIN orders o ON c.customer_id = o.customer_id
    JOIN order_items oi ON o.order_id = oi.order_id
    WHERE o.status = 'completed'
    GROUP BY c.customer_id, customer, c.city
),
top_category AS (
    SELECT 
        c.customer_id,
        p.category,
        SUM(oi.quantity) AS qty,
        RANK() OVER (PARTITION BY c.customer_id ORDER BY SUM(oi.quantity) DESC) AS rn
    FROM customers c
    JOIN orders o ON c.customer_id = o.customer_id
    JOIN order_items oi ON o.order_id = oi.order_id
    JOIN products p ON oi.product_id = p.product_id
    WHERE o.status = 'completed'
    GROUP BY c.customer_id, p.category
)
SELECT 
    cs.customer_id,
    cs.customer,
    cs.city,
    cs.order_count,
    cs.total_spent,
    cs.last_order_date,
    tc.category AS favorite_category,
    CASE 
        WHEN cs.total_spent >= 50000 THEN 'VIP'
        WHEN cs.total_spent >= 20000 THEN 'Gold'
        WHEN cs.total_spent >= 5000  THEN 'Silver'
        ELSE 'Bronze'
    END AS customer_tier
FROM customer_stats cs
LEFT JOIN top_category tc ON cs.customer_id = tc.customer_id AND tc.rn = 1
ORDER BY cs.total_spent DESC;
```

### ตัวอย่างที่ 17: Inventory Reorder Report

```sql
-- รายงานสต็อกที่ต้องสั่งซื้อเพิ่ม
SELECT 
    p.product_id,
    p.product_name,
    p.category,
    p.stock_quantity AS current_stock,
    p.price,
    -- ยอดขายใน 30 วันที่ผ่านมา
    COALESCE(recent.qty_30d, 0) AS sold_last_30_days,
    COALESCE(recent.qty_30d, 0) / 30.0 AS daily_avg_sales,
    -- จำนวนวันที่สต็อกจะหมด
    CASE 
        WHEN COALESCE(recent.qty_30d, 0) = 0 THEN 999
        ELSE ROUND(p.stock_quantity / (COALESCE(recent.qty_30d, 0) / 30.0))
    END AS days_of_stock,
    -- Reorder recommendation
    CASE 
        WHEN p.stock_quantity = 0 THEN 'URGENT - OUT OF STOCK'
        WHEN COALESCE(recent.qty_30d, 0) = 0 THEN 'MONITOR - NO RECENT SALES'
        WHEN p.stock_quantity < COALESCE(recent.qty_30d, 0) / 30.0 * 14 THEN 'REORDER NOW'
        WHEN p.stock_quantity < COALESCE(recent.qty_30d, 0) / 30.0 * 30 THEN 'REORDER SOON'
        ELSE 'OK'
    END AS reorder_action
FROM products p
LEFT JOIN (
    SELECT 
        oi.product_id,
        SUM(oi.quantity) AS qty_30d
    FROM order_items oi
    JOIN orders o ON oi.order_id = o.order_id
    WHERE o.order_date >= CURRENT_DATE - INTERVAL '30 days'
      AND o.status IN ('completed', 'shipped', 'processing')
    GROUP BY oi.product_id
) recent ON p.product_id = recent.product_id
ORDER BY 
    CASE 
        WHEN p.stock_quantity = 0 THEN 0
        WHEN recent.qty_30d > 0 AND p.stock_quantity < recent.qty_30d / 30.0 * 14 THEN 1
        ELSE 2
    END,
    p.category;
```

### ตัวอย่างที่ 18: Sales Team Performance

```sql
-- Performance ของทีม Sales
SELECT 
    d.dept_name,
    CONCAT(m.first_name, ' ', m.last_name) AS team_manager,
    COUNT(DISTINCT e.emp_id) AS team_size,
    -- สมมติ Sales reps responsible for orders (linked by city assignment)
    COUNT(DISTINCT o.order_id) AS orders_this_month,
    SUM(o.total_amount) AS team_revenue,
    ROUND(SUM(o.total_amount) / COUNT(DISTINCT e.emp_id), 2) AS revenue_per_rep,
    ROUND(AVG(e.salary), 2) AS avg_salary,
    -- Commission estimate: 3% of revenue
    ROUND(SUM(o.total_amount) * 0.03, 2) AS estimated_commission
FROM departments d
JOIN employees m ON d.dept_id = m.dept_id AND m.job_title LIKE '%Manager%'
JOIN employees e ON e.manager_id = m.emp_id
CROSS JOIN (
    -- Sales data (สมมติ linked)
    SELECT o.order_id, o.total_amount, c.city
    FROM orders o
    JOIN customers c ON o.customer_id = c.customer_id
    WHERE o.status = 'completed'
      AND EXTRACT(MONTH FROM o.order_date) = EXTRACT(MONTH FROM CURRENT_DATE)
) o ON TRUE  -- CROSS JOIN + WHERE = simulate assignment
WHERE d.dept_name = 'Sales'
GROUP BY d.dept_name, m.emp_id, m.first_name, m.last_name
ORDER BY team_revenue DESC;
```

### ตัวอย่างที่ 19: Complete Order Lifecycle with Timestamps

```sql
-- ติดตาม lifecycle ของ order ครบถ้วน
SELECT 
    o.order_id,
    -- Customer
    c.customer_id,
    CONCAT(c.first_name, ' ', c.last_name) AS customer,
    c.city,
    -- Order
    o.order_date,
    o.status,
    o.shipped_date,
    -- Items summary
    COUNT(oi.item_id) AS item_count,
    SUM(oi.quantity) AS total_qty,
    -- Products
    STRING_AGG(DISTINCT p.product_name, ', ') AS products_ordered,
    STRING_AGG(DISTINCT p.category, ', ') AS categories,
    -- Financial
    SUM(oi.quantity * oi.unit_price) AS gross_amount,
    SUM(oi.discount) AS total_discount,
    SUM(oi.quantity * oi.unit_price - oi.discount) AS net_amount,
    o.total_amount AS invoiced_amount,
    -- Timing
    CASE WHEN o.shipped_date IS NOT NULL 
         THEN o.shipped_date - o.order_date 
    END AS processing_days
FROM orders o
JOIN customers c ON o.customer_id = c.customer_id
JOIN order_items oi ON o.order_id = oi.order_id
JOIN products p ON oi.product_id = p.product_id
GROUP BY o.order_id, c.customer_id, customer, c.city,
         o.order_date, o.status, o.shipped_date, o.total_amount
ORDER BY o.order_date DESC;
```

### ตัวอย่างที่ 20: Top 5 Products per Customer

```sql
-- สินค้าที่แต่ละลูกค้าซื้อมากที่สุด Top 5
WITH customer_product_sales AS (
    SELECT 
        c.customer_id,
        CONCAT(c.first_name, ' ', c.last_name) AS customer,
        p.product_id,
        p.product_name,
        p.category,
        SUM(oi.quantity) AS qty_bought,
        SUM(oi.quantity * oi.unit_price - oi.discount) AS amount_spent,
        RANK() OVER (
            PARTITION BY c.customer_id 
            ORDER BY SUM(oi.quantity * oi.unit_price - oi.discount) DESC
        ) AS product_rank
    FROM customers c
    JOIN orders o ON c.customer_id = o.customer_id
    JOIN order_items oi ON o.order_id = oi.order_id
    JOIN products p ON oi.product_id = p.product_id
    WHERE o.status = 'completed'
    GROUP BY c.customer_id, customer, p.product_id, p.product_name, p.category
)
SELECT 
    customer_id,
    customer,
    product_rank AS rank,
    product_name,
    category,
    qty_bought,
    amount_spent
FROM customer_product_sales
WHERE product_rank <= 5
ORDER BY customer_id, product_rank;
```

---

## แบบฝึกหัดภาค 27

**ข้อ 1:** เขียน query JOIN 3 ตาราง: orders + customers + order_items สรุปยอดต่อลูกค้า

```sql
-- เฉลย
SELECT 
    c.customer_id,
    CONCAT(c.first_name, ' ', c.last_name) AS customer,
    c.city,
    COUNT(DISTINCT o.order_id) AS total_orders,
    COUNT(oi.item_id) AS total_line_items,
    SUM(oi.quantity) AS units_purchased,
    SUM(oi.quantity * oi.unit_price - oi.discount) AS total_spent
FROM customers c
JOIN orders o ON c.customer_id = o.customer_id
JOIN order_items oi ON o.order_id = oi.order_id
WHERE o.status = 'completed'
GROUP BY c.customer_id, customer, c.city
ORDER BY total_spent DESC;
```

**ข้อ 2:** เขียน query JOIN 4 ตาราง แสดงทุก line item พร้อมชื่อลูกค้าและสินค้า

```sql
-- เฉลย
SELECT 
    o.order_id,
    o.order_date,
    CONCAT(c.first_name, ' ', c.last_name) AS customer,
    p.product_name,
    p.category,
    oi.quantity,
    oi.unit_price,
    oi.discount,
    oi.quantity * oi.unit_price - oi.discount AS line_total,
    o.status
FROM orders o
JOIN customers c ON o.customer_id = c.customer_id
JOIN order_items oi ON o.order_id = oi.order_id
JOIN products p ON oi.product_id = p.product_id
ORDER BY o.order_id, p.product_name;
```

**ข้อ 3:** หายอดขายต่อ department ใน Engineering

```sql
-- เฉลย: employees + departments + manager chain
SELECT 
    d.dept_name,
    COUNT(DISTINCT e.emp_id) AS staff_count,
    COUNT(DISTINCT CASE WHEN m.job_title LIKE '%Manager%' THEN m.emp_id END) AS managers,
    SUM(e.salary) AS monthly_payroll,
    MIN(e.hire_date) AS oldest_employee_date,
    MAX(e.hire_date) AS newest_employee_date
FROM departments d
JOIN employees e ON d.dept_id = e.dept_id
LEFT JOIN employees m ON e.manager_id = m.emp_id
WHERE d.dept_name = 'Engineering'
GROUP BY d.dept_id, d.dept_name;
```

**ข้อ 4:** แสดง Top 3 products ต่อ city ของลูกค้า

```sql
-- เฉลย
WITH city_product AS (
    SELECT 
        c.city,
        p.product_name,
        p.category,
        SUM(oi.quantity * oi.unit_price - oi.discount) AS revenue,
        RANK() OVER (PARTITION BY c.city ORDER BY SUM(oi.quantity * oi.unit_price - oi.discount) DESC) AS rn
    FROM customers c
    JOIN orders o ON c.customer_id = o.customer_id
    JOIN order_items oi ON o.order_id = oi.order_id
    JOIN products p ON oi.product_id = p.product_id
    WHERE o.status IN ('completed', 'shipped')
    GROUP BY c.city, p.product_id, p.product_name, p.category
)
SELECT city, rn AS rank, product_name, category, revenue
FROM city_product
WHERE rn <= 3
ORDER BY city, rn;
```

**ข้อ 5:** สร้าง Monthly Sales Report ต่อ category ด้วย 4 ตาราง

```sql
-- เฉลย
SELECT 
    TO_CHAR(o.order_date, 'YYYY-MM') AS month,
    p.category,
    COUNT(DISTINCT o.order_id) AS orders,
    COUNT(DISTINCT o.customer_id) AS unique_customers,
    SUM(oi.quantity) AS units,
    SUM(oi.quantity * oi.unit_price - oi.discount) AS revenue
FROM orders o
JOIN customers c ON o.customer_id = c.customer_id
JOIN order_items oi ON o.order_id = oi.order_id
JOIN products p ON oi.product_id = p.product_id
WHERE o.status IN ('completed', 'shipped')
GROUP BY TO_CHAR(o.order_date, 'YYYY-MM'), p.category
ORDER BY month, p.category;
```

**ข้อ 6:** หา customers ที่ไม่เคยสั่ง Electronics เลย

```sql
-- เฉลย
SELECT DISTINCT
    c.customer_id,
    CONCAT(c.first_name, ' ', c.last_name) AS customer,
    c.city
FROM customers c
JOIN orders o ON c.customer_id = o.customer_id
JOIN order_items oi ON o.order_id = oi.order_id
JOIN products p ON oi.product_id = p.product_id
WHERE c.customer_id NOT IN (
    SELECT DISTINCT c2.customer_id
    FROM customers c2
    JOIN orders o2 ON c2.customer_id = o2.customer_id
    JOIN order_items oi2 ON o2.order_id = oi2.order_id
    JOIN products p2 ON oi2.product_id = p2.product_id
    WHERE p2.category = 'Electronics'
)
ORDER BY c.customer_id;
```

**ข้อ 7:** คำนวณ payroll burden ต่อ department (salary + estimated overhead)

```sql
-- เฉลย
SELECT 
    d.dept_id,
    d.dept_name,
    d.location,
    d.budget,
    COUNT(e.emp_id) AS headcount,
    SUM(e.salary) AS monthly_base,
    SUM(e.salary) * 12 AS annual_base,
    SUM(e.salary) * 12 * 1.30 AS total_cost_estimate,  -- 30% overhead (benefits, etc.)
    d.budget - SUM(e.salary) * 12 * 1.30 AS budget_after_payroll,
    CONCAT(m.first_name, ' ', m.last_name) AS dept_head
FROM departments d
JOIN employees e ON d.dept_id = e.dept_id
LEFT JOIN employees m ON d.dept_id = m.dept_id 
    AND (m.job_title LIKE '%Director%' OR m.job_title LIKE '%VP%')
GROUP BY d.dept_id, d.dept_name, d.location, d.budget, m.emp_id, m.first_name, m.last_name
ORDER BY headcount DESC;
```

**ข้อ 8:** สรุป order ทุก order พร้อม detail ครบถ้วนใน 1 query

```sql
-- เฉลย
SELECT 
    o.order_id,
    o.order_date,
    o.status,
    o.shipped_date,
    -- Customer
    CONCAT(c.first_name, ' ', c.last_name) AS customer,
    c.city,
    c.country,
    -- Items
    COUNT(oi.item_id) AS line_items,
    SUM(oi.quantity) AS total_units,
    -- Financials
    SUM(oi.quantity * oi.unit_price) AS subtotal,
    SUM(oi.discount) AS total_discounts,
    SUM(oi.quantity * oi.unit_price - oi.discount) AS net_total,
    o.total_amount AS invoiced,
    -- Categories ordered
    STRING_AGG(DISTINCT p.category, ', ' ORDER BY p.category) AS categories
FROM orders o
JOIN customers c ON o.customer_id = c.customer_id
JOIN order_items oi ON o.order_id = oi.order_id
JOIN products p ON oi.product_id = p.product_id
GROUP BY o.order_id, o.order_date, o.status, o.shipped_date, 
         customer, c.city, c.country, o.total_amount
ORDER BY o.order_date;
```

**ข้อ 9:** หา employee ที่จัดการ product categories เกิน 1 category

```sql
-- เฉลย (สมมติ employees รับผิดชอบ products ตาม dept assignment)
SELECT 
    d.dept_name,
    COUNT(DISTINCT e.emp_id) AS employees_in_dept,
    COUNT(DISTINCT p.category) AS product_categories_handled,
    STRING_AGG(DISTINCT p.category, ', ' ORDER BY p.category) AS categories
FROM departments d
JOIN employees e ON d.dept_id = e.dept_id
CROSS JOIN products p  -- cross ให้เห็นว่า dept handle ทุก category
JOIN order_items oi ON p.product_id = oi.product_id
JOIN orders o ON oi.order_id = o.order_id
WHERE o.status = 'completed'
  AND d.dept_name IN ('Sales', 'Marketing')
GROUP BY d.dept_id, d.dept_name
HAVING COUNT(DISTINCT p.category) > 3
ORDER BY product_categories_handled DESC;
```

**ข้อ 10:** สร้าง Complete KPI Dashboard Query

```sql
-- เฉลย: KPI ทั้งหมดใน 1 query ด้วย UNION ALL
SELECT 'Total Revenue (Completed)' AS kpi, 
       TO_CHAR(SUM(o.total_amount), 'FM999,999,999.00') AS value
FROM orders o WHERE o.status = 'completed'

UNION ALL

SELECT 'Total Orders', COUNT(*)::TEXT
FROM orders WHERE status IN ('completed', 'shipped')

UNION ALL

SELECT 'Active Customers', COUNT(DISTINCT customer_id)::TEXT
FROM orders WHERE status = 'completed'

UNION ALL

SELECT 'Total Products', COUNT(*)::TEXT
FROM products

UNION ALL

SELECT 'Avg Order Value', TO_CHAR(AVG(total_amount), 'FM999,999.00')
FROM orders WHERE status = 'completed'

UNION ALL

SELECT 'Total Employees', COUNT(*)::TEXT
FROM employees

UNION ALL

SELECT 'Departments', COUNT(*)::TEXT
FROM departments

UNION ALL

SELECT 'Best Selling Product', p.product_name
FROM products p
JOIN order_items oi ON p.product_id = oi.product_id
JOIN orders o ON oi.order_id = o.order_id
WHERE o.status = 'completed'
GROUP BY p.product_id, p.product_name
ORDER BY SUM(oi.quantity) DESC
LIMIT 1;
```

---

## สรุปภาค 27

1. **Multi-Table JOIN** — JOIN ทีละตารางต่อกันได้ไม่จำกัด
2. **Meaningful Aliases** — ใช้ชื่อที่มีความหมาย (e, d, o, c, p, oi, m)
3. **Column Ambiguity** — ระบุ table prefix เสมอสำหรับ columns ที่อาจ ambiguous
4. **Query Optimizer** — SQL engine จัดการลำดับ JOIN เอง
5. **Readability** — เรียง JOIN ให้อ่านเหมือนเล่าเรื่อง: ตารางหลัก → details
6. **Aggregation** — GROUP BY + HAVING ใช้ได้ปกติกับ multi-table JOIN

**ในภาคถัดไป** จะเรียน Advanced JOIN Techniques รวมถึง Non-equi JOIN และ LATERAL JOIN!
