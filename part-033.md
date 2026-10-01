# Part 033: HAVING - การกรองข้อมูลที่ถูกจัดกลุ่มแล้ว

## บทนำ (Introduction)

`HAVING` คือคำสั่งที่ใช้กรองผลลัพธ์ **หลัง** จากการทำ GROUP BY แล้ว ต่างจาก `WHERE` ที่กรอง **ก่อน** การจัดกลุ่ม

**ความแตกต่างหลัก:**
- `WHERE` → กรองแถวก่อนจัดกลุ่ม (ทำงานกับ raw data)
- `HAVING` → กรองกลุ่มหลังจัดกลุ่มแล้ว (ทำงานกับ aggregated data)

---

## การตั้งค่าฐานข้อมูล

```sql
USE ecommerce_db;

-- เพิ่มข้อมูลเพิ่มเติม
INSERT INTO orders (customer_id, order_date, status, shipping_fee, discount_amt, total_amount, payment_method) VALUES
(1,  '2024-05-10 10:00:00', 'delivered',  50,    0, 46000, 'บัตรเครดิต'),
(11, '2024-05-15 14:00:00', 'delivered',   0, 1000, 38950, 'บัตรเครดิต'),
(6,  '2024-05-20 09:00:00', 'processing', 50,    0, 24950, 'บัตรเดบิต'),
(3,  '2024-05-25 15:00:00', 'delivered',  50,  500,  8540, 'PromptPay'),
(10, '2024-06-01 11:00:00', 'delivered',   0,    0, 14950, 'บัตรเครดิต'),
(2,  '2024-06-05 13:00:00', 'shipped',    50,    0, 35950, 'บัตรเดบิต'),
(11, '2024-06-10 10:00:00', 'delivered',   0, 2000, 57950, 'บัตรเครดิต'),
(1,  '2024-06-15 16:00:00', 'delivered',  50,    0,  9090, 'บัตรเครดิต');

-- เพิ่ม order_items
INSERT INTO order_items (order_id, product_id, quantity, unit_price, discount_pct) VALUES
(38, 1,  1, 45900, 0),
(39, 3,  1, 42900, 0),
(40, 10, 1, 24900, 0),
(41, 14, 1,  5490, 0),
(41, 19, 1,  3490, 0),
(42, 10, 1, 24900, 0),
(43, 2,  1, 35900, 0),
(44, 8,  1, 59900, 0),
(45, 6,  1,  8990, 0);
```

---

## 1. HAVING Syntax พื้นฐาน

```sql
-- รูปแบบ:
SELECT   column1, aggregate_function(column2) AS alias
FROM     table
WHERE    row_condition           -- กรองก่อน GROUP BY
GROUP BY column1
HAVING   aggregate_condition    -- กรองหลัง GROUP BY
ORDER BY column1;
```

---

## 2. HAVING พื้นฐาน - ตัวอย่างทั่วไป

```sql
-- ตัวอย่างที่ 1: หาแผนกที่มีพนักงานมากกว่า 2 คน
SELECT 
    department_id,
    COUNT(*) AS employee_count
FROM employees
GROUP BY department_id
HAVING COUNT(*) > 2;
```

```sql
-- ตัวอย่างที่ 2: หาลูกค้าที่ซื้อสินค้ามากกว่า 3 ครั้ง
SELECT 
    customer_id,
    COUNT(*) AS order_count,
    SUM(total_amount) AS total_spent
FROM orders
GROUP BY customer_id
HAVING COUNT(*) > 3
ORDER BY order_count DESC;
```

```sql
-- ตัวอย่างที่ 3: หา category สินค้าที่มีราคาเฉลี่ยมากกว่า 20,000 บาท
SELECT 
    category,
    COUNT(*) AS product_count,
    ROUND(AVG(price), 2) AS avg_price
FROM products
GROUP BY category
HAVING AVG(price) > 20000
ORDER BY avg_price DESC;
```

```sql
-- ตัวอย่างที่ 4: หาเดือนที่มียอดขายมากกว่า 100,000 บาท
SELECT 
    DATE_FORMAT(order_date, '%Y-%m') AS month,
    COUNT(*) AS order_count,
    SUM(total_amount) AS monthly_revenue
FROM orders
WHERE status NOT IN ('cancelled')
GROUP BY DATE_FORMAT(order_date, '%Y-%m')
HAVING SUM(total_amount) > 100000
ORDER BY month;
```

```sql
-- ตัวอย่างที่ 5: หา brand ที่มีสินค้ามากกว่า 2 รายการ
SELECT 
    brand,
    COUNT(*) AS product_count,
    AVG(price) AS avg_price
FROM products
GROUP BY brand
HAVING COUNT(*) > 2
ORDER BY product_count DESC;
```

---

## 3. HAVING vs WHERE - ความแตกต่างที่สำคัญ

```sql
-- ตัวอย่างที่ 6: WHERE กรองก่อน GROUP BY
-- หาแผนกใน location 'กรุงเทพฯ ชั้น 3' ที่มีพนักงาน >= 2 คน
SELECT 
    d.department_name,
    COUNT(e.employee_id) AS headcount
FROM departments d
JOIN employees e ON d.department_id = e.department_id
WHERE d.location LIKE 'กรุงเทพฯ%'    -- กรองแผนกในกรุงเทพฯ ก่อน
GROUP BY d.department_id, d.department_name
HAVING COUNT(e.employee_id) >= 2;    -- แล้วกรองเฉพาะที่มีพนักงาน >= 2
```

```sql
-- ตัวอย่างที่ 7: ใช้ทั้ง WHERE และ HAVING ร่วมกัน
-- หาลูกค้า (เฉพาะจากกรุงเทพฯ) ที่ใช้จ่ายมากกว่า 50,000 บาท
SELECT 
    c.customer_id,
    CONCAT(c.first_name, ' ', c.last_name) AS customer_name,
    c.city,
    COUNT(o.order_id)    AS order_count,
    SUM(o.total_amount)  AS total_spent
FROM customers c
JOIN orders o ON c.customer_id = o.customer_id
WHERE c.city = 'กรุงเทพฯ'           -- WHERE: กรองเฉพาะกรุงเทพฯ
    AND o.status != 'cancelled'      -- WHERE: กรองออเดอร์ที่ไม่ถูกยกเลิก
GROUP BY c.customer_id, c.first_name, c.last_name, c.city
HAVING SUM(o.total_amount) > 50000   -- HAVING: กรองผู้ที่ใช้จ่ายมากกว่า 50,000
ORDER BY total_spent DESC;
```

```sql
-- ตัวอย่างที่ 8: ข้อผิดพลาดที่พบบ่อย - ใช้ Aggregate ใน WHERE (ผิด!)
-- ❌ ผิด!
SELECT department_id, COUNT(*)
FROM employees
WHERE COUNT(*) > 2   -- ERROR!
GROUP BY department_id;

-- ✅ ถูก!
SELECT department_id, COUNT(*) AS headcount
FROM employees
GROUP BY department_id
HAVING COUNT(*) > 2;
```

---

## 4. HAVING กับหลายเงื่อนไข

```sql
-- ตัวอย่างที่ 9: HAVING กับ AND
SELECT 
    category,
    COUNT(*) AS product_count,
    AVG(price) AS avg_price,
    SUM(stock_qty * price) AS stock_value
FROM products
GROUP BY category
HAVING COUNT(*) >= 2             -- มีสินค้าอย่างน้อย 2 ชนิด
    AND AVG(price) > 15000       -- ราคาเฉลี่ยมากกว่า 15,000
ORDER BY avg_price DESC;
```

```sql
-- ตัวอย่างที่ 10: HAVING กับ OR
SELECT 
    status,
    COUNT(*) AS order_count,
    SUM(total_amount) AS revenue
FROM orders
GROUP BY status
HAVING COUNT(*) > 5             -- มีออเดอร์มากกว่า 5 ครั้ง
    OR SUM(total_amount) > 200000;  -- หรือยอดขายมากกว่า 200,000
```

```sql
-- ตัวอย่างที่ 11: HAVING กับ BETWEEN
SELECT 
    department_id,
    COUNT(*) AS headcount,
    AVG(salary) AS avg_salary
FROM employees
GROUP BY department_id
HAVING AVG(salary) BETWEEN 40000 AND 70000
ORDER BY avg_salary;
```

```sql
-- ตัวอย่างที่ 12: HAVING กับ IN
SELECT 
    YEAR(order_date) AS year,
    MONTH(order_date) AS month,
    SUM(total_amount) AS revenue
FROM orders
GROUP BY YEAR(order_date), MONTH(order_date)
HAVING MONTH(order_date) IN (1, 4, 7, 10)  -- เฉพาะไตรมาสแรก
ORDER BY year, month;
```

---

## 5. HAVING กับ Subquery

```sql
-- ตัวอย่างที่ 13: หา category ที่มียอดขายสูงกว่าค่าเฉลี่ยของทุก category
SELECT 
    p.category,
    SUM(oi.quantity * oi.unit_price) AS total_revenue
FROM products p
JOIN order_items oi ON p.product_id = oi.product_id
GROUP BY p.category
HAVING SUM(oi.quantity * oi.unit_price) > (
    SELECT AVG(cat_total)
    FROM (
        SELECT SUM(oi2.quantity * oi2.unit_price) AS cat_total
        FROM products p2
        JOIN order_items oi2 ON p2.product_id = oi2.product_id
        GROUP BY p2.category
    ) AS category_totals
)
ORDER BY total_revenue DESC;
```

```sql
-- ตัวอย่างที่ 14: หาพนักงานที่มีเงินเดือนสูงกว่าค่าเฉลี่ยของแผนกตัวเอง
SELECT 
    e.department_id,
    e.first_name,
    e.last_name,
    e.salary,
    dept_avg.avg_dept_salary
FROM employees e
JOIN (
    SELECT department_id, AVG(salary) AS avg_dept_salary
    FROM employees
    GROUP BY department_id
) AS dept_avg ON e.department_id = dept_avg.department_id
WHERE e.salary > dept_avg.avg_dept_salary
ORDER BY e.department_id, e.salary DESC;
```

---

## 6. HAVING Patterns ที่ใช้บ่อย

### Pattern 1: หา Top Groups

```sql
-- ตัวอย่างที่ 15: หา Top 3 ลูกค้าที่ซื้อมากที่สุด
SELECT 
    c.customer_id,
    CONCAT(c.first_name, ' ', c.last_name) AS customer_name,
    COUNT(o.order_id) AS order_count,
    SUM(o.total_amount) AS total_spent
FROM customers c
JOIN orders o ON c.customer_id = o.customer_id
WHERE o.status != 'cancelled'
GROUP BY c.customer_id, c.first_name, c.last_name
ORDER BY total_spent DESC
LIMIT 3;
```

### Pattern 2: หา Outliers (ค่าผิดปกติ)

```sql
-- ตัวอย่างที่ 16: หาลูกค้าที่สั่งซื้อน้อยผิดปกติ (เพียง 1 ครั้ง)
SELECT 
    c.customer_id,
    CONCAT(c.first_name, ' ', c.last_name) AS customer_name,
    c.registered_at,
    COUNT(o.order_id) AS order_count
FROM customers c
LEFT JOIN orders o ON c.customer_id = o.customer_id
GROUP BY c.customer_id, c.first_name, c.last_name, c.registered_at
HAVING COUNT(o.order_id) <= 1
ORDER BY c.registered_at;
```

### Pattern 3: หากลุ่มที่มีค่าเฉพาะ

```sql
-- ตัวอย่างที่ 17: หาแผนกที่มีทั้ง Manager และ Staff
SELECT 
    department_id,
    COUNT(*) AS total,
    COUNT(CASE WHEN manager_id IS NULL THEN 1 END) AS managers,
    COUNT(CASE WHEN manager_id IS NOT NULL THEN 1 END) AS staff
FROM employees
GROUP BY department_id
HAVING COUNT(CASE WHEN manager_id IS NULL THEN 1 END) > 0
    AND COUNT(CASE WHEN manager_id IS NOT NULL THEN 1 END) > 0;
```

### Pattern 4: กรองด้วยค่า MAX/MIN

```sql
-- ตัวอย่างที่ 18: หา category ที่มีสินค้าราคาสูงสุดมากกว่า 40,000 บาท
SELECT 
    category,
    MAX(price) AS max_price,
    COUNT(*) AS product_count
FROM products
GROUP BY category
HAVING MAX(price) > 40000
ORDER BY max_price DESC;
```

---

## 7. HAVING กับ NULL

```sql
-- ตัวอย่างที่ 19: จัดการ NULL ใน HAVING
-- หาแผนกที่มีพนักงานที่ไม่มีเงินเดือน (salary IS NULL)
SELECT 
    department_id,
    COUNT(*) AS total_employees,
    COUNT(salary) AS employees_with_salary,
    COUNT(*) - COUNT(salary) AS employees_without_salary
FROM employees
GROUP BY department_id
HAVING COUNT(*) - COUNT(salary) > 0;
```

```sql
-- ตัวอย่างที่ 20: หา category ที่มีสินค้า inactive
SELECT 
    category,
    COUNT(*) AS total,
    COUNT(CASE WHEN is_active = FALSE THEN 1 END) AS inactive_count
FROM products
GROUP BY category
HAVING COUNT(CASE WHEN is_active = FALSE THEN 1 END) > 0;
```

---

## 8. Performance Tips สำหรับ HAVING

```sql
-- ตัวอย่างที่ 21: ใช้ WHERE แทน HAVING เมื่อเป็นไปได้ (เร็วกว่า)

-- 🐌 ช้า: ใช้ HAVING กับเงื่อนไขที่ไม่ใช่ Aggregate
SELECT category, COUNT(*) AS cnt
FROM products
GROUP BY category
HAVING category = 'สมาร์ทโฟน';   -- ❌ ควรใช้ WHERE แทน

-- ⚡ เร็ว: ใช้ WHERE สำหรับเงื่อนไขปกติ
SELECT category, COUNT(*) AS cnt
FROM products
WHERE category = 'สมาร์ทโฟน'    -- ✅ กรองก่อน ลดข้อมูล
GROUP BY category;
```

```sql
-- ตัวอย่างที่ 22: Compound Filter - ใช้ WHERE และ HAVING ร่วมกันอย่างถูกต้อง
SELECT 
    p.category,
    COUNT(DISTINCT p.product_id) AS product_count,
    SUM(oi.quantity * oi.unit_price) AS revenue,
    AVG(oi.unit_price) AS avg_selling_price
FROM products p
JOIN order_items oi ON p.product_id = oi.product_id
JOIN orders o ON oi.order_id = o.order_id
WHERE o.status IN ('delivered', 'shipped')     -- WHERE: กรองออเดอร์ที่สำเร็จ
    AND o.order_date >= '2024-01-01'            -- WHERE: กรองปี 2024
GROUP BY p.category
HAVING SUM(oi.quantity * oi.unit_price) > 50000  -- HAVING: กรองยอดขาย
ORDER BY revenue DESC;
```

---

## 9. Common HAVING Patterns สำหรับธุรกิจ

```sql
-- ตัวอย่างที่ 23: หา VIP Customers (ซื้อมากกว่า 3 ครั้งและยอดรวม > 50,000)
SELECT 
    c.customer_id,
    CONCAT(c.first_name, ' ', c.last_name) AS customer_name,
    COUNT(o.order_id)    AS order_count,
    SUM(o.total_amount)  AS lifetime_value
FROM customers c
JOIN orders o ON c.customer_id = o.customer_id
WHERE o.status != 'cancelled'
GROUP BY c.customer_id, c.first_name, c.last_name
HAVING COUNT(o.order_id) >= 3
    AND SUM(o.total_amount) >= 50000
ORDER BY lifetime_value DESC;
```

```sql
-- ตัวอย่างที่ 24: หาสินค้าที่ขายดี (ขายได้มากกว่า 2 ชิ้น)
SELECT 
    p.product_id,
    p.product_name,
    p.category,
    SUM(oi.quantity) AS total_sold,
    SUM(oi.quantity * oi.unit_price) AS revenue
FROM products p
JOIN order_items oi ON p.product_id = oi.product_id
GROUP BY p.product_id, p.product_name, p.category
HAVING SUM(oi.quantity) >= 2
ORDER BY total_sold DESC;
```

```sql
-- ตัวอย่างที่ 25: หาแผนกที่มีงบประมาณเกินกว่าเงินเดือนรวม
SELECT 
    d.department_name,
    d.budget,
    SUM(e.salary) * 12 AS annual_payroll,
    d.budget - SUM(e.salary) * 12 AS remaining_budget
FROM departments d
JOIN employees e ON d.department_id = e.department_id
GROUP BY d.department_id, d.department_name, d.budget
HAVING d.budget > SUM(e.salary) * 12
ORDER BY remaining_budget DESC;
```

```sql
-- ตัวอย่างที่ 26: หาเดือนที่มีออเดอร์ที่ถูกยกเลิกมากกว่า 0
SELECT 
    YEAR(order_date)  AS year,
    MONTH(order_date) AS month,
    COUNT(*) AS total_orders,
    COUNT(CASE WHEN status = 'cancelled' THEN 1 END) AS cancelled_count,
    ROUND(
        COUNT(CASE WHEN status = 'cancelled' THEN 1 END) * 100.0 / COUNT(*),
        2
    ) AS cancellation_rate
FROM orders
GROUP BY YEAR(order_date), MONTH(order_date)
HAVING COUNT(CASE WHEN status = 'cancelled' THEN 1 END) > 0
ORDER BY year, month;
```

```sql
-- ตัวอย่างที่ 27: หา Products ที่มีส่วนลดเฉลี่ยมากกว่า 5%
SELECT 
    p.product_name,
    p.category,
    COUNT(oi.item_id) AS times_ordered,
    AVG(oi.discount_pct) AS avg_discount,
    SUM(oi.quantity * oi.unit_price) AS gross_revenue,
    SUM(oi.quantity * oi.unit_price * (1 - oi.discount_pct/100)) AS net_revenue
FROM products p
JOIN order_items oi ON p.product_id = oi.product_id
GROUP BY p.product_id, p.product_name, p.category
HAVING AVG(oi.discount_pct) > 2
ORDER BY avg_discount DESC;
```

```sql
-- ตัวอย่างที่ 28: หาลูกค้าที่ไม่ได้ซื้อในช่วง 3 เดือนล่าสุด (Lapsed Customers)
SELECT 
    c.customer_id,
    CONCAT(c.first_name, ' ', c.last_name) AS customer_name,
    MAX(o.order_date) AS last_order_date,
    DATEDIFF(CURDATE(), MAX(o.order_date)) AS days_since_last_order,
    COUNT(o.order_id) AS total_orders
FROM customers c
JOIN orders o ON c.customer_id = o.customer_id
WHERE o.status != 'cancelled'
GROUP BY c.customer_id, c.first_name, c.last_name
HAVING DATEDIFF(CURDATE(), MAX(o.order_date)) > 90
ORDER BY days_since_last_order DESC;
```

```sql
-- ตัวอย่างที่ 29: หา Payment Method ที่มี Avg Order Value ต่ำกว่าค่าเฉลี่ยรวม
SELECT 
    payment_method,
    COUNT(*) AS order_count,
    ROUND(AVG(total_amount), 2) AS avg_order_value
FROM orders
WHERE status NOT IN ('cancelled')
GROUP BY payment_method
HAVING AVG(total_amount) < (
    SELECT AVG(total_amount) 
    FROM orders 
    WHERE status NOT IN ('cancelled')
)
ORDER BY avg_order_value;
```

```sql
-- ตัวอย่างที่ 30: Complex HAVING - หา Category ที่มีทั้งสินค้าแพงและถูก
SELECT 
    category,
    COUNT(*) AS product_count,
    MIN(price) AS min_price,
    MAX(price) AS max_price,
    MAX(price) - MIN(price) AS price_range
FROM products
GROUP BY category
HAVING MIN(price) < 10000       -- มีสินค้าราคาต่ำกว่า 10,000
    AND MAX(price) > 30000      -- และมีสินค้าราคาสูงกว่า 30,000
ORDER BY price_range DESC;
```

---

## 10. HAVING กับ Multiple Aggregates

```sql
-- ตัวอย่างที่ 31: หาแผนกที่มีเงินเดือนเฉลี่ยสูงและมีพนักงานมาก
SELECT 
    d.department_name,
    COUNT(e.employee_id) AS headcount,
    ROUND(AVG(e.salary), 2) AS avg_salary,
    SUM(e.salary) AS total_salary
FROM departments d
JOIN employees e ON d.department_id = e.department_id
GROUP BY d.department_id, d.department_name
HAVING COUNT(e.employee_id) >= 2           -- อย่างน้อย 2 คน
    AND AVG(e.salary) >= 50000             -- เงินเดือนเฉลี่ย >= 50,000
ORDER BY avg_salary DESC;
```

---

## สรุปบทที่ 33

| เงื่อนไข | ใช้ WHERE หรือ HAVING? | เหตุผล |
|---------|---------------------|--------|
| `city = 'Bangkok'` | WHERE | เป็น column ปกติ |
| `price > 1000` | WHERE | เป็น column ปกติ |
| `COUNT(*) > 5` | HAVING | เป็น Aggregate |
| `SUM(amount) > 1000` | HAVING | เป็น Aggregate |
| `AVG(salary) > 50000` | HAVING | เป็น Aggregate |
| `MAX(date) > '2024-01-01'` | HAVING | เป็น Aggregate |

---

## แบบฝึกหัดท้ายบท

### แบบฝึกหัดที่ 1
หา category สินค้าที่มีจำนวนสินค้ามากกว่า 2 ชนิด

```sql
-- เฉลย:
SELECT category, COUNT(*) AS product_count
FROM products
GROUP BY category
HAVING COUNT(*) > 2
ORDER BY product_count DESC;
```

### แบบฝึกหัดที่ 2
หาลูกค้าที่ซื้อสินค้ารวมมากกว่า 100,000 บาท

```sql
-- เฉลย:
SELECT 
    c.customer_id,
    CONCAT(c.first_name, ' ', c.last_name) AS name,
    SUM(o.total_amount) AS total_spent
FROM customers c
JOIN orders o ON c.customer_id = o.customer_id
WHERE o.status != 'cancelled'
GROUP BY c.customer_id, c.first_name, c.last_name
HAVING SUM(o.total_amount) > 100000
ORDER BY total_spent DESC;
```

### แบบฝึกหัดที่ 3
หาเดือนที่มียอดขายต่ำกว่า 50,000 บาท (อาจต้องการกระตุ้นยอดขาย)

```sql
-- เฉลย:
SELECT 
    DATE_FORMAT(order_date, '%Y-%m') AS month,
    COUNT(*) AS order_count,
    SUM(total_amount) AS monthly_revenue
FROM orders
WHERE status NOT IN ('cancelled', 'pending')
GROUP BY DATE_FORMAT(order_date, '%Y-%m')
HAVING SUM(total_amount) < 50000
ORDER BY month;
```

### แบบฝึกหัดที่ 4
หา brand ที่มีราคาสินค้าเฉลี่ยสูงกว่าค่าเฉลี่ยของทุก brand

```sql
-- เฉลย:
SELECT 
    brand,
    ROUND(AVG(price), 2) AS avg_price
FROM products
GROUP BY brand
HAVING AVG(price) > (SELECT AVG(price) FROM products)
ORDER BY avg_price DESC;
```

### แบบฝึกหัดที่ 5
หา payment_method ที่มีจำนวนออเดอร์ที่ถูกยกเลิกมากกว่า 0

```sql
-- เฉลย:
SELECT 
    payment_method,
    COUNT(*) AS total_orders,
    COUNT(CASE WHEN status = 'cancelled' THEN 1 END) AS cancelled_count
FROM orders
GROUP BY payment_method
HAVING COUNT(CASE WHEN status = 'cancelled' THEN 1 END) > 0
ORDER BY cancelled_count DESC;
```

### แบบฝึกหัดที่ 6
หาสินค้าที่ถูกสั่งซื้อมากกว่า 1 ครั้งและมียอดขายมากกว่า 50,000 บาท

```sql
-- เฉลย:
SELECT 
    p.product_name,
    COUNT(DISTINCT oi.order_id) AS times_ordered,
    SUM(oi.quantity * oi.unit_price) AS total_revenue
FROM products p
JOIN order_items oi ON p.product_id = oi.product_id
GROUP BY p.product_id, p.product_name
HAVING COUNT(DISTINCT oi.order_id) > 1
    AND SUM(oi.quantity * oi.unit_price) > 50000
ORDER BY total_revenue DESC;
```

### แบบฝึกหัดที่ 7
หาแผนกที่มีพนักงานที่ไม่มีเงินเดือนระบุ

```sql
-- เฉลย:
SELECT 
    d.department_name,
    COUNT(e.employee_id) AS total_employees,
    COUNT(e.salary) AS with_salary,
    COUNT(e.employee_id) - COUNT(e.salary) AS without_salary
FROM departments d
JOIN employees e ON d.department_id = e.department_id
GROUP BY d.department_id, d.department_name
HAVING COUNT(e.employee_id) - COUNT(e.salary) > 0;
```

### แบบฝึกหัดที่ 8
หาจังหวัดที่มีลูกค้ามากกว่า 1 คน

```sql
-- เฉลย:
SELECT 
    province,
    COUNT(*) AS customer_count
FROM customers
GROUP BY province
HAVING COUNT(*) > 1
ORDER BY customer_count DESC;
```

### แบบฝึกหัดที่ 9
หาวันในสัปดาห์ที่มียอดขายเฉลี่ยต่อออเดอร์สูงกว่า 30,000 บาท

```sql
-- เฉลย:
SELECT 
    DAYNAME(order_date) AS day_name,
    COUNT(*) AS order_count,
    ROUND(AVG(total_amount), 2) AS avg_order_value
FROM orders
WHERE status NOT IN ('cancelled')
GROUP BY DAYNAME(order_date), DAYOFWEEK(order_date)
HAVING AVG(total_amount) > 30000
ORDER BY avg_order_value DESC;
```

### แบบฝึกหัดที่ 10
สร้าง Report: หา VIP Customers ที่ซื้อ >= 3 ครั้ง และใช้จ่ายรวม >= 50,000 บาท ในปี 2024

```sql
-- เฉลย:
SELECT 
    c.customer_id,
    CONCAT(c.first_name, ' ', c.last_name) AS customer_name,
    c.city,
    COUNT(o.order_id) AS order_count,
    SUM(o.total_amount) AS total_spent,
    MAX(o.order_date) AS last_purchase,
    c.loyalty_points
FROM customers c
JOIN orders o ON c.customer_id = o.customer_id
WHERE YEAR(o.order_date) = 2024
    AND o.status NOT IN ('cancelled')
GROUP BY c.customer_id, c.first_name, c.last_name, c.city, c.loyalty_points
HAVING COUNT(o.order_id) >= 3
    AND SUM(o.total_amount) >= 50000
ORDER BY total_spent DESC;
```

---

*จบบทที่ 033 - HAVING: การกรองข้อมูลที่ถูกจัดกลุ่มแล้ว*

*บทถัดไป: Part 034 - Advanced Aggregation Patterns*
