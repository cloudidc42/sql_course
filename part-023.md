# ภาค 23: LEFT JOIN และ RIGHT JOIN

## ทบทวน INNER JOIN vs LEFT JOIN

ก่อนเรียน LEFT JOIN มาดูความแตกต่างกับ INNER JOIN ก่อน:

```
ข้อมูลตัวอย่าง:

departments:           employees:
┌─────┬────────────┐   ┌─────┬──────────┬─────────┐
│ id  │ name       │   │ id  │ name     │ dept_id │
├─────┼────────────┤   ├─────┼──────────┼─────────┤
│ 1   │ Engineering│   │ 1   │ Somchai  │ 10      │
│ 2   │ Marketing  │   │ 2   │ Wanchai  │ 10      │
│ 10  │ Executive  │   │ 4   │ Narong   │ 1       │
│ 99  │ Ghost Dept │← ไม่มีพนักงาน
└─────┴────────────┘   └─────┴──────────┴─────────┘

INNER JOIN result:          LEFT JOIN (dept LEFT JOIN emp):
┌────────────┬──────────┐   ┌────────────┬──────────┐
│ name       │ emp_name │   │ name       │ emp_name │
├────────────┼──────────┤   ├────────────┼──────────┤
│ Engineering│ Narong   │   │ Engineering│ Narong   │
│ Executive  │ Somchai  │   │ Executive  │ Somchai  │
│ Executive  │ Wanchai  │   │ Executive  │ Wanchai  │
└────────────┴──────────┘   │ Marketing  │ NULL     │ ← คงไว้!
Ghost Dept หายไป!           │ Ghost Dept │ NULL     │ ← คงไว้!
Marketing หายไป!            └────────────┴──────────┘
```

---

## LEFT JOIN (LEFT OUTER JOIN)

**LEFT JOIN** คือการรวมตารางที่:
- เก็บ **ทุกแถว** จากตาราง **ซ้าย** (LEFT table)
- ถ้าตาราง **ขวา** ไม่มีข้อมูลตรง → แสดง **NULL**

```
แผนภาพ Venn Diagram:

Table A          Table B
  ┌─────────────────────────┐
  │ ╔═══════════════╗       │
  │ ║  A all + A∩B  ║  B    │
  │ ║ (LEFT JOIN)   ║ only  │
  │ ╚═══════════════╝       │
  └─────────────────────────┘

LEFT JOIN = ทั้งหมดจาก A + ส่วนที่ match กับ B
```

### Syntax

```sql
SELECT columns
FROM left_table
LEFT JOIN right_table ON left_table.key = right_table.key;

-- หรือ LEFT OUTER JOIN (เหมือนกัน)
SELECT columns
FROM left_table
LEFT OUTER JOIN right_table ON left_table.key = right_table.key;
```

---

## ตัวอย่าง LEFT JOIN พื้นฐาน (1-10)

### ตัวอย่างที่ 1: แสดงแผนกทั้งหมดรวมถึงที่ไม่มีพนักงาน

```sql
-- INNER JOIN: แสดงเฉพาะแผนกที่มีพนักงาน
SELECT d.dept_name, COUNT(e.emp_id) AS emp_count
FROM departments d
INNER JOIN employees e ON d.dept_id = e.dept_id
GROUP BY d.dept_name;

-- LEFT JOIN: แสดงทุกแผนก รวมถึงที่ไม่มีพนักงาน
SELECT 
    d.dept_id,
    d.dept_name,
    d.location,
    COUNT(e.emp_id) AS emp_count  -- จะเป็น 0 ถ้าไม่มีพนักงาน
FROM departments d
LEFT JOIN employees e ON d.dept_id = e.dept_id
GROUP BY d.dept_id, d.dept_name, d.location
ORDER BY emp_count DESC;
```

### ตัวอย่างที่ 2: ลูกค้าทั้งหมดพร้อม orders (รวมที่ยังไม่เคยสั่ง)

```sql
-- แสดงลูกค้าทุกคน รวมที่ยังไม่เคยสั่งสินค้า
SELECT 
    c.customer_id,
    CONCAT(c.first_name, ' ', c.last_name) AS customer_name,
    c.city,
    c.created_at::DATE AS joined_date,
    COUNT(o.order_id) AS order_count,
    COALESCE(SUM(o.total_amount), 0) AS total_spent,
    CASE 
        WHEN COUNT(o.order_id) = 0 THEN 'Never Ordered'
        ELSE 'Has Orders'
    END AS customer_status
FROM customers c
LEFT JOIN orders o ON c.customer_id = o.customer_id
GROUP BY c.customer_id, customer_name, c.city, c.created_at
ORDER BY order_count DESC, customer_name;
```

### ตัวอย่างที่ 3: Products ที่ยังไม่เคยถูกสั่งซื้อ

```sql
SELECT 
    p.product_id,
    p.product_name,
    p.category,
    p.price,
    p.stock_quantity,
    oi.order_id  -- จะเป็น NULL ถ้าไม่เคยถูกสั่ง
FROM products p
LEFT JOIN order_items oi ON p.product_id = oi.product_id
ORDER BY oi.order_id NULLS FIRST;

-- เห็น NULL ในคอลัมน์ order_id = products ที่ไม่เคยถูกสั่ง
```

### ตัวอย่างที่ 4: COALESCE กับ LEFT JOIN

```sql
-- แทน NULL ด้วยค่าที่มีความหมาย
SELECT 
    e.emp_id,
    CONCAT(e.first_name, ' ', e.last_name) AS employee,
    e.salary,
    COALESCE(d.dept_name, 'No Department') AS department,
    COALESCE(d.location, 'Unknown') AS location
FROM employees e
LEFT JOIN departments d ON e.dept_id = d.dept_id
ORDER BY e.dept_id NULLS LAST;
```

### ตัวอย่างที่ 5: LEFT JOIN กับ WHERE

```sql
-- แสดงเฉพาะพนักงานที่มีเงินเดือนสูงในแผนกที่มีพนักงาน
SELECT 
    e.first_name,
    e.last_name,
    e.salary,
    d.dept_name
FROM employees e
LEFT JOIN departments d ON e.dept_id = d.dept_id
WHERE e.salary > 150000  -- filter หลัง join
ORDER BY e.salary DESC;
```

### ตัวอย่างที่ 6: LEFT JOIN กับ Aggregate

```sql
-- สถิติ order ต่อลูกค้า
SELECT 
    c.customer_id,
    c.first_name || ' ' || c.last_name AS customer,
    c.city,
    COUNT(o.order_id) AS total_orders,
    COUNT(CASE WHEN o.status = 'completed' THEN 1 END) AS completed,
    COUNT(CASE WHEN o.status = 'pending' THEN 1 END) AS pending,
    COUNT(CASE WHEN o.status = 'cancelled' THEN 1 END) AS cancelled,
    COALESCE(SUM(o.total_amount), 0) AS lifetime_value
FROM customers c
LEFT JOIN orders o ON c.customer_id = o.customer_id
GROUP BY c.customer_id, customer, c.city
ORDER BY total_orders DESC, customer;
```

### ตัวอย่างที่ 7: NULL ใน LEFT JOIN — เข้าใจอย่างถูกต้อง

```sql
-- เมื่อ NULL มาจาก LEFT JOIN
SELECT 
    c.customer_id,
    c.first_name,
    o.order_id,
    o.total_amount
FROM customers c
LEFT JOIN orders o ON c.customer_id = o.customer_id
WHERE c.customer_id BETWEEN 55 AND 65;

-- บาง customer จะมี o.order_id = NULL
-- เพราะพวกเขายังไม่มี order

-- NULL ไม่ใช่ error — มันแปลว่า "ไม่มีข้อมูลที่ match"
```

### ตัวอย่างที่ 8: LEFT JOIN หลายตาราง

```sql
-- ลูกค้า → orders → order_items (เก็บลูกค้าที่ไม่มี order ด้วย)
SELECT 
    c.customer_id,
    c.first_name || ' ' || c.last_name AS customer,
    COUNT(DISTINCT o.order_id) AS orders,
    COUNT(oi.item_id) AS total_items,
    COALESCE(SUM(oi.quantity * oi.unit_price - oi.discount), 0) AS total_revenue
FROM customers c
LEFT JOIN orders o ON c.customer_id = o.customer_id
LEFT JOIN order_items oi ON o.order_id = oi.order_id
GROUP BY c.customer_id, customer
ORDER BY total_revenue DESC;
```

### ตัวอย่างที่ 9: สำคัญ! ON vs WHERE ใน LEFT JOIN

```sql
-- กรณี 1: Condition ใน ON → เก็บ LEFT rows ทั้งหมด, filter เฉพาะ RIGHT side
SELECT 
    c.customer_id,
    c.first_name,
    o.order_id,
    o.status
FROM customers c
LEFT JOIN orders o ON c.customer_id = o.customer_id 
                  AND o.status = 'completed';  -- filter ใน ON
-- ผล: ลูกค้าทุกคน, แต่แสดง order เฉพาะที่ completed (ถ้าไม่มี completed → NULL)

-- กรณี 2: Condition ใน WHERE → filter หลัง join (กลายเป็นเหมือน INNER JOIN!)
SELECT 
    c.customer_id,
    c.first_name,
    o.order_id,
    o.status
FROM customers c
LEFT JOIN orders o ON c.customer_id = o.customer_id
WHERE o.status = 'completed';  -- filter ใน WHERE
-- ผล: แสดงเฉพาะลูกค้าที่มี completed orders! (ลูกค้าที่ไม่มี order หายไป)
```

### ตัวอย่างที่ 10: Comparing ON vs WHERE ชัดเจน

```sql
-- ON: แสดงลูกค้าทุกคน + orders ที่ completed
-- ลูกค้าที่ไม่มี completed order จะยังปรากฏ แต่มี NULL ใน order columns
SELECT 
    c.customer_id,
    c.first_name,
    COUNT(o.order_id) AS completed_order_count
FROM customers c
LEFT JOIN orders o ON c.customer_id = o.customer_id 
                  AND o.status = 'completed'
GROUP BY c.customer_id, c.first_name
ORDER BY completed_order_count DESC;

-- WHERE: แสดงเฉพาะลูกค้าที่มี completed order
-- ลูกค้าที่ไม่มี completed order จะหายไปเลย
SELECT 
    c.customer_id,
    c.first_name,
    COUNT(o.order_id) AS completed_order_count
FROM customers c
LEFT JOIN orders o ON c.customer_id = o.customer_id
WHERE o.status = 'completed'  -- ทำให้กลายเป็น INNER JOIN!
GROUP BY c.customer_id, c.first_name
ORDER BY completed_order_count DESC;
```

---

## Anti-Join Pattern — หา "ที่ไม่มี" ด้วย LEFT JOIN

**Anti-Join** คือ pattern ที่ใช้ LEFT JOIN + `WHERE right.id IS NULL` เพื่อหาแถวในตาราง LEFT ที่ **ไม่มีคู่** ในตาราง RIGHT

```
Anti-Join Diagram:

Table A          Table B
  ┌─────────────────────────┐
  │ ╔═════════╗             │
  │ ║ A only  ║   A∩B  B   │
  │ ║(Anti-   ║             │
  │ ║ Join)   ║             │
  │ ╚═════════╝             │
  └─────────────────────────┘
Anti-Join = เฉพาะส่วน A ที่ไม่ match กับ B
```

### ตัวอย่างที่ 11: ลูกค้าที่ยังไม่เคยสั่งสินค้า

```sql
-- Anti-Join: ลูกค้าที่ไม่มี order เลย
SELECT 
    c.customer_id,
    c.first_name || ' ' || c.last_name AS customer_name,
    c.email,
    c.city,
    c.created_at::DATE AS joined_date
FROM customers c
LEFT JOIN orders o ON c.customer_id = o.customer_id
WHERE o.order_id IS NULL  -- Anti-Join condition!
ORDER BY c.created_at DESC;

-- ผล: ลูกค้าที่ไม่มี order แม้แต่รายการเดียว
```

### ตัวอย่างที่ 12: Products ที่ไม่เคยถูกสั่ง

```sql
-- สินค้าที่ไม่มีใน order_items เลย
SELECT 
    p.product_id,
    p.product_name,
    p.category,
    p.price,
    p.stock_quantity
FROM products p
LEFT JOIN order_items oi ON p.product_id = oi.product_id
WHERE oi.item_id IS NULL  -- ไม่เคยถูกสั่ง
ORDER BY p.category, p.product_name;
```

### ตัวอย่างที่ 13: แผนกที่ไม่มีพนักงาน

```sql
-- แผนกว่าง
SELECT 
    d.dept_id,
    d.dept_name,
    d.location,
    d.budget
FROM departments d
LEFT JOIN employees e ON d.dept_id = e.dept_id
WHERE e.emp_id IS NULL  -- ไม่มีพนักงาน
ORDER BY d.dept_id;
```

### ตัวอย่างที่ 14: Anti-Join เทียบกับ NOT IN และ NOT EXISTS

```sql
-- 3 วิธีที่ให้ผลเหมือนกัน: หา customers ที่ไม่มี order

-- วิธี 1: LEFT JOIN + IS NULL (Anti-Join) — เร็วที่สุดส่วนใหญ่
SELECT c.customer_id, c.first_name
FROM customers c
LEFT JOIN orders o ON c.customer_id = o.customer_id
WHERE o.customer_id IS NULL;

-- วิธี 2: NOT IN — ระวัง NULL! ถ้า subquery มี NULL จะได้ผลผิด
SELECT c.customer_id, c.first_name
FROM customers c
WHERE c.customer_id NOT IN (SELECT customer_id FROM orders WHERE customer_id IS NOT NULL);

-- วิธี 3: NOT EXISTS — ปลอดภัยกว่า NOT IN แต่อาจช้ากว่า
SELECT c.customer_id, c.first_name
FROM customers c
WHERE NOT EXISTS (
    SELECT 1 FROM orders o WHERE o.customer_id = c.customer_id
);

-- เปรียบเทียบ performance:
-- LEFT JOIN + IS NULL: ดีที่สุดสำหรับข้อมูลใหญ่
-- NOT EXISTS: ดีเมื่อมี index ที่เหมาะสม
-- NOT IN: หลีกเลี่ยงถ้า subquery อาจมี NULL
```

### ตัวอย่างที่ 15: Anti-Join สำหรับ Data Reconciliation

```sql
-- หา order_items ที่ไม่มี order (orphan records)
SELECT oi.*
FROM order_items oi
LEFT JOIN orders o ON oi.order_id = o.order_id
WHERE o.order_id IS NULL;  -- ควรไม่มีผลถ้า FK ทำงานถูกต้อง

-- หา orders ที่ไม่มี order_items
SELECT o.order_id, o.order_date, o.total_amount, o.status
FROM orders o
LEFT JOIN order_items oi ON o.order_id = oi.order_id
WHERE oi.item_id IS NULL;
```

---

## LEFT JOIN กับ Aggregate Functions (รวม NULL)

### ตัวอย่างที่ 16: COUNT กับ NULL ใน LEFT JOIN

```sql
-- สำคัญ! COUNT(*) vs COUNT(column) กับ NULL

SELECT 
    d.dept_name,
    COUNT(*) AS count_star,        -- นับทุกแถว (รวม NULL)
    COUNT(e.emp_id) AS count_emp,  -- นับเฉพาะ non-NULL
    COALESCE(COUNT(e.emp_id), 0) AS emp_count  -- ปลอดภัยกว่า
FROM departments d
LEFT JOIN employees e ON d.dept_id = e.dept_id
GROUP BY d.dept_id, d.dept_name;

-- ผล:
-- dept ที่มีพนักงาน: count_star = count_emp (เท่ากัน)
-- dept ที่ไม่มีพนักงาน: count_star = 1, count_emp = 0 (ต่างกัน!)
```

### ตัวอย่างที่ 17: SUM + COALESCE กับ NULL

```sql
-- SUM กับ NULL → SUM คืน NULL ถ้าทุกค่าเป็น NULL
SELECT 
    d.dept_name,
    SUM(e.salary) AS total_salary,           -- NULL ถ้าไม่มีพนักงาน
    COALESCE(SUM(e.salary), 0) AS total_salary_safe  -- 0 ถ้าไม่มีพนักงาน
FROM departments d
LEFT JOIN employees e ON d.dept_id = e.dept_id
GROUP BY d.dept_id, d.dept_name
ORDER BY total_salary_safe DESC;
```

### ตัวอย่างที่ 18: AVG + NULL

```sql
-- AVG ไม่นับ NULL (ผลถูกต้องเอง)
SELECT 
    d.dept_name,
    COUNT(e.emp_id) AS employee_count,
    AVG(e.salary) AS avg_salary,  -- NULL ถ้าไม่มีพนักงาน
    COALESCE(AVG(e.salary), 0) AS avg_salary_safe
FROM departments d
LEFT JOIN employees e ON d.dept_id = e.dept_id
GROUP BY d.dept_id, d.dept_name;
```

---

## ตัวอย่าง LEFT JOIN ขั้นสูง (19-30)

### ตัวอย่างที่ 19: Multiple LEFT JOINs

```sql
-- ข้อมูลพนักงานครบถ้วน ทั้งที่มีและไม่มี department/manager
SELECT 
    e.emp_id,
    CONCAT(e.first_name, ' ', e.last_name) AS employee,
    e.job_title,
    COALESCE(d.dept_name, 'No Dept') AS department,
    COALESCE(d.location, 'Unknown') AS location,
    COALESCE(
        CONCAT(m.first_name, ' ', m.last_name), 
        'No Manager'
    ) AS manager
FROM employees e
LEFT JOIN departments d ON e.dept_id = d.dept_id
LEFT JOIN employees m ON e.manager_id = m.emp_id
ORDER BY d.dept_name NULLS LAST, e.emp_id;
```

### ตัวอย่างที่ 20: Customer Engagement Analysis

```sql
-- วิเคราะห์ลูกค้าแบบละเอียด (รวมคนที่ไม่เคยซื้อ)
SELECT 
    c.customer_id,
    c.first_name || ' ' || c.last_name AS customer,
    c.city,
    c.country,
    c.created_at::DATE AS registered,
    COUNT(o.order_id) AS total_orders,
    COUNT(DISTINCT CASE WHEN o.status = 'completed' THEN o.order_id END) AS completed,
    COUNT(DISTINCT CASE WHEN o.status = 'cancelled' THEN o.order_id END) AS cancelled,
    COALESCE(SUM(CASE WHEN o.status = 'completed' THEN o.total_amount END), 0) AS spent,
    CASE 
        WHEN COUNT(o.order_id) = 0 THEN 'New'
        WHEN SUM(CASE WHEN o.status = 'completed' THEN o.total_amount ELSE 0 END) > 50000 THEN 'VIP'
        WHEN SUM(CASE WHEN o.status = 'completed' THEN o.total_amount ELSE 0 END) > 20000 THEN 'Regular'
        ELSE 'Occasional'
    END AS customer_tier
FROM customers c
LEFT JOIN orders o ON c.customer_id = o.customer_id
GROUP BY c.customer_id, customer, c.city, c.country, c.created_at
ORDER BY spent DESC;
```

### ตัวอย่างที่ 21: Inventory Report พร้อมข้อมูลการขาย

```sql
-- ทุก product + ยอดขาย (รวมที่ไม่เคยขาย)
SELECT 
    p.product_id,
    p.product_name,
    p.category,
    p.price,
    p.stock_quantity,
    COALESCE(COUNT(oi.item_id), 0) AS times_ordered,
    COALESCE(SUM(oi.quantity), 0) AS total_sold,
    p.stock_quantity + COALESCE(SUM(oi.quantity), 0) AS original_stock_estimate,
    CASE 
        WHEN COUNT(oi.item_id) = 0 THEN 'Never Sold'
        WHEN p.stock_quantity < 20 THEN 'Low Stock'
        WHEN p.stock_quantity = 0 THEN 'Out of Stock'
        ELSE 'In Stock'
    END AS inventory_status
FROM products p
LEFT JOIN order_items oi ON p.product_id = oi.product_id
GROUP BY p.product_id, p.product_name, p.category, p.price, p.stock_quantity
ORDER BY times_ordered ASC, p.category;
```

### ตัวอย่างที่ 22: Employee Department Summary (แม้แผนกว่าง)

```sql
-- รายงานแผนกครบถ้วน
SELECT 
    d.dept_id,
    d.dept_name,
    d.location,
    d.budget,
    COUNT(e.emp_id) AS headcount,
    COALESCE(SUM(e.salary), 0) AS monthly_payroll,
    COALESCE(SUM(e.salary) * 12, 0) AS annual_payroll,
    d.budget - COALESCE(SUM(e.salary) * 12, 0) AS remaining_budget,
    CASE 
        WHEN COUNT(e.emp_id) = 0 THEN 'Empty Department'
        WHEN SUM(e.salary) * 12 > d.budget THEN 'Over Budget!'
        WHEN SUM(e.salary) * 12 > d.budget * 0.8 THEN 'Near Limit'
        ELSE 'OK'
    END AS budget_status
FROM departments d
LEFT JOIN employees e ON d.dept_id = e.dept_id
GROUP BY d.dept_id, d.dept_name, d.location, d.budget
ORDER BY headcount DESC;
```

### ตัวอย่างที่ 23: หา "Gap" ในข้อมูล

```sql
-- หาช่วงวันที่ไม่มี order (ใช้ generate_series + LEFT JOIN)
-- PostgreSQL specific
SELECT 
    date_series.dt AS date,
    COUNT(o.order_id) AS order_count,
    COALESCE(SUM(o.total_amount), 0) AS daily_revenue
FROM generate_series(
    '2024-01-01'::DATE,
    '2024-09-30'::DATE,
    '1 day'::INTERVAL
) AS date_series(dt)
LEFT JOIN orders o ON o.order_date = date_series.dt
GROUP BY date_series.dt
ORDER BY date_series.dt;
-- วันที่ไม่มี order จะมี order_count = 0
```

### ตัวอย่างที่ 24: Hierarchical Data ด้วย LEFT JOIN

```sql
-- แสดง hierarchy ของพนักงานทุกระดับ
SELECT 
    e.emp_id,
    CONCAT(e.first_name, ' ', e.last_name) AS employee,
    e.job_title,
    COALESCE(CONCAT(m.first_name, ' ', m.last_name), 'Top Level') AS manager,
    COALESCE(m.job_title, 'No Manager') AS manager_title,
    CASE 
        WHEN e.manager_id IS NULL THEN 0
        WHEN m.manager_id IS NULL THEN 1
        ELSE 2
    END AS hierarchy_level
FROM employees e
LEFT JOIN employees m ON e.manager_id = m.emp_id
ORDER BY hierarchy_level, e.dept_id, e.emp_id;
```

### ตัวอย่างที่ 25: Sales Funnel Analysis

```sql
-- วิเคราะห์ funnel: ลงทะเบียน → สั่งซื้อ → ชำระเงิน → ส่งของ
SELECT 
    COUNT(DISTINCT c.customer_id) AS registered_customers,
    COUNT(DISTINCT o.customer_id) AS customers_with_orders,
    COUNT(DISTINCT CASE WHEN o.status != 'cancelled' THEN o.customer_id END) AS paying_customers,
    COUNT(DISTINCT CASE WHEN o.status = 'completed' THEN o.customer_id END) AS completed_customers,
    ROUND(
        COUNT(DISTINCT o.customer_id)::NUMERIC / COUNT(DISTINCT c.customer_id) * 100, 1
    ) AS order_conversion_pct,
    ROUND(
        COUNT(DISTINCT CASE WHEN o.status = 'completed' THEN o.customer_id END)::NUMERIC / 
        COUNT(DISTINCT c.customer_id) * 100, 1
    ) AS completion_rate_pct
FROM customers c
LEFT JOIN orders o ON c.customer_id = o.customer_id;
```

---

## RIGHT JOIN

**RIGHT JOIN** (RIGHT OUTER JOIN) คือ LEFT JOIN กลับหัว:
- เก็บ **ทุกแถว** จากตาราง **ขวา** (RIGHT table)
- ถ้าตาราง **ซ้าย** ไม่มีข้อมูลตรง → แสดง **NULL**

```
แผนภาพ Venn Diagram:

Table A          Table B
  ┌─────────────────────────┐
  │       ╔═══════════════╗ │
  │  A    ║  A∩B + B all  ║ │
  │ only  ║  (RIGHT JOIN) ║ │
  │       ╚═══════════════╝ │
  └─────────────────────────┘
```

### Syntax

```sql
SELECT columns
FROM left_table
RIGHT JOIN right_table ON left_table.key = right_table.key;
```

### ตัวอย่างที่ 26: RIGHT JOIN พื้นฐาน

```sql
-- แสดงพนักงานทั้งหมด และแผนกที่ตรงกัน (รวมแผนกว่าง)
-- (ผลเหมือนกับ departments LEFT JOIN employees)
SELECT 
    COALESCE(e.first_name || ' ' || e.last_name, 'No Employee') AS employee,
    d.dept_name,
    d.location
FROM employees e
RIGHT JOIN departments d ON e.dept_id = d.dept_id;

-- เทียบเท่ากับ:
SELECT 
    COALESCE(e.first_name || ' ' || e.last_name, 'No Employee') AS employee,
    d.dept_name,
    d.location
FROM departments d
LEFT JOIN employees e ON d.dept_id = e.dept_id;
```

### ตัวอย่างที่ 27: แปลง RIGHT JOIN เป็น LEFT JOIN

```sql
-- RIGHT JOIN (อ่านยาก)
SELECT c.first_name, o.order_id
FROM orders o
RIGHT JOIN customers c ON o.customer_id = c.customer_id;

-- แปลงเป็น LEFT JOIN (อ่านง่ายกว่า)
SELECT c.first_name, o.order_id
FROM customers c
LEFT JOIN orders o ON c.customer_id = o.customer_id;

-- ทั้งสองให้ผลเหมือนกัน!
-- แนะนำให้ใช้ LEFT JOIN เสมอ เพื่อความสม่ำเสมอ
```

### ตัวอย่างที่ 28: เมื่อ RIGHT JOIN มีประโยชน์

```sql
-- สมมติต้องการ report ที่ products เป็นหลัก
-- แสดง products ทั้งหมด แม้ไม่มีใน order_items
SELECT 
    p.product_name,
    p.category,
    COALESCE(oi.order_id, 0) AS order_id,
    COALESCE(oi.quantity, 0) AS qty
FROM order_items oi
RIGHT JOIN products p ON oi.product_id = p.product_id
ORDER BY p.category, p.product_name;

-- เทียบเท่า (Left JOIN แนะนำกว่า):
SELECT 
    p.product_name,
    p.category,
    COALESCE(oi.order_id, 0) AS order_id,
    COALESCE(oi.quantity, 0) AS qty
FROM products p
LEFT JOIN order_items oi ON p.product_id = oi.product_id
ORDER BY p.category, p.product_name;
```

---

## Anti-Join Pattern ขั้นสูง

### ตัวอย่างที่ 29: Double Anti-Join

```sql
-- หาพนักงานที่ไม่มีทั้ง manager และ department
SELECT 
    e.emp_id,
    e.first_name,
    e.last_name,
    e.email
FROM employees e
LEFT JOIN departments d ON e.dept_id = d.dept_id
LEFT JOIN employees m ON e.manager_id = m.emp_id
WHERE d.dept_id IS NULL 
   OR e.manager_id IS NULL;
```

### ตัวอย่างที่ 30: Customers ที่ไม่ได้ซื้อใน 30 วันที่ผ่านมา

```sql
-- Re-engagement targets: ลูกค้าที่เคยซื้อแต่ไม่ได้ซื้อนาน
SELECT 
    c.customer_id,
    c.first_name || ' ' || c.last_name AS customer,
    c.email,
    c.city,
    MAX(o.order_date) AS last_order_date,
    CURRENT_DATE - MAX(o.order_date) AS days_since_last_order,
    COUNT(o.order_id) AS total_orders_ever
FROM customers c
JOIN orders o ON c.customer_id = o.customer_id  -- INNER: เฉพาะที่เคยซื้อ
WHERE o.status = 'completed'
GROUP BY c.customer_id, customer, c.email, c.city
HAVING MAX(o.order_date) < CURRENT_DATE - INTERVAL '30 days'
ORDER BY last_order_date ASC;
```

### ตัวอย่างที่ 31: Product นอก order_items ใน range ราคา

```sql
-- สินค้าราคา 1000-10000 ที่ไม่เคยขายในปี 2024
SELECT 
    p.product_id,
    p.product_name,
    p.category,
    p.price,
    p.stock_quantity
FROM products p
LEFT JOIN order_items oi ON p.product_id = oi.product_id
LEFT JOIN orders o ON oi.order_id = o.order_id 
                  AND EXTRACT(YEAR FROM o.order_date) = 2024
WHERE p.price BETWEEN 1000 AND 10000
  AND o.order_id IS NULL  -- Anti-Join
ORDER BY p.price DESC;
```

### ตัวอย่างที่ 32: Conditional Anti-Join

```sql
-- ลูกค้าที่มี order แต่ไม่มี completed order
SELECT 
    c.customer_id,
    c.first_name || ' ' || c.last_name AS customer,
    COUNT(all_orders.order_id) AS total_orders,
    COUNT(completed.order_id) AS completed_orders
FROM customers c
LEFT JOIN orders all_orders ON c.customer_id = all_orders.customer_id
LEFT JOIN orders completed ON c.customer_id = completed.customer_id 
                           AND completed.status = 'completed'
WHERE all_orders.order_id IS NOT NULL   -- มี orders
  AND completed.order_id IS NULL        -- แต่ไม่มี completed
GROUP BY c.customer_id, customer
ORDER BY total_orders DESC;
```

---

## ตัวอย่างธุรกิจ LEFT JOIN ขั้นสูง (33-35)

### ตัวอย่างที่ 33: Full Sales Report with Gaps

```sql
-- รายงานการขายครบถ้วนรวม products ที่ไม่ขาย
SELECT 
    p.category,
    p.product_name,
    p.price,
    COUNT(oi.item_id) AS order_count,
    COALESCE(SUM(oi.quantity), 0) AS units_sold,
    COALESCE(SUM(oi.quantity * oi.unit_price - oi.discount), 0) AS revenue,
    p.stock_quantity AS current_stock,
    CASE 
        WHEN COUNT(oi.item_id) = 0 THEN 'No Sales'
        WHEN SUM(oi.quantity) < 5 THEN 'Slow Mover'
        WHEN SUM(oi.quantity) < 20 THEN 'Average'
        ELSE 'Fast Mover'
    END AS sales_velocity
FROM products p
LEFT JOIN order_items oi ON p.product_id = oi.product_id
LEFT JOIN orders o ON oi.order_id = o.order_id 
                  AND o.status IN ('completed', 'shipped')
GROUP BY p.product_id, p.category, p.product_name, p.price, p.stock_quantity
ORDER BY revenue DESC, p.category;
```

### ตัวอย่างที่ 34: Employee-Manager Gap Analysis

```sql
-- หาพนักงานที่ยังไม่มีผู้จัดการ (นอกจาก CEO)
SELECT 
    e.emp_id,
    e.first_name || ' ' || e.last_name AS employee,
    e.job_title,
    d.dept_name,
    CASE 
        WHEN e.manager_id IS NULL AND e.job_title LIKE '%CEO%' THEN 'CEO - Top Level'
        WHEN e.manager_id IS NULL THEN 'NO MANAGER ASSIGNED!'
        ELSE CONCAT(m.first_name, ' ', m.last_name)
    END AS manager_status
FROM employees e
LEFT JOIN employees m ON e.manager_id = m.emp_id
LEFT JOIN departments d ON e.dept_id = d.dept_id
ORDER BY e.manager_id NULLS FIRST;
```

### ตัวอย่างที่ 35: Complete Order Lifecycle Report

```sql
-- รายงาน lifecycle ของ order ครบถ้วน
SELECT 
    o.order_id,
    c.first_name || ' ' || c.last_name AS customer,
    c.city,
    o.order_date,
    o.status,
    o.shipped_date,
    CASE 
        WHEN o.shipped_date IS NOT NULL 
        THEN o.shipped_date - o.order_date 
        ELSE NULL
    END AS days_to_ship,
    COUNT(oi.item_id) AS item_count,
    COALESCE(SUM(oi.quantity), 0) AS total_qty,
    o.total_amount,
    CASE 
        WHEN o.shipped_date IS NULL AND o.status NOT IN ('cancelled', 'pending') 
        THEN 'Needs Attention'
        ELSE 'OK'
    END AS fulfillment_alert
FROM orders o
LEFT JOIN customers c ON o.customer_id = c.customer_id
LEFT JOIN order_items oi ON o.order_id = oi.order_id
GROUP BY o.order_id, customer, c.city, o.order_date, o.status, 
         o.shipped_date, o.total_amount
ORDER BY o.order_date;
```

---

## แบบฝึกหัดภาค 23

**ข้อ 1:** แสดงลูกค้าทุกคนพร้อมจำนวน orders รวมทั้งที่ไม่มี order

```sql
-- เฉลย
SELECT 
    c.customer_id,
    c.first_name || ' ' || c.last_name AS customer,
    c.city,
    COUNT(o.order_id) AS order_count
FROM customers c
LEFT JOIN orders o ON c.customer_id = o.customer_id
GROUP BY c.customer_id, customer, c.city
ORDER BY order_count DESC, customer;
```

**ข้อ 2:** หาลูกค้าที่ไม่เคยสั่งสินค้าเลย

```sql
-- เฉลย
SELECT 
    c.customer_id,
    c.first_name || ' ' || c.last_name AS customer,
    c.email,
    c.created_at::DATE AS registered
FROM customers c
LEFT JOIN orders o ON c.customer_id = o.customer_id
WHERE o.order_id IS NULL
ORDER BY c.created_at DESC;
```

**ข้อ 3:** หาสินค้าที่ไม่เคยถูกสั่งซื้อเลย

```sql
-- เฉลย
SELECT 
    p.product_id,
    p.product_name,
    p.category,
    p.price,
    p.stock_quantity
FROM products p
LEFT JOIN order_items oi ON p.product_id = oi.product_id
WHERE oi.item_id IS NULL
ORDER BY p.category, p.price DESC;
```

**ข้อ 4:** แสดงทุกแผนกพร้อมจำนวนพนักงาน รวมแผนกที่ว่าง

```sql
-- เฉลย
SELECT 
    d.dept_id,
    d.dept_name,
    d.budget,
    COUNT(e.emp_id) AS headcount,
    COALESCE(SUM(e.salary), 0) AS monthly_payroll,
    CASE 
        WHEN COUNT(e.emp_id) = 0 THEN 'EMPTY'
        ELSE CAST(COUNT(e.emp_id) AS VARCHAR)
    END AS status
FROM departments d
LEFT JOIN employees e ON d.dept_id = e.dept_id
GROUP BY d.dept_id, d.dept_name, d.budget
ORDER BY headcount DESC;
```

**ข้อ 5:** แปลง RIGHT JOIN ต่อไปนี้เป็น LEFT JOIN

```sql
-- ต้นฉบับ (RIGHT JOIN)
SELECT p.product_name, oi.quantity, oi.unit_price
FROM order_items oi
RIGHT JOIN products p ON oi.product_id = p.product_id;

-- เฉลย (LEFT JOIN)
SELECT p.product_name, oi.quantity, oi.unit_price
FROM products p
LEFT JOIN order_items oi ON p.product_id = oi.product_id;
```

**ข้อ 6:** อธิบายความแตกต่างระหว่าง ON กับ WHERE ใน LEFT JOIN พร้อมตัวอย่าง

```sql
-- เฉลย: ตัวอย่างความแตกต่าง
-- ON: filter ฝั่ง RIGHT ก่อน join → เก็บ LEFT ไว้ทั้งหมด
SELECT c.customer_id, c.first_name, o.order_id
FROM customers c
LEFT JOIN orders o ON c.customer_id = o.customer_id
    AND o.status = 'completed';
-- ลูกค้าทุกคนปรากฏ แต่แสดงเฉพาะ completed orders (หรือ NULL)

-- WHERE: filter หลัง join → กรอง NULL ออก = เหมือน INNER JOIN
SELECT c.customer_id, c.first_name, o.order_id
FROM customers c
LEFT JOIN orders o ON c.customer_id = o.customer_id
WHERE o.status = 'completed';
-- เฉพาะลูกค้าที่มี completed orders เท่านั้น
```

**ข้อ 7:** หาลูกค้าที่ยังไม่ได้สั่งซื้อในปี 2024

```sql
-- เฉลย
SELECT 
    c.customer_id,
    c.first_name || ' ' || c.last_name AS customer,
    c.email,
    c.city
FROM customers c
LEFT JOIN orders o ON c.customer_id = o.customer_id
    AND EXTRACT(YEAR FROM o.order_date) = 2024  -- filter ใน ON!
WHERE o.order_id IS NULL  -- ไม่มี order ในปี 2024
ORDER BY c.customer_id;
```

**ข้อ 8:** รายงานสินค้าทุกชิ้นพร้อมยอดขาย (ราคาต่อหน่วย, จำนวน, รายได้)

```sql
-- เฉลย
SELECT 
    p.product_id,
    p.product_name,
    p.category,
    p.price,
    COALESCE(COUNT(oi.item_id), 0) AS times_ordered,
    COALESCE(SUM(oi.quantity), 0) AS total_units_sold,
    COALESCE(SUM(oi.quantity * oi.unit_price - oi.discount), 0) AS total_revenue,
    p.stock_quantity AS current_stock
FROM products p
LEFT JOIN order_items oi ON p.product_id = oi.product_id
GROUP BY p.product_id, p.product_name, p.category, p.price, p.stock_quantity
ORDER BY total_revenue DESC;
```

**ข้อ 9:** หาพนักงานที่ไม่มีผู้จัดการ (ยกเว้น CEO)

```sql
-- เฉลย
SELECT 
    e.emp_id,
    e.first_name || ' ' || e.last_name AS employee,
    e.job_title,
    d.dept_name
FROM employees e
LEFT JOIN employees m ON e.manager_id = m.emp_id
LEFT JOIN departments d ON e.dept_id = d.dept_id
WHERE e.manager_id IS NULL
  AND e.job_title NOT LIKE '%CEO%'
ORDER BY e.emp_id;
```

**ข้อ 10:** วิเคราะห์ customer segments: New (0 orders), Active (1-2), Loyal (3+)

```sql
-- เฉลย
SELECT 
    segment,
    COUNT(*) AS customer_count,
    ROUND(COUNT(*)::NUMERIC / SUM(COUNT(*)) OVER () * 100, 1) AS percentage
FROM (
    SELECT 
        c.customer_id,
        CASE 
            WHEN COUNT(o.order_id) = 0 THEN 'New (No Orders)'
            WHEN COUNT(o.order_id) <= 2 THEN 'Active (1-2 Orders)'
            ELSE 'Loyal (3+ Orders)'
        END AS segment
    FROM customers c
    LEFT JOIN orders o ON c.customer_id = o.customer_id
    GROUP BY c.customer_id
) segmented
GROUP BY segment
ORDER BY customer_count DESC;
```

---

## สรุปภาค 23

1. **LEFT JOIN** — เก็บทุกแถวจากตารางซ้าย, NULL สำหรับตารางขวาที่ไม่ match
2. **RIGHT JOIN** — เก็บทุกแถวจากตารางขวา (แนะนำให้แปลงเป็น LEFT JOIN)
3. **ON vs WHERE** — สำคัญมาก! ON filter ก่อน join, WHERE filter หลัง join
4. **Anti-Join** — `LEFT JOIN ... WHERE right.id IS NULL` หาที่ "ไม่มีคู่"
5. **COALESCE** — แทน NULL ด้วยค่าที่มีความหมาย
6. **COUNT(*) vs COUNT(col)** — ต่างกัน! COUNT(*) นับ NULL ด้วย

**ในภาคถัดไป** จะเรียน FULL OUTER JOIN ที่รวมข้อมูลจากทั้งสองตาราง ไม่ว่าจะมีคู่หรือไม่!
