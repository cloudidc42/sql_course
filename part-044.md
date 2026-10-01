# Part 44: Derived Tables - Subqueries in FROM

## 44.1 Derived Table คืออะไร?

**Derived Table** (หรือ Inline View) คือ subquery ที่อยู่ใน `FROM` clause ซึ่งจะถูกประมวลผลก่อนแล้วนำผลลัพธ์มาใช้เป็น "ตารางชั่วคราว" ใน query หลัก

```
SELECT *
FROM (
    SELECT ...         ← Derived Table (subquery ใน FROM)
    FROM   ...
    WHERE  ...
) AS alias_name        ← ต้องมี alias เสมอ!
WHERE ...
```

### ข้อดีของ Derived Table

```
1. สามารถใช้ aggregate function ซ้อน aggregate ได้
2. แบ่ง logic ที่ซับซ้อนเป็นขั้นตอน
3. สามารถ filter ผลลัพธ์ของ GROUP BY ได้อีกครั้ง
4. ช่วยให้ query อ่านง่ายขึ้น
```

---

## 44.2 Derived Table พื้นฐาน

### ตัวอย่างที่ 1: Derived Table ง่ายๆ

```sql
-- แสดงพนักงานที่มีเงินเดือนสูงกว่า 80,000 พร้อมชื่อแผนก
SELECT high_earners.first_name,
       high_earners.last_name,
       high_earners.salary,
       d.department_name
FROM (
    SELECT employee_id, first_name, last_name, salary, department_id
    FROM   employees
    WHERE  salary > 80000
) AS high_earners
JOIN departments d ON d.department_id = high_earners.department_id;
```

### ตัวอย่างที่ 2: Derived Table สำหรับ Aggregation

```sql
-- สรุปยอดขายต่อหมวดหมู่ แล้วหาเฉพาะที่ยอดสูง
SELECT category, total_revenue
FROM (
    SELECT p.category,
           SUM(oi.quantity * oi.unit_price) AS total_revenue
    FROM   products p
    JOIN   order_items oi ON oi.product_id = p.product_id
    GROUP  BY p.category
) AS category_revenue
WHERE total_revenue > 30000
ORDER BY total_revenue DESC;
```

### ตัวอย่างที่ 3: Derived Table เพื่อ Filter หลัง GROUP BY

```sql
-- หาแผนกที่มีพนักงานมากกว่า 2 คนและเงินเดือนเฉลี่ยสูงกว่า 80,000
SELECT dept_stats.department_id,
       d.department_name,
       dept_stats.emp_count,
       dept_stats.avg_salary
FROM (
    SELECT department_id,
           COUNT(*)    AS emp_count,
           AVG(salary) AS avg_salary
    FROM   employees
    GROUP  BY department_id
    HAVING COUNT(*) > 2
) AS dept_stats
JOIN departments d ON d.department_id = dept_stats.department_id
WHERE dept_stats.avg_salary > 80000;
```

### ตัวอย่างที่ 4: ตั้งชื่อ Derived Table และคอลัมน์

```sql
-- ตั้งชื่อคอลัมน์ใน derived table
SELECT order_summary.month_year,
       order_summary.order_count,
       order_summary.revenue
FROM (
    SELECT DATE_FORMAT(order_date, '%Y-%m') AS month_year,
           COUNT(*)                          AS order_count,
           SUM(total_amount)                AS revenue
    FROM   orders
    GROUP  BY DATE_FORMAT(order_date, '%Y-%m')
) AS order_summary
WHERE order_summary.order_count >= 3
ORDER BY order_summary.month_year;
```

---

## 44.3 Aggregating Aggregates (ซ้อน Aggregate)

```sql
-- ⚠️ ไม่สามารถทำแบบนี้ได้โดยตรง:
-- SELECT AVG(SUM(total_amount)) FROM orders GROUP BY customer_id;  -- ERROR!

-- ✓ ต้องใช้ Derived Table:
SELECT AVG(customer_total) AS avg_customer_total
FROM (
    SELECT customer_id, SUM(total_amount) AS customer_total
    FROM   orders
    GROUP  BY customer_id
) AS customer_totals;
```

### ตัวอย่างที่ 5: MAX of SUMs

```sql
-- ลูกค้าที่ใช้จ่ายมากที่สุด (MAX of SUM)
SELECT MAX(customer_total) AS max_spent
FROM (
    SELECT customer_id, SUM(total_amount) AS customer_total
    FROM   orders
    GROUP  BY customer_id
) AS customer_spending;
```

### ตัวอย่างที่ 6: Average of Counts

```sql
-- ค่าเฉลี่ยจำนวนออเดอร์ต่อลูกค้า
SELECT AVG(order_count) AS avg_orders_per_customer
FROM (
    SELECT customer_id, COUNT(*) AS order_count
    FROM   orders
    GROUP  BY customer_id
) AS order_counts;
```

### ตัวอย่างที่ 7: SUM of MAXes

```sql
-- ยอดออเดอร์สูงสุดของแต่ละลูกค้า รวมกัน
SELECT SUM(max_order) AS sum_of_max_orders
FROM (
    SELECT customer_id, MAX(total_amount) AS max_order
    FROM   orders
    GROUP  BY customer_id
) AS max_orders;
```

---

## 44.4 Multi-level Derived Tables

### ตัวอย่างที่ 8: 2 ชั้น

```sql
-- ชั้น 1: สรุปต่อ category และเดือน
-- ชั้น 2: หาเดือนที่ขายดีในแต่ละ category
SELECT cat_monthly.*
FROM (
    SELECT category, month_str, monthly_revenue,
           RANK() OVER (PARTITION BY category ORDER BY monthly_revenue DESC) AS revenue_rank
    FROM (
        SELECT p.category,
               DATE_FORMAT(o.order_date, '%Y-%m') AS month_str,
               SUM(oi.quantity * oi.unit_price)   AS monthly_revenue
        FROM   products p
        JOIN   order_items oi ON oi.product_id = p.product_id
        JOIN   orders o       ON o.order_id = oi.order_id
        GROUP  BY p.category, DATE_FORMAT(o.order_date, '%Y-%m')
    ) AS category_monthly
) AS cat_monthly
WHERE revenue_rank = 1;
```

### ตัวอย่างที่ 9: 3 ชั้น - Customer Tier Analysis

```sql
-- วิเคราะห์ระดับลูกค้า
SELECT tier, COUNT(*) AS customer_count, AVG(total_spent) AS avg_spend
FROM (
    SELECT customer_id, total_spent,
           CASE
               WHEN total_spent > high_threshold THEN 'Gold'
               WHEN total_spent > low_threshold  THEN 'Silver'
               ELSE 'Bronze'
           END AS tier
    FROM (
        SELECT cs.customer_id, cs.total_spent,
               thresholds.high_threshold,
               thresholds.low_threshold
        FROM (
            SELECT customer_id, SUM(total_amount) AS total_spent
            FROM   orders
            GROUP  BY customer_id
        ) AS cs
        CROSS JOIN (
            SELECT AVG(customer_total) * 1.5 AS high_threshold,
                   AVG(customer_total) * 0.5 AS low_threshold
            FROM (
                SELECT customer_id, SUM(total_amount) AS customer_total
                FROM   orders
                GROUP  BY customer_id
            ) AS ct
        ) AS thresholds
    ) AS customer_with_thresholds
) AS tiered_customers
GROUP BY tier
ORDER BY AVG(total_spent) DESC;
```

### ตัวอย่างที่ 10: Derived Table พร้อม JOIN หลายตาราง

```sql
-- รายงาน: top product ในแต่ละแผนก (ผ่านพนักงาน)
SELECT
    dept_prod.department_name,
    dept_prod.product_name,
    dept_prod.total_revenue
FROM (
    SELECT d.department_name,
           p.product_name,
           SUM(oi.quantity * oi.unit_price) AS total_revenue,
           ROW_NUMBER() OVER (
               PARTITION BY d.department_id
               ORDER BY SUM(oi.quantity * oi.unit_price) DESC
           ) AS rn
    FROM   departments d
    JOIN   employees e   ON e.department_id = d.department_id
    JOIN   orders o      ON o.customer_id   = e.employee_id  -- สมมติ
    JOIN   order_items oi ON oi.order_id    = o.order_id
    JOIN   products p    ON p.product_id    = oi.product_id
    GROUP  BY d.department_id, d.department_name, p.product_id, p.product_name
) AS dept_prod
WHERE rn = 1;
```

---

## 44.5 Complex Data Transformation

### ตัวอย่างที่ 11: Pivot ด้วย Derived Table

```sql
-- แสดงยอดขายแต่ละ category ในแต่ละเดือน (Pivot)
SELECT
    month_str,
    MAX(CASE WHEN category = 'Electronics' THEN monthly_rev ELSE 0 END) AS Electronics,
    MAX(CASE WHEN category = 'Furniture'   THEN monthly_rev ELSE 0 END) AS Furniture,
    MAX(CASE WHEN category = 'Appliances'  THEN monthly_rev ELSE 0 END) AS Appliances,
    MAX(CASE WHEN category = 'Stationery'  THEN monthly_rev ELSE 0 END) AS Stationery
FROM (
    SELECT DATE_FORMAT(o.order_date, '%Y-%m') AS month_str,
           p.category,
           SUM(oi.quantity * oi.unit_price)   AS monthly_rev
    FROM   orders o
    JOIN   order_items oi ON oi.order_id = o.order_id
    JOIN   products p     ON p.product_id = oi.product_id
    GROUP  BY DATE_FORMAT(o.order_date, '%Y-%m'), p.category
) AS monthly_category
GROUP BY month_str
ORDER BY month_str;
```

### ตัวอย่างที่ 12: Running Total ด้วย Derived Table

```sql
-- Running total ยอดขาย (ก่อน MySQL 8 มี Window Functions)
SELECT
    a.order_date,
    a.daily_revenue,
    SUM(b.daily_revenue) AS running_total
FROM (
    SELECT order_date, SUM(total_amount) AS daily_revenue
    FROM   orders
    GROUP  BY order_date
) AS a
JOIN (
    SELECT order_date, SUM(total_amount) AS daily_revenue
    FROM   orders
    GROUP  BY order_date
) AS b ON b.order_date <= a.order_date
GROUP BY a.order_date, a.daily_revenue
ORDER BY a.order_date;
```

### ตัวอย่างที่ 13: Percentile Calculation

```sql
-- คำนวณ percentile เงินเดือน
SELECT
    first_name,
    last_name,
    salary,
    ROUND(
        (SELECT COUNT(*) FROM employees e2 WHERE e2.salary <= e1.salary)
        * 100.0 / (SELECT COUNT(*) FROM employees),
        1
    ) AS percentile
FROM employees e1
ORDER BY salary DESC;
```

### ตัวอย่างที่ 14: Year-over-Year Comparison

```sql
-- เปรียบเทียบยอดขายปีต่อปี
SELECT
    curr.month_num,
    curr.revenue    AS current_year,
    prev.revenue    AS prev_year,
    ROUND(
        (curr.revenue - COALESCE(prev.revenue, 0))
        / NULLIF(prev.revenue, 0) * 100,
        1
    ) AS yoy_growth_pct
FROM (
    SELECT MONTH(order_date) AS month_num,
           SUM(total_amount) AS revenue
    FROM   orders
    WHERE  YEAR(order_date) = 2024
    GROUP  BY MONTH(order_date)
) AS curr
LEFT JOIN (
    SELECT MONTH(order_date) AS month_num,
           SUM(total_amount) AS revenue
    FROM   orders
    WHERE  YEAR(order_date) = 2023
    GROUP  BY MONTH(order_date)
) AS prev ON prev.month_num = curr.month_num
ORDER BY curr.month_num;
```

---

## 44.6 Derived Table กับ CROSS JOIN

### ตัวอย่างที่ 15: CROSS JOIN กับ Derived Table สำหรับ Statistics

```sql
-- เปรียบเทียบทุกสินค้ากับค่าเฉลี่ยรวม
SELECT
    p.product_name,
    p.price,
    stats.avg_price,
    stats.max_price,
    stats.min_price,
    ROUND(p.price / stats.avg_price * 100, 1) AS price_index
FROM products p
CROSS JOIN (
    SELECT AVG(price) AS avg_price,
           MAX(price) AS max_price,
           MIN(price) AS min_price
    FROM   products
) AS stats
ORDER BY price_index DESC;
```

### ตัวอย่างที่ 16: CROSS JOIN เพื่อ Normalize

```sql
-- Normalize เงินเดือนพนักงาน (0-100 scale)
SELECT
    e.first_name,
    e.last_name,
    e.salary,
    ROUND(
        (e.salary - s.min_sal) / (s.max_sal - s.min_sal) * 100,
        1
    ) AS normalized_salary
FROM employees e
CROSS JOIN (
    SELECT MIN(salary) AS min_sal, MAX(salary) AS max_sal
    FROM   employees
) AS s
ORDER BY normalized_salary DESC;
```

---

## 44.7 Derived Table vs CTE (Preview)

### ตัวอย่างที่ 17: Derived Table

```sql
-- แบบ Derived Table
SELECT dept_avg.department_id,
       d.department_name,
       dept_avg.avg_salary
FROM (
    SELECT department_id, AVG(salary) AS avg_salary
    FROM   employees
    GROUP  BY department_id
) AS dept_avg
JOIN departments d ON d.department_id = dept_avg.department_id
WHERE dept_avg.avg_salary > 80000;
```

```sql
-- แบบ CTE (อ่านง่ายกว่า)
WITH dept_avg AS (
    SELECT department_id, AVG(salary) AS avg_salary
    FROM   employees
    GROUP  BY department_id
)
SELECT dept_avg.department_id,
       d.department_name,
       dept_avg.avg_salary
FROM   dept_avg
JOIN   departments d ON d.department_id = dept_avg.department_id
WHERE  dept_avg.avg_salary > 80000;
```

### ตัวอย่างที่ 18: เมื่อ Derived Table เหมาะกว่า CTE

```sql
-- ใช้ Derived Table เมื่อใช้แค่ครั้งเดียวในท่อน FROM
SELECT product_name, total_qty
FROM (
    SELECT product_id, SUM(quantity) AS total_qty
    FROM   order_items
    GROUP  BY product_id
    HAVING SUM(quantity) > 1
) AS popular_products
JOIN products USING (product_id)
ORDER BY total_qty DESC;
```

---

## 44.8 Derived Table กับ Filtering

### ตัวอย่างที่ 19: Filter หลัง Aggregation

```sql
-- HAVING ใน derived table + WHERE ใน outer query
SELECT product_stats.category,
       product_stats.product_count,
       product_stats.avg_price
FROM (
    SELECT category,
           COUNT(*)    AS product_count,
           AVG(price)  AS avg_price,
           MAX(price)  AS max_price
    FROM   products
    GROUP  BY category
    HAVING COUNT(*) >= 3  -- Filter ใน derived table
) AS product_stats
WHERE avg_price > 5000     -- Filter ใน outer query
ORDER BY avg_price DESC;
```

### ตัวอย่างที่ 20: Chaining Derived Tables

```sql
-- Step 1 → Step 2 (Derived table ซ้อนกัน)
SELECT final.department_name,
       final.tier,
       final.emp_count
FROM (
    SELECT dept_data.department_id,
           d.department_name,
           dept_data.avg_salary,
           dept_data.emp_count,
           CASE
               WHEN dept_data.avg_salary > 85000 THEN 'High Pay'
               WHEN dept_data.avg_salary > 75000 THEN 'Med Pay'
               ELSE 'Low Pay'
           END AS tier
    FROM (
        SELECT department_id,
               AVG(salary) AS avg_salary,
               COUNT(*)    AS emp_count
        FROM   employees
        GROUP  BY department_id
    ) AS dept_data
    JOIN departments d ON d.department_id = dept_data.department_id
) AS final
WHERE final.tier IN ('High Pay', 'Med Pay')
ORDER BY final.avg_salary DESC;
```

---

## 44.9 Performance และ MySQL การจัดการ Derived Table

### ตัวอย่างที่ 21: Materialization

```sql
-- MySQL 8+: Derived table มักถูก materialize (สร้าง temp table)
-- ใช้ EXPLAIN เพื่อดู

EXPLAIN SELECT *
FROM (
    SELECT department_id, AVG(salary) AS avg_sal
    FROM   employees
    GROUP  BY department_id
) AS dept_avg
WHERE avg_sal > 80000;
-- ดู "DERIVED" ใน type column
```

### ตัวอย่างที่ 22: เปรียบเทียบแผน Execution

```sql
-- EXPLAIN FORMAT=JSON สำหรับรายละเอียดมากขึ้น
EXPLAIN FORMAT=JSON
SELECT dt.department_id, dt.avg_sal, d.department_name
FROM (
    SELECT department_id, AVG(salary) AS avg_sal
    FROM   employees GROUP BY department_id
) AS dt
JOIN departments d ON d.department_id = dt.department_id
WHERE dt.avg_sal > 75000;
```

---

## 44.10 Real-world Complex Examples

### ตัวอย่างที่ 23: Sales Performance Report

```sql
-- รายงานประสิทธิภาพการขายรายเดือน
SELECT
    monthly.month_label,
    monthly.total_orders,
    monthly.total_revenue,
    monthly.avg_order_value,
    ROUND(
        (monthly.total_revenue - LAG(monthly.total_revenue) OVER (ORDER BY monthly.month_label))
        / LAG(monthly.total_revenue) OVER (ORDER BY monthly.month_label) * 100,
        1
    ) AS mom_growth_pct
FROM (
    SELECT DATE_FORMAT(order_date, '%Y-%m')   AS month_label,
           COUNT(*)                            AS total_orders,
           SUM(total_amount)                   AS total_revenue,
           ROUND(AVG(total_amount), 2)         AS avg_order_value
    FROM   orders
    GROUP  BY DATE_FORMAT(order_date, '%Y-%m')
) AS monthly
ORDER BY month_label;
```

### ตัวอย่างที่ 24: Product Performance Matrix

```sql
-- Matrix ประสิทธิภาพสินค้า
SELECT
    perf.product_name,
    perf.category,
    perf.total_revenue,
    perf.total_quantity,
    perf.avg_unit_price,
    perf.order_count,
    CASE
        WHEN perf.total_revenue > rev_stats.high_rev AND perf.total_quantity > qty_stats.high_qty
            THEN 'Star'
        WHEN perf.total_revenue > rev_stats.high_rev
            THEN 'High Revenue'
        WHEN perf.total_quantity > qty_stats.high_qty
            THEN 'High Volume'
        ELSE 'Normal'
    END AS performance_tier
FROM (
    SELECT p.product_id,
           p.product_name,
           p.category,
           SUM(oi.quantity * oi.unit_price) AS total_revenue,
           SUM(oi.quantity)                 AS total_quantity,
           AVG(oi.unit_price)               AS avg_unit_price,
           COUNT(DISTINCT oi.order_id)      AS order_count
    FROM   products p
    LEFT   JOIN order_items oi ON oi.product_id = p.product_id
    GROUP  BY p.product_id, p.product_name, p.category
) AS perf
CROSS JOIN (
    SELECT AVG(total_revenue) AS high_rev
    FROM (
        SELECT product_id, SUM(quantity * unit_price) AS total_revenue
        FROM   order_items GROUP BY product_id
    ) AS t
) AS rev_stats
CROSS JOIN (
    SELECT AVG(total_quantity) AS high_qty
    FROM (
        SELECT product_id, SUM(quantity) AS total_quantity
        FROM   order_items GROUP BY product_id
    ) AS t
) AS qty_stats
ORDER BY total_revenue DESC;
```

### ตัวอย่างที่ 25: Customer Cohort Analysis

```sql
-- วิเคราะห์ Cohort ลูกค้าตามเดือนที่สมัคร
SELECT
    cohort.cohort_month,
    cohort.customer_count,
    orders_in_first_3m.ordering_count,
    ROUND(
        orders_in_first_3m.ordering_count * 100.0 / cohort.customer_count,
        1
    ) AS conversion_rate
FROM (
    SELECT DATE_FORMAT(created_at, '%Y-%m') AS cohort_month,
           COUNT(*) AS customer_count
    FROM   customers
    GROUP  BY DATE_FORMAT(created_at, '%Y-%m')
) AS cohort
LEFT JOIN (
    SELECT DATE_FORMAT(c.created_at, '%Y-%m') AS cohort_month,
           COUNT(DISTINCT c.customer_id)        AS ordering_count
    FROM   customers c
    JOIN   orders o ON o.customer_id = c.customer_id
    WHERE  o.order_date <= DATE_ADD(c.created_at, INTERVAL 90 DAY)
    GROUP  BY DATE_FORMAT(c.created_at, '%Y-%m')
) AS orders_in_first_3m ON orders_in_first_3m.cohort_month = cohort.cohort_month
ORDER BY cohort.cohort_month;
```

### ตัวอย่างที่ 26: Inventory Turnover

```sql
-- วิเคราะห์การหมุนเวียนสินค้าคงคลัง
SELECT
    inv.product_name,
    inv.category,
    inv.stock_qty            AS current_stock,
    sales.avg_monthly_sales,
    ROUND(inv.stock_qty / NULLIF(sales.avg_monthly_sales, 0), 1) AS months_of_stock,
    CASE
        WHEN inv.stock_qty / NULLIF(sales.avg_monthly_sales, 0) < 1 THEN 'Reorder Now'
        WHEN inv.stock_qty / NULLIF(sales.avg_monthly_sales, 0) < 2 THEN 'Reorder Soon'
        WHEN sales.avg_monthly_sales IS NULL                         THEN 'No Sales'
        ELSE 'OK'
    END AS stock_status
FROM products AS inv
LEFT JOIN (
    SELECT product_id,
           SUM(quantity) / COUNT(DISTINCT DATE_FORMAT(
               (SELECT order_date FROM orders WHERE order_id = oi.order_id LIMIT 1),
               '%Y-%m'
           )) AS avg_monthly_sales
    FROM   order_items oi
    GROUP  BY product_id
) AS sales ON sales.product_id = inv.product_id
ORDER BY months_of_stock ASC;
```

### ตัวอย่างที่ 27: Employee Salary Band Analysis

```sql
-- วิเคราะห์การกระจายเงินเดือนในแต่ละแบนด์
SELECT
    salary_band,
    COUNT(*)       AS employee_count,
    MIN(salary)    AS min_salary,
    MAX(salary)    AS max_salary,
    AVG(salary)    AS avg_salary
FROM (
    SELECT
        employee_id,
        salary,
        CASE
            WHEN salary < 70000 THEN '1: Under 70K'
            WHEN salary < 80000 THEN '2: 70K-80K'
            WHEN salary < 90000 THEN '3: 80K-90K'
            ELSE                     '4: 90K+'
        END AS salary_band
    FROM employees
) AS banded
GROUP BY salary_band
ORDER BY salary_band;
```

---

## 44.11 Derived Table กับ Window Functions

### ตัวอย่างที่ 28: Filter Window Function Result

```sql
-- Window function ไม่สามารถ filter ได้ตรงๆ ต้องใช้ derived table
SELECT *
FROM (
    SELECT
        employee_id,
        first_name,
        last_name,
        salary,
        department_id,
        RANK() OVER (PARTITION BY department_id ORDER BY salary DESC) AS dept_rank
    FROM employees
) AS ranked
WHERE dept_rank <= 2  -- ← filter บน window function result
ORDER BY department_id, dept_rank;
```

### ตัวอย่างที่ 29: Multi-step Window Calculation

```sql
-- หาพนักงานที่มีเงินเดือนสูงกว่า percentile ที่ 75
SELECT first_name, last_name, salary, pct_rank
FROM (
    SELECT
        first_name,
        last_name,
        salary,
        ROUND(PERCENT_RANK() OVER (ORDER BY salary) * 100, 1) AS pct_rank
    FROM employees
) AS ranked
WHERE pct_rank >= 75
ORDER BY pct_rank DESC;
```

### ตัวอย่างที่ 30: Top N per Group

```sql
-- Top 2 สินค้าขายดีในแต่ละหมวดหมู่
SELECT product_name, category, total_qty
FROM (
    SELECT p.product_name,
           p.category,
           COALESCE(SUM(oi.quantity), 0) AS total_qty,
           RANK() OVER (
               PARTITION BY p.category
               ORDER BY COALESCE(SUM(oi.quantity), 0) DESC
           ) AS rnk
    FROM   products p
    LEFT   JOIN order_items oi ON oi.product_id = p.product_id
    GROUP  BY p.product_id, p.product_name, p.category
) AS ranked
WHERE rnk <= 2
ORDER BY category, rnk;
```

---

## แบบฝึกหัดบทที่ 44

**ข้อ 1:** ใช้ derived table หาลูกค้าที่มียอดซื้อสูงกว่าค่าเฉลี่ย

```sql
-- เฉลย:
SELECT c.first_name, c.last_name, cust_spending.total_spent
FROM (
    SELECT customer_id, SUM(total_amount) AS total_spent
    FROM   orders
    GROUP  BY customer_id
) AS cust_spending
JOIN customers c ON c.customer_id = cust_spending.customer_id
WHERE cust_spending.total_spent > (
    SELECT AVG(total_spent)
    FROM (
        SELECT customer_id, SUM(total_amount) AS total_spent
        FROM   orders
        GROUP  BY customer_id
    ) AS t
)
ORDER BY total_spent DESC;
```

**ข้อ 2:** คำนวณ AVG ของ SUM ยอดขายต่อเดือน โดยใช้ derived table

```sql
-- เฉลย:
SELECT ROUND(AVG(monthly_revenue), 2) AS avg_monthly_revenue
FROM (
    SELECT YEAR(order_date)  AS yr,
           MONTH(order_date) AS mo,
           SUM(total_amount) AS monthly_revenue
    FROM   orders
    GROUP  BY YEAR(order_date), MONTH(order_date)
) AS monthly;
```

**ข้อ 3:** ใช้ derived table และ CROSS JOIN แสดงราคาสินค้าเทียบกับค่าเฉลี่ยในหมวดหมู่เดียวกัน

```sql
-- เฉลย:
SELECT p.product_name, p.category, p.price,
       cat_avg.avg_price,
       ROUND(p.price - cat_avg.avg_price, 2) AS diff_from_cat_avg
FROM products p
JOIN (
    SELECT category, AVG(price) AS avg_price
    FROM   products
    GROUP  BY category
) AS cat_avg ON cat_avg.category = p.category
ORDER BY p.category, diff_from_cat_avg DESC;
```

**ข้อ 4:** ใช้ derived table ที่ซ้อนสองชั้น สรุปยอดขายต่อเดือนต่อ category แล้วหาเดือนที่ยอดขายสูงสุดของแต่ละ category

```sql
-- เฉลย:
SELECT category, month_label, monthly_revenue
FROM (
    SELECT category, month_label, monthly_revenue,
           RANK() OVER (PARTITION BY category ORDER BY monthly_revenue DESC) AS rnk
    FROM (
        SELECT p.category,
               DATE_FORMAT(o.order_date, '%Y-%m') AS month_label,
               SUM(oi.quantity * oi.unit_price)   AS monthly_revenue
        FROM   products p
        JOIN   order_items oi ON oi.product_id = p.product_id
        JOIN   orders o       ON o.order_id = oi.order_id
        GROUP  BY p.category, DATE_FORMAT(o.order_date, '%Y-%m')
    ) AS cat_monthly
) AS ranked
WHERE rnk = 1;
```

**ข้อ 5:** ใช้ derived table สร้าง report เงินเดือนพนักงาน: ชื่อ, เงินเดือน, % เทียบกับ MAX ของแผนก, % เทียบกับ MAX ของบริษัท

```sql
-- เฉลย:
SELECT e.first_name, e.last_name, e.salary,
       dept_max.max_salary AS dept_max,
       company_max.max_salary AS company_max,
       ROUND(e.salary / dept_max.max_salary * 100, 1) AS pct_of_dept_max,
       ROUND(e.salary / company_max.max_salary * 100, 1) AS pct_of_company_max
FROM employees e
JOIN (
    SELECT department_id, MAX(salary) AS max_salary
    FROM   employees
    GROUP  BY department_id
) AS dept_max ON dept_max.department_id = e.department_id
CROSS JOIN (
    SELECT MAX(salary) AS max_salary FROM employees
) AS company_max;
```

**ข้อ 6:** ใช้ derived table + window function หา Top 1 ลูกค้าในแต่ละ city

```sql
-- เฉลย:
SELECT city, customer_name, total_spent
FROM (
    SELECT c.city,
           CONCAT(c.first_name, ' ', c.last_name) AS customer_name,
           COALESCE(SUM(o.total_amount), 0)        AS total_spent,
           RANK() OVER (PARTITION BY c.city ORDER BY COALESCE(SUM(o.total_amount), 0) DESC) AS rnk
    FROM   customers c
    LEFT   JOIN orders o ON o.customer_id = c.customer_id
    GROUP  BY c.customer_id, c.city, c.first_name, c.last_name
) AS ranked
WHERE rnk = 1
ORDER BY total_spent DESC;
```

**ข้อ 7:** คำนวณ Salary Band Distribution (ร้อยละของพนักงานในแต่ละ band)

```sql
-- เฉลย:
SELECT salary_band,
       emp_count,
       ROUND(emp_count * 100.0 / total_emp, 1) AS pct
FROM (
    SELECT salary_band,
           COUNT(*) AS emp_count,
           SUM(COUNT(*)) OVER () AS total_emp
    FROM (
        SELECT CASE
                   WHEN salary < 70000 THEN 'Under 70K'
                   WHEN salary < 80000 THEN '70K-80K'
                   WHEN salary < 90000 THEN '80K-90K'
                   ELSE '90K+'
               END AS salary_band
        FROM employees
    ) AS banded
    GROUP BY salary_band
) AS band_counts
ORDER BY salary_band;
```

**ข้อ 8:** ใช้ derived table แสดงสินค้าพร้อม stock status (กี่เดือนถึงจะขายหมด)

```sql
-- เฉลย:
SELECT p.product_name, p.stock_qty, sales.avg_monthly_sales,
       CASE
           WHEN sales.avg_monthly_sales IS NULL OR sales.avg_monthly_sales = 0 THEN 'No Recent Sales'
           WHEN p.stock_qty / sales.avg_monthly_sales < 1 THEN 'Critical'
           WHEN p.stock_qty / sales.avg_monthly_sales < 3 THEN 'Low'
           ELSE 'Adequate'
       END AS stock_status
FROM products p
LEFT JOIN (
    SELECT product_id, AVG(monthly_qty) AS avg_monthly_sales
    FROM (
        SELECT product_id,
               DATE_FORMAT((SELECT order_date FROM orders WHERE order_id = oi.order_id LIMIT 1), '%Y-%m') AS m,
               SUM(quantity) AS monthly_qty
        FROM order_items oi
        GROUP BY product_id, m
    ) AS monthly_sales
    GROUP BY product_id
) AS sales ON sales.product_id = p.product_id
ORDER BY stock_status, p.product_name;
```

**ข้อ 9:** ใช้ derived table วิเคราะห์ revenue growth month-over-month

```sql
-- เฉลย:
SELECT
    curr.month_label,
    curr.revenue,
    COALESCE(prev.revenue, 0)    AS prev_month_revenue,
    curr.revenue - COALESCE(prev.revenue, 0) AS abs_growth,
    ROUND(
        (curr.revenue - COALESCE(prev.revenue, 0)) / NULLIF(prev.revenue, 0) * 100,
        1
    ) AS pct_growth
FROM (
    SELECT DATE_FORMAT(order_date, '%Y-%m') AS month_label,
           SUM(total_amount) AS revenue
    FROM   orders
    GROUP  BY DATE_FORMAT(order_date, '%Y-%m')
) AS curr
LEFT JOIN (
    SELECT DATE_FORMAT(order_date, '%Y-%m') AS month_label,
           SUM(total_amount) AS revenue
    FROM   orders
    GROUP  BY DATE_FORMAT(order_date, '%Y-%m')
) AS prev ON prev.month_label = DATE_FORMAT(DATE_SUB(STR_TO_DATE(CONCAT(curr.month_label, '-01'), '%Y-%m-%d'), INTERVAL 1 MONTH), '%Y-%m')
ORDER BY curr.month_label;
```

**ข้อ 10:** สร้าง comprehensive report แสดง: department, avg_salary, emp_count, top_earner_name, top_salary

```sql
-- เฉลย:
SELECT
    d.department_name,
    dept_stats.avg_salary,
    dept_stats.emp_count,
    top_earners.top_name AS top_earner,
    top_earners.top_salary
FROM (
    SELECT department_id,
           ROUND(AVG(salary), 2) AS avg_salary,
           COUNT(*) AS emp_count
    FROM   employees
    GROUP  BY department_id
) AS dept_stats
JOIN departments d ON d.department_id = dept_stats.department_id
JOIN (
    SELECT department_id,
           CONCAT(first_name, ' ', last_name) AS top_name,
           salary AS top_salary
    FROM employees e
    WHERE salary = (
        SELECT MAX(salary) FROM employees WHERE department_id = e.department_id
    )
) AS top_earners ON top_earners.department_id = dept_stats.department_id
ORDER BY dept_stats.avg_salary DESC;
```

---

*จบบทที่ 44: Derived Tables - Subqueries in FROM*
*บทถัดไป: Part 45 - Correlated Subqueries*
