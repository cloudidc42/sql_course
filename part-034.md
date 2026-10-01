# Part 034: Advanced Aggregation Patterns

## บทนำ (Introduction)

เมื่อเรียนรู้ Aggregate Functions พื้นฐานแล้ว ถึงเวลาก้าวไปสู่ **Advanced Aggregation Patterns** ซึ่งช่วยให้เราสร้าง queries ที่ซับซ้อนและทรงพลังขึ้น ได้แก่:

- **Conditional Aggregation** - คำนวณเฉพาะบางเงื่อนไข
- **Pivot Tables** - เปลี่ยน Rows เป็น Columns
- **Percentage Calculations** - คำนวณสัดส่วน
- **Running Totals** - ยอดสะสม
- **NULL Handling** ขั้นสูง

---

## การตั้งค่าฐานข้อมูล

```sql
USE ecommerce_db;

-- เพิ่มข้อมูลเพิ่มเติมสำหรับ Advanced Examples
-- เพิ่มข้อมูลออเดอร์รายไตรมาส 2023-2024
INSERT INTO orders (customer_id, order_date, status, shipping_fee, discount_amt, total_amount, payment_method) VALUES
-- Q1 2023
(1,  '2023-01-15', 'delivered',  50,    0, 45950, 'บัตรเครดิต'),
(3,  '2023-02-20', 'delivered',   0,  500,  8540, 'PromptPay'),
(5,  '2023-03-10', 'delivered',  50,    0, 35950, 'บัตรเดบิต'),
-- Q2 2023
(7,  '2023-04-05', 'delivered',  50,    0, 14950, 'บัตรเครดิต'),
(9,  '2023-05-12', 'cancelled',   0,    0, 24950, 'PromptPay'),
(11, '2023-06-18', 'delivered',  50, 1000, 59000, 'บัตรเครดิต'),
-- Q3 2023
(2,  '2023-07-22', 'delivered',   0,    0, 38950, 'บัตรเดบิต'),
(4,  '2023-08-30', 'delivered',  50,    0,  9090, 'PromptPay'),
(6,  '2023-09-15', 'shipped',    50,  500, 43500, 'บัตรเครดิต'),
-- Q4 2023
(8,  '2023-10-20', 'delivered',   0,    0, 21950, 'บัตรเดบิต'),
(10, '2023-11-11', 'delivered',  50, 2000, 57950, 'บัตรเครดิต'),
(12, '2023-12-25', 'delivered',   0,    0, 15050, 'PromptPay'),
-- Q1 2024 (เพิ่มเติม)
(13, '2024-01-18', 'delivered',  50,    0, 29000, 'บัตรเครดิต'),
(14, '2024-02-22', 'delivered',   0,  500, 24450, 'บัตรเดบิต'),
(15, '2024-03-28', 'delivered',  50,    0,  6990, 'PromptPay');

-- เพิ่ม order_items สำหรับออเดอร์ใหม่
INSERT INTO order_items (order_id, product_id, quantity, unit_price, discount_pct) VALUES
(46, 1,  1, 45900, 0),
(47, 14, 1,  5490, 0),
(47, 19, 1,  3490, 0),
(48, 2,  1, 35900, 0),
(49, 12, 1, 14900, 0),
(50, 9,  1, 29900, 0),
(51, 8,  1, 59900, 0),
(51, 6,  1,  8990, 0),
(52, 3,  1, 42900, 0),
(52, 7,  1,  9990, 0),
(53, 10, 1, 24900, 0),
(54, 5,  1, 38900, 0),
(54, 6,  1,  8990, 0),
(55, 10, 1, 24900, 0),
(55, 12, 1, 14900, 0),
(56, 9,  1, 29900, 0),
(57, 17, 1, 36900, 0),
(58, 7,  1,  9990, 0),
(58, 18, 1, 21900, 0),
(59, 19, 1, 14900, 0),
(60, 14, 1,  5490, 0),
(61, 4,  1, 28900, 0);
```

---

## 1. Conditional Aggregation (SUM CASE WHEN)

Conditional Aggregation ช่วยให้เราคำนวณสถิติที่มีเงื่อนไขในคำสั่ง SELECT เดียว

### 1.1 นับแบบมีเงื่อนไข

```sql
-- ตัวอย่างที่ 1: นับออเดอร์แยกตาม status ในแถวเดียว
SELECT 
    COUNT(*) AS total_orders,
    COUNT(CASE WHEN status = 'delivered'   THEN 1 END) AS delivered,
    COUNT(CASE WHEN status = 'processing'  THEN 1 END) AS processing,
    COUNT(CASE WHEN status = 'shipped'     THEN 1 END) AS shipped,
    COUNT(CASE WHEN status = 'pending'     THEN 1 END) AS pending,
    COUNT(CASE WHEN status = 'cancelled'   THEN 1 END) AS cancelled
FROM orders;
```

```sql
-- ตัวอย่างที่ 2: ใช้ SUM แทน COUNT สำหรับ Boolean Aggregation
-- (ให้ผลเหมือนกัน แต่ SUM อ่านได้ชัดเจนกว่าในบางกรณี)
SELECT 
    SUM(CASE WHEN status = 'delivered' THEN 1 ELSE 0 END) AS delivered,
    SUM(CASE WHEN status = 'cancelled' THEN 1 ELSE 0 END) AS cancelled,
    SUM(CASE WHEN is_active = TRUE     THEN 1 ELSE 0 END) AS active_products
FROM orders
CROSS JOIN (SELECT 1) AS dummy;  -- ตัวอย่างนี้ไม่ได้ใช้งานจริง ดูต่อไปด้านล่าง
```

```sql
-- ตัวอย่างที่ 3: Conditional COUNT ที่ถูกต้อง
SELECT 
    -- สถิติออเดอร์
    COUNT(*) AS total_orders,
    COUNT(CASE WHEN status = 'delivered' THEN 1 END) AS completed,
    COUNT(CASE WHEN status = 'cancelled' THEN 1 END) AS cancelled,
    
    -- สถิติทางการเงิน
    SUM(CASE WHEN status != 'cancelled' THEN total_amount ELSE 0 END) AS active_revenue,
    SUM(CASE WHEN status = 'cancelled'  THEN total_amount ELSE 0 END) AS lost_revenue,
    
    -- สถิติการชำระเงิน
    COUNT(CASE WHEN payment_method = 'บัตรเครดิต' THEN 1 END) AS credit_card_orders,
    COUNT(CASE WHEN payment_method = 'PromptPay'  THEN 1 END) AS promptpay_orders,
    COUNT(CASE WHEN payment_method = 'บัตรเดบิต'  THEN 1 END) AS debit_card_orders
FROM orders;
```

### 1.2 SUM ด้วยเงื่อนไข

```sql
-- ตัวอย่างที่ 4: รวมยอดขายแยกตาม payment method
SELECT 
    DATE_FORMAT(order_date, '%Y-%m') AS month,
    SUM(CASE WHEN payment_method = 'บัตรเครดิต' THEN total_amount ELSE 0 END) AS credit_card,
    SUM(CASE WHEN payment_method = 'PromptPay'  THEN total_amount ELSE 0 END) AS promptpay,
    SUM(CASE WHEN payment_method = 'บัตรเดบิต'  THEN total_amount ELSE 0 END) AS debit_card,
    SUM(total_amount) AS total
FROM orders
WHERE status NOT IN ('cancelled')
GROUP BY DATE_FORMAT(order_date, '%Y-%m')
ORDER BY month;
```

```sql
-- ตัวอย่างที่ 5: รวมยอดขายแยกตาม category ในแต่ละเดือน
SELECT 
    DATE_FORMAT(o.order_date, '%Y-%m') AS month,
    SUM(CASE WHEN p.category = 'สมาร์ทโฟน'      THEN oi.quantity * oi.unit_price ELSE 0 END) AS smartphone,
    SUM(CASE WHEN p.category = 'โน้ตบุ๊ค'         THEN oi.quantity * oi.unit_price ELSE 0 END) AS notebook,
    SUM(CASE WHEN p.category = 'แท็บเล็ต'         THEN oi.quantity * oi.unit_price ELSE 0 END) AS tablet,
    SUM(CASE WHEN p.category = 'หูฟัง'            THEN oi.quantity * oi.unit_price ELSE 0 END) AS headphone,
    SUM(CASE WHEN p.category = 'ทีวี'             THEN oi.quantity * oi.unit_price ELSE 0 END) AS tv,
    SUM(CASE WHEN p.category = 'เกมคอนโซล'        THEN oi.quantity * oi.unit_price ELSE 0 END) AS gaming,
    SUM(oi.quantity * oi.unit_price) AS total
FROM orders o
JOIN order_items oi ON o.order_id = oi.order_id
JOIN products p ON oi.product_id = p.product_id
WHERE o.status NOT IN ('cancelled')
GROUP BY DATE_FORMAT(o.order_date, '%Y-%m')
ORDER BY month;
```

---

## 2. Pivot Table ด้วย Conditional Aggregation

Pivot Table เปลี่ยนข้อมูลแบบ Rows ให้กลายเป็น Columns

```sql
-- ตัวอย่างที่ 6: Pivot - เงินเดือนแยกตามแผนกในแต่ละ Salary Band
SELECT 
    CASE 
        WHEN salary < 45000 THEN 'Band A (< 45K)'
        WHEN salary < 65000 THEN 'Band B (45K-65K)'
        WHEN salary < 85000 THEN 'Band C (65K-85K)'
        ELSE 'Band D (85K+)'
    END AS salary_band,
    COUNT(CASE WHEN department_id = 1 THEN 1 END) AS dept_sales,
    COUNT(CASE WHEN department_id = 2 THEN 1 END) AS dept_marketing,
    COUNT(CASE WHEN department_id = 3 THEN 1 END) AS dept_it,
    COUNT(CASE WHEN department_id = 4 THEN 1 END) AS dept_finance,
    COUNT(CASE WHEN department_id = 5 THEN 1 END) AS dept_warehouse,
    COUNT(*) AS total
FROM employees
WHERE salary IS NOT NULL
GROUP BY 
    CASE 
        WHEN salary < 45000 THEN 'Band A (< 45K)'
        WHEN salary < 65000 THEN 'Band B (45K-65K)'
        WHEN salary < 85000 THEN 'Band C (65K-85K)'
        ELSE 'Band D (85K+)'
    END
ORDER BY MIN(salary);
```

```sql
-- ตัวอย่างที่ 7: Pivot ยอดขายรายไตรมาส
SELECT 
    YEAR(order_date) AS year,
    SUM(CASE WHEN QUARTER(order_date) = 1 THEN total_amount ELSE 0 END) AS Q1,
    SUM(CASE WHEN QUARTER(order_date) = 2 THEN total_amount ELSE 0 END) AS Q2,
    SUM(CASE WHEN QUARTER(order_date) = 3 THEN total_amount ELSE 0 END) AS Q3,
    SUM(CASE WHEN QUARTER(order_date) = 4 THEN total_amount ELSE 0 END) AS Q4,
    SUM(total_amount) AS annual_total
FROM orders
WHERE status NOT IN ('cancelled')
GROUP BY YEAR(order_date)
ORDER BY year;
```

```sql
-- ตัวอย่างที่ 8: Crosstab - จำนวนออเดอร์แยกตาม status และเดือน
SELECT 
    MONTH(order_date) AS month,
    COUNT(CASE WHEN status = 'delivered'  THEN 1 END) AS delivered,
    COUNT(CASE WHEN status = 'shipped'    THEN 1 END) AS shipped,
    COUNT(CASE WHEN status = 'processing' THEN 1 END) AS processing,
    COUNT(CASE WHEN status = 'pending'    THEN 1 END) AS pending,
    COUNT(CASE WHEN status = 'cancelled'  THEN 1 END) AS cancelled,
    COUNT(*) AS total
FROM orders
WHERE YEAR(order_date) = 2024
GROUP BY MONTH(order_date)
ORDER BY month;
```

---

## 3. Percentage Calculations

```sql
-- ตัวอย่างที่ 9: เปอร์เซ็นต์ยอดขายต่อ category
SELECT 
    p.category,
    SUM(oi.quantity * oi.unit_price) AS category_revenue,
    SUM(SUM(oi.quantity * oi.unit_price)) OVER () AS total_revenue,
    ROUND(
        SUM(oi.quantity * oi.unit_price) * 100.0 / 
        SUM(SUM(oi.quantity * oi.unit_price)) OVER (),
        2
    ) AS revenue_pct
FROM products p
JOIN order_items oi ON p.product_id = oi.product_id
JOIN orders o ON oi.order_id = o.order_id
WHERE o.status NOT IN ('cancelled')
GROUP BY p.category
ORDER BY category_revenue DESC;
```

```sql
-- ตัวอย่างที่ 10: เปอร์เซ็นต์ยอดขายต่อ payment method
SELECT 
    payment_method,
    COUNT(*) AS order_count,
    SUM(total_amount) AS revenue,
    ROUND(COUNT(*) * 100.0 / (SELECT COUNT(*) FROM orders), 2) AS pct_of_orders,
    ROUND(SUM(total_amount) * 100.0 / (SELECT SUM(total_amount) FROM orders WHERE status != 'cancelled'), 2) AS pct_of_revenue
FROM orders
WHERE status != 'cancelled'
GROUP BY payment_method
ORDER BY revenue DESC;
```

```sql
-- ตัวอย่างที่ 11: Market Share ต่อ brand
SELECT 
    brand,
    COUNT(*) AS products,
    SUM(stock_qty * price) AS inventory_value,
    ROUND(
        SUM(stock_qty * price) * 100.0 / (SELECT SUM(stock_qty * price) FROM products WHERE is_active = TRUE),
        2
    ) AS market_share_pct
FROM products
WHERE is_active = TRUE
GROUP BY brand
ORDER BY inventory_value DESC;
```

---

## 4. Running Totals (ก่อนใช้ Window Functions)

```sql
-- ตัวอย่างที่ 12: Running Total ด้วย Self-Join (วิธีเก่าก่อน Window Functions)
SELECT 
    o1.order_id,
    o1.order_date,
    o1.total_amount,
    SUM(o2.total_amount) AS running_total
FROM orders o1
JOIN orders o2 ON o2.order_id <= o1.order_id
    AND o2.status NOT IN ('cancelled')
WHERE o1.status NOT IN ('cancelled')
GROUP BY o1.order_id, o1.order_date, o1.total_amount
ORDER BY o1.order_id
LIMIT 10;

-- หมายเหตุ: วิธีนี้ช้ามาก ใน Part 038 จะเรียน Window Functions ที่ดีกว่า
```

```sql
-- ตัวอย่างที่ 13: Running Total ต่อเดือนด้วย Subquery
SELECT 
    month,
    monthly_revenue,
    (SELECT SUM(monthly_revenue)
     FROM (
         SELECT DATE_FORMAT(order_date, '%Y-%m') AS month,
                SUM(total_amount) AS monthly_revenue
         FROM orders
         WHERE status NOT IN ('cancelled')
         GROUP BY DATE_FORMAT(order_date, '%Y-%m')
     ) AS inner_calc
     WHERE inner_calc.month <= outer_calc.month
    ) AS running_total
FROM (
    SELECT DATE_FORMAT(order_date, '%Y-%m') AS month,
           SUM(total_amount) AS monthly_revenue
    FROM orders
    WHERE status NOT IN ('cancelled')
    GROUP BY DATE_FORMAT(order_date, '%Y-%m')
) AS outer_calc
ORDER BY month;
```

---

## 5. NULL Handling ขั้นสูง

```sql
-- ตัวอย่างที่ 14: COALESCE เพื่อแทน NULL
SELECT 
    employee_id,
    first_name,
    last_name,
    COALESCE(salary, 0) AS salary_or_zero,
    COALESCE(CAST(salary AS CHAR), 'ไม่ระบุ') AS salary_display
FROM employees;
```

```sql
-- ตัวอย่างที่ 15: NULL-safe Comparison ใน Aggregation
-- SUM ของค่าที่อาจมี NULL
SELECT 
    department_id,
    COUNT(*) AS total_employees,
    COUNT(salary) AS with_salary,
    SUM(COALESCE(salary, 0)) AS total_salary_incl_null,
    SUM(salary) AS total_salary_excl_null,
    AVG(salary) AS avg_salary_excl_null,
    SUM(COALESCE(salary, 0)) / COUNT(*) AS avg_salary_incl_null
FROM employees
GROUP BY department_id;
```

```sql
-- ตัวอย่างที่ 16: NULLIF เพื่อป้องกัน Division by Zero
SELECT 
    p.category,
    SUM(oi.quantity * oi.unit_price) AS revenue,
    SUM(oi.quantity * p.cost) AS cost,
    -- ป้องกัน Division by Zero ด้วย NULLIF
    ROUND(
        (SUM(oi.quantity * oi.unit_price) - SUM(oi.quantity * p.cost)) * 100.0 /
        NULLIF(SUM(oi.quantity * oi.unit_price), 0),
        2
    ) AS gross_margin_pct
FROM products p
JOIN order_items oi ON p.product_id = oi.product_id
GROUP BY p.category;
```

---

## 6. Aggregating Multiple Levels

```sql
-- ตัวอย่างที่ 17: Multi-level Aggregation ด้วย Subquery
-- ยอดเฉลี่ยต่อลูกค้าต่อเดือน
SELECT 
    month,
    COUNT(DISTINCT customer_id) AS active_customers,
    SUM(monthly_spent) AS total_revenue,
    AVG(monthly_spent) AS avg_per_customer
FROM (
    SELECT 
        DATE_FORMAT(order_date, '%Y-%m') AS month,
        customer_id,
        SUM(total_amount) AS monthly_spent
    FROM orders
    WHERE status != 'cancelled'
    GROUP BY DATE_FORMAT(order_date, '%Y-%m'), customer_id
) AS customer_monthly
GROUP BY month
ORDER BY month;
```

```sql
-- ตัวอย่างที่ 18: Nested Aggregation - ค่าเฉลี่ยของค่าเฉลี่ย
SELECT 
    AVG(dept_avg) AS avg_department_salary
FROM (
    SELECT department_id, AVG(salary) AS dept_avg
    FROM employees
    WHERE salary IS NOT NULL
    GROUP BY department_id
) AS dept_averages;
```

```sql
-- ตัวอย่างที่ 19: สถิติการขายต่อ Category ต่อ Brand
SELECT 
    p.category,
    p.brand,
    COUNT(DISTINCT p.product_id) AS products,
    SUM(oi.quantity) AS total_sold,
    SUM(oi.quantity * oi.unit_price) AS revenue,
    AVG(oi.unit_price) AS avg_selling_price,
    -- เปรียบเทียบกับค่าเฉลี่ยของ category
    AVG(oi.unit_price) - AVG(AVG(oi.unit_price)) OVER (PARTITION BY p.category) AS vs_category_avg
FROM products p
JOIN order_items oi ON p.product_id = oi.product_id
GROUP BY p.category, p.brand
ORDER BY p.category, revenue DESC;
```

---

## 7. FILTER Clause (PostgreSQL)

```sql
-- หมายเหตุ: FILTER Clause ใช้ได้เฉพาะ PostgreSQL
-- MySQL และ SQL Server ใช้ CASE WHEN แทน

-- PostgreSQL FILTER syntax:
-- SELECT 
--     COUNT(*) AS total,
--     COUNT(*) FILTER (WHERE status = 'delivered') AS delivered,
--     SUM(total_amount) FILTER (WHERE status = 'delivered') AS delivered_revenue
-- FROM orders;

-- MySQL equivalent:
SELECT 
    COUNT(*) AS total,
    COUNT(CASE WHEN status = 'delivered' THEN 1 END) AS delivered,
    SUM(CASE WHEN status = 'delivered' THEN total_amount ELSE 0 END) AS delivered_revenue
FROM orders;
```

---

## 8. Advanced Business Intelligence Queries

```sql
-- ตัวอย่างที่ 20: Year-over-Year Growth Analysis
SELECT 
    YEAR(order_date) AS year,
    SUM(total_amount) AS revenue,
    LAG(SUM(total_amount)) OVER (ORDER BY YEAR(order_date)) AS prev_year_revenue,
    ROUND(
        (SUM(total_amount) - LAG(SUM(total_amount)) OVER (ORDER BY YEAR(order_date))) * 100.0 /
        LAG(SUM(total_amount)) OVER (ORDER BY YEAR(order_date)),
        2
    ) AS yoy_growth_pct
FROM orders
WHERE status NOT IN ('cancelled')
GROUP BY YEAR(order_date)
ORDER BY year;
```

```sql
-- ตัวอย่างที่ 21: Customer Lifetime Value (CLV) Analysis
SELECT 
    c.customer_id,
    CONCAT(c.first_name, ' ', c.last_name) AS customer_name,
    -- Frequency
    COUNT(o.order_id) AS purchase_count,
    -- Recency
    DATEDIFF(CURDATE(), MAX(o.order_date)) AS days_since_last_purchase,
    -- Monetary
    SUM(o.total_amount) AS lifetime_value,
    AVG(o.total_amount) AS avg_order_value,
    -- CLV Segment
    CASE 
        WHEN COUNT(o.order_id) >= 5 AND SUM(o.total_amount) >= 100000 THEN 'Champions'
        WHEN COUNT(o.order_id) >= 3 AND SUM(o.total_amount) >= 50000  THEN 'Loyal Customers'
        WHEN DATEDIFF(CURDATE(), MAX(o.order_date)) <= 90              THEN 'Recent Customers'
        WHEN DATEDIFF(CURDATE(), MAX(o.order_date)) > 180              THEN 'At Risk'
        ELSE 'Potential Loyalists'
    END AS customer_segment
FROM customers c
JOIN orders o ON c.customer_id = o.customer_id
WHERE o.status != 'cancelled'
GROUP BY c.customer_id, c.first_name, c.last_name
ORDER BY lifetime_value DESC;
```

```sql
-- ตัวอย่างที่ 22: Basket Analysis - ขนาดตะกร้าเฉลี่ย
SELECT 
    COUNT(DISTINCT o.order_id) AS total_orders,
    SUM(oi.quantity) AS total_items,
    COUNT(DISTINCT oi.product_id) AS distinct_products,
    ROUND(SUM(oi.quantity) / COUNT(DISTINCT o.order_id), 2) AS avg_items_per_order,
    ROUND(COUNT(DISTINCT oi.product_id) / COUNT(DISTINCT o.order_id), 2) AS avg_products_per_order,
    ROUND(SUM(o.total_amount) / COUNT(DISTINCT o.order_id), 2) AS avg_order_value
FROM orders o
JOIN order_items oi ON o.order_id = oi.order_id
WHERE o.status NOT IN ('cancelled');
```

```sql
-- ตัวอย่างที่ 23: สินค้าที่ขายพร้อมกัน (Frequently Bought Together)
SELECT 
    a.product_id AS product_a,
    b.product_id AS product_b,
    pa.product_name AS product_a_name,
    pb.product_name AS product_b_name,
    COUNT(*) AS times_bought_together
FROM order_items a
JOIN order_items b ON a.order_id = b.order_id 
    AND a.product_id < b.product_id    -- ป้องกัน duplicate pairs
JOIN products pa ON a.product_id = pa.product_id
JOIN products pb ON b.product_id = pb.product_id
GROUP BY a.product_id, b.product_id, pa.product_name, pb.product_name
HAVING COUNT(*) >= 2
ORDER BY times_bought_together DESC;
```

```sql
-- ตัวอย่างที่ 24: Revenue Attribution - ยอดขายจาก Repeat vs New Customers
SELECT 
    order_type,
    COUNT(*) AS order_count,
    SUM(total_amount) AS revenue,
    ROUND(AVG(total_amount), 2) AS avg_order_value
FROM (
    SELECT 
        o.order_id,
        o.total_amount,
        CASE 
            WHEN ROW_NUMBER() OVER (PARTITION BY o.customer_id ORDER BY o.order_date) = 1
            THEN 'First Purchase'
            ELSE 'Repeat Purchase'
        END AS order_type
    FROM orders o
    WHERE o.status != 'cancelled'
) AS classified_orders
GROUP BY order_type;
```

```sql
-- ตัวอย่างที่ 25: Price Sensitivity Analysis
SELECT 
    CASE 
        WHEN p.price < 10000  THEN 'Budget (< 10K)'
        WHEN p.price < 25000  THEN 'Mid-range (10K-25K)'
        WHEN p.price < 50000  THEN 'Premium (25K-50K)'
        ELSE 'Luxury (50K+)'
    END AS price_segment,
    COUNT(DISTINCT p.product_id) AS product_count,
    SUM(oi.quantity) AS units_sold,
    SUM(oi.quantity * oi.unit_price) AS revenue,
    ROUND(AVG(oi.unit_price), 0) AS avg_selling_price
FROM products p
JOIN order_items oi ON p.product_id = oi.product_id
GROUP BY 
    CASE 
        WHEN p.price < 10000  THEN 'Budget (< 10K)'
        WHEN p.price < 25000  THEN 'Mid-range (10K-25K)'
        WHEN p.price < 50000  THEN 'Premium (25K-50K)'
        ELSE 'Luxury (50K+)'
    END
ORDER BY MIN(p.price);
```

```sql
-- ตัวอย่างที่ 26: เปรียบเทียบยอดขาย Q1 2024 vs Q1 2023
SELECT 
    p.category,
    SUM(CASE WHEN YEAR(o.order_date) = 2024 AND QUARTER(o.order_date) = 1 
             THEN oi.quantity * oi.unit_price ELSE 0 END) AS q1_2024,
    SUM(CASE WHEN YEAR(o.order_date) = 2023 AND QUARTER(o.order_date) = 1 
             THEN oi.quantity * oi.unit_price ELSE 0 END) AS q1_2023,
    SUM(CASE WHEN YEAR(o.order_date) = 2024 AND QUARTER(o.order_date) = 1 
             THEN oi.quantity * oi.unit_price ELSE 0 END) -
    SUM(CASE WHEN YEAR(o.order_date) = 2023 AND QUARTER(o.order_date) = 1 
             THEN oi.quantity * oi.unit_price ELSE 0 END) AS growth,
    ROUND(
        (SUM(CASE WHEN YEAR(o.order_date) = 2024 AND QUARTER(o.order_date) = 1 
                  THEN oi.quantity * oi.unit_price ELSE 0 END) -
         SUM(CASE WHEN YEAR(o.order_date) = 2023 AND QUARTER(o.order_date) = 1 
                  THEN oi.quantity * oi.unit_price ELSE 0 END)) * 100.0 /
        NULLIF(SUM(CASE WHEN YEAR(o.order_date) = 2023 AND QUARTER(o.order_date) = 1 
                        THEN oi.quantity * oi.unit_price ELSE 0 END), 0),
        2
    ) AS growth_pct
FROM products p
JOIN order_items oi ON p.product_id = oi.product_id
JOIN orders o ON oi.order_id = o.order_id
WHERE o.status NOT IN ('cancelled')
GROUP BY p.category
ORDER BY q1_2024 DESC;
```

```sql
-- ตัวอย่างที่ 27: Complete Crosstab Report - ยอดขายแต่ละ Category ต่อเดือน (Pivot)
SELECT 
    DATE_FORMAT(o.order_date, '%Y-%m') AS month,
    SUM(CASE WHEN p.category = 'สมาร์ทโฟน'  THEN oi.quantity * oi.unit_price ELSE 0 END) AS smartphone_rev,
    SUM(CASE WHEN p.category = 'โน้ตบุ๊ค'    THEN oi.quantity * oi.unit_price ELSE 0 END) AS notebook_rev,
    SUM(CASE WHEN p.category = 'แท็บเล็ต'    THEN oi.quantity * oi.unit_price ELSE 0 END) AS tablet_rev,
    SUM(CASE WHEN p.category = 'หูฟัง'       THEN oi.quantity * oi.unit_price ELSE 0 END) AS headphone_rev,
    SUM(CASE WHEN p.category = 'ทีวี'        THEN oi.quantity * oi.unit_price ELSE 0 END) AS tv_rev,
    SUM(CASE WHEN p.category = 'เกมคอนโซล'   THEN oi.quantity * oi.unit_price ELSE 0 END) AS gaming_rev,
    SUM(oi.quantity * oi.unit_price) AS total_rev,
    COUNT(DISTINCT o.order_id) AS order_count
FROM orders o
JOIN order_items oi ON o.order_id = oi.order_id
JOIN products p ON oi.product_id = p.product_id
WHERE o.status NOT IN ('cancelled')
GROUP BY DATE_FORMAT(o.order_date, '%Y-%m')
ORDER BY month;
```

```sql
-- ตัวอย่างที่ 28: ABC Analysis สินค้า (Pareto Principle)
WITH product_revenue AS (
    SELECT 
        p.product_id,
        p.product_name,
        SUM(oi.quantity * oi.unit_price) AS revenue
    FROM products p
    JOIN order_items oi ON p.product_id = oi.product_id
    GROUP BY p.product_id, p.product_name
),
ranked AS (
    SELECT 
        product_id,
        product_name,
        revenue,
        SUM(revenue) OVER () AS total_revenue,
        SUM(revenue) OVER (ORDER BY revenue DESC) AS cumulative_revenue
    FROM product_revenue
)
SELECT 
    product_id,
    product_name,
    revenue,
    ROUND(revenue * 100.0 / total_revenue, 2) AS pct_of_total,
    ROUND(cumulative_revenue * 100.0 / total_revenue, 2) AS cumulative_pct,
    CASE 
        WHEN cumulative_revenue * 100.0 / total_revenue <= 80  THEN 'A - Top Products'
        WHEN cumulative_revenue * 100.0 / total_revenue <= 95  THEN 'B - Mid Products'
        ELSE 'C - Low Products'
    END AS abc_class
FROM ranked
ORDER BY revenue DESC;
```

```sql
-- ตัวอย่างที่ 29: Campaign Effectiveness - ส่วนลดเฉลี่ยและผลต่อยอดขาย
SELECT 
    CASE 
        WHEN discount_amt = 0              THEN 'ไม่มีส่วนลด'
        WHEN discount_amt BETWEEN 1 AND 500 THEN 'ส่วนลดน้อย (1-500)'
        WHEN discount_amt BETWEEN 501 AND 1500 THEN 'ส่วนลดปานกลาง (501-1500)'
        ELSE 'ส่วนลดมาก (1500+)'
    END AS discount_tier,
    COUNT(*) AS order_count,
    AVG(total_amount) AS avg_order_value,
    SUM(total_amount) AS total_revenue,
    AVG(discount_amt) AS avg_discount
FROM orders
WHERE status NOT IN ('cancelled')
GROUP BY 
    CASE 
        WHEN discount_amt = 0              THEN 'ไม่มีส่วนลด'
        WHEN discount_amt BETWEEN 1 AND 500 THEN 'ส่วนลดน้อย (1-500)'
        WHEN discount_amt BETWEEN 501 AND 1500 THEN 'ส่วนลดปานกลาง (501-1500)'
        ELSE 'ส่วนลดมาก (1500+)'
    END
ORDER BY AVG(discount_amt);
```

```sql
-- ตัวอย่างที่ 30: Cohort Retention Analysis
SELECT 
    cohort_month,
    COUNT(DISTINCT customer_id) AS cohort_size,
    COUNT(DISTINCT CASE WHEN order_month = cohort_month THEN customer_id END) AS month_0,
    COUNT(DISTINCT CASE WHEN PERIOD_DIFF(
        EXTRACT(YEAR_MONTH FROM order_month),
        EXTRACT(YEAR_MONTH FROM cohort_month)
    ) = 1 THEN customer_id END) AS month_1,
    COUNT(DISTINCT CASE WHEN PERIOD_DIFF(
        EXTRACT(YEAR_MONTH FROM order_month),
        EXTRACT(YEAR_MONTH FROM cohort_month)
    ) = 2 THEN customer_id END) AS month_2
FROM (
    SELECT 
        c.customer_id,
        MIN(DATE_FORMAT(o.order_date, '%Y-%m-01')) AS cohort_month,
        DATE_FORMAT(o.order_date, '%Y-%m-01') AS order_month
    FROM customers c
    JOIN orders o ON c.customer_id = o.customer_id
    WHERE o.status != 'cancelled'
    GROUP BY c.customer_id, DATE_FORMAT(o.order_date, '%Y-%m-01')
) AS cohort_data
GROUP BY cohort_month
ORDER BY cohort_month;
```

```sql
-- ตัวอย่างที่ 31: Stock Coverage Analysis
SELECT 
    p.category,
    p.product_name,
    p.stock_qty,
    p.min_stock,
    COALESCE(SUM(oi.quantity), 0) AS total_sold,
    CASE 
        WHEN COALESCE(SUM(oi.quantity), 0) = 0 THEN NULL
        ELSE ROUND(p.stock_qty / (COALESCE(SUM(oi.quantity), 0) / 
             NULLIF(COUNT(DISTINCT MONTH(o.order_date)), 0)), 1)
    END AS months_of_stock_coverage,
    CASE 
        WHEN p.stock_qty <= p.min_stock THEN '🔴 ต้องสั่งซื้อเร่งด่วน'
        WHEN p.stock_qty <= p.min_stock * 2 THEN '🟡 ควรสั่งซื้อเร็วๆ นี้'
        ELSE '🟢 สต็อกปกติ'
    END AS stock_status
FROM products p
LEFT JOIN order_items oi ON p.product_id = oi.product_id
LEFT JOIN orders o ON oi.order_id = o.order_id 
    AND o.status NOT IN ('cancelled')
WHERE p.is_active = TRUE
GROUP BY p.product_id, p.category, p.product_name, p.stock_qty, p.min_stock
ORDER BY p.category, months_of_stock_coverage;
```

```sql
-- ตัวอย่างที่ 32: Funnel Analysis
SELECT 
    'Total Customers Registered' AS stage,
    COUNT(*) AS count,
    100.0 AS pct
FROM customers
UNION ALL
SELECT 
    'Customers with at least 1 Order',
    COUNT(DISTINCT customer_id),
    COUNT(DISTINCT customer_id) * 100.0 / (SELECT COUNT(*) FROM customers)
FROM orders
WHERE status != 'cancelled'
UNION ALL
SELECT 
    'Customers with 2+ Orders',
    COUNT(*),
    COUNT(*) * 100.0 / (SELECT COUNT(*) FROM customers)
FROM (
    SELECT customer_id
    FROM orders
    WHERE status != 'cancelled'
    GROUP BY customer_id
    HAVING COUNT(*) >= 2
) AS repeat_buyers;
```

```sql
-- ตัวอย่างที่ 33: Revenue Mix Analysis - Weighted Average
SELECT 
    p.category,
    SUM(oi.quantity * oi.unit_price) AS revenue,
    SUM(oi.quantity * p.cost) AS total_cost,
    SUM(oi.quantity * (oi.unit_price - p.cost)) AS gross_profit,
    -- Weighted Average Margin
    ROUND(
        SUM(oi.quantity * (oi.unit_price - p.cost)) * 100.0 /
        NULLIF(SUM(oi.quantity * oi.unit_price), 0),
        2
    ) AS weighted_avg_margin_pct,
    -- Revenue Share
    ROUND(
        SUM(oi.quantity * oi.unit_price) * 100.0 /
        SUM(SUM(oi.quantity * oi.unit_price)) OVER (),
        2
    ) AS revenue_share_pct
FROM products p
JOIN order_items oi ON p.product_id = oi.product_id
JOIN orders o ON oi.order_id = o.order_id
WHERE o.status NOT IN ('cancelled')
GROUP BY p.category
ORDER BY revenue DESC;
```

```sql
-- ตัวอย่างที่ 34: Customer Purchase Pattern - Weekday vs Weekend
SELECT 
    CASE 
        WHEN DAYOFWEEK(order_date) IN (1, 7) THEN 'Weekend'
        ELSE 'Weekday'
    END AS day_type,
    COUNT(*) AS order_count,
    COUNT(DISTINCT customer_id) AS unique_customers,
    SUM(total_amount) AS revenue,
    ROUND(AVG(total_amount), 2) AS avg_order_value,
    ROUND(COUNT(*) * 100.0 / (SELECT COUNT(*) FROM orders), 2) AS pct_of_orders
FROM orders
WHERE status NOT IN ('cancelled')
GROUP BY 
    CASE 
        WHEN DAYOFWEEK(order_date) IN (1, 7) THEN 'Weekend'
        ELSE 'Weekday'
    END;
```

```sql
-- ตัวอย่างที่ 35: Complete Executive Summary Dashboard
SELECT 
    -- Revenue Metrics
    SUM(CASE WHEN YEAR(order_date) = 2024 AND status != 'cancelled' THEN total_amount ELSE 0 END) AS ytd_revenue_2024,
    SUM(CASE WHEN YEAR(order_date) = 2023 AND status != 'cancelled' THEN total_amount ELSE 0 END) AS ytd_revenue_2023,
    
    -- Order Metrics
    COUNT(CASE WHEN YEAR(order_date) = 2024 THEN 1 END) AS orders_2024,
    COUNT(CASE WHEN YEAR(order_date) = 2023 THEN 1 END) AS orders_2023,
    
    -- Customer Metrics
    COUNT(DISTINCT CASE WHEN YEAR(order_date) = 2024 THEN customer_id END) AS customers_2024,
    COUNT(DISTINCT CASE WHEN YEAR(order_date) = 2023 THEN customer_id END) AS customers_2023,
    
    -- Quality Metrics
    COUNT(CASE WHEN status = 'cancelled' THEN 1 END) AS total_cancellations,
    ROUND(COUNT(CASE WHEN status = 'cancelled' THEN 1 END) * 100.0 / COUNT(*), 2) AS cancellation_rate
FROM orders;
```

---

## สรุปบทที่ 34

### Patterns สำคัญที่เรียนในบทนี้

1. **Conditional Aggregation**: `SUM(CASE WHEN condition THEN value END)`
2. **Pivot Table**: ใช้ Conditional Aggregation เปลี่ยน Rows เป็น Columns
3. **Percentage**: `value * 100.0 / total` หรือ `value / SUM(value) OVER()`
4. **Running Total**: Self-join หรือ Window Function
5. **NULL Handling**: `COALESCE()`, `NULLIF()`, `ISNULL()`

---

## แบบฝึกหัดท้ายบท

### แบบฝึกหัดที่ 1
นับจำนวนออเดอร์แยกตาม status (delivered, pending, shipped, cancelled) ในแถวเดียว

```sql
-- เฉลย:
SELECT 
    COUNT(*) AS total,
    COUNT(CASE WHEN status = 'delivered' THEN 1 END) AS delivered,
    COUNT(CASE WHEN status = 'pending' THEN 1 END) AS pending,
    COUNT(CASE WHEN status = 'shipped' THEN 1 END) AS shipped,
    COUNT(CASE WHEN status = 'cancelled' THEN 1 END) AS cancelled
FROM orders;
```

### แบบฝึกหัดที่ 2
สร้าง Pivot Table แสดงยอดขายแต่ละ category แยกตาม brand

```sql
-- เฉลย:
SELECT 
    brand,
    SUM(CASE WHEN category = 'สมาร์ทโฟน' THEN price * stock_qty ELSE 0 END) AS smartphone,
    SUM(CASE WHEN category = 'โน้ตบุ๊ค'   THEN price * stock_qty ELSE 0 END) AS notebook,
    SUM(CASE WHEN category = 'แท็บเล็ต'   THEN price * stock_qty ELSE 0 END) AS tablet,
    SUM(price * stock_qty) AS total
FROM products
GROUP BY brand
ORDER BY total DESC;
```

### แบบฝึกหัดที่ 3
คำนวณสัดส่วนยอดขายต่อ category (เป็นเปอร์เซ็นต์)

```sql
-- เฉลย:
SELECT 
    p.category,
    SUM(oi.quantity * oi.unit_price) AS revenue,
    ROUND(
        SUM(oi.quantity * oi.unit_price) * 100.0 /
        (SELECT SUM(quantity * unit_price) FROM order_items),
        2
    ) AS revenue_pct
FROM products p
JOIN order_items oi ON p.product_id = oi.product_id
GROUP BY p.category
ORDER BY revenue DESC;
```

### แบบฝึกหัดที่ 4
หายอดขายรายไตรมาสแบบ Pivot (Q1, Q2, Q3, Q4)

```sql
-- เฉลย:
SELECT 
    YEAR(order_date) AS year,
    SUM(CASE WHEN QUARTER(order_date) = 1 THEN total_amount ELSE 0 END) AS Q1,
    SUM(CASE WHEN QUARTER(order_date) = 2 THEN total_amount ELSE 0 END) AS Q2,
    SUM(CASE WHEN QUARTER(order_date) = 3 THEN total_amount ELSE 0 END) AS Q3,
    SUM(CASE WHEN QUARTER(order_date) = 4 THEN total_amount ELSE 0 END) AS Q4,
    SUM(total_amount) AS annual
FROM orders
WHERE status NOT IN ('cancelled')
GROUP BY YEAR(order_date)
ORDER BY year;
```

### แบบฝึกหัดที่ 5
คำนวณ Gross Margin % แต่ละ category (ป้องกัน Division by Zero ด้วย NULLIF)

```sql
-- เฉลย:
SELECT 
    p.category,
    SUM(oi.quantity * oi.unit_price) AS revenue,
    SUM(oi.quantity * p.cost) AS cost,
    SUM(oi.quantity * (oi.unit_price - p.cost)) AS gross_profit,
    ROUND(
        SUM(oi.quantity * (oi.unit_price - p.cost)) * 100.0 /
        NULLIF(SUM(oi.quantity * oi.unit_price), 0),
        2
    ) AS gross_margin_pct
FROM products p
JOIN order_items oi ON p.product_id = oi.product_id
GROUP BY p.category
ORDER BY gross_margin_pct DESC;
```

### แบบฝึกหัดที่ 6
วิเคราะห์การซื้อ weekday vs weekend

```sql
-- เฉลย:
SELECT 
    CASE WHEN DAYOFWEEK(order_date) IN (1,7) THEN 'Weekend' ELSE 'Weekday' END AS type,
    COUNT(*) AS orders,
    SUM(total_amount) AS revenue,
    ROUND(AVG(total_amount), 2) AS avg_value
FROM orders
WHERE status != 'cancelled'
GROUP BY CASE WHEN DAYOFWEEK(order_date) IN (1,7) THEN 'Weekend' ELSE 'Weekday' END;
```

### แบบฝึกหัดที่ 7
Segment ลูกค้าตาม RFM (Recency, Frequency, Monetary)

```sql
-- เฉลย:
SELECT 
    c.customer_id,
    CONCAT(c.first_name, ' ', c.last_name) AS name,
    DATEDIFF(CURDATE(), MAX(o.order_date)) AS recency_days,
    COUNT(o.order_id) AS frequency,
    SUM(o.total_amount) AS monetary,
    CASE 
        WHEN DATEDIFF(CURDATE(), MAX(o.order_date)) <= 30 
            AND COUNT(o.order_id) >= 3 
            AND SUM(o.total_amount) >= 50000 THEN 'Champions'
        WHEN COUNT(o.order_id) >= 3 THEN 'Loyal'
        WHEN DATEDIFF(CURDATE(), MAX(o.order_date)) <= 30 THEN 'Recent'
        ELSE 'Others'
    END AS segment
FROM customers c
JOIN orders o ON c.customer_id = o.customer_id
WHERE o.status != 'cancelled'
GROUP BY c.customer_id, c.first_name, c.last_name
ORDER BY monetary DESC;
```

### แบบฝึกหัดที่ 8
นับจำนวนสินค้าที่ stock ต่ำกว่า min_stock แยกตาม category

```sql
-- เฉลย:
SELECT 
    category,
    COUNT(*) AS total_products,
    COUNT(CASE WHEN stock_qty < min_stock THEN 1 END) AS low_stock,
    COUNT(CASE WHEN stock_qty >= min_stock THEN 1 END) AS normal_stock
FROM products
WHERE is_active = TRUE
GROUP BY category
ORDER BY low_stock DESC;
```

### แบบฝึกหัดที่ 9
คำนวณเปอร์เซ็นต์การเติบโต YoY (Year over Year)

```sql
-- เฉลย:
SELECT 
    y2024.month_num AS month,
    y2024.revenue AS revenue_2024,
    y2023.revenue AS revenue_2023,
    ROUND((y2024.revenue - y2023.revenue) * 100.0 / NULLIF(y2023.revenue, 0), 2) AS growth_pct
FROM (
    SELECT MONTH(order_date) AS month_num, SUM(total_amount) AS revenue
    FROM orders WHERE YEAR(order_date) = 2024 AND status != 'cancelled'
    GROUP BY MONTH(order_date)
) AS y2024
LEFT JOIN (
    SELECT MONTH(order_date) AS month_num, SUM(total_amount) AS revenue
    FROM orders WHERE YEAR(order_date) = 2023 AND status != 'cancelled'
    GROUP BY MONTH(order_date)
) AS y2023 ON y2024.month_num = y2023.month_num
ORDER BY month;
```

### แบบฝึกหัดที่ 10
สร้าง Executive Dashboard สรุปข้อมูลสำคัญทั้งหมดในคำสั่งเดียว

```sql
-- เฉลย:
SELECT 
    SUM(CASE WHEN status != 'cancelled' THEN total_amount ELSE 0 END) AS total_revenue,
    COUNT(*) AS total_orders,
    COUNT(CASE WHEN status = 'delivered' THEN 1 END) AS completed_orders,
    COUNT(CASE WHEN status = 'cancelled' THEN 1 END) AS cancelled_orders,
    ROUND(COUNT(CASE WHEN status = 'cancelled' THEN 1 END) * 100.0 / COUNT(*), 2) AS cancellation_rate,
    COUNT(DISTINCT customer_id) AS unique_customers,
    ROUND(AVG(CASE WHEN status != 'cancelled' THEN total_amount END), 2) AS avg_order_value,
    SUM(CASE WHEN payment_method = 'บัตรเครดิต' THEN total_amount ELSE 0 END) AS credit_card_revenue,
    SUM(CASE WHEN payment_method = 'PromptPay' THEN total_amount ELSE 0 END) AS promptpay_revenue
FROM orders;
```

---

*จบบทที่ 034 - Advanced Aggregation Patterns*

*บทถัดไป: Part 035 - ROLLUP: การสร้าง Multi-level Subtotals*
