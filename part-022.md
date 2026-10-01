# ภาค 22: INNER JOIN — การรวมตารางที่พบบ่อยที่สุด

## INNER JOIN คืออะไร?

**INNER JOIN** คือการรวมข้อมูลจาก 2 ตารางขึ้นไป โดยเอาเฉพาะแถวที่มีค่าตรงกันใน **ทั้งสองตาราง** เท่านั้น

```
แผนภาพ Venn Diagram ของ INNER JOIN:

Table A          Table B
  ┌─────────────────────────┐
  │      ╔═══════╗          │
  │   A  ║  A∩B  ║  B only  │
  │  only║(JOIN) ║          │
  │      ╚═══════╝          │
  └─────────────────────────┘

INNER JOIN = เฉพาะส่วนตัดกัน (A∩B)
- แถวที่มีค่า key ตรงกันในทั้งสองตาราง
- แถวที่ไม่มีคู่ = ถูกตัดทิ้ง!
```

### ตัวอย่างเข้าใจง่าย

```
employees:                    departments:
┌─────┬──────────┬─────────┐  ┌─────────┬──────────────┐
│ id  │ name     │ dept_id │  │ dept_id │ dept_name    │
├─────┼──────────┼─────────┤  ├─────────┼──────────────┤
│ 1   │ Somchai  │ 10      │  │ 1       │ Engineering  │
│ 2   │ Wanchai  │ 10      │  │ 2       │ Marketing    │
│ 3   │ Pranee   │ 10      │  │ 10      │ Executive    │
│ 4   │ Narong   │ 1       │  │ 99      │ Ghost Dept   │ ← ไม่มีพนักงาน
│NULL │ Ghost Emp│ NULL    │  └─────────┴──────────────┘
└─────┴──────────┴─────────┘

INNER JOIN result (dept_id ตรงกัน):
┌─────┬──────────┬──────────────┐
│ id  │ name     │ dept_name    │
├─────┼──────────┼──────────────┤
│ 1   │ Somchai  │ Executive    │ ← dept_id = 10 ตรงกัน
│ 2   │ Wanchai  │ Executive    │ ← dept_id = 10 ตรงกัน
│ 3   │ Pranee   │ Executive    │ ← dept_id = 10 ตรงกัน
│ 4   │ Narong   │ Engineering  │ ← dept_id = 1 ตรงกัน
└─────┴──────────┴──────────────┘

หายไป:
- Ghost Emp (dept_id = NULL) → ไม่ match
- Ghost Dept (dept_id = 99) → ไม่มีพนักงาน
```

---

## Syntax ของ INNER JOIN

### แบบ Explicit (แนะนำ)

```sql
-- Syntax ทั่วไป
SELECT columns
FROM table_a
INNER JOIN table_b ON table_a.key = table_b.key;

-- INNER keyword ไม่จำเป็น (JOIN อย่างเดียวก็ได้)
SELECT columns
FROM table_a
JOIN table_b ON table_a.key = table_b.key;
```

### แบบ Implicit (รูปแบบเก่า — ไม่แนะนำ)

```sql
-- รูปแบบเก่า: ใช้ comma คั่นตาราง + WHERE
SELECT columns
FROM table_a, table_b
WHERE table_a.key = table_b.key;

-- เทียบเท่ากับ:
SELECT columns
FROM table_a
INNER JOIN table_b ON table_a.key = table_b.key;
```

---

## ตัวอย่าง INNER JOIN พื้นฐาน (1-10)

### ตัวอย่างที่ 1: JOIN employees กับ departments

```sql
-- แสดงชื่อพนักงานพร้อมชื่อแผนก
SELECT 
    e.emp_id,
    e.first_name,
    e.last_name,
    e.job_title,
    d.dept_name
FROM employees e
INNER JOIN departments d ON e.dept_id = d.dept_id;

-- ผลลัพธ์ (50 แถว):
-- emp_id | first_name | last_name | job_title | dept_name
-- -------+------------+-----------+-----------+----------
-- 1      | Somchai    | Jaidee    | CEO       | Executive
-- 2      | Wanchai    | Thongdee  | CTO       | Executive
-- ...
```

### ตัวอย่างที่ 2: ใช้ Table Aliases

```sql
-- Alias ทำให้ query อ่านง่ายขึ้น
SELECT 
    e.first_name || ' ' || e.last_name AS full_name,
    e.salary,
    d.dept_name,
    d.location
FROM employees e
JOIN departments d ON e.dept_id = d.dept_id
ORDER BY d.dept_name, e.salary DESC;
```

### ตัวอย่างที่ 3: เลือก columns เฉพาะที่ต้องการ

```sql
-- ดูพนักงานในแผนก Engineering
SELECT 
    e.emp_id,
    CONCAT(e.first_name, ' ', e.last_name) AS name,
    e.job_title,
    e.salary,
    e.hire_date
FROM employees e
JOIN departments d ON e.dept_id = d.dept_id
WHERE d.dept_name = 'Engineering'
ORDER BY e.salary DESC;
```

### ตัวอย่างที่ 4: JOIN orders กับ customers

```sql
-- แสดงออร์เดอร์พร้อมชื่อลูกค้า
SELECT 
    o.order_id,
    o.order_date,
    o.total_amount,
    o.status,
    c.first_name || ' ' || c.last_name AS customer_name,
    c.city
FROM orders o
JOIN customers c ON o.customer_id = c.customer_id
ORDER BY o.order_date;
```

### ตัวอย่างที่ 5: JOIN order_items กับ products

```sql
-- แสดงรายการสินค้าในแต่ละ order
SELECT 
    oi.order_id,
    p.product_name,
    p.category,
    oi.quantity,
    oi.unit_price,
    oi.discount,
    (oi.quantity * oi.unit_price - oi.discount) AS line_total
FROM order_items oi
JOIN products p ON oi.product_id = p.product_id
ORDER BY oi.order_id, p.product_name;
```

### ตัวอย่างที่ 6: JOIN ด้วย USING (เมื่อชื่อ column เหมือนกัน)

```sql
-- ใช้ USING แทน ON เมื่อชื่อ column เหมือนกัน
SELECT 
    e.first_name,
    e.last_name,
    d.dept_name
FROM employees e
JOIN departments d USING (dept_id);  -- แทน ON e.dept_id = d.dept_id

-- JOIN หลายตารางด้วย USING
SELECT 
    o.order_id,
    o.order_date,
    oi.product_id,
    oi.quantity
FROM orders o
JOIN order_items oi USING (order_id);
```

### ตัวอย่างที่ 7: INNER JOIN กับ WHERE clause

```sql
-- หา orders ที่ completed ของลูกค้าใน Bangkok
SELECT 
    o.order_id,
    o.order_date,
    o.total_amount,
    c.first_name || ' ' || c.last_name AS customer,
    c.city
FROM orders o
JOIN customers c ON o.customer_id = c.customer_id
WHERE c.city = 'Bangkok'
  AND o.status = 'completed'
ORDER BY o.total_amount DESC;
```

### ตัวอย่างที่ 8: WHERE หลัง JOIN เทียบกับ ON

```sql
-- ทั้งสองแบบนี้ให้ผลเหมือนกันสำหรับ INNER JOIN
-- แบบที่ 1: condition ใน WHERE
SELECT e.first_name, d.dept_name
FROM employees e
JOIN departments d ON e.dept_id = d.dept_id
WHERE d.dept_name = 'Engineering';

-- แบบที่ 2: condition ใน ON (ไม่แนะนำสำหรับ INNER JOIN - อ่านยาก)
SELECT e.first_name, d.dept_name
FROM employees e
JOIN departments d ON e.dept_id = d.dept_id AND d.dept_name = 'Engineering';

-- หมายเหตุ: สำหรับ LEFT JOIN, การวาง condition ใน ON หรือ WHERE
-- ให้ผลต่างกัน! (เรียนในบทถัดไป)
```

### ตัวอย่างที่ 9: JOIN หลาย conditions (Multiple ON)

```sql
-- JOIN บน multiple columns (composite key)
SELECT 
    oi.item_id,
    oi.order_id,
    oi.product_id,
    p.product_name
FROM order_items oi
JOIN products p ON oi.product_id = p.product_id
                AND oi.unit_price = p.price;  -- ราคาต้องตรงกัน

-- JOIN ด้วย condition ที่ซับซ้อน
SELECT 
    e.first_name,
    e.salary,
    d.dept_name,
    d.budget
FROM employees e
JOIN departments d ON e.dept_id = d.dept_id
                  AND e.salary > 100000;  -- เงินเดือนต้องมากกว่า 100000
```

### ตัวอย่างที่ 10: JOIN กับ computed columns

```sql
-- คำนวณค่าใน JOIN
SELECT 
    e.first_name || ' ' || e.last_name AS employee_name,
    e.salary,
    d.dept_name,
    d.budget,
    ROUND(e.salary / d.budget * 100, 2) AS salary_pct_of_budget
FROM employees e
JOIN departments d ON e.dept_id = d.dept_id
ORDER BY salary_pct_of_budget DESC;
```

---

## JOIN หลายตาราง (Multiple Table JOINs)

### ตัวอย่างที่ 11: JOIN 3 ตาราง — orders + customers + order_items

```sql
-- แสดงรายละเอียด order ครบถ้วน
SELECT 
    o.order_id,
    c.first_name || ' ' || c.last_name AS customer_name,
    o.order_date,
    COUNT(oi.item_id) AS items_count,
    SUM(oi.quantity) AS total_qty,
    o.total_amount
FROM orders o
JOIN customers c ON o.customer_id = c.customer_id
JOIN order_items oi ON o.order_id = oi.order_id
GROUP BY o.order_id, customer_name, o.order_date, o.total_amount
ORDER BY o.order_date;
```

### ตัวอย่างที่ 12: JOIN 4 ตาราง — orders + customers + order_items + products

```sql
-- รายงานการขายแบบเต็ม
SELECT 
    o.order_id,
    c.first_name || ' ' || c.last_name AS customer,
    p.product_name,
    p.category,
    oi.quantity,
    oi.unit_price,
    oi.discount,
    (oi.quantity * oi.unit_price - oi.discount) AS line_total
FROM orders o
JOIN customers c ON o.customer_id = c.customer_id
JOIN order_items oi ON o.order_id = oi.order_id
JOIN products p ON oi.product_id = p.product_id
ORDER BY o.order_id, p.product_name;
```

### ตัวอย่างที่ 13: JOIN employees กับ departments และ managers

```sql
-- แสดงข้อมูลพนักงาน + แผนก + ผู้จัดการ
SELECT 
    e.emp_id,
    CONCAT(e.first_name, ' ', e.last_name) AS employee,
    e.job_title,
    d.dept_name,
    CONCAT(m.first_name, ' ', m.last_name) AS manager
FROM employees e
JOIN departments d ON e.dept_id = d.dept_id
JOIN employees m ON e.manager_id = m.emp_id  -- Self JOIN สำหรับ manager
ORDER BY d.dept_name, e.emp_id;
```

---

## INNER JOIN กับ Aggregate Functions

### ตัวอย่างที่ 14: COUNT + JOIN

```sql
-- นับจำนวนพนักงานในแต่ละแผนก
SELECT 
    d.dept_name,
    d.location,
    COUNT(e.emp_id) AS employee_count
FROM departments d
JOIN employees e ON d.dept_id = e.dept_id  -- INNER JOIN: แผนกที่มีพนักงานเท่านั้น
GROUP BY d.dept_id, d.dept_name, d.location
ORDER BY employee_count DESC;

-- ผลลัพธ์ (เฉพาะแผนกที่มีพนักงาน):
-- Engineering    : 12 คน
-- Executive      :  3 คน
-- ...
```

### ตัวอย่างที่ 15: SUM + JOIN

```sql
-- ยอดขายรวมต่อ category
SELECT 
    p.category,
    COUNT(DISTINCT oi.order_id) AS orders_count,
    SUM(oi.quantity) AS total_units_sold,
    SUM(oi.quantity * oi.unit_price - oi.discount) AS total_revenue
FROM products p
JOIN order_items oi ON p.product_id = oi.product_id
GROUP BY p.category
ORDER BY total_revenue DESC;
```

### ตัวอย่างที่ 16: AVG + JOIN

```sql
-- เงินเดือนเฉลี่ยต่อแผนก
SELECT 
    d.dept_name,
    COUNT(e.emp_id) AS headcount,
    ROUND(AVG(e.salary), 2) AS avg_salary,
    MIN(e.salary) AS min_salary,
    MAX(e.salary) AS max_salary,
    SUM(e.salary) AS total_salary
FROM departments d
JOIN employees e ON d.dept_id = e.dept_id
GROUP BY d.dept_id, d.dept_name
ORDER BY avg_salary DESC;
```

### ตัวอย่างที่ 17: HAVING กับ JOIN

```sql
-- แผนกที่มีเงินเดือนเฉลี่ยมากกว่า 120,000
SELECT 
    d.dept_name,
    COUNT(e.emp_id) AS employee_count,
    ROUND(AVG(e.salary), 2) AS avg_salary
FROM departments d
JOIN employees e ON d.dept_id = e.dept_id
GROUP BY d.dept_id, d.dept_name
HAVING AVG(e.salary) > 120000
ORDER BY avg_salary DESC;
```

### ตัวอย่างที่ 18: ยอดขายต่อลูกค้า

```sql
-- สรุปยอดซื้อของลูกค้าแต่ละคน
SELECT 
    c.customer_id,
    CONCAT(c.first_name, ' ', c.last_name) AS customer_name,
    c.city,
    COUNT(o.order_id) AS order_count,
    SUM(o.total_amount) AS total_spent,
    AVG(o.total_amount) AS avg_order_value,
    MAX(o.total_amount) AS largest_order
FROM customers c
JOIN orders o ON c.customer_id = o.customer_id
GROUP BY c.customer_id, customer_name, c.city
ORDER BY total_spent DESC;
```

### ตัวอย่างที่ 19: Top Products ที่ขายดีที่สุด

```sql
-- สินค้าที่ขายดีที่สุด (ตามยอดเงิน)
SELECT 
    p.product_id,
    p.product_name,
    p.category,
    p.price AS list_price,
    COUNT(oi.item_id) AS times_in_orders,
    SUM(oi.quantity) AS total_quantity,
    SUM(oi.quantity * oi.unit_price - oi.discount) AS total_revenue,
    RANK() OVER (ORDER BY SUM(oi.quantity * oi.unit_price - oi.discount) DESC) AS revenue_rank
FROM products p
JOIN order_items oi ON p.product_id = oi.product_id
GROUP BY p.product_id, p.product_name, p.category, p.price
ORDER BY total_revenue DESC
LIMIT 10;
```

---

## INNER JOIN ขั้นสูง

### ตัวอย่างที่ 20: JOIN กับ Subquery

```sql
-- JOIN กับ subquery: หาพนักงานที่มีเงินเดือนสูงกว่าค่าเฉลี่ยของแผนก
SELECT 
    e.first_name,
    e.last_name,
    e.salary,
    dept_avg.avg_salary,
    e.salary - dept_avg.avg_salary AS above_average
FROM employees e
JOIN (
    SELECT dept_id, AVG(salary) AS avg_salary
    FROM employees
    GROUP BY dept_id
) dept_avg ON e.dept_id = dept_avg.dept_id
WHERE e.salary > dept_avg.avg_salary
ORDER BY above_average DESC;
```

### ตัวอย่างที่ 21: JOIN กับ CASE WHEN

```sql
-- จัดกลุ่มพนักงานตามระดับเงินเดือน
SELECT 
    e.first_name,
    e.last_name,
    d.dept_name,
    e.salary,
    CASE 
        WHEN e.salary >= 200000 THEN 'Executive'
        WHEN e.salary >= 150000 THEN 'Senior Management'
        WHEN e.salary >= 100000 THEN 'Management'
        WHEN e.salary >= 80000  THEN 'Staff'
        ELSE 'Junior'
    END AS salary_band
FROM employees e
JOIN departments d ON e.dept_id = d.dept_id
ORDER BY e.salary DESC;
```

### ตัวอย่างที่ 22: JOIN กับ DATE functions

```sql
-- หาพนักงานที่ทำงานมาแล้วมากกว่า 5 ปี พร้อมข้อมูลแผนก
SELECT 
    e.first_name || ' ' || e.last_name AS name,
    e.hire_date,
    EXTRACT(YEAR FROM AGE(CURRENT_DATE, e.hire_date)) AS years_worked,
    e.job_title,
    d.dept_name
FROM employees e
JOIN departments d ON e.dept_id = d.dept_id
WHERE e.hire_date <= CURRENT_DATE - INTERVAL '5 years'
ORDER BY e.hire_date;
```

### ตัวอย่างที่ 23: NATURAL JOIN (ระวัง!)

```sql
-- NATURAL JOIN: Join โดยอัตโนมัติบน columns ที่ชื่อเหมือนกัน
-- ระวัง! ถ้าชื่อตรงกันหลาย columns อาจให้ผลผิดพลาด

SELECT e.first_name, d.dept_name
FROM employees e
NATURAL JOIN departments d;
-- เทียบเท่า: JOIN departments d ON e.dept_id = d.dept_id
-- (ถ้ามีแค่ dept_id เป็น column ที่ชื่อตรงกัน)

-- ไม่แนะนำใช้ NATURAL JOIN เพราะ:
-- 1. ไม่ชัดเจน ว่า join บน column อะไร
-- 2. ถ้าเพิ่ม column ชื่อเดิมในอนาคต อาจพัง
```

### ตัวอย่างที่ 24: JOIN กับ ORDER ที่สำคัญ

```sql
-- ลำดับการ JOIN สำหรับ 3 ตาราง
-- (SQL engine จะหาลำดับที่ดีที่สุดเอง แต่เราควรเรียงให้อ่านง่าย)

-- แบบที่ 1: จาก orders ออกไป
SELECT 
    o.order_id,
    c.first_name,
    oi.product_id
FROM orders o
JOIN customers c ON o.customer_id = c.customer_id
JOIN order_items oi ON o.order_id = oi.order_id;

-- แบบที่ 2: เริ่มจาก customers
SELECT 
    o.order_id,
    c.first_name,
    oi.product_id
FROM customers c
JOIN orders o ON c.customer_id = o.customer_id
JOIN order_items oi ON o.order_id = oi.order_id;

-- ทั้งสองแบบให้ผลเหมือนกัน (SQL ไม่สนใจลำดับการเขียน)
```

### ตัวอย่างที่ 25: INNER JOIN ที่ซับซ้อน — Sales Report

```sql
-- รายงานการขายรายเดือน
SELECT 
    EXTRACT(YEAR FROM o.order_date) AS year,
    EXTRACT(MONTH FROM o.order_date) AS month,
    TO_CHAR(o.order_date, 'YYYY-MM') AS month_label,
    COUNT(DISTINCT o.order_id) AS total_orders,
    COUNT(DISTINCT o.customer_id) AS unique_customers,
    SUM(o.total_amount) AS monthly_revenue
FROM orders o
JOIN customers c ON o.customer_id = c.customer_id
WHERE o.status IN ('completed', 'shipped')
GROUP BY year, month, month_label
ORDER BY year, month;
```

---

## Pattern ที่ใช้บ่อยใน INNER JOIN

### ตัวอย่างที่ 26: Look up / Decode pattern

```sql
-- ใช้ JOIN เป็น lookup table (แทนที่ CASE WHEN)
-- สมมติมีตาราง lookup สำหรับ status
CREATE TEMP TABLE status_labels (
    status_code VARCHAR(20),
    status_thai VARCHAR(50),
    sort_order INT
);
INSERT INTO status_labels VALUES
('completed',  'สำเร็จ',    1),
('shipped',    'จัดส่งแล้ว', 2),
('processing', 'กำลังดำเนิน',3),
('pending',    'รอดำเนิน',   4),
('cancelled',  'ยกเลิก',    5);

SELECT 
    o.order_id,
    o.order_date,
    o.total_amount,
    sl.status_thai
FROM orders o
JOIN status_labels sl ON o.status = sl.status_code
ORDER BY sl.sort_order, o.order_date;
```

### ตัวอย่างที่ 27: JOIN สำหรับ Data Validation

```sql
-- ตรวจสอบว่า order_items ทั้งหมดมี product ที่ valid
SELECT 
    oi.item_id,
    oi.order_id,
    oi.product_id,
    p.product_name,
    oi.unit_price,
    p.price AS current_price,
    ABS(oi.unit_price - p.price) AS price_difference
FROM order_items oi
JOIN products p ON oi.product_id = p.product_id
WHERE ABS(oi.unit_price - p.price) > 1000  -- ราคาต่างกันมาก
ORDER BY price_difference DESC;
```

### ตัวอย่างที่ 28: JOIN แล้ว GROUP BY เพื่อหา duplicates

```sql
-- หา customers ที่มี email ซ้ำกัน (สมมติ)
SELECT 
    c1.email,
    COUNT(*) AS duplicate_count
FROM customers c1
JOIN customers c2 ON c1.email = c2.email 
                  AND c1.customer_id != c2.customer_id
GROUP BY c1.email
HAVING COUNT(*) > 1;
```

### ตัวอย่างที่ 29: JOIN สำหรับ Reporting ขั้นสูง

```sql
-- Customer Lifetime Value (CLV) report
SELECT 
    c.customer_id,
    CONCAT(c.first_name, ' ', c.last_name) AS customer_name,
    c.city,
    c.country,
    c.created_at::DATE AS joined_date,
    MIN(o.order_date) AS first_order_date,
    MAX(o.order_date) AS last_order_date,
    COUNT(o.order_id) AS total_orders,
    SUM(o.total_amount) AS lifetime_value,
    AVG(o.total_amount) AS avg_order_value,
    SUM(o.total_amount) / 
        NULLIF(DATE_PART('day', MAX(o.order_date) - MIN(o.order_date)), 0) * 30 
        AS monthly_avg_spend
FROM customers c
JOIN orders o ON c.customer_id = o.customer_id
WHERE o.status = 'completed'
GROUP BY c.customer_id, customer_name, c.city, c.country, c.created_at
ORDER BY lifetime_value DESC;
```

### ตัวอย่างที่ 30: Department Budget vs Actual Salary

```sql
-- เปรียบเทียบ Budget กับยอดเงินเดือนจริง
SELECT 
    d.dept_name,
    d.budget AS dept_budget,
    COUNT(e.emp_id) AS headcount,
    SUM(e.salary) AS annual_salary_cost,
    SUM(e.salary) * 12 AS projected_annual_salary,
    d.budget - SUM(e.salary) * 12 AS budget_variance,
    ROUND(SUM(e.salary) * 12 / d.budget * 100, 1) AS salary_to_budget_pct
FROM departments d
JOIN employees e ON d.dept_id = e.dept_id
GROUP BY d.dept_id, d.dept_name, d.budget
ORDER BY salary_to_budget_pct DESC;
```

---

## ตัวอย่างที่ 31-40: ธุรกิจ E-commerce

### ตัวอย่างที่ 31: สินค้าขายดีแต่ละ category

```sql
SELECT 
    p.category,
    p.product_name,
    SUM(oi.quantity) AS total_sold,
    SUM(oi.quantity * oi.unit_price - oi.discount) AS revenue,
    RANK() OVER (PARTITION BY p.category ORDER BY SUM(oi.quantity) DESC) AS rank_in_category
FROM products p
JOIN order_items oi ON p.product_id = oi.product_id
JOIN orders o ON oi.order_id = o.order_id
WHERE o.status IN ('completed', 'shipped')
GROUP BY p.category, p.product_id, p.product_name
ORDER BY p.category, rank_in_category;
```

### ตัวอย่างที่ 32: Monthly Customer Acquisition

```sql
-- ลูกค้าใหม่ที่สั่งซื้อในเดือนแรก
SELECT 
    TO_CHAR(c.created_at, 'YYYY-MM') AS registration_month,
    COUNT(DISTINCT c.customer_id) AS new_customers,
    COUNT(DISTINCT o.order_id) AS orders_in_first_month,
    ROUND(COUNT(DISTINCT o.order_id)::NUMERIC / COUNT(DISTINCT c.customer_id) * 100, 1) 
        AS conversion_rate_pct
FROM customers c
JOIN orders o ON c.customer_id = o.customer_id
    AND o.order_date <= c.created_at::DATE + INTERVAL '30 days'
GROUP BY registration_month
ORDER BY registration_month;
```

### ตัวอย่างที่ 33: สินค้าที่ order ร่วมกันบ่อยที่สุด

```sql
-- Product pairs ที่ถูกซื้อพร้อมกัน
SELECT 
    p1.product_name AS product_1,
    p2.product_name AS product_2,
    COUNT(*) AS times_bought_together
FROM order_items oi1
JOIN order_items oi2 ON oi1.order_id = oi2.order_id 
                     AND oi1.product_id < oi2.product_id  -- ป้องกัน duplicate pairs
JOIN products p1 ON oi1.product_id = p1.product_id
JOIN products p2 ON oi2.product_id = p2.product_id
GROUP BY p1.product_id, p1.product_name, p2.product_id, p2.product_name
ORDER BY times_bought_together DESC
LIMIT 10;
```

### ตัวอย่างที่ 34: Revenue by Employee Department

```sql
-- เชื่อมโยงแผนกกับ revenue (สมมติว่า Sales dept รับผิดชอบ orders)
SELECT 
    d.dept_name,
    d.location,
    COUNT(DISTINCT e.emp_id) AS sales_reps,
    COUNT(DISTINCT o.order_id) AS total_orders,
    SUM(o.total_amount) AS total_revenue,
    ROUND(SUM(o.total_amount) / COUNT(DISTINCT e.emp_id), 2) AS revenue_per_employee
FROM departments d
JOIN employees e ON d.dept_id = e.dept_id
JOIN orders o ON o.status IN ('completed', 'shipped')  -- สมมติ
WHERE d.dept_name = 'Sales'
GROUP BY d.dept_id, d.dept_name, d.location;
```

### ตัวอย่างที่ 35: Cohort Analysis พื้นฐาน

```sql
-- วิเคราะห์ลูกค้าตามเดือนที่สมัคร
SELECT 
    TO_CHAR(c.created_at, 'YYYY-MM') AS cohort_month,
    EXTRACT(MONTH FROM AGE(o.order_date::TIMESTAMP, c.created_at)) AS months_since_join,
    COUNT(DISTINCT c.customer_id) AS active_customers,
    SUM(o.total_amount) AS revenue
FROM customers c
JOIN orders o ON c.customer_id = o.customer_id
WHERE o.status = 'completed'
GROUP BY cohort_month, months_since_join
ORDER BY cohort_month, months_since_join;
```

### ตัวอย่างที่ 36: Stock Alert Report

```sql
-- สินค้าที่ stock น้อย vs ยอดขาย
SELECT 
    p.product_id,
    p.product_name,
    p.category,
    p.stock_quantity,
    COALESCE(SUM(oi.quantity), 0) AS total_sold,
    p.stock_quantity - COALESCE(SUM(oi.quantity), 0) AS remaining_stock,
    CASE 
        WHEN p.stock_quantity < 20 THEN 'CRITICAL'
        WHEN p.stock_quantity < 50 THEN 'LOW'
        ELSE 'OK'
    END AS stock_status
FROM products p
JOIN order_items oi ON p.product_id = oi.product_id
GROUP BY p.product_id, p.product_name, p.category, p.stock_quantity
ORDER BY remaining_stock ASC;
```

### ตัวอย่างที่ 37: Employee Performance Metrics

```sql
-- ดู metrics ของพนักงานแต่ละคน
SELECT 
    e.emp_id,
    CONCAT(e.first_name, ' ', e.last_name) AS employee,
    d.dept_name,
    e.salary,
    e.hire_date,
    EXTRACT(YEAR FROM AGE(CURRENT_DATE, e.hire_date)) AS years_experience,
    ROUND(e.salary / d.budget * 100, 3) AS salary_budget_share_pct,
    RANK() OVER (PARTITION BY e.dept_id ORDER BY e.salary DESC) AS salary_rank_in_dept
FROM employees e
JOIN departments d ON e.dept_id = d.dept_id
ORDER BY d.dept_name, salary_rank_in_dept;
```

### ตัวอย่างที่ 38: Customer Order History Summary

```sql
-- ประวัติการสั่งซื้อสรุปต่อลูกค้า
SELECT 
    c.customer_id,
    CONCAT(c.first_name, ' ', c.last_name) AS customer,
    COUNT(o.order_id) AS total_orders,
    COUNT(CASE WHEN o.status = 'completed' THEN 1 END) AS completed_orders,
    COUNT(CASE WHEN o.status = 'cancelled' THEN 1 END) AS cancelled_orders,
    SUM(CASE WHEN o.status = 'completed' THEN o.total_amount ELSE 0 END) AS total_spent,
    ROUND(
        COUNT(CASE WHEN o.status = 'cancelled' THEN 1 END)::NUMERIC / 
        COUNT(o.order_id) * 100, 1
    ) AS cancellation_rate_pct
FROM customers c
JOIN orders o ON c.customer_id = o.customer_id
GROUP BY c.customer_id, customer
ORDER BY total_spent DESC;
```

### ตัวอย่างที่ 39: Order Fulfillment Analysis

```sql
-- วิเคราะห์ระยะเวลาการจัดส่ง
SELECT 
    o.order_id,
    CONCAT(c.first_name, ' ', c.last_name) AS customer,
    o.order_date,
    o.shipped_date,
    o.status,
    CASE 
        WHEN o.shipped_date IS NOT NULL 
        THEN o.shipped_date - o.order_date
        ELSE NULL
    END AS days_to_ship,
    o.total_amount
FROM orders o
JOIN customers c ON o.customer_id = c.customer_id
WHERE o.status IN ('completed', 'shipped')
ORDER BY days_to_ship DESC NULLS LAST;
```

### ตัวอย่างที่ 40: Full Business Dashboard Query

```sql
-- KPI Dashboard ภาพรวมธุรกิจ
SELECT 
    'Total Revenue' AS metric,
    CONCAT('฿', TO_CHAR(SUM(o.total_amount), 'FM999,999,999.00')) AS value
FROM orders o
WHERE o.status = 'completed'

UNION ALL

SELECT 
    'Total Orders',
    COUNT(o.order_id)::TEXT
FROM orders o
WHERE o.status = 'completed'

UNION ALL

SELECT 
    'Unique Customers',
    COUNT(DISTINCT o.customer_id)::TEXT
FROM orders o
WHERE o.status = 'completed'

UNION ALL

SELECT 
    'Average Order Value',
    CONCAT('฿', TO_CHAR(AVG(o.total_amount), 'FM999,999.00'))
FROM orders o
WHERE o.status = 'completed'

UNION ALL

SELECT 
    'Top Product',
    p.product_name
FROM products p
JOIN order_items oi ON p.product_id = oi.product_id
JOIN orders o ON oi.order_id = o.order_id
WHERE o.status = 'completed'
GROUP BY p.product_id, p.product_name
ORDER BY SUM(oi.quantity) DESC
LIMIT 1;
```

---

## เปรียบเทียบ JOIN ON vs USING vs NATURAL JOIN

```sql
-- 3 วิธีในการ JOIN employees กับ departments

-- 1. ON (แนะนำมากที่สุด - ชัดเจน)
SELECT e.first_name, d.dept_name
FROM employees e
JOIN departments d ON e.dept_id = d.dept_id;

-- 2. USING (ใช้ได้เมื่อชื่อ column เหมือนกัน - ห้ามใส่ table alias กับ column นั้น)
SELECT e.first_name, d.dept_name
FROM employees e
JOIN departments d USING (dept_id);

-- 3. NATURAL JOIN (ไม่แนะนำ - เสี่ยง)
SELECT e.first_name, d.dept_name
FROM employees e
NATURAL JOIN departments d;

-- ข้อแตกต่าง:
-- ON:      ระบุ condition เองชัดเจน ใช้ table.column ได้
-- USING:   ต้องชื่อ column เหมือนกัน ผลลัพธ์แสดง column นั้นแค่ครั้งเดียว
-- NATURAL: Auto-detect columns ชื่อเดียวกัน อันตราย!
```

---

## แบบฝึกหัดภาค 22

**ข้อ 1:** แสดงรายชื่อพนักงานทั้งหมดพร้อมชื่อแผนกและ location

```sql
-- เฉลย
SELECT 
    e.emp_id,
    CONCAT(e.first_name, ' ', e.last_name) AS full_name,
    e.job_title,
    e.salary,
    d.dept_name,
    d.location
FROM employees e
INNER JOIN departments d ON e.dept_id = d.dept_id
ORDER BY d.dept_name, e.last_name;
```

**ข้อ 2:** แสดง orders ทั้งหมดพร้อมชื่อลูกค้าและเมือง เฉพาะที่ completed

```sql
-- เฉลย
SELECT 
    o.order_id,
    o.order_date,
    o.total_amount,
    c.first_name || ' ' || c.last_name AS customer,
    c.city
FROM orders o
JOIN customers c ON o.customer_id = c.customer_id
WHERE o.status = 'completed'
ORDER BY o.total_amount DESC;
```

**ข้อ 3:** หาสินค้าที่ขายดีที่สุด 5 อันดับแรกตามยอดเงิน

```sql
-- เฉลย
SELECT 
    p.product_name,
    p.category,
    SUM(oi.quantity) AS units_sold,
    SUM(oi.quantity * oi.unit_price - oi.discount) AS total_revenue
FROM products p
JOIN order_items oi ON p.product_id = oi.product_id
GROUP BY p.product_id, p.product_name, p.category
ORDER BY total_revenue DESC
LIMIT 5;
```

**ข้อ 4:** นับจำนวน orders ต่อเดือนในปี 2024 (เฉพาะที่ completed)

```sql
-- เฉลย
SELECT 
    EXTRACT(MONTH FROM o.order_date) AS month,
    TO_CHAR(o.order_date, 'Month') AS month_name,
    COUNT(o.order_id) AS order_count,
    SUM(o.total_amount) AS revenue
FROM orders o
JOIN customers c ON o.customer_id = c.customer_id
WHERE o.status = 'completed'
  AND EXTRACT(YEAR FROM o.order_date) = 2024
GROUP BY month, month_name
ORDER BY month;
```

**ข้อ 5:** แสดงรายละเอียด order 18 ครบถ้วน (ชื่อลูกค้า, สินค้าแต่ละรายการ)

```sql
-- เฉลย
SELECT 
    o.order_id,
    c.first_name || ' ' || c.last_name AS customer,
    o.order_date,
    o.status,
    p.product_name,
    p.category,
    oi.quantity,
    oi.unit_price,
    oi.discount,
    (oi.quantity * oi.unit_price - oi.discount) AS line_amount
FROM orders o
JOIN customers c ON o.customer_id = c.customer_id
JOIN order_items oi ON o.order_id = oi.order_id
JOIN products p ON oi.product_id = p.product_id
WHERE o.order_id = 18;
```

**ข้อ 6:** หาแผนกที่มีค่าเฉลี่ยเงินเดือนมากที่สุด 3 อันดับแรก

```sql
-- เฉลย
SELECT 
    d.dept_name,
    COUNT(e.emp_id) AS headcount,
    ROUND(AVG(e.salary), 2) AS avg_salary,
    SUM(e.salary) AS total_salary
FROM departments d
JOIN employees e ON d.dept_id = e.dept_id
GROUP BY d.dept_id, d.dept_name
ORDER BY avg_salary DESC
LIMIT 3;
```

**ข้อ 7:** หาลูกค้าที่มียอดซื้อรวมมากที่สุด 10 อันดับแรก

```sql
-- เฉลย
SELECT 
    c.customer_id,
    c.first_name || ' ' || c.last_name AS customer,
    c.city,
    COUNT(o.order_id) AS order_count,
    SUM(o.total_amount) AS total_spent,
    RANK() OVER (ORDER BY SUM(o.total_amount) DESC) AS spending_rank
FROM customers c
JOIN orders o ON c.customer_id = o.customer_id
WHERE o.status = 'completed'
GROUP BY c.customer_id, customer, c.city
ORDER BY total_spent DESC
LIMIT 10;
```

**ข้อ 8:** แสดงสินค้าใน Electronics ที่ถูกสั่งซื้อในปี 2024

```sql
-- เฉลย
SELECT DISTINCT
    p.product_id,
    p.product_name,
    p.price,
    p.stock_quantity,
    COUNT(DISTINCT oi.order_id) AS times_ordered
FROM products p
JOIN order_items oi ON p.product_id = oi.product_id
JOIN orders o ON oi.order_id = o.order_id
WHERE p.category = 'Electronics'
  AND EXTRACT(YEAR FROM o.order_date) = 2024
GROUP BY p.product_id, p.product_name, p.price, p.stock_quantity
ORDER BY times_ordered DESC;
```

**ข้อ 9:** หาพนักงานที่มีผู้จัดการเป็น VP (มี VP ใน job_title)

```sql
-- เฉลย
SELECT 
    e.emp_id,
    CONCAT(e.first_name, ' ', e.last_name) AS employee,
    e.job_title AS emp_title,
    d.dept_name,
    m.first_name || ' ' || m.last_name AS manager_name,
    m.job_title AS manager_title
FROM employees e
JOIN employees m ON e.manager_id = m.emp_id
JOIN departments d ON e.dept_id = d.dept_id
WHERE m.job_title LIKE '%VP%'
ORDER BY d.dept_name, e.emp_id;
```

**ข้อ 10:** วิเคราะห์ยอดขายต่อ category และ subcategory (ใช้ INNER JOIN)

```sql
-- เฉลย
SELECT 
    p.category,
    COUNT(DISTINCT p.product_id) AS products_in_category,
    COUNT(DISTINCT oi.order_id) AS orders_count,
    SUM(oi.quantity) AS total_units_sold,
    SUM(oi.quantity * oi.unit_price - oi.discount) AS total_revenue,
    ROUND(AVG(oi.unit_price), 2) AS avg_unit_price,
    ROUND(
        SUM(oi.quantity * oi.unit_price - oi.discount) / 
        SUM(SUM(oi.quantity * oi.unit_price - oi.discount)) OVER () * 100, 1
    ) AS revenue_share_pct
FROM products p
JOIN order_items oi ON p.product_id = oi.product_id
JOIN orders o ON oi.order_id = o.order_id
WHERE o.status IN ('completed', 'shipped')
GROUP BY p.category
ORDER BY total_revenue DESC;
```

---

## สรุปภาค 22

INNER JOIN เป็น JOIN ที่ใช้บ่อยที่สุดใน SQL:

1. **หลักการ:** เอาเฉพาะแถวที่มีค่าตรงกันในทั้งสองตาราง
2. **Syntax:** `JOIN table ON condition` หรือ `INNER JOIN table ON condition`
3. **USING:** ใช้เมื่อชื่อ column เหมือนกัน
4. **Multiple Tables:** JOIN ต่อกันได้ไม่จำกัด
5. **กับ Aggregates:** ใช้ GROUP BY, HAVING ได้ปกติ
6. **ข้อควรระวัง:** แถวที่ไม่มีคู่จะหายไป (ใช้ LEFT JOIN ถ้าต้องการเก็บไว้)

**ในภาคถัดไป** เราจะเรียน LEFT JOIN ที่เก็บแถวที่ไม่มีคู่ไว้ด้วย!
