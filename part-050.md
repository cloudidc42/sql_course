# Part 50: Subquery Projects - Real-world Complex Queries

## บทนำ

บทนี้รวบรวม 10 โปรเจคจริงที่ใช้ subquery ขั้นสูง แต่ละโปรเจคมีบริบททางธุรกิจ ความท้าทาย วิธีแก้ปัญหาทีละขั้น และ query สมบูรณ์

---

## Project 1: Multi-level Business Analytics Dashboard

**บริบท:** ผู้บริหารต้องการ dashboard ที่แสดงภาพรวมธุรกิจแบบรวดเดียว

**ความท้าทาย:** ต้องรวมข้อมูลจากหลายมุมมองในคำสั่งเดียว

```sql
-- Business KPI Dashboard Query
SELECT
    -- Revenue Metrics
    (SELECT SUM(total_amount) FROM orders WHERE status = 'completed')
        AS total_revenue,
    (SELECT AVG(total_amount) FROM orders WHERE status = 'completed')
        AS avg_order_value,
    (SELECT SUM(total_amount)
     FROM   orders
     WHERE  status = 'completed'
       AND  order_date >= DATE_SUB(CURDATE(), INTERVAL 30 DAY))
        AS revenue_last_30_days,

    -- Customer Metrics
    (SELECT COUNT(*) FROM customers) AS total_customers,
    (SELECT COUNT(DISTINCT customer_id) FROM orders) AS active_customers,
    (SELECT COUNT(*) FROM customers
     WHERE customer_id NOT IN (SELECT DISTINCT customer_id FROM orders))
        AS never_ordered_customers,

    -- Product Metrics
    (SELECT COUNT(*) FROM products) AS total_products,
    (SELECT COUNT(DISTINCT product_id) FROM order_items) AS products_ever_sold,
    (SELECT COUNT(*) FROM products
     WHERE product_id NOT IN (SELECT DISTINCT product_id FROM order_items))
        AS unsold_products,
    (SELECT COUNT(*) FROM products WHERE stock_qty < 30) AS low_stock_count,

    -- Order Metrics
    (SELECT COUNT(*) FROM orders) AS total_orders,
    (SELECT COUNT(*) FROM orders WHERE status = 'pending') AS pending_orders,
    (SELECT COUNT(*) FROM orders WHERE status = 'completed') AS completed_orders,

    -- Top Performance
    (SELECT product_name FROM products p
     WHERE p.product_id = (
         SELECT product_id FROM order_items
         GROUP BY product_id ORDER BY SUM(quantity) DESC LIMIT 1
     )) AS best_selling_product,
    (SELECT CONCAT(first_name, ' ', last_name) FROM customers c
     WHERE c.customer_id = (
         SELECT customer_id FROM orders
         GROUP BY customer_id ORDER BY SUM(total_amount) DESC LIMIT 1
     )) AS top_customer
;
```

**ผลลัพธ์ที่ได้:**
```
total_revenue | avg_order_value | active_customers | pending_orders | best_selling_product | ...
217870.00     | 14524.67        | 9                | 2              | Laptop Pro 15        | ...
```

---

## Project 2: Customer Purchase Pattern Analysis

**บริบท:** ทีม CRM ต้องการเข้าใจพฤติกรรมการซื้อของลูกค้าแต่ละราย

**ความท้าทาย:** คำนวณ metrics หลายอย่างต่อลูกค้าพร้อมกัน

```sql
-- Customer Purchase Pattern Analysis
SELECT
    c.customer_id,
    CONCAT(c.first_name, ' ', c.last_name)        AS customer_name,
    c.city,

    -- Volume Metrics
    COALESCE((SELECT COUNT(*) FROM orders WHERE customer_id = c.customer_id), 0)
        AS total_orders,
    COALESCE((SELECT SUM(total_amount) FROM orders WHERE customer_id = c.customer_id), 0)
        AS total_spent,
    COALESCE((SELECT AVG(total_amount) FROM orders WHERE customer_id = c.customer_id), 0)
        AS avg_order_value,

    -- Time Metrics
    (SELECT MIN(order_date) FROM orders WHERE customer_id = c.customer_id)
        AS first_order_date,
    (SELECT MAX(order_date) FROM orders WHERE customer_id = c.customer_id)
        AS last_order_date,
    DATEDIFF(
        COALESCE((SELECT MAX(order_date) FROM orders WHERE customer_id = c.customer_id), CURDATE()),
        COALESCE((SELECT MIN(order_date) FROM orders WHERE customer_id = c.customer_id), CURDATE())
    ) AS days_as_customer,

    -- Category Preference
    (SELECT p.category
     FROM   order_items oi
     JOIN   orders o ON o.order_id = oi.order_id
     JOIN   products p ON p.product_id = oi.product_id
     WHERE  o.customer_id = c.customer_id
     GROUP  BY p.category
     ORDER  BY SUM(oi.quantity * oi.unit_price) DESC
     LIMIT  1) AS preferred_category,

    -- Segment
    CASE
        WHEN COALESCE((SELECT SUM(total_amount) FROM orders WHERE customer_id = c.customer_id), 0)
             > (SELECT AVG(cust_total) * 1.5
                FROM (SELECT customer_id, SUM(total_amount) AS cust_total
                      FROM orders GROUP BY customer_id) AS t)
        THEN 'VIP'
        WHEN COALESCE((SELECT COUNT(*) FROM orders WHERE customer_id = c.customer_id), 0) = 0
        THEN 'Prospect'
        ELSE 'Regular'
    END AS segment

FROM customers c
ORDER BY total_spent DESC;
```

---

## Project 3: Finding Customers Who Bought Everything in a Category (Relational Division)

**บริบท:** Marketing ต้องการหาลูกค้าที่ซื้อครบทุกสินค้าใน Electronics เพื่อโปรโมท bundle ใหม่

**ความท้าทาย:** Relational Division - หนึ่งในปัญหาที่ยากที่สุดใน SQL

**วิธีคิด:**
```
"ลูกค้า X ซื้อครบทุกสินค้าใน Electronics"
= "ไม่มีสินค้า Electronics ใดที่ลูกค้า X ยังไม่ได้ซื้อ"
= NOT EXISTS (สินค้า Electronics ที่ลูกค้า X ยังไม่ได้ซื้อ)
```

```sql
-- Step 1: หาสินค้าทั้งหมดใน Electronics
SELECT product_id, product_name FROM products WHERE category = 'Electronics';
-- ผล: 8 ชิ้น (product_id: 1, 2, 3, 10, 11, 12, 13, 14, 15)

-- Step 2: หาสินค้าที่ลูกค้าแต่ละคนเคยซื้อ
SELECT DISTINCT o.customer_id, oi.product_id
FROM   orders o JOIN order_items oi ON oi.order_id = o.order_id;

-- Step 3: Relational Division ด้วย NOT EXISTS ซ้อน
SELECT c.customer_id,
       CONCAT(c.first_name, ' ', c.last_name) AS customer_name
FROM   customers c
WHERE  NOT EXISTS (
    -- หาสินค้า Electronics ที่ลูกค้ายังไม่ได้ซื้อ
    SELECT p.product_id
    FROM   products p
    WHERE  p.category = 'Electronics'
      AND  NOT EXISTS (
          -- ตรวจว่าลูกค้าเคยซื้อสินค้านี้หรือไม่
          SELECT 1
          FROM   orders o
          JOIN   order_items oi ON oi.order_id = o.order_id
          WHERE  o.customer_id = c.customer_id
            AND  oi.product_id = p.product_id
      )
);

-- Alternative: Division ด้วย COUNT (เร็วกว่าบางครั้ง)
SELECT c.customer_id, CONCAT(c.first_name, ' ', c.last_name) AS customer_name
FROM   customers c
WHERE  (
    SELECT COUNT(DISTINCT oi.product_id)
    FROM   orders o
    JOIN   order_items oi ON oi.order_id = o.order_id
    JOIN   products p     ON p.product_id = oi.product_id
    WHERE  o.customer_id = c.customer_id
      AND  p.category = 'Electronics'
) = (
    SELECT COUNT(*)
    FROM   products
    WHERE  category = 'Electronics'
);
```

---

## Project 4: Inventory Reorder Analysis

**บริบท:** ทีม Logistics ต้องการ report สินค้าที่ควรสั่งซื้อเพิ่ม พร้อมข้อมูลประกอบการตัดสินใจ

```sql
-- Inventory Reorder Analysis Report
SELECT
    p.product_id,
    p.product_name,
    p.category,
    p.stock_qty AS current_stock,

    -- ยอดขายใน 30 วันล่าสุด
    COALESCE((
        SELECT SUM(oi.quantity)
        FROM   order_items oi
        JOIN   orders o ON o.order_id = oi.order_id
        WHERE  oi.product_id = p.product_id
          AND  o.order_date >= DATE_SUB(CURDATE(), INTERVAL 30 DAY)
    ), 0) AS sold_last_30d,

    -- ยอดขายเฉลี่ยต่อวัน
    ROUND(COALESCE((
        SELECT SUM(oi.quantity) / 30.0
        FROM   order_items oi
        JOIN   orders o ON o.order_id = oi.order_id
        WHERE  oi.product_id = p.product_id
          AND  o.order_date >= DATE_SUB(CURDATE(), INTERVAL 30 DAY)
    ), 0), 2) AS avg_daily_sales,

    -- วันที่ stock จะหมด (Days of Supply)
    CASE
        WHEN COALESCE((
            SELECT SUM(oi.quantity) / 30.0
            FROM   order_items oi
            JOIN   orders o ON o.order_id = oi.order_id
            WHERE  oi.product_id = p.product_id
              AND  o.order_date >= DATE_SUB(CURDATE(), INTERVAL 30 DAY)
        ), 0) = 0
        THEN 999
        ELSE ROUND(p.stock_qty / (
            SELECT SUM(oi.quantity) / 30.0
            FROM   order_items oi
            JOIN   orders o ON o.order_id = oi.order_id
            WHERE  oi.product_id = p.product_id
              AND  o.order_date >= DATE_SUB(CURDATE(), INTERVAL 30 DAY)
        ), 0)
    END AS days_of_supply,

    -- จำนวน pending orders
    COALESCE((
        SELECT SUM(oi.quantity)
        FROM   order_items oi
        JOIN   orders o ON o.order_id = oi.order_id
        WHERE  oi.product_id = p.product_id
          AND  o.status = 'pending'
    ), 0) AS pending_qty,

    -- คำแนะนำ
    CASE
        WHEN p.stock_qty = 0 THEN 'URGENT: Out of Stock'
        WHEN p.stock_qty < (
            SELECT COALESCE(SUM(oi.quantity) / 30.0 * 7, 5)
            FROM   order_items oi
            JOIN   orders o ON o.order_id = oi.order_id
            WHERE  oi.product_id = p.product_id
              AND  o.order_date >= DATE_SUB(CURDATE(), INTERVAL 30 DAY)
        ) THEN 'REORDER: Less than 7 days stock'
        WHEN p.stock_qty < (
            SELECT COALESCE(SUM(oi.quantity) / 30.0 * 14, 10)
            FROM   order_items oi
            JOIN   orders o ON o.order_id = oi.order_id
            WHERE  oi.product_id = p.product_id
              AND  o.order_date >= DATE_SUB(CURDATE(), INTERVAL 30 DAY)
        ) THEN 'MONITOR: Less than 14 days stock'
        ELSE 'OK'
    END AS reorder_action

FROM products p
ORDER BY days_of_supply ASC, current_stock ASC;
```

---

## Project 5: Employee Performance Ranking

**บริบท:** HR ต้องการ report ประเมินผลพนักงานเทียบกับ benchmark ต่างๆ

```sql
-- Employee Performance Ranking Report
SELECT
    e.employee_id,
    CONCAT(e.first_name, ' ', e.last_name)     AS employee_name,
    d.department_name,
    e.salary,

    -- เทียบกับ company average
    ROUND((SELECT AVG(salary) FROM employees), 2)
        AS company_avg,
    ROUND(e.salary / (SELECT AVG(salary) FROM employees) * 100 - 100, 1)
        AS pct_above_company_avg,

    -- เทียบกับ department average
    ROUND((SELECT AVG(salary) FROM employees WHERE department_id = e.department_id), 2)
        AS dept_avg,
    ROUND(e.salary / (SELECT AVG(salary) FROM employees WHERE department_id = e.department_id) * 100 - 100, 1)
        AS pct_above_dept_avg,

    -- Rank ในแผนก (1 = สูงสุด)
    (SELECT COUNT(*) + 1
     FROM   employees e2
     WHERE  e2.department_id = e.department_id
       AND  e2.salary > e.salary)
        AS dept_salary_rank,

    -- จำนวนพนักงานในแผนก
    (SELECT COUNT(*) FROM employees WHERE department_id = e.department_id)
        AS dept_size,

    -- Percentile ในบริษัท
    ROUND(
        (SELECT COUNT(*) FROM employees WHERE salary <= e.salary)
        * 100.0 / (SELECT COUNT(*) FROM employees),
        1
    ) AS company_percentile,

    -- ระดับเงินเดือน
    CASE
        WHEN e.salary >= (SELECT PERCENTILE FROM (
            SELECT salary AS PERCENTILE
            FROM employees
            ORDER BY salary DESC
            LIMIT 1 OFFSET FLOOR((SELECT COUNT(*) FROM employees) * 0.25)
        ) AS t) THEN 'Top 25%'
        WHEN e.salary >= (SELECT AVG(salary) FROM employees) THEN 'Above Average'
        ELSE 'Below Average'
    END AS salary_tier,

    -- วันทำงาน
    DATEDIFF(CURDATE(), e.hire_date) AS days_employed,
    ROUND(DATEDIFF(CURDATE(), e.hire_date) / 365.25, 1) AS years_employed

FROM employees e
JOIN departments d ON d.department_id = e.department_id
ORDER BY d.department_name, dept_salary_rank;
```

---

## Project 6: Sales Funnel Analysis

**บริบท:** Marketing ต้องการดู conversion funnel จากลูกค้าที่สมัครถึงการซื้อ

```sql
-- Sales Funnel Analysis
SELECT
    funnel_stage,
    customer_count,
    LAG(customer_count) OVER (ORDER BY stage_order) AS prev_stage_count,
    ROUND(
        customer_count * 100.0
        / FIRST_VALUE(customer_count) OVER (ORDER BY stage_order),
        1
    ) AS pct_of_top,
    ROUND(
        customer_count * 100.0
        / LAG(customer_count) OVER (ORDER BY stage_order),
        1
    ) AS conversion_from_prev
FROM (
    SELECT 1 AS stage_order, 'All Registered Customers' AS funnel_stage,
           (SELECT COUNT(*) FROM customers) AS customer_count

    UNION ALL
    SELECT 2, 'Customers Who Visited (any order attempt)',
           (SELECT COUNT(DISTINCT customer_id) FROM orders)

    UNION ALL
    SELECT 3, 'Customers With Completed Order',
           (SELECT COUNT(DISTINCT customer_id) FROM orders WHERE status = 'completed')

    UNION ALL
    SELECT 4, 'Customers With 2+ Completed Orders',
           (SELECT COUNT(*) FROM (
               SELECT customer_id FROM orders WHERE status = 'completed'
               GROUP BY customer_id HAVING COUNT(*) >= 2
           ) AS t)

    UNION ALL
    SELECT 5, 'Customers With Orders > 20000',
           (SELECT COUNT(DISTINCT customer_id) FROM orders
            WHERE status = 'completed' AND total_amount > 20000)
) AS funnel
ORDER BY stage_order;
```

---

## Project 7: Product Cross-selling Opportunities

**บริบท:** ต้องการหาคู่สินค้าที่มักถูกซื้อพร้อมกัน เพื่อแนะนำ cross-selling

```sql
-- Product Co-purchase Analysis (Market Basket Analysis)
SELECT
    p1.product_name  AS product_a,
    p2.product_name  AS product_b,
    p1.category      AS category_a,
    p2.category      AS category_b,
    co_purchase.times_bought_together,

    -- Support: % ของ orders ที่มีทั้งสองสินค้า
    ROUND(
        co_purchase.times_bought_together * 100.0
        / (SELECT COUNT(*) FROM orders),
        1
    ) AS support_pct,

    -- Lift: ซื้อร่วมกันมากกว่าที่คาดเท่าไหร่
    ROUND(
        co_purchase.times_bought_together * 1.0
        / (
            (SELECT COUNT(DISTINCT oi1.order_id) FROM order_items oi1 WHERE oi1.product_id = p1.product_id)
            * (SELECT COUNT(DISTINCT oi2.order_id) FROM order_items oi2 WHERE oi2.product_id = p2.product_id)
            / (SELECT COUNT(*) FROM orders)
        ),
        2
    ) AS lift

FROM (
    -- หาทุกคู่สินค้าที่ถูกสั่งพร้อมกันในออเดอร์เดียว
    SELECT oi1.product_id AS product_id_a,
           oi2.product_id AS product_id_b,
           COUNT(DISTINCT oi1.order_id) AS times_bought_together
    FROM   order_items oi1
    JOIN   order_items oi2 ON oi2.order_id = oi1.order_id
                           AND oi2.product_id > oi1.product_id  -- หลีกเลี่ยง duplicate
    GROUP  BY oi1.product_id, oi2.product_id
    HAVING COUNT(DISTINCT oi1.order_id) >= 1
) AS co_purchase
JOIN products p1 ON p1.product_id = co_purchase.product_id_a
JOIN products p2 ON p2.product_id = co_purchase.product_id_b
ORDER BY times_bought_together DESC, lift DESC;
```

---

## Project 8: Revenue Attribution by Customer Cohort

**บริบท:** Finance ต้องการดูว่า revenue แต่ละเดือนมาจาก cohort ลูกค้าไหน

```sql
-- Cohort Revenue Attribution
SELECT
    cohort.cohort_month,
    cohort.cohort_size,

    -- Revenue ใน 3 เดือนแรกของ cohort
    COALESCE((
        SELECT SUM(o.total_amount)
        FROM   customers c
        JOIN   orders o ON o.customer_id = c.customer_id
        WHERE  DATE_FORMAT(c.created_at, '%Y-%m') = cohort.cohort_month
          AND  o.order_date <= DATE_ADD(
                   DATE_FORMAT(c.created_at, '%Y-%m-01'),
                   INTERVAL 3 MONTH
               )
    ), 0) AS revenue_first_3m,

    -- Lifetime Revenue ของ cohort
    COALESCE((
        SELECT SUM(o.total_amount)
        FROM   customers c
        JOIN   orders o ON o.customer_id = c.customer_id
        WHERE  DATE_FORMAT(c.created_at, '%Y-%m') = cohort.cohort_month
    ), 0) AS lifetime_revenue,

    -- Average Revenue per Customer ใน cohort
    ROUND(COALESCE((
        SELECT SUM(o.total_amount) / COUNT(DISTINCT c.customer_id)
        FROM   customers c
        JOIN   orders o ON o.customer_id = c.customer_id
        WHERE  DATE_FORMAT(c.created_at, '%Y-%m') = cohort.cohort_month
    ), 0), 2) AS arpc,

    -- % ที่ซื้อใน 3 เดือนแรก
    ROUND(COALESCE((
        SELECT COUNT(DISTINCT c.customer_id) * 100.0 / cohort.cohort_size
        FROM   customers c
        WHERE  DATE_FORMAT(c.created_at, '%Y-%m') = cohort.cohort_month
          AND  EXISTS (
              SELECT 1 FROM orders o
              WHERE o.customer_id = c.customer_id
                AND o.order_date <= DATE_ADD(
                        DATE_FORMAT(c.created_at, '%Y-%m-01'),
                        INTERVAL 3 MONTH
                    )
          )
    ), 0), 1) AS conversion_rate_3m

FROM (
    SELECT DATE_FORMAT(created_at, '%Y-%m') AS cohort_month,
           COUNT(*) AS cohort_size
    FROM   customers
    GROUP  BY DATE_FORMAT(created_at, '%Y-%m')
) AS cohort
ORDER BY cohort_month;
```

---

## Project 9: Competitive Price Analysis

**บริบท:** ทีม Product ต้องการ report ราคาสินค้าพร้อมการวิเคราะห์เชิงลึก

```sql
-- Competitive Price Analysis Dashboard
SELECT
    p.product_id,
    p.product_name,
    p.category,
    p.price AS current_price,

    -- Category Statistics
    (SELECT MIN(price) FROM products WHERE category = p.category) AS cat_min_price,
    (SELECT MAX(price) FROM products WHERE category = p.category) AS cat_max_price,
    ROUND((SELECT AVG(price) FROM products WHERE category = p.category), 2) AS cat_avg_price,
    ROUND((SELECT STDDEV(price) FROM products WHERE category = p.category), 2) AS cat_price_stddev,

    -- Position within category
    (SELECT COUNT(*) + 1 FROM products WHERE category = p.category AND price > p.price)
        AS rank_lowest_in_cat,
    (SELECT COUNT(*) FROM products WHERE category = p.category) AS cat_product_count,

    -- Price vs Average
    ROUND(p.price - (SELECT AVG(price) FROM products WHERE category = p.category), 2)
        AS vs_cat_avg,
    ROUND(
        (p.price - (SELECT AVG(price) FROM products WHERE category = p.category))
        / NULLIF((SELECT AVG(price) FROM products WHERE category = p.category), 0) * 100,
        1
    ) AS pct_vs_cat_avg,

    -- Sales performance (revenue per price point)
    COALESCE((
        SELECT SUM(oi.quantity * oi.unit_price)
        FROM   order_items oi
        WHERE  oi.product_id = p.product_id
    ), 0) AS total_revenue,

    COALESCE((
        SELECT SUM(oi.quantity)
        FROM   order_items oi
        WHERE  oi.product_id = p.product_id
    ), 0) AS units_sold,

    -- Price elasticity indicator
    CASE
        WHEN p.price > (SELECT AVG(price) FROM products WHERE category = p.category) * 1.2
            AND COALESCE((SELECT SUM(quantity) FROM order_items WHERE product_id = p.product_id), 0)
                > (SELECT AVG(qty) FROM (
                    SELECT product_id, SUM(quantity) AS qty FROM order_items GROUP BY product_id
                   ) AS t)
        THEN 'Premium & Popular'
        WHEN p.price > (SELECT AVG(price) FROM products WHERE category = p.category) * 1.2
        THEN 'Premium, Consider Price Review'
        WHEN p.price < (SELECT AVG(price) FROM products WHERE category = p.category) * 0.8
        THEN 'Value, Room to Increase'
        ELSE 'Market Rate'
    END AS pricing_strategy

FROM products p
ORDER BY p.category, p.price DESC;
```

---

## Project 10: Customer Lifetime Value Prediction Model

**บริบท:** ทีม Data Analytics ต้องการคำนวณและพยากรณ์ Customer Lifetime Value (CLV)

```sql
-- Customer Lifetime Value Analysis
SELECT
    c.customer_id,
    CONCAT(c.first_name, ' ', c.last_name) AS customer_name,
    c.city,
    c.created_at AS join_date,

    -- Historical Metrics
    COALESCE((
        SELECT COUNT(*)
        FROM orders WHERE customer_id = c.customer_id
    ), 0) AS total_orders,

    COALESCE((
        SELECT SUM(total_amount)
        FROM orders WHERE customer_id = c.customer_id
    ), 0) AS total_revenue,

    COALESCE((
        SELECT AVG(total_amount)
        FROM orders WHERE customer_id = c.customer_id
    ), 0) AS avg_order_value,

    -- Purchase Frequency (orders per month since joining)
    ROUND(COALESCE((
        SELECT COUNT(*) / GREATEST(DATEDIFF(CURDATE(), MIN(c2.created_at)) / 30.0, 1)
        FROM orders o
        JOIN customers c2 ON c2.customer_id = o.customer_id
        WHERE o.customer_id = c.customer_id
    ), 0), 2) AS orders_per_month,

    -- Recency (days since last order)
    COALESCE(
        DATEDIFF(CURDATE(), (
            SELECT MAX(order_date) FROM orders WHERE customer_id = c.customer_id
        )),
        DATEDIFF(CURDATE(), c.created_at)
    ) AS days_since_last_order,

    -- Projected Annual Value (avg_order_value * orders_per_month * 12)
    ROUND(
        COALESCE((SELECT AVG(total_amount) FROM orders WHERE customer_id = c.customer_id), 0)
        * COALESCE((
            SELECT COUNT(*) / GREATEST(DATEDIFF(CURDATE(), c.created_at) / 30.0, 1)
            FROM orders WHERE customer_id = c.customer_id
        ), 0) * 12,
        2
    ) AS projected_annual_value,

    -- RFM Score (Recency, Frequency, Monetary)
    CONCAT(
        -- Recency score (1-3, 3 = most recent)
        CASE
            WHEN COALESCE(DATEDIFF(CURDATE(), (SELECT MAX(order_date) FROM orders WHERE customer_id = c.customer_id)), 999) <= 30 THEN '3'
            WHEN COALESCE(DATEDIFF(CURDATE(), (SELECT MAX(order_date) FROM orders WHERE customer_id = c.customer_id)), 999) <= 90 THEN '2'
            ELSE '1'
        END,
        '-',
        -- Frequency score (1-3, 3 = most frequent)
        CASE
            WHEN COALESCE((SELECT COUNT(*) FROM orders WHERE customer_id = c.customer_id), 0) >= 3 THEN '3'
            WHEN COALESCE((SELECT COUNT(*) FROM orders WHERE customer_id = c.customer_id), 0) >= 2 THEN '2'
            ELSE '1'
        END,
        '-',
        -- Monetary score (1-3, 3 = highest value)
        CASE
            WHEN COALESCE((SELECT SUM(total_amount) FROM orders WHERE customer_id = c.customer_id), 0)
                 >= (SELECT AVG(cust_total) * 1.5 FROM (SELECT customer_id, SUM(total_amount) AS cust_total FROM orders GROUP BY customer_id) AS t)
            THEN '3'
            WHEN COALESCE((SELECT SUM(total_amount) FROM orders WHERE customer_id = c.customer_id), 0)
                 >= (SELECT AVG(cust_total) FROM (SELECT customer_id, SUM(total_amount) AS cust_total FROM orders GROUP BY customer_id) AS t)
            THEN '2'
            ELSE '1'
        END
    ) AS rfm_score,

    -- Customer Health
    CASE
        WHEN COALESCE((SELECT COUNT(*) FROM orders WHERE customer_id = c.customer_id), 0) = 0
        THEN 'Inactive - Never Bought'
        WHEN DATEDIFF(CURDATE(), (SELECT MAX(order_date) FROM orders WHERE customer_id = c.customer_id)) <= 30
          AND COALESCE((SELECT COUNT(*) FROM orders WHERE customer_id = c.customer_id), 0) >= 2
        THEN 'Champion'
        WHEN DATEDIFF(CURDATE(), (SELECT MAX(order_date) FROM orders WHERE customer_id = c.customer_id)) <= 60
        THEN 'Loyal Customer'
        WHEN DATEDIFF(CURDATE(), (SELECT MAX(order_date) FROM orders WHERE customer_id = c.customer_id)) <= 90
        THEN 'At Risk'
        ELSE 'Churned'
    END AS customer_health

FROM customers c
ORDER BY projected_annual_value DESC;
```

---

## สรุป Patterns ที่ใช้ใน Projects เหล่านี้

```sql
-- Pattern 1: Scalar Subquery ใน SELECT สำหรับ KPI
SELECT (SELECT COUNT(*) FROM table) AS metric FROM dual;

-- Pattern 2: Correlated Subquery สำหรับ per-row calculation
SELECT x, (SELECT f(y) FROM other WHERE other.key = x.key) FROM x;

-- Pattern 3: NOT EXISTS สำหรับ Relational Division
SELECT * FROM a WHERE NOT EXISTS (SELECT 1 FROM b WHERE condition_linking_b_to_a);

-- Pattern 4: CROSS JOIN กับ Scalar ลด redundancy
SELECT col / stats.total FROM table CROSS JOIN (SELECT SUM(col) AS total FROM table) stats;

-- Pattern 5: Derived Table + Window Function สำหรับ Ranking
SELECT * FROM (SELECT *, RANK() OVER (...) AS rnk FROM ...) t WHERE rnk = 1;

-- Pattern 6: UNION ALL สำหรับ Multi-stage Reports
SELECT 'Stage 1', COUNT(*) FROM ... UNION ALL SELECT 'Stage 2', COUNT(*) FROM ...;

-- Pattern 7: Nested NOT EXISTS สำหรับ Division
SELECT * FROM a WHERE NOT EXISTS (
    SELECT * FROM b WHERE NOT EXISTS (
        SELECT * FROM a_b_relationship WHERE a_id = a.id AND b_id = b.id
    )
);
```

---

## แบบฝึกหัดบทที่ 50

**ข้อ 1:** สร้าง Mini Dashboard แสดง: จำนวนสินค้าต่อ category, ราคาเฉลี่ย, ยอดขายรวม

```sql
-- เฉลย:
SELECT
    p.category,
    COUNT(*) AS product_count,
    ROUND(AVG(p.price), 2) AS avg_price,
    COALESCE((
        SELECT SUM(oi.quantity * oi.unit_price)
        FROM   order_items oi
        JOIN   products p2 ON p2.product_id = oi.product_id
        WHERE  p2.category = p.category
    ), 0) AS total_revenue,
    COALESCE((
        SELECT SUM(oi.quantity)
        FROM   order_items oi
        JOIN   products p2 ON p2.product_id = oi.product_id
        WHERE  p2.category = p.category
    ), 0) AS total_units_sold
FROM products p
GROUP BY p.category
ORDER BY total_revenue DESC;
```

**ข้อ 2:** หาลูกค้าที่ซื้อสินค้าครบทุกหมวดหมู่ที่มีในระบบ

```sql
-- เฉลย (Relational Division):
SELECT c.first_name, c.last_name
FROM   customers c
WHERE  NOT EXISTS (
    SELECT DISTINCT p.category
    FROM   products p
    WHERE  NOT EXISTS (
        SELECT 1
        FROM   orders o
        JOIN   order_items oi ON oi.order_id = o.order_id
        JOIN   products p2    ON p2.product_id = oi.product_id
        WHERE  o.customer_id = c.customer_id
          AND  p2.category = p.category
    )
);
```

**ข้อ 3:** วิเคราะห์ Top 3 สินค้าขายดีในแต่ละ category พร้อมยอดขาย

```sql
-- เฉลย:
SELECT category, product_name, total_qty, total_revenue
FROM (
    SELECT p.category, p.product_name,
           COALESCE(SUM(oi.quantity), 0)           AS total_qty,
           COALESCE(SUM(oi.quantity * oi.unit_price), 0) AS total_revenue,
           RANK() OVER (PARTITION BY p.category ORDER BY COALESCE(SUM(oi.quantity), 0) DESC) AS rnk
    FROM   products p
    LEFT   JOIN order_items oi ON oi.product_id = p.product_id
    GROUP  BY p.category, p.product_id, p.product_name
) AS ranked
WHERE rnk <= 3
ORDER BY category, rnk;
```

**ข้อ 4:** สร้าง Funnel Report: Registered → Ordered → Repeat Customer

```sql
-- เฉลย:
SELECT stage, customer_count,
       ROUND(customer_count * 100.0 / MAX(customer_count) OVER (), 1) AS pct_of_total
FROM (
    SELECT 1 AS ord, 'Registered' AS stage,
           (SELECT COUNT(*) FROM customers) AS customer_count
    UNION ALL
    SELECT 2, 'Made 1+ Order',
           (SELECT COUNT(DISTINCT customer_id) FROM orders)
    UNION ALL
    SELECT 3, 'Made 2+ Orders',
           (SELECT COUNT(*) FROM (
               SELECT customer_id FROM orders
               GROUP BY customer_id HAVING COUNT(*) >= 2
           ) AS t)
    UNION ALL
    SELECT 4, 'Spent 20000+',
           (SELECT COUNT(*) FROM (
               SELECT customer_id FROM orders
               GROUP BY customer_id HAVING SUM(total_amount) >= 20000
           ) AS t)
) AS funnel
ORDER BY ord;
```

**ข้อ 5:** คำนวณ RFM Score สำหรับลูกค้าทุกคน

```sql
-- เฉลย:
SELECT
    c.customer_id,
    CONCAT(c.first_name, ' ', c.last_name) AS name,
    COALESCE(DATEDIFF(CURDATE(), (SELECT MAX(order_date) FROM orders WHERE customer_id = c.customer_id)), 999) AS recency,
    COALESCE((SELECT COUNT(*) FROM orders WHERE customer_id = c.customer_id), 0) AS frequency,
    COALESCE((SELECT SUM(total_amount) FROM orders WHERE customer_id = c.customer_id), 0) AS monetary
FROM customers c
ORDER BY monetary DESC, frequency DESC, recency ASC;
```

**ข้อ 6:** หา 'Market Basket' - สินค้าคู่ไหนถูกซื้อพร้อมกันบ่อยที่สุด

```sql
-- เฉลย:
SELECT
    p1.product_name AS product_a,
    p2.product_name AS product_b,
    COUNT(DISTINCT oi1.order_id) AS co_purchases
FROM order_items oi1
JOIN order_items oi2 ON oi2.order_id = oi1.order_id AND oi2.product_id > oi1.product_id
JOIN products p1 ON p1.product_id = oi1.product_id
JOIN products p2 ON p2.product_id = oi2.product_id
GROUP BY oi1.product_id, oi2.product_id, p1.product_name, p2.product_name
ORDER BY co_purchases DESC
LIMIT 10;
```

**ข้อ 7:** วิเคราะห์ Revenue ต่อ city ว่า city ไหนมีลูกค้าที่ใช้จ่ายมากที่สุด

```sql
-- เฉลย:
SELECT
    c.city,
    COUNT(DISTINCT c.customer_id) AS customer_count,
    COALESCE(SUM(o.total_amount), 0) AS total_revenue,
    ROUND(COALESCE(AVG(o.total_amount), 0), 2) AS avg_order_value,
    ROUND(COALESCE(SUM(o.total_amount), 0) / COUNT(DISTINCT c.customer_id), 2) AS revenue_per_customer,
    (SELECT CONCAT(first_name, ' ', last_name) FROM customers c2
     WHERE c2.city = c.city
       AND (SELECT COALESCE(SUM(total_amount), 0) FROM orders WHERE customer_id = c2.customer_id)
           = (SELECT MAX(COALESCE(SUM(o3.total_amount), 0))
              FROM customers c3
              LEFT JOIN orders o3 ON o3.customer_id = c3.customer_id
              WHERE c3.city = c.city
              GROUP BY c3.customer_id)
     LIMIT 1) AS top_customer
FROM customers c
LEFT JOIN orders o ON o.customer_id = c.customer_id
GROUP BY c.city
ORDER BY total_revenue DESC;
```

**ข้อ 8:** สร้าง Employee Department Summary พร้อม Top/Bottom Earner

```sql
-- เฉลย:
SELECT
    d.department_name,
    COUNT(e.employee_id) AS emp_count,
    MIN(e.salary) AS min_salary,
    ROUND(AVG(e.salary), 2) AS avg_salary,
    MAX(e.salary) AS max_salary,
    SUM(e.salary) AS total_salary_cost,
    (SELECT CONCAT(first_name, ' ', last_name)
     FROM employees WHERE department_id = d.department_id
     ORDER BY salary DESC LIMIT 1) AS highest_paid,
    (SELECT CONCAT(first_name, ' ', last_name)
     FROM employees WHERE department_id = d.department_id
     ORDER BY salary ASC LIMIT 1) AS lowest_paid
FROM departments d
LEFT JOIN employees e ON e.department_id = d.department_id
GROUP BY d.department_id, d.department_name
ORDER BY avg_salary DESC;
```

**ข้อ 9:** หาออเดอร์ที่ "ผิดปกติ" - ยอดสูงกว่า 2 SD จากค่าเฉลี่ย

```sql
-- เฉลย:
SELECT order_id, customer_id, order_date, total_amount,
       stats.avg_amount,
       stats.std_amount,
       ROUND((total_amount - stats.avg_amount) / NULLIF(stats.std_amount, 0), 2) AS z_score
FROM orders
CROSS JOIN (
    SELECT AVG(total_amount) AS avg_amount,
           STDDEV(total_amount) AS std_amount
    FROM orders
) AS stats
WHERE ABS((total_amount - stats.avg_amount) / NULLIF(stats.std_amount, 0)) > 2
ORDER BY z_score DESC;
```

**ข้อ 10:** สร้าง Comprehensive Business Report รวมทุก metric สำคัญ

```sql
-- เฉลย: One-page Business Report
SELECT
    -- === Revenue ===
    (SELECT SUM(total_amount) FROM orders WHERE status = 'completed') AS total_revenue,
    (SELECT SUM(total_amount) FROM orders
     WHERE status = 'completed' AND YEAR(order_date) = YEAR(CURDATE())) AS ytd_revenue,
    (SELECT COUNT(*) FROM orders) AS total_orders,
    (SELECT AVG(total_amount) FROM orders WHERE status = 'completed') AS avg_order_value,

    -- === Customers ===
    (SELECT COUNT(*) FROM customers) AS total_customers,
    (SELECT COUNT(DISTINCT customer_id) FROM orders) AS buying_customers,
    (SELECT COUNT(*) FROM customers
     WHERE customer_id NOT IN (SELECT DISTINCT customer_id FROM orders)) AS inactive_customers,

    -- === Products ===
    (SELECT COUNT(*) FROM products) AS total_products,
    (SELECT SUM(stock_qty * price) FROM products) AS inventory_value,
    (SELECT COUNT(*) FROM products WHERE stock_qty < 30) AS low_stock_items,

    -- === Top Performers ===
    (SELECT p.product_name FROM products p
     WHERE p.product_id = (
         SELECT product_id FROM order_items
         GROUP BY product_id ORDER BY SUM(quantity * unit_price) DESC LIMIT 1
     )) AS top_revenue_product,
    (SELECT CONCAT(c.first_name, ' ', c.last_name) FROM customers c
     WHERE c.customer_id = (
         SELECT customer_id FROM orders
         GROUP BY customer_id ORDER BY SUM(total_amount) DESC LIMIT 1
     )) AS top_customer,
    (SELECT d.department_name FROM departments d
     WHERE d.department_id = (
         SELECT department_id FROM employees GROUP BY department_id
         ORDER BY AVG(salary) DESC LIMIT 1
     )) AS highest_paid_department
;
```

---

## สรุปบทที่ 41-50: Subqueries Series

```
บทที่ 41: Introduction to Subqueries
  └── ประเภท: Scalar, Row, Table, Correlated
  └── ตำแหน่ง: SELECT, FROM, WHERE, HAVING

บทที่ 42: Scalar Subqueries
  └── ใน SELECT: computed column
  └── ใน WHERE: เปรียบเทียบค่าเดียว

บทที่ 43: IN / NOT IN
  └── Multi-row subquery
  └── ⚠️ NULL Trap กับ NOT IN

บทที่ 44: Derived Tables
  └── Subquery ใน FROM
  └── Aggregating Aggregates

บทที่ 45: Correlated Subqueries
  └── อ้างอิง outer query
  └── ทำงาน row-by-row

บทที่ 46: EXISTS / NOT EXISTS
  └── Semi-join / Anti-join
  └── Short-circuit evaluation

บทที่ 47: ALL / ANY / SOME
  └── ALL ≈ MAX, ANY ≈ MIN
  └── = ANY ≈ IN

บทที่ 48: DML กับ Subqueries
  └── UPDATE, DELETE, INSERT ... SELECT

บทที่ 49: Optimization
  └── EXPLAIN, Index, Rewriting

บทที่ 50: Real-world Projects
  └── 10 complex business queries
```

---

*จบบทที่ 50: Subquery Projects - Real-world Complex Queries*
*จบ Series Subqueries (Part 41-50)*
*บทถัดไป: Part 51 - Introduction to Common Table Expressions (CTEs)*
