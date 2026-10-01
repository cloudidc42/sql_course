# Part 032: GROUP BY - การจัดกลุ่มข้อมูล

## บทนำ (Introduction)

`GROUP BY` เป็นคำสั่งที่ทรงพลังที่สุดอย่างหนึ่งใน SQL ช่วยให้เราสามารถ **จัดกลุ่มข้อมูล** และ **คำนวณสถิติ** ของแต่ละกลุ่มได้ เช่น:

- ยอดขายรายเดือน
- จำนวนพนักงานแต่ละแผนก
- สินค้าขายดีแยกตามหมวดหมู่

ในบทนี้เราจะเรียนรู้ GROUP BY อย่างครบถ้วน พร้อมตัวอย่างการสร้างรายงานธุรกิจจริงกว่า 40 ตัวอย่าง

---

## การตั้งค่าฐานข้อมูล (Database Setup)

```sql
-- ใช้ฐานข้อมูลเดิมจาก Part 031
USE ecommerce_db;

-- เพิ่มข้อมูลเพิ่มเติมเพื่อให้ตัวอย่างสมบูรณ์ขึ้น

-- เพิ่มออเดอร์เพิ่มเติม (ปี 2023 สำหรับเปรียบเทียบ)
INSERT INTO orders (customer_id, order_date, status, shipping_fee, discount_amt, total_amount, payment_method) VALUES
(1,  '2023-01-10 10:00:00', 'delivered', 50,    0, 35950, 'บัตรเครดิต'),
(3,  '2023-02-14 14:00:00', 'delivered', 50,  500, 42450, 'PromptPay'),
(5,  '2023-03-20 09:00:00', 'delivered',  0,    0,  9090, 'บัตรเดบิต'),
(7,  '2023-04-05 15:00:00', 'delivered', 50,    0, 59950, 'บัตรเครดิต'),
(9,  '2023-05-12 11:00:00', 'cancelled', 50,    0, 14950, 'PromptPay'),
(11, '2023-06-18 13:00:00', 'delivered',  0, 1000, 44900, 'บัตรเครดิต'),
(2,  '2023-07-22 10:00:00', 'delivered', 50,    0, 24950, 'บัตรเดบิต'),
(4,  '2023-08-30 16:00:00', 'delivered', 50,  500,  8540, 'PromptPay'),
(6,  '2023-09-15 09:00:00', 'shipped',    0,    0, 38950, 'บัตรเครดิต'),
(8,  '2023-10-20 14:00:00', 'delivered', 50,    0, 21950, 'บัตรเดบิต'),
(10, '2023-11-11 11:00:00', 'delivered',  0, 2000, 57950, 'บัตรเครดิต'),
(12, '2023-12-25 10:00:00', 'delivered', 50,    0, 15050, 'PromptPay');

-- เพิ่มข้อมูล order_items สำหรับออเดอร์ใหม่
INSERT INTO order_items (order_id, product_id, quantity, unit_price, discount_pct) VALUES
(26, 2,  1, 35900, 0),
(27, 3,  1, 42900, 0),
(28, 6,  1,  8990, 0),
(29, 8,  1, 59900, 0),
(30, 12, 1, 14900, 0),
(31, 5,  1, 38900, 0),
(31, 6,  1,  8990, 0),
(32, 10, 1, 24900, 0),
(33, 14, 1,  5490, 0),
(33, 19, 1,  3490, 0),
(34, 3,  1, 42900, 0),
(34, 7,  1,  9990, 0),
(35, 10, 1, 24900, 0),
(36, 12, 1, 14900, 0),
(36, 9,  1, 29900, 0),
(37, 8,  1, 59900, 0);
```

---

## 1. GROUP BY Syntax พื้นฐาน

### รูปแบบคำสั่ง

```sql
SELECT   column1, column2, aggregate_function(column3)
FROM     table_name
WHERE    condition              -- กรองก่อน GROUP BY
GROUP BY column1, column2      -- จัดกลุ่ม
HAVING   aggregate_condition   -- กรองหลัง GROUP BY
ORDER BY column1;              -- เรียงลำดับ
```

---

## 2. GROUP BY แบบคอลัมน์เดียว

```sql
-- ตัวอย่างที่ 1: จำนวนพนักงานแต่ละแผนก
SELECT 
    department_id,
    COUNT(*) AS employee_count
FROM employees
GROUP BY department_id
ORDER BY employee_count DESC;
```

```sql
-- ตัวอย่างที่ 2: ยอดขายแยกตาม status ออเดอร์
SELECT 
    status,
    COUNT(*)          AS order_count,
    SUM(total_amount) AS total_revenue
FROM orders
GROUP BY status
ORDER BY order_count DESC;
```

```sql
-- ตัวอย่างที่ 3: จำนวนสินค้าแต่ละหมวดหมู่
SELECT 
    category,
    COUNT(*)              AS product_count,
    SUM(stock_qty)        AS total_stock,
    AVG(price)            AS avg_price,
    MIN(price)            AS min_price,
    MAX(price)            AS max_price
FROM products
WHERE is_active = TRUE
GROUP BY category
ORDER BY product_count DESC;
```

```sql
-- ตัวอย่างที่ 4: จำนวนลูกค้าแต่ละจังหวัด
SELECT 
    province,
    COUNT(*) AS customer_count
FROM customers
GROUP BY province
ORDER BY customer_count DESC;
```

```sql
-- ตัวอย่างที่ 5: ยอดขายแยกตามวิธีชำระเงิน
SELECT 
    payment_method,
    COUNT(*)          AS order_count,
    SUM(total_amount) AS total_revenue,
    AVG(total_amount) AS avg_order_value
FROM orders
GROUP BY payment_method
ORDER BY total_revenue DESC;
```

---

## 3. GROUP BY หลายคอลัมน์

```sql
-- ตัวอย่างที่ 6: ยอดขายแยกตามปีและเดือน
SELECT 
    YEAR(order_date)  AS year,
    MONTH(order_date) AS month,
    COUNT(*)          AS order_count,
    SUM(total_amount) AS monthly_revenue
FROM orders
GROUP BY YEAR(order_date), MONTH(order_date)
ORDER BY year, month;
```

```sql
-- ตัวอย่างที่ 7: ยอดขายแยกตาม status และ payment_method
SELECT 
    status,
    payment_method,
    COUNT(*) AS count,
    SUM(total_amount) AS total
FROM orders
GROUP BY status, payment_method
ORDER BY status, payment_method;
```

```sql
-- ตัวอย่างที่ 8: สินค้าแยกตาม category และ brand
SELECT 
    category,
    brand,
    COUNT(*) AS product_count,
    AVG(price) AS avg_price,
    SUM(stock_qty * price) AS stock_value
FROM products
GROUP BY category, brand
ORDER BY category, brand;
```

```sql
-- ตัวอย่างที่ 9: พนักงานแยกตาม department และ job_title
SELECT 
    d.department_name,
    e.job_title,
    COUNT(*)       AS headcount,
    AVG(e.salary)  AS avg_salary,
    SUM(e.salary)  AS total_salary
FROM employees e
JOIN departments d ON e.department_id = d.department_id
GROUP BY d.department_name, e.job_title
ORDER BY d.department_name, headcount DESC;
```

```sql
-- ตัวอย่างที่ 10: ยอดขายรายปีและ status พร้อมเรียงลำดับ
SELECT 
    YEAR(order_date)   AS year,
    status,
    COUNT(*)           AS order_count,
    SUM(total_amount)  AS revenue,
    AVG(total_amount)  AS avg_order
FROM orders
GROUP BY YEAR(order_date), status
ORDER BY year DESC, revenue DESC;
```

---

## 4. GROUP BY กับ Expression (การคำนวณ)

```sql
-- ตัวอย่างที่ 11: จัดกลุ่มตามช่วงราคา
SELECT 
    CASE 
        WHEN price < 10000  THEN 'ต่ำกว่า 10,000'
        WHEN price < 25000  THEN '10,000-24,999'
        WHEN price < 50000  THEN '25,000-49,999'
        ELSE '50,000 ขึ้นไป'
    END AS price_range,
    COUNT(*) AS product_count,
    AVG(price) AS avg_price
FROM products
GROUP BY 
    CASE 
        WHEN price < 10000  THEN 'ต่ำกว่า 10,000'
        WHEN price < 25000  THEN '10,000-24,999'
        WHEN price < 50000  THEN '25,000-49,999'
        ELSE '50,000 ขึ้นไป'
    END
ORDER BY MIN(price);
```

```sql
-- ตัวอย่างที่ 12: GROUP BY วันในสัปดาห์ (Day of Week Analysis)
SELECT 
    DAYNAME(order_date) AS day_of_week,
    DAYOFWEEK(order_date) AS day_number,
    COUNT(*) AS order_count,
    SUM(total_amount) AS daily_revenue,
    AVG(total_amount) AS avg_order_value
FROM orders
GROUP BY DAYNAME(order_date), DAYOFWEEK(order_date)
ORDER BY day_number;
```

```sql
-- ตัวอย่างที่ 13: GROUP BY ไตรมาส
SELECT 
    YEAR(order_date)    AS year,
    QUARTER(order_date) AS quarter,
    COUNT(*)            AS order_count,
    SUM(total_amount)   AS quarterly_revenue
FROM orders
GROUP BY YEAR(order_date), QUARTER(order_date)
ORDER BY year, quarter;
```

```sql
-- ตัวอย่างที่ 14: GROUP BY ช่วงเวลาของวัน
SELECT 
    CASE 
        WHEN HOUR(order_date) BETWEEN  6 AND 11 THEN 'เช้า (06:00-11:59)'
        WHEN HOUR(order_date) BETWEEN 12 AND 17 THEN 'บ่าย (12:00-17:59)'
        WHEN HOUR(order_date) BETWEEN 18 AND 22 THEN 'เย็น (18:00-22:59)'
        ELSE 'ดึก (23:00-05:59)'
    END AS time_period,
    COUNT(*) AS order_count,
    SUM(total_amount) AS revenue
FROM orders
GROUP BY 
    CASE 
        WHEN HOUR(order_date) BETWEEN  6 AND 11 THEN 'เช้า (06:00-11:59)'
        WHEN HOUR(order_date) BETWEEN 12 AND 17 THEN 'บ่าย (12:00-17:59)'
        WHEN HOUR(order_date) BETWEEN 18 AND 22 THEN 'เย็น (18:00-22:59)'
        ELSE 'ดึก (23:00-05:59)'
    END
ORDER BY order_count DESC;
```

---

## 5. GROUP BY กับ CASE WHEN

```sql
-- ตัวอย่างที่ 15: จัดกลุ่มลูกค้าตาม Loyalty Tier
SELECT 
    CASE 
        WHEN loyalty_points >= 5000 THEN 'Platinum'
        WHEN loyalty_points >= 2000 THEN 'Gold'
        WHEN loyalty_points >= 500  THEN 'Silver'
        ELSE 'Bronze'
    END AS loyalty_tier,
    COUNT(*) AS customer_count,
    AVG(loyalty_points) AS avg_points
FROM customers
GROUP BY 
    CASE 
        WHEN loyalty_points >= 5000 THEN 'Platinum'
        WHEN loyalty_points >= 2000 THEN 'Gold'
        WHEN loyalty_points >= 500  THEN 'Silver'
        ELSE 'Bronze'
    END
ORDER BY MIN(loyalty_points) DESC;
```

```sql
-- ตัวอย่างที่ 16: จัดกลุ่มพนักงานตามช่วงเงินเดือน
SELECT 
    CASE 
        WHEN salary < 40000           THEN 'น้อยกว่า 40,000'
        WHEN salary BETWEEN 40000 AND 59999 THEN '40,000-59,999'
        WHEN salary BETWEEN 60000 AND 79999 THEN '60,000-79,999'
        WHEN salary >= 80000          THEN '80,000 ขึ้นไป'
        ELSE 'ไม่มีข้อมูล'
    END AS salary_band,
    COUNT(*) AS employee_count,
    MIN(salary) AS min_salary,
    MAX(salary) AS max_salary
FROM employees
GROUP BY 
    CASE 
        WHEN salary < 40000           THEN 'น้อยกว่า 40,000'
        WHEN salary BETWEEN 40000 AND 59999 THEN '40,000-59,999'
        WHEN salary BETWEEN 60000 AND 79999 THEN '60,000-79,999'
        WHEN salary >= 80000          THEN '80,000 ขึ้นไป'
        ELSE 'ไม่มีข้อมูล'
    END
ORDER BY MIN(COALESCE(salary, 0));
```

---

## 6. GROUP BY กับ JOIN

```sql
-- ตัวอย่างที่ 17: ยอดขายรวมต่อลูกค้า (พร้อมชื่อ)
SELECT 
    c.customer_id,
    CONCAT(c.first_name, ' ', c.last_name) AS customer_name,
    c.city,
    COUNT(o.order_id)    AS total_orders,
    SUM(o.total_amount)  AS total_spent,
    AVG(o.total_amount)  AS avg_order_value,
    MAX(o.order_date)    AS last_order_date
FROM customers c
LEFT JOIN orders o ON c.customer_id = o.customer_id
GROUP BY c.customer_id, c.first_name, c.last_name, c.city
ORDER BY total_spent DESC;
```

```sql
-- ตัวอย่างที่ 18: ยอดขายต่อสินค้า (พร้อมชื่อและ category)
SELECT 
    p.category,
    p.product_name,
    COUNT(oi.item_id)                  AS times_ordered,
    SUM(oi.quantity)                   AS total_qty_sold,
    SUM(oi.quantity * oi.unit_price)   AS total_revenue,
    p.stock_qty                        AS current_stock
FROM products p
LEFT JOIN order_items oi ON p.product_id = oi.product_id
GROUP BY p.product_id, p.category, p.product_name, p.stock_qty
ORDER BY total_revenue DESC;
```

```sql
-- ตัวอย่างที่ 19: เงินเดือนรวมต่อแผนก
SELECT 
    d.department_id,
    d.department_name,
    d.location,
    COUNT(e.employee_id) AS headcount,
    SUM(e.salary)        AS total_salary,
    AVG(e.salary)        AS avg_salary,
    d.budget,
    SUM(e.salary) * 12   AS annual_payroll,
    ROUND(SUM(e.salary) * 12 / d.budget * 100, 2) AS budget_utilization_pct
FROM departments d
LEFT JOIN employees e ON d.department_id = e.department_id
GROUP BY d.department_id, d.department_name, d.location, d.budget
ORDER BY total_salary DESC;
```

```sql
-- ตัวอย่างที่ 20: ยอดขายแยกตามเมืองของลูกค้า
SELECT 
    c.city,
    c.province,
    COUNT(DISTINCT c.customer_id) AS customer_count,
    COUNT(o.order_id)             AS total_orders,
    SUM(o.total_amount)           AS total_revenue,
    AVG(o.total_amount)           AS avg_order_value
FROM customers c
LEFT JOIN orders o ON c.customer_id = o.customer_id
GROUP BY c.city, c.province
ORDER BY total_revenue DESC;
```

---

## 7. GROUP BY Rules - กฎที่ต้องรู้

### กฎสำคัญ: คอลัมน์ที่ไม่ได้ Aggregate ต้องอยู่ใน GROUP BY

```sql
-- ❌ ผิด! first_name, last_name ไม่อยู่ใน GROUP BY
SELECT first_name, last_name, department_id, COUNT(*)
FROM employees
GROUP BY department_id;  -- ERROR!

-- ✅ ถูก!
SELECT department_id, COUNT(*) AS headcount
FROM employees
GROUP BY department_id;

-- ✅ ถูก! ใส่ทุกคอลัมน์ที่ไม่ใช่ Aggregate
SELECT first_name, last_name, department_id, COUNT(*) AS orders_handled
FROM employees
GROUP BY first_name, last_name, department_id;
```

```sql
-- ตัวอย่างที่ 21: วิธีถูกต้องในการ GROUP BY หลายคอลัมน์
SELECT 
    d.department_name,
    d.location,
    COUNT(e.employee_id) AS headcount,
    SUM(e.salary)        AS total_salary
FROM departments d
JOIN employees e ON d.department_id = e.department_id
GROUP BY d.department_id, d.department_name, d.location  -- ต้องรวม d.location ด้วย
ORDER BY headcount DESC;
```

---

## 8. GROUP BY กับ HAVING

```sql
-- ตัวอย่างที่ 22: หาแผนกที่มีพนักงานมากกว่า 2 คน
SELECT 
    department_id,
    COUNT(*) AS headcount
FROM employees
GROUP BY department_id
HAVING COUNT(*) > 2
ORDER BY headcount DESC;
```

```sql
-- ตัวอย่างที่ 23: หาลูกค้าที่ซื้อมากกว่า 2 ครั้ง
SELECT 
    customer_id,
    COUNT(*) AS order_count,
    SUM(total_amount) AS total_spent
FROM orders
WHERE status != 'cancelled'
GROUP BY customer_id
HAVING COUNT(*) >= 2
ORDER BY total_spent DESC;
```

```sql
-- ตัวอย่างที่ 24: หา category ที่มีมูลค่าสต็อกมากกว่า 500,000 บาท
SELECT 
    category,
    COUNT(*) AS product_count,
    SUM(price * stock_qty) AS stock_value
FROM products
WHERE is_active = TRUE
GROUP BY category
HAVING SUM(price * stock_qty) > 500000
ORDER BY stock_value DESC;
```

---

## 9. GROUP BY Performance

### เทคนิคการเพิ่มประสิทธิภาพ

```sql
-- ตัวอย่างที่ 25: Index ช่วย GROUP BY
-- ควรสร้าง Index บนคอลัมน์ที่ GROUP BY บ่อยๆ
CREATE INDEX idx_orders_customer ON orders(customer_id);
CREATE INDEX idx_orders_status ON orders(status);
CREATE INDEX idx_employees_dept ON employees(department_id);

-- ตอนนี้ Query เหล่านี้จะเร็วขึ้น:
SELECT customer_id, COUNT(*) FROM orders GROUP BY customer_id;
SELECT status, COUNT(*) FROM orders GROUP BY status;
SELECT department_id, COUNT(*) FROM employees GROUP BY department_id;
```

```sql
-- ตัวอย่างที่ 26: กรองด้วย WHERE ก่อน GROUP BY เพื่อลดข้อมูล
-- ดีกว่า: กรองก่อน
SELECT 
    YEAR(order_date) AS year,
    SUM(total_amount) AS revenue
FROM orders
WHERE order_date >= '2024-01-01'    -- กรองก่อน ลดแถวที่ต้องประมวลผล
GROUP BY YEAR(order_date);

-- แย่กว่า: ไม่กรองก่อน แล้วค่อยกรองใน HAVING
SELECT 
    YEAR(order_date) AS year,
    SUM(total_amount) AS revenue
FROM orders
GROUP BY YEAR(order_date)
HAVING YEAR(order_date) >= 2024;   -- ช้ากว่า เพราะต้อง process ข้อมูลทั้งหมดก่อน
```

---

## 10. GROUP BY vs DISTINCT

```sql
-- ตัวอย่างที่ 27: GROUP BY vs DISTINCT - ใช้เมื่อไหร่?

-- DISTINCT: ใช้เมื่อต้องการค่าไม่ซ้ำโดยไม่ต้อง Aggregate
SELECT DISTINCT category FROM products ORDER BY category;

-- GROUP BY: ใช้เมื่อต้องการ Aggregate ด้วย
SELECT category, COUNT(*) AS count FROM products GROUP BY category;

-- ทั้งสองแบบให้ผลเหมือนกันเมื่อไม่มี Aggregate:
SELECT DISTINCT department_id FROM employees ORDER BY department_id;
SELECT department_id FROM employees GROUP BY department_id ORDER BY department_id;
-- แต่ GROUP BY มักช้ากว่าสำหรับแค่ค่าไม่ซ้ำ (DISTINCT เร็วกว่า)
```

---

## 11. รายงานธุรกิจจริง (Real Business Reports)

### Report 1: Sales Dashboard รายเดือน

```sql
-- ตัวอย่างที่ 28: Monthly Sales Report
SELECT 
    DATE_FORMAT(order_date, '%Y-%m') AS month,
    COUNT(*)                         AS total_orders,
    COUNT(DISTINCT customer_id)      AS unique_customers,
    SUM(total_amount)                AS gross_revenue,
    SUM(discount_amt)                AS total_discounts,
    SUM(total_amount) - SUM(discount_amt) AS net_revenue,
    AVG(total_amount)                AS avg_order_value,
    SUM(shipping_fee)                AS shipping_revenue
FROM orders
WHERE status NOT IN ('cancelled')
GROUP BY DATE_FORMAT(order_date, '%Y-%m')
ORDER BY month;
```

### Report 2: Product Performance Report

```sql
-- ตัวอย่างที่ 29: สรุปประสิทธิภาพสินค้า
SELECT 
    p.category,
    COUNT(DISTINCT p.product_id)          AS products_in_category,
    SUM(oi.quantity)                      AS total_units_sold,
    SUM(oi.quantity * oi.unit_price)      AS total_revenue,
    SUM(oi.quantity * p.cost)             AS total_cost,
    SUM(oi.quantity * oi.unit_price) - 
    SUM(oi.quantity * p.cost)             AS gross_profit,
    ROUND(
        (SUM(oi.quantity * oi.unit_price) - SUM(oi.quantity * p.cost)) 
        / SUM(oi.quantity * oi.unit_price) * 100, 2
    )                                     AS gp_margin_pct
FROM products p
JOIN order_items oi ON p.product_id = oi.product_id
GROUP BY p.category
ORDER BY gross_profit DESC;
```

### Report 3: Customer Behavior Analysis

```sql
-- ตัวอย่างที่ 30: วิเคราะห์พฤติกรรมลูกค้า
SELECT 
    c.province,
    COUNT(DISTINCT c.customer_id)        AS total_customers,
    COUNT(o.order_id)                    AS total_orders,
    ROUND(COUNT(o.order_id) / 
          COUNT(DISTINCT c.customer_id), 2) AS orders_per_customer,
    SUM(o.total_amount)                  AS total_revenue,
    ROUND(SUM(o.total_amount) / 
          COUNT(DISTINCT c.customer_id), 2) AS revenue_per_customer,
    AVG(o.total_amount)                  AS avg_order_value
FROM customers c
LEFT JOIN orders o ON c.customer_id = o.customer_id 
    AND o.status != 'cancelled'
GROUP BY c.province
ORDER BY total_revenue DESC;
```

### Report 4: Employee Performance (Sales Staff)

```sql
-- ตัวอย่างที่ 31: สรุปข้อมูลพนักงานแต่ละแผนก
SELECT 
    d.department_name,
    COUNT(e.employee_id)                  AS headcount,
    ROUND(AVG(e.salary), 0)               AS avg_salary,
    MIN(e.salary)                         AS min_salary,
    MAX(e.salary)                         AS max_salary,
    SUM(e.salary)                         AS total_payroll_monthly,
    SUM(e.salary) * 12                    AS total_payroll_annual,
    COUNT(CASE WHEN e.manager_id IS NULL THEN 1 END) AS manager_count,
    COUNT(CASE WHEN e.manager_id IS NOT NULL THEN 1 END) AS staff_count
FROM departments d
JOIN employees e ON d.department_id = e.department_id
GROUP BY d.department_id, d.department_name
ORDER BY headcount DESC;
```

### Report 5: Inventory Report

```sql
-- ตัวอย่างที่ 32: รายงานสต็อกสินค้าแยก Category
SELECT 
    category,
    brand,
    COUNT(*) AS skus,
    SUM(stock_qty) AS total_units,
    SUM(CASE WHEN stock_qty < min_stock THEN 1 ELSE 0 END) AS low_stock_items,
    SUM(stock_qty * cost) AS inventory_cost,
    SUM(stock_qty * price) AS inventory_retail,
    SUM(stock_qty * (price - cost)) AS potential_profit
FROM products
WHERE is_active = TRUE
GROUP BY category, brand
ORDER BY category, inventory_retail DESC;
```

---

## 12. GROUP BY กับ Subquery

```sql
-- ตัวอย่างที่ 33: หา Category ที่มียอดขายสูงกว่าค่าเฉลี่ยของทุก Category
SELECT category, total_revenue
FROM (
    SELECT 
        p.category,
        SUM(oi.quantity * oi.unit_price) AS total_revenue
    FROM products p
    JOIN order_items oi ON p.product_id = oi.product_id
    GROUP BY p.category
) AS category_sales
WHERE total_revenue > (
    SELECT AVG(cat_rev) 
    FROM (
        SELECT SUM(oi.quantity * oi.unit_price) AS cat_rev
        FROM products p
        JOIN order_items oi ON p.product_id = oi.product_id
        GROUP BY p.category
    ) AS avg_calc
)
ORDER BY total_revenue DESC;
```

```sql
-- ตัวอย่างที่ 34: จัดอันดับแผนกตามเงินเดือนเฉลี่ย
SELECT 
    department_name,
    avg_salary,
    RANK() OVER (ORDER BY avg_salary DESC) AS salary_rank
FROM (
    SELECT 
        d.department_name,
        AVG(e.salary) AS avg_salary
    FROM departments d
    JOIN employees e ON d.department_id = e.department_id
    GROUP BY d.department_id, d.department_name
) AS dept_salary;
```

---

## 13. GROUP BY กับ WITH ROLLUP (Preview)

```sql
-- ตัวอย่างที่ 35: ดูผลรวมพร้อมยอดรวมทั้งหมด
SELECT 
    COALESCE(category, 'รวมทั้งหมด') AS category,
    COUNT(*) AS product_count,
    SUM(stock_qty * price) AS total_value
FROM products
WHERE is_active = TRUE
GROUP BY category WITH ROLLUP
ORDER BY category;
```

---

## 14. Advanced GROUP BY Patterns

```sql
-- ตัวอย่างที่ 36: Cohort Analysis - ยอดซื้อแยกตามปีที่ลงทะเบียน
SELECT 
    YEAR(c.registered_at)            AS cohort_year,
    COUNT(DISTINCT c.customer_id)    AS customers_in_cohort,
    COUNT(o.order_id)                AS total_orders,
    SUM(o.total_amount)              AS total_revenue,
    ROUND(SUM(o.total_amount) / COUNT(DISTINCT c.customer_id), 2) AS avg_revenue_per_customer
FROM customers c
LEFT JOIN orders o ON c.customer_id = o.customer_id
GROUP BY YEAR(c.registered_at)
ORDER BY cohort_year;
```

```sql
-- ตัวอย่างที่ 37: Running Revenue by Month (Ordered GROUP BY)
SELECT 
    DATE_FORMAT(order_date, '%Y-%m') AS month,
    COUNT(*) AS orders,
    SUM(total_amount) AS monthly_revenue
FROM orders
WHERE status NOT IN ('cancelled', 'pending')
GROUP BY DATE_FORMAT(order_date, '%Y-%m')
ORDER BY month;
```

```sql
-- ตัวอย่างที่ 38: Top Products ในแต่ละ Category
SELECT 
    p.category,
    p.product_name,
    SUM(oi.quantity * oi.unit_price) AS revenue
FROM products p
JOIN order_items oi ON p.product_id = oi.product_id
GROUP BY p.category, p.product_id, p.product_name
HAVING SUM(oi.quantity * oi.unit_price) = (
    SELECT MAX(cat_rev)
    FROM (
        SELECT p2.category, SUM(oi2.quantity * oi2.unit_price) AS cat_rev
        FROM products p2
        JOIN order_items oi2 ON p2.product_id = oi2.product_id
        WHERE p2.category = p.category
        GROUP BY p2.product_id
    ) AS max_calc
)
ORDER BY p.category;
```

```sql
-- ตัวอย่างที่ 39: วิเคราะห์การซื้อซ้ำ (Repeat Purchase Rate)
SELECT 
    purchase_frequency,
    COUNT(*) AS customer_count,
    ROUND(COUNT(*) * 100.0 / SUM(COUNT(*)) OVER (), 2) AS percentage
FROM (
    SELECT 
        customer_id,
        CASE 
            WHEN COUNT(*) = 1  THEN 'ซื้อครั้งเดียว'
            WHEN COUNT(*) = 2  THEN 'ซื้อ 2 ครั้ง'
            WHEN COUNT(*) <= 4 THEN 'ซื้อ 3-4 ครั้ง'
            ELSE 'ซื้อ 5+ ครั้ง'
        END AS purchase_frequency
    FROM orders
    WHERE status != 'cancelled'
    GROUP BY customer_id
) AS freq_data
GROUP BY purchase_frequency
ORDER BY MIN(
    CASE purchase_frequency
        WHEN 'ซื้อครั้งเดียว' THEN 1
        WHEN 'ซื้อ 2 ครั้ง'   THEN 2
        WHEN 'ซื้อ 3-4 ครั้ง' THEN 3
        ELSE 4
    END
);
```

```sql
-- ตัวอย่างที่ 40: Full Sales Analysis Report
SELECT 
    DATE_FORMAT(o.order_date, '%Y-%m')  AS month,
    p.category,
    COUNT(DISTINCT o.order_id)          AS order_count,
    COUNT(DISTINCT o.customer_id)       AS unique_customers,
    SUM(oi.quantity)                    AS units_sold,
    SUM(oi.quantity * oi.unit_price)    AS gross_revenue,
    SUM(o.discount_amt)                 AS discounts,
    SUM(oi.quantity * oi.unit_price) - SUM(o.discount_amt) AS net_revenue
FROM orders o
JOIN order_items oi ON o.order_id = oi.order_id
JOIN products p ON oi.product_id = p.product_id
WHERE o.status NOT IN ('cancelled')
GROUP BY DATE_FORMAT(o.order_date, '%Y-%m'), p.category
ORDER BY month, gross_revenue DESC;
```

---

## สรุปบทที่ 32 (Summary)

### กฎของ GROUP BY

1. **ทุกคอลัมน์ใน SELECT ที่ไม่ใช่ Aggregate ต้องอยู่ใน GROUP BY**
2. **ลำดับการทำงาน**: FROM → WHERE → GROUP BY → HAVING → SELECT → ORDER BY
3. **ใช้ WHERE กรองก่อน GROUP BY** เพื่อประสิทธิภาพที่ดีขึ้น
4. **ใช้ HAVING กรองหลัง GROUP BY** สำหรับเงื่อนไขที่เกี่ยวกับ Aggregate

---

## แบบฝึกหัดท้ายบท (Exercises)

### แบบฝึกหัดที่ 1
นับจำนวนพนักงานในแต่ละแผนก พร้อมแสดงชื่อแผนก เรียงตามจำนวนพนักงานมากที่สุดก่อน

```sql
-- เฉลย:
SELECT 
    d.department_name,
    COUNT(e.employee_id) AS employee_count
FROM departments d
LEFT JOIN employees e ON d.department_id = e.department_id
GROUP BY d.department_id, d.department_name
ORDER BY employee_count DESC;
```

### แบบฝึกหัดที่ 2
หายอดขายรวมแต่ละเดือนในปี 2024 เรียงตามเดือน

```sql
-- เฉลย:
SELECT 
    MONTH(order_date) AS month,
    COUNT(*) AS order_count,
    SUM(total_amount) AS monthly_revenue
FROM orders
WHERE YEAR(order_date) = 2024
    AND status NOT IN ('cancelled')
GROUP BY MONTH(order_date)
ORDER BY month;
```

### แบบฝึกหัดที่ 3
หาจำนวนสินค้าและราคาเฉลี่ยแต่ละ brand เรียงตามราคาเฉลี่ยมากที่สุดก่อน

```sql
-- เฉลย:
SELECT 
    brand,
    COUNT(*) AS product_count,
    ROUND(AVG(price), 2) AS avg_price,
    MIN(price) AS min_price,
    MAX(price) AS max_price
FROM products
WHERE is_active = TRUE
GROUP BY brand
ORDER BY avg_price DESC;
```

### แบบฝึกหัดที่ 4
หาลูกค้าที่มียอดซื้อรวมมากกว่า 50,000 บาท

```sql
-- เฉลย:
SELECT 
    c.customer_id,
    CONCAT(c.first_name, ' ', c.last_name) AS customer_name,
    COUNT(o.order_id) AS order_count,
    SUM(o.total_amount) AS total_spent
FROM customers c
JOIN orders o ON c.customer_id = o.customer_id
WHERE o.status != 'cancelled'
GROUP BY c.customer_id, c.first_name, c.last_name
HAVING SUM(o.total_amount) > 50000
ORDER BY total_spent DESC;
```

### แบบฝึกหัดที่ 5
นับจำนวนออเดอร์แยกตาม status และแสดงสัดส่วนเปอร์เซ็นต์

```sql
-- เฉลย:
SELECT 
    status,
    COUNT(*) AS order_count,
    ROUND(COUNT(*) * 100.0 / (SELECT COUNT(*) FROM orders), 2) AS percentage
FROM orders
GROUP BY status
ORDER BY order_count DESC;
```

### แบบฝึกหัดที่ 6
หา Top 3 category ที่มียอดขายสูงสุด

```sql
-- เฉลย:
SELECT 
    p.category,
    SUM(oi.quantity * oi.unit_price) AS total_revenue,
    SUM(oi.quantity) AS total_units
FROM products p
JOIN order_items oi ON p.product_id = oi.product_id
GROUP BY p.category
ORDER BY total_revenue DESC
LIMIT 3;
```

### แบบฝึกหัดที่ 7
วิเคราะห์ยอดขายแยกตามวันในสัปดาห์

```sql
-- เฉลย:
SELECT 
    DAYNAME(order_date) AS day_name,
    COUNT(*) AS order_count,
    SUM(total_amount) AS revenue,
    ROUND(AVG(total_amount), 2) AS avg_order_value
FROM orders
GROUP BY DAYNAME(order_date), DAYOFWEEK(order_date)
ORDER BY DAYOFWEEK(order_date);
```

### แบบฝึกหัดที่ 8
หาแผนกที่มีเงินเดือนเฉลี่ยสูงกว่าเงินเดือนเฉลี่ยของทั้งบริษัท

```sql
-- เฉลย:
SELECT 
    d.department_name,
    ROUND(AVG(e.salary), 2) AS avg_salary
FROM departments d
JOIN employees e ON d.department_id = e.department_id
GROUP BY d.department_id, d.department_name
HAVING AVG(e.salary) > (SELECT AVG(salary) FROM employees WHERE salary IS NOT NULL)
ORDER BY avg_salary DESC;
```

### แบบฝึกหัดที่ 9
สร้างรายงานสรุปยอดขายแยกตามจังหวัดของลูกค้า

```sql
-- เฉลย:
SELECT 
    c.province,
    COUNT(DISTINCT c.customer_id) AS customers,
    COUNT(o.order_id) AS total_orders,
    SUM(o.total_amount) AS revenue,
    ROUND(AVG(o.total_amount), 2) AS avg_order
FROM customers c
LEFT JOIN orders o ON c.customer_id = o.customer_id
    AND o.status NOT IN ('cancelled')
GROUP BY c.province
ORDER BY revenue DESC;
```

### แบบฝึกหัดที่ 10
จัดกลุ่มสินค้าตามช่วงราคา (< 10K, 10K-30K, 30K-50K, > 50K) และนับจำนวน

```sql
-- เฉลย:
SELECT 
    CASE 
        WHEN price < 10000  THEN 'ต่ำกว่า 10,000'
        WHEN price < 30000  THEN '10,000-29,999'
        WHEN price < 50000  THEN '30,000-49,999'
        ELSE '50,000 ขึ้นไป'
    END AS price_range,
    COUNT(*) AS product_count,
    ROUND(AVG(price), 0) AS avg_price
FROM products
WHERE is_active = TRUE
GROUP BY 
    CASE 
        WHEN price < 10000  THEN 'ต่ำกว่า 10,000'
        WHEN price < 30000  THEN '10,000-29,999'
        WHEN price < 50000  THEN '30,000-49,999'
        ELSE '50,000 ขึ้นไป'
    END
ORDER BY MIN(price);
```

---

*จบบทที่ 032 - GROUP BY: การจัดกลุ่มข้อมูล*

*บทถัดไป: Part 033 - HAVING: การกรองข้อมูลที่ถูกจัดกลุ่มแล้ว*
