# Part 120: Advanced SQL Interview Questions (50 Problems)

## บทนำ (Introduction)

Part สุดท้ายของ SQL Course รวบรวมโจทย์ Interview SQL ระดับ Hard จาก Tech Companies ชั้นนำ แต่ละข้อมีคำอธิบาย Approach และ Solution พร้อม Alternative Approaches เพื่อให้เข้าใจอย่างลึกซึ้ง

---

## Setup: Database สำหรับทำโจทย์

```sql
CREATE DATABASE interview_db CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
USE interview_db;

-- Employees
CREATE TABLE employees (
    emp_id      INT PRIMARY KEY,
    name        VARCHAR(100),
    dept_id     INT,
    manager_id  INT,
    salary      DECIMAL(10,2),
    hire_date   DATE,
    job_title   VARCHAR(100)
);

CREATE TABLE departments (
    dept_id     INT PRIMARY KEY,
    dept_name   VARCHAR(100),
    location    VARCHAR(100)
);

-- Sales
CREATE TABLE orders (
    order_id    INT PRIMARY KEY,
    customer_id INT,
    product_id  INT,
    order_date  DATE,
    amount      DECIMAL(10,2),
    status      VARCHAR(20)
);

CREATE TABLE customers (
    customer_id INT PRIMARY KEY,
    name        VARCHAR(100),
    country     VARCHAR(50),
    joined_date DATE
);

CREATE TABLE products (
    product_id  INT PRIMARY KEY,
    name        VARCHAR(100),
    category    VARCHAR(50),
    price       DECIMAL(10,2)
);

-- Logs/Activities
CREATE TABLE user_activity (
    activity_id INT PRIMARY KEY,
    user_id     INT,
    activity    VARCHAR(50),
    activity_date DATE
);

-- Insert sample data
INSERT INTO departments VALUES
(1,'Engineering','Bangkok'),(2,'Marketing','Bangkok'),
(3,'Sales','Chiang Mai'),(4,'HR','Bangkok'),(5,'Finance','Bangkok');

INSERT INTO employees VALUES
(1,'Alice',1,NULL,120000,'2018-01-15','CTO'),
(2,'Bob',1,1,95000,'2019-03-20','Senior Dev'),
(3,'Charlie',1,1,85000,'2020-06-10','Developer'),
(4,'Diana',2,NULL,90000,'2018-05-01','CMO'),
(5,'Eve',2,4,75000,'2021-01-10','Marketing Mgr'),
(6,'Frank',3,NULL,80000,'2019-08-15','Sales Dir'),
(7,'Grace',3,6,65000,'2022-01-01','Sales Rep'),
(8,'Henry',3,6,62000,'2022-03-15','Sales Rep'),
(9,'Iris',4,NULL,70000,'2020-02-20','HR Manager'),
(10,'Jack',5,NULL,85000,'2019-11-01','CFO'),
(11,'Kate',1,2,78000,'2021-07-01','Developer'),
(12,'Liam',1,2,72000,'2022-09-01','Junior Dev'),
(13,'Mona',2,4,68000,'2022-04-15','Marketing Analyst'),
(14,'Noah',3,6,60000,'2023-01-10','Sales Rep');

INSERT INTO customers VALUES
(1,'Somchai','Thailand','2020-01-15'),
(2,'Malee','Thailand','2020-03-20'),
(3,'John','USA','2021-06-01'),
(4,'Kenji','Japan','2021-08-15'),
(5,'Priya','India','2022-01-10'),
(6,'Emma','UK','2022-03-01'),
(7,'Carlos','Mexico','2022-06-15'),
(8,'Nisa','Thailand','2023-01-01');

INSERT INTO products VALUES
(1,'Laptop Pro','Electronics',45000),
(2,'Wireless Mouse','Electronics',850),
(3,'Office Chair','Furniture',5500),
(4,'Standing Desk','Furniture',12000),
(5,'Web Cam 4K','Electronics',3200),
(6,'Notebook Pack','Stationery',250),
(7,'Mechanical Keyboard','Electronics',3800),
(8,'Monitor 27"','Electronics',9500);

INSERT INTO orders VALUES
(1,1,1,'2023-01-15',45000,'completed'),
(2,1,2,'2023-01-20',850,'completed'),
(3,2,3,'2023-02-01',5500,'completed'),
(4,3,1,'2023-02-15',45000,'completed'),
(5,2,4,'2023-02-20',12000,'completed'),
(6,4,7,'2023-03-01',3800,'completed'),
(7,1,8,'2023-03-15',9500,'completed'),
(8,5,5,'2023-04-01',3200,'completed'),
(9,3,2,'2023-04-15',850,'completed'),
(10,6,1,'2023-05-01',45000,'refunded'),
(11,7,6,'2023-05-15',250,'completed'),
(12,2,8,'2023-06-01',9500,'completed'),
(13,8,1,'2023-06-15',45000,'completed'),
(14,4,3,'2023-07-01',5500,'completed'),
(15,1,7,'2023-07-15',3800,'completed'),
(16,5,4,'2023-08-01',12000,'completed'),
(17,3,8,'2023-08-15',9500,'completed'),
(18,2,2,'2023-09-01',850,'completed'),
(19,6,5,'2023-09-15',3200,'pending'),
(20,7,1,'2023-10-01',45000,'completed');

INSERT INTO user_activity VALUES
(1,1,'login','2024-01-01'),(2,1,'purchase','2024-01-01'),
(3,1,'login','2024-01-02'),(4,2,'login','2024-01-01'),
(5,2,'login','2024-01-03'),(6,3,'login','2024-01-01'),
(7,3,'login','2024-01-02'),(8,3,'login','2024-01-03'),
(9,3,'purchase','2024-01-03'),(10,4,'login','2024-01-05'),
(11,1,'login','2024-01-07'),(12,2,'login','2024-01-09'),
(13,3,'login','2024-01-08'),(14,5,'login','2024-01-01'),
(15,5,'login','2024-01-02'),(16,5,'login','2024-01-04');
```

---

## Problems 1-10: Window Functions

### Problem 1: Rank Employees by Salary Within Department

**โจทย์**: จัดอันดับพนักงานตาม Salary ในแต่ละ Department โดยแสดง rank, dense_rank และ % ของ max salary ในแผนก

```sql
SELECT 
    e.emp_id,
    e.name,
    d.dept_name,
    e.salary,
    RANK() OVER (PARTITION BY e.dept_id ORDER BY e.salary DESC) AS rnk,
    DENSE_RANK() OVER (PARTITION BY e.dept_id ORDER BY e.salary DESC) AS dense_rnk,
    ROW_NUMBER() OVER (PARTITION BY e.dept_id ORDER BY e.salary DESC) AS row_num,
    ROUND(e.salary * 100.0 / MAX(e.salary) OVER (PARTITION BY e.dept_id), 1) AS pct_of_max
FROM employees e
JOIN departments d ON e.dept_id = d.dept_id
ORDER BY d.dept_name, rnk;
```

---

### Problem 2: Find Second Highest Salary in Each Department

**โจทย์**: หาพนักงานที่ได้รับเงินเดือนสูงสุดอันดับ 2 ของแต่ละแผนก

```sql
-- Method 1: Using DENSE_RANK
WITH ranked AS (
    SELECT 
        e.name,
        d.dept_name,
        e.salary,
        DENSE_RANK() OVER (PARTITION BY e.dept_id ORDER BY e.salary DESC) AS dr
    FROM employees e
    JOIN departments d ON e.dept_id = d.dept_id
)
SELECT dept_name, name, salary
FROM ranked
WHERE dr = 2;

-- Method 2: Correlated Subquery (older style)
SELECT e.name, d.dept_name, e.salary
FROM employees e
JOIN departments d ON e.dept_id = d.dept_id
WHERE e.salary = (
    SELECT MAX(e2.salary) 
    FROM employees e2 
    WHERE e2.dept_id = e.dept_id 
      AND e2.salary < (SELECT MAX(salary) FROM employees e3 WHERE e3.dept_id = e.dept_id)
);
```

---

### Problem 3: Running Total with Reset

**โจทย์**: คำนวณ Running Total ของ Sales แต่ให้ Reset ทุกต้นเดือน

```sql
SELECT 
    o.order_id,
    o.customer_id,
    o.order_date,
    o.amount,
    DATE_FORMAT(o.order_date, '%Y-%m') AS month,
    SUM(o.amount) OVER (
        PARTITION BY DATE_FORMAT(o.order_date, '%Y-%m')  -- Reset per month
        ORDER BY o.order_date, o.order_id
        ROWS UNBOUNDED PRECEDING
    ) AS running_total_this_month,
    SUM(o.amount) OVER (
        ORDER BY o.order_date, o.order_id  -- No reset
        ROWS UNBOUNDED PRECEDING
    ) AS cumulative_all_time
FROM orders o
WHERE o.status = 'completed'
ORDER BY o.order_date, o.order_id;
```

---

### Problem 4: Moving Average (3-month)

**โจทย์**: คำนวณ 3-month Moving Average ของ Monthly Revenue

```sql
WITH monthly_revenue AS (
    SELECT 
        DATE_FORMAT(order_date, '%Y-%m') AS month,
        SUM(amount) AS revenue
    FROM orders
    WHERE status = 'completed'
    GROUP BY DATE_FORMAT(order_date, '%Y-%m')
)
SELECT 
    month,
    revenue,
    ROUND(AVG(revenue) OVER (
        ORDER BY month
        ROWS BETWEEN 2 PRECEDING AND CURRENT ROW
    ), 2) AS moving_avg_3m,
    LAG(revenue) OVER (ORDER BY month) AS prev_month,
    ROUND(
        (revenue - LAG(revenue) OVER (ORDER BY month)) * 100.0 / 
        NULLIF(LAG(revenue) OVER (ORDER BY month), 0),
        1
    ) AS mom_growth_pct
FROM monthly_revenue
ORDER BY month;
```

---

### Problem 5: Percentile Distribution

**โจทย์**: แบ่งพนักงานเป็น Salary Quartiles และหา Percentile

```sql
SELECT 
    name,
    dept_id,
    salary,
    NTILE(4) OVER (ORDER BY salary) AS salary_quartile,
    PERCENT_RANK() OVER (ORDER BY salary) AS percentile_rank,
    CUME_DIST() OVER (ORDER BY salary) AS cumulative_dist,
    CASE NTILE(4) OVER (ORDER BY salary)
        WHEN 1 THEN 'Bottom 25%'
        WHEN 2 THEN '25-50%'
        WHEN 3 THEN '50-75%'
        WHEN 4 THEN 'Top 25%'
    END AS salary_band
FROM employees
ORDER BY salary DESC;
```

---

## Problems 6-10: Advanced Aggregations

### Problem 6: Pivoting Data

**โจทย์**: Pivot ยอดขายจาก Rows เป็น Columns แยกตาม Category

```sql
SELECT 
    DATE_FORMAT(o.order_date, '%Y-%m') AS month,
    SUM(CASE WHEN p.category = 'Electronics' THEN o.amount ELSE 0 END) AS electronics_sales,
    SUM(CASE WHEN p.category = 'Furniture' THEN o.amount ELSE 0 END) AS furniture_sales,
    SUM(CASE WHEN p.category = 'Stationery' THEN o.amount ELSE 0 END) AS stationery_sales,
    SUM(o.amount) AS total_sales,
    COUNT(DISTINCT o.customer_id) AS unique_customers
FROM orders o
JOIN products p ON o.product_id = p.product_id
WHERE o.status = 'completed'
GROUP BY DATE_FORMAT(o.order_date, '%Y-%m')
ORDER BY month;
```

---

### Problem 7: Consecutive Days Active

**โจทย์**: หา Users ที่ Active ต่อเนื่องกันกี่วัน (Streaks)

```sql
WITH daily_active AS (
    SELECT DISTINCT user_id, activity_date
    FROM user_activity
),
with_groups AS (
    SELECT 
        user_id,
        activity_date,
        -- Gap: this date minus row number = constant for consecutive dates
        DATE_SUB(activity_date, INTERVAL 
            ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY activity_date) 
        DAY) AS grp
    FROM daily_active
),
streaks AS (
    SELECT 
        user_id,
        grp,
        MIN(activity_date) AS streak_start,
        MAX(activity_date) AS streak_end,
        COUNT(*) AS streak_length
    FROM with_groups
    GROUP BY user_id, grp
)
SELECT 
    user_id,
    streak_start,
    streak_end,
    streak_length,
    RANK() OVER (PARTITION BY user_id ORDER BY streak_length DESC) AS streak_rank
FROM streaks
ORDER BY streak_length DESC, user_id;
```

---

### Problem 8: Finding Gaps in Sequential IDs

**โจทย์**: หา Order IDs ที่ขาดหายไปในลำดับ

```sql
WITH RECURSIVE id_sequence AS (
    SELECT MIN(order_id) AS id FROM orders
    UNION ALL
    SELECT id + 1
    FROM id_sequence
    WHERE id < (SELECT MAX(order_id) FROM orders)
)
SELECT id AS missing_order_id
FROM id_sequence
WHERE id NOT IN (SELECT order_id FROM orders)
ORDER BY id;

-- Alternative: Using self-join to find gaps
SELECT 
    o1.order_id AS current_id,
    o1.order_id + 1 AS expected_next,
    o2.order_id AS actual_next,
    o2.order_id - o1.order_id - 1 AS gap_size
FROM orders o1
LEFT JOIN orders o2 ON o2.order_id = (
    SELECT MIN(order_id) FROM orders WHERE order_id > o1.order_id
)
WHERE o2.order_id != o1.order_id + 1 OR o2.order_id IS NULL
ORDER BY o1.order_id;
```

---

### Problem 9: Median Calculation

**โจทย์**: คำนวณ Median Salary ของแต่ละแผนก (MySQL ไม่มี MEDIAN() function)

```sql
WITH ranked AS (
    SELECT 
        dept_id,
        salary,
        ROW_NUMBER() OVER (PARTITION BY dept_id ORDER BY salary) AS row_asc,
        COUNT(*) OVER (PARTITION BY dept_id) AS total_count
    FROM employees
)
SELECT 
    d.dept_name,
    ROUND(AVG(r.salary), 2) AS median_salary
FROM ranked r
JOIN departments d ON r.dept_id = d.dept_id
WHERE r.row_asc IN (
    FLOOR((r.total_count + 1) / 2),
    CEIL((r.total_count + 1) / 2)
)
GROUP BY d.dept_name;
```

---

### Problem 10: YoY Growth Comparison

**โจทย์**: เปรียบเทียบยอดขายเดือนนี้กับเดือนเดียวกันปีที่แล้ว

```sql
WITH monthly AS (
    SELECT 
        YEAR(order_date) AS yr,
        MONTH(order_date) AS mo,
        DATE_FORMAT(order_date, '%Y-%m') AS month_label,
        SUM(amount) AS revenue,
        COUNT(*) AS order_count
    FROM orders
    WHERE status = 'completed'
    GROUP BY YEAR(order_date), MONTH(order_date)
)
SELECT 
    curr.yr,
    curr.mo,
    curr.month_label,
    ROUND(curr.revenue, 2) AS current_revenue,
    ROUND(prev.revenue, 2) AS prev_year_revenue,
    ROUND(curr.revenue - COALESCE(prev.revenue, 0), 2) AS yoy_change,
    ROUND(
        (curr.revenue - prev.revenue) * 100.0 / NULLIF(prev.revenue, 0),
        1
    ) AS yoy_growth_pct
FROM monthly curr
LEFT JOIN monthly prev ON curr.mo = prev.mo AND curr.yr = prev.yr + 1
ORDER BY curr.yr DESC, curr.mo DESC;
```

---

## Problems 11-20: CTEs and Subqueries

### Problem 11: Org Chart - Full Hierarchy

**โจทย์**: แสดง Organizational Chart ทั้งหมดพร้อม Level และ Path

```sql
WITH RECURSIVE org_chart AS (
    -- Root: Employees without manager
    SELECT 
        emp_id,
        name,
        job_title,
        manager_id,
        dept_id,
        salary,
        0 AS level,
        CAST(name AS CHAR(1000)) AS hierarchy_path
    FROM employees
    WHERE manager_id IS NULL
    
    UNION ALL
    
    -- Recursive: Employees with manager
    SELECT 
        e.emp_id,
        e.name,
        e.job_title,
        e.manager_id,
        e.dept_id,
        e.salary,
        oc.level + 1,
        CONCAT(oc.hierarchy_path, ' > ', e.name)
    FROM employees e
    JOIN org_chart oc ON e.manager_id = oc.emp_id
)
SELECT 
    emp_id,
    CONCAT(REPEAT('  ', level), name) AS indented_name,
    job_title,
    level AS org_level,
    hierarchy_path,
    salary,
    -- Count direct reports
    (SELECT COUNT(*) FROM employees sub WHERE sub.manager_id = org_chart.emp_id) AS direct_reports
FROM org_chart
ORDER BY hierarchy_path;
```

---

### Problem 12: Self-Referencing: Find All Subordinates

**โจทย์**: หา Subordinates ทั้งหมดของ Employee คนหนึ่ง (ทุก Level)

```sql
WITH RECURSIVE subordinates AS (
    SELECT emp_id, name, manager_id, job_title, 1 AS depth
    FROM employees
    WHERE emp_id = 1  -- Find all reports of emp_id = 1 (Alice/CTO)
    
    UNION ALL
    
    SELECT e.emp_id, e.name, e.manager_id, e.job_title, s.depth + 1
    FROM employees e
    JOIN subordinates s ON e.manager_id = s.emp_id
)
SELECT 
    emp_id,
    name,
    job_title,
    depth AS org_levels_below,
    (SELECT name FROM employees WHERE emp_id = subordinates.manager_id) AS reports_to
FROM subordinates
WHERE emp_id != 1  -- Exclude the root
ORDER BY depth, name;
```

---

### Problem 13: Products Bought Together (Market Basket)

**โจทย์**: หาสินค้าคู่ไหนถูกซื้อพร้อมกัน (Market Basket Analysis)

```sql
-- หาคู่สินค้าที่ลูกค้าคนเดียวกันซื้อ (ภายใน 30 วัน)
SELECT 
    p1.name AS product_1,
    p2.name AS product_2,
    COUNT(DISTINCT o1.customer_id) AS customers_bought_both,
    ROUND(
        COUNT(DISTINCT o1.customer_id) * 100.0 / 
        (SELECT COUNT(DISTINCT customer_id) FROM orders WHERE status = 'completed'),
        1
    ) AS support_pct
FROM orders o1
JOIN orders o2 ON o1.customer_id = o2.customer_id 
    AND o1.product_id < o2.product_id  -- Avoid duplicates
    AND ABS(DATEDIFF(o1.order_date, o2.order_date)) <= 30
    AND o1.status = 'completed' AND o2.status = 'completed'
JOIN products p1 ON o1.product_id = p1.product_id
JOIN products p2 ON o2.product_id = p2.product_id
GROUP BY p1.product_id, p2.product_id, p1.name, p2.name
ORDER BY customers_bought_both DESC
LIMIT 10;
```

---

### Problem 14: First and Last Purchase Per Customer

**โจทย์**: หา First Purchase, Last Purchase และ Days Between สำหรับแต่ละ Customer

```sql
SELECT 
    c.customer_id,
    c.name AS customer_name,
    c.country,
    MIN(o.order_date) AS first_purchase,
    MAX(o.order_date) AS last_purchase,
    DATEDIFF(MAX(o.order_date), MIN(o.order_date)) AS days_between_first_last,
    COUNT(o.order_id) AS total_orders,
    ROUND(SUM(o.amount), 2) AS total_spent,
    ROUND(AVG(o.amount), 2) AS avg_order_value,
    -- First product bought
    (SELECT p.name FROM orders o2 JOIN products p ON o2.product_id = p.product_id
     WHERE o2.customer_id = c.customer_id AND o2.status = 'completed'
     ORDER BY o2.order_date LIMIT 1) AS first_product_bought
FROM customers c
LEFT JOIN orders o ON c.customer_id = o.customer_id AND o.status = 'completed'
GROUP BY c.customer_id, c.name, c.country
ORDER BY total_spent DESC;
```

---

### Problem 15: Employee Salary Increase History

**โจทย์**: จาก Salary History ให้หาว่าพนักงานได้ขึ้นเงินเดือนกี่ครั้ง และ % เพิ่มรวม

```sql
-- สมมุติว่า employees table มี salary ปัจจุบัน และมี salary_history
-- ใช้ employees table จำลอง history จาก hire_date
WITH salary_history AS (
    SELECT emp_id, name, salary, hire_date,
           salary * 0.85 AS starting_salary  -- Assume started at 85% of current
    FROM employees
)
SELECT 
    sh.emp_id,
    sh.name,
    ROUND(sh.starting_salary, 2) AS starting_salary,
    ROUND(sh.salary, 2) AS current_salary,
    ROUND(sh.salary - sh.starting_salary, 2) AS total_increase,
    ROUND((sh.salary - sh.starting_salary) * 100.0 / sh.starting_salary, 1) AS total_pct_increase,
    DATEDIFF(CURDATE(), sh.hire_date) / 365 AS years_employed,
    ROUND(
        (sh.salary - sh.starting_salary) * 100.0 / sh.starting_salary / 
        NULLIF(DATEDIFF(CURDATE(), sh.hire_date) / 365, 0),
        1
    ) AS avg_annual_raise_pct
FROM salary_history sh
ORDER BY total_pct_increase DESC;
```

---

## Problems 16-25: Complex Business Logic

### Problem 16: Cohort Retention Analysis

**โจทย์**: คำนวณ Monthly Cohort Retention Rate

```sql
WITH cohorts AS (
    SELECT 
        customer_id,
        DATE_FORMAT(MIN(order_date), '%Y-%m') AS cohort_month
    FROM orders
    WHERE status = 'completed'
    GROUP BY customer_id
),
monthly_activity AS (
    SELECT DISTINCT
        o.customer_id,
        DATE_FORMAT(o.order_date, '%Y-%m') AS active_month
    FROM orders o
    WHERE o.status = 'completed'
),
cohort_data AS (
    SELECT 
        c.cohort_month,
        ma.active_month,
        COUNT(DISTINCT c.customer_id) AS customers,
        -- Month number since cohort start
        PERIOD_DIFF(
            EXTRACT(YEAR_MONTH FROM STR_TO_DATE(CONCAT(ma.active_month, '-01'), '%Y-%m-%d')),
            EXTRACT(YEAR_MONTH FROM STR_TO_DATE(CONCAT(c.cohort_month, '-01'), '%Y-%m-%d'))
        ) AS months_since_cohort
    FROM cohorts c
    JOIN monthly_activity ma ON c.customer_id = ma.customer_id
    GROUP BY c.cohort_month, ma.active_month
)
SELECT 
    cohort_month,
    months_since_cohort,
    customers,
    ROUND(customers * 100.0 / FIRST_VALUE(customers) OVER (
        PARTITION BY cohort_month ORDER BY months_since_cohort
    ), 1) AS retention_pct
FROM cohort_data
ORDER BY cohort_month, months_since_cohort;
```

---

### Problem 17: RFM Segmentation

**โจทย์**: จัด Segment ลูกค้าด้วย RFM Model

```sql
WITH rfm_raw AS (
    SELECT 
        customer_id,
        DATEDIFF(CURDATE(), MAX(order_date)) AS recency_days,
        COUNT(order_id) AS frequency,
        SUM(amount) AS monetary
    FROM orders
    WHERE status = 'completed'
    GROUP BY customer_id
),
rfm_scores AS (
    SELECT 
        customer_id,
        recency_days,
        frequency,
        ROUND(monetary, 2) AS monetary,
        NTILE(5) OVER (ORDER BY recency_days ASC) AS r_score,
        NTILE(5) OVER (ORDER BY frequency ASC) AS f_score,
        NTILE(5) OVER (ORDER BY monetary ASC) AS m_score
    FROM rfm_raw
)
SELECT 
    rs.customer_id,
    c.name,
    recency_days,
    frequency,
    monetary,
    r_score,
    f_score,
    m_score,
    r_score + f_score + m_score AS rfm_total,
    CASE 
        WHEN r_score >= 4 AND f_score >= 4 AND m_score >= 4 THEN 'Champions'
        WHEN r_score >= 4 AND f_score >= 3 THEN 'Loyal Customers'
        WHEN r_score >= 4 AND f_score <= 2 THEN 'New Customers'
        WHEN r_score >= 3 AND f_score >= 3 THEN 'Potential Loyalists'
        WHEN r_score <= 2 AND f_score >= 4 THEN 'At Risk'
        WHEN r_score <= 2 AND f_score <= 2 THEN 'Lost Customers'
        ELSE 'Need Attention'
    END AS rfm_segment
FROM rfm_scores rs
JOIN customers c ON rs.customer_id = c.customer_id
ORDER BY rfm_total DESC;
```

---

### Problem 18: Sessionization

**โจทย์**: แบ่ง User Activity เป็น Sessions (30 นาที inactivity = new session)

```sql
WITH activity_with_gap AS (
    SELECT 
        user_id,
        activity,
        activity_date,
        LAG(activity_date) OVER (PARTITION BY user_id ORDER BY activity_date) AS prev_date,
        DATEDIFF(activity_date, LAG(activity_date) OVER (PARTITION BY user_id ORDER BY activity_date)) AS gap_days
    FROM user_activity
),
session_flags AS (
    SELECT 
        user_id,
        activity,
        activity_date,
        -- New session if gap > 1 day (using days since we only have dates)
        CASE WHEN gap_days > 1 OR gap_days IS NULL THEN 1 ELSE 0 END AS is_new_session
    FROM activity_with_gap
),
sessions AS (
    SELECT 
        user_id,
        activity,
        activity_date,
        SUM(is_new_session) OVER (PARTITION BY user_id ORDER BY activity_date) AS session_num
    FROM session_flags
)
SELECT 
    user_id,
    session_num,
    MIN(activity_date) AS session_start,
    MAX(activity_date) AS session_end,
    COUNT(*) AS activities_in_session,
    DATEDIFF(MAX(activity_date), MIN(activity_date)) AS session_duration_days,
    GROUP_CONCAT(activity ORDER BY activity_date SEPARATOR ' -> ') AS activity_flow
FROM sessions
GROUP BY user_id, session_num
ORDER BY user_id, session_num;
```

---

### Problem 19: Inventory with Expiration (FEFO)

**โจทย์**: First Expired First Out - คำนวณ Stock ที่เหลือหลังขายตาม FEFO

```sql
-- สร้างตารางสมมุติเพื่อแสดง FEFO
CREATE TEMPORARY TABLE inventory_lots (
    lot_id INT, product_id INT, qty INT, expiry_date DATE, unit_cost DECIMAL(10,2)
);
INSERT INTO inventory_lots VALUES
(1, 1, 50, '2024-06-01', 42000),
(2, 1, 30, '2024-03-15', 43000),  -- Earlier expiry
(3, 1, 40, '2024-08-01', 44000);

-- Sell 60 units using FEFO (earliest expiry first)
WITH fefo_lots AS (
    SELECT 
        lot_id,
        product_id,
        qty,
        expiry_date,
        unit_cost,
        SUM(qty) OVER (PARTITION BY product_id ORDER BY expiry_date ASC 
                       ROWS UNBOUNDED PRECEDING) AS cumulative_qty
    FROM inventory_lots
    WHERE product_id = 1
),
fefo_consumption AS (
    SELECT 
        lot_id,
        expiry_date,
        qty,
        unit_cost,
        cumulative_qty,
        60 AS qty_to_sell,  -- Selling 60 units
        CASE 
            WHEN cumulative_qty - qty >= 60 THEN 0
            WHEN cumulative_qty <= 60 THEN qty
            ELSE 60 - (cumulative_qty - qty)
        END AS qty_sold_from_lot,
        CASE 
            WHEN cumulative_qty - qty >= 60 THEN qty
            WHEN cumulative_qty <= 60 THEN 0
            ELSE qty - (60 - (cumulative_qty - qty))
        END AS qty_remaining
    FROM fefo_lots
)
SELECT 
    lot_id,
    expiry_date,
    qty AS original_qty,
    qty_sold_from_lot,
    qty_remaining,
    ROUND(qty_sold_from_lot * unit_cost, 2) AS cogs_from_lot
FROM fefo_consumption
ORDER BY expiry_date;
```

---

### Problem 20: Customer Lifetime Value Prediction

**โจทย์**: คำนวณ LTV และทำนาย Expected Future Value

```sql
WITH customer_metrics AS (
    SELECT 
        c.customer_id,
        c.name,
        c.joined_date,
        DATEDIFF(CURDATE(), c.joined_date) / 30 AS months_as_customer,
        COUNT(o.order_id) AS total_orders,
        SUM(o.amount) AS total_revenue,
        AVG(o.amount) AS avg_order_value,
        DATEDIFF(MAX(o.order_date), MIN(o.order_date)) / 30 AS months_active,
        -- Purchase frequency (orders per month)
        COUNT(o.order_id) / NULLIF(DATEDIFF(MAX(o.order_date), c.joined_date) / 30, 0) AS orders_per_month
    FROM customers c
    LEFT JOIN orders o ON c.customer_id = o.customer_id AND o.status = 'completed'
    GROUP BY c.customer_id, c.name, c.joined_date
)
SELECT 
    customer_id,
    name,
    ROUND(months_as_customer, 1) AS months_as_customer,
    total_orders,
    ROUND(total_revenue, 2) AS actual_ltv,
    ROUND(avg_order_value, 2) AS avg_order_value,
    ROUND(orders_per_month, 2) AS orders_per_month,
    -- Predicted 12-month future value = frequency * avg_value * 12 months * retention_rate
    ROUND(
        orders_per_month * avg_order_value * 12 * 0.85,  -- 85% retention assumption
        2
    ) AS predicted_12m_value,
    -- Customer value score
    CASE 
        WHEN total_revenue > 50000 THEN 'Premium'
        WHEN total_revenue > 20000 THEN 'High'
        WHEN total_revenue > 5000 THEN 'Medium'
        ELSE 'Low'
    END AS value_tier
FROM customer_metrics
ORDER BY actual_ltv DESC;
```

---

## Problems 21-30: Advanced Techniques

### Problem 21: Duplicate Detection

```sql
-- หา Duplicate Orders (ลูกค้าคนเดียว, สินค้าเดียว, วันเดียว - อาจเป็น double-charge)
SELECT 
    customer_id,
    product_id,
    order_date,
    COUNT(*) AS duplicate_count,
    SUM(amount) AS total_amount,
    GROUP_CONCAT(order_id ORDER BY order_id SEPARATOR ', ') AS duplicate_order_ids,
    MIN(order_id) AS keep_order_id,
    GROUP_CONCAT(
        CASE WHEN order_id != MIN(order_id) OVER (PARTITION BY customer_id, product_id, order_date)
             THEN order_id END
        ORDER BY order_id SEPARATOR ', '
    ) AS cancel_order_ids
FROM orders
GROUP BY customer_id, product_id, order_date
HAVING COUNT(*) > 1
ORDER BY duplicate_count DESC;
```

---

### Problem 22: Top N Per Group

```sql
-- Top 3 Products by Revenue Per Category
WITH ranked AS (
    SELECT 
        p.category,
        p.name AS product_name,
        ROUND(SUM(o.amount), 2) AS revenue,
        COUNT(o.order_id) AS orders,
        RANK() OVER (PARTITION BY p.category ORDER BY SUM(o.amount) DESC) AS rnk
    FROM products p
    LEFT JOIN orders o ON p.product_id = o.product_id AND o.status = 'completed'
    GROUP BY p.product_id, p.category, p.name
)
SELECT category, product_name, revenue, orders, rnk
FROM ranked
WHERE rnk <= 3
ORDER BY category, rnk;
```

---

### Problem 23: Cumulative Distribution

```sql
-- Pareto Analysis: 80/20 Rule on Products
WITH product_revenue AS (
    SELECT 
        p.product_id,
        p.name,
        ROUND(SUM(o.amount), 2) AS revenue
    FROM products p
    JOIN orders o ON p.product_id = o.product_id AND o.status = 'completed'
    GROUP BY p.product_id, p.name
),
pareto AS (
    SELECT 
        name,
        revenue,
        SUM(revenue) OVER () AS total_revenue,
        SUM(revenue) OVER (ORDER BY revenue DESC ROWS UNBOUNDED PRECEDING) AS cumulative_revenue,
        ROUND(revenue * 100.0 / SUM(revenue) OVER (), 2) AS revenue_pct,
        ROUND(
            SUM(revenue) OVER (ORDER BY revenue DESC ROWS UNBOUNDED PRECEDING) * 100.0 / 
            SUM(revenue) OVER (),
            2
        ) AS cumulative_pct,
        RANK() OVER (ORDER BY revenue DESC) AS product_rank,
        COUNT(*) OVER () AS total_products
    FROM product_revenue
)
SELECT 
    product_rank,
    name,
    revenue,
    revenue_pct,
    cumulative_pct,
    ROUND(product_rank * 100.0 / total_products, 1) AS product_pct,
    CASE WHEN cumulative_pct <= 80 THEN 'Top 80% Revenue' ELSE 'Remaining 20%' END AS pareto_group
FROM pareto
ORDER BY product_rank;
```

---

### Problem 24: Island Detection (Date Ranges)

```sql
-- หาช่วงวันที่พนักงานทำงานติดต่อกัน (จาก attendance records)
CREATE TEMPORARY TABLE attendance (emp_id INT, work_date DATE);
INSERT INTO attendance VALUES
(1,'2024-01-01'),(1,'2024-01-02'),(1,'2024-01-03'),
(1,'2024-01-07'),(1,'2024-01-08'),  -- Gap on 4-6
(1,'2024-01-10'),(1,'2024-01-11'),(1,'2024-01-12');

WITH grouped AS (
    SELECT 
        emp_id,
        work_date,
        DATE_SUB(work_date, INTERVAL 
            ROW_NUMBER() OVER (PARTITION BY emp_id ORDER BY work_date) 
        DAY) AS island_group
    FROM attendance
)
SELECT 
    emp_id,
    MIN(work_date) AS period_start,
    MAX(work_date) AS period_end,
    COUNT(*) AS days_worked,
    DATEDIFF(MAX(work_date), MIN(work_date)) + 1 AS calendar_days
FROM grouped
GROUP BY emp_id, island_group
ORDER BY emp_id, period_start;
```

---

### Problem 25: String Aggregation and Parsing

```sql
-- Concatenate all products bought by each customer
SELECT 
    c.customer_id,
    c.name AS customer_name,
    GROUP_CONCAT(
        DISTINCT p.name 
        ORDER BY p.name 
        SEPARATOR ' | '
    ) AS products_purchased,
    GROUP_CONCAT(
        DISTINCT p.category 
        ORDER BY p.category 
        SEPARATOR ', '
    ) AS categories_bought,
    COUNT(DISTINCT o.product_id) AS unique_products,
    COUNT(DISTINCT p.category) AS unique_categories
FROM customers c
JOIN orders o ON c.customer_id = o.customer_id AND o.status = 'completed'
JOIN products p ON o.product_id = p.product_id
GROUP BY c.customer_id, c.name
ORDER BY unique_products DESC;
```

---

## Problems 26-35: Performance-Focused

### Problem 26: Avoid N+1 with Single Query

```sql
-- ดึง Employees พร้อม Department Info, Manager Name, Subordinate Count
-- ใน 1 Query แทนที่จะทำหลาย Queries
SELECT 
    e.emp_id,
    e.name AS employee,
    e.job_title,
    e.salary,
    d.dept_name,
    mgr.name AS manager_name,
    mgr.job_title AS manager_title,
    COALESCE(sub_count.cnt, 0) AS direct_reports,
    -- Department stats
    dept_avg.avg_salary AS dept_avg_salary,
    ROUND(e.salary / dept_avg.avg_salary * 100, 1) AS salary_vs_dept_avg_pct
FROM employees e
JOIN departments d ON e.dept_id = d.dept_id
LEFT JOIN employees mgr ON e.manager_id = mgr.emp_id
LEFT JOIN (
    SELECT manager_id, COUNT(*) AS cnt
    FROM employees
    WHERE manager_id IS NOT NULL
    GROUP BY manager_id
) sub_count ON e.emp_id = sub_count.manager_id
LEFT JOIN (
    SELECT dept_id, AVG(salary) AS avg_salary
    FROM employees
    GROUP BY dept_id
) dept_avg ON e.dept_id = dept_avg.dept_id
ORDER BY d.dept_name, e.salary DESC;
```

---

### Problem 27: Efficient Pagination

```sql
-- Keyset Pagination (faster than OFFSET for large tables)
-- First page
SELECT order_id, customer_id, order_date, amount
FROM orders
WHERE order_id > 0  -- Start after this ID (0 = beginning)
ORDER BY order_id ASC
LIMIT 5;

-- Next page (pass last_id from previous result)
-- SELECT order_id, customer_id, order_date, amount
-- FROM orders
-- WHERE order_id > 5  -- Last seen order_id
-- ORDER BY order_id ASC
-- LIMIT 5;

-- Traditional OFFSET (SLOW for large offsets)
SELECT order_id, customer_id, order_date, amount
FROM orders
ORDER BY order_id
LIMIT 5 OFFSET 10;  -- Slow: scans 15 rows, returns 5
```

---

### Problem 28: Optimizing EXISTS vs IN vs JOIN

```sql
-- หา Customers ที่มี order อย่างน้อย 1 รายการ (Completed)

-- Method 1: EXISTS (usually fastest for large datasets)
SELECT c.customer_id, c.name
FROM customers c
WHERE EXISTS (
    SELECT 1 FROM orders o 
    WHERE o.customer_id = c.customer_id 
    AND o.status = 'completed'
);

-- Method 2: IN
SELECT customer_id, name
FROM customers
WHERE customer_id IN (
    SELECT DISTINCT customer_id FROM orders WHERE status = 'completed'
);

-- Method 3: JOIN with DISTINCT
SELECT DISTINCT c.customer_id, c.name
FROM customers c
JOIN orders o ON c.customer_id = o.customer_id
WHERE o.status = 'completed';

-- Method 4: Customers without completed orders (NOT EXISTS)
SELECT c.customer_id, c.name
FROM customers c
WHERE NOT EXISTS (
    SELECT 1 FROM orders o 
    WHERE o.customer_id = c.customer_id 
    AND o.status = 'completed'
);
```

---

### Problem 29: Query Plan Analysis

```sql
-- ตรวจสอบ Query Execution Plan
EXPLAIN SELECT 
    e.name, d.dept_name, e.salary
FROM employees e
JOIN departments d ON e.dept_id = d.dept_id
WHERE e.salary > 80000
ORDER BY e.salary DESC;

-- ดู Extended info
EXPLAIN FORMAT=JSON SELECT 
    e.name, d.dept_name, e.salary
FROM employees e
JOIN departments d ON e.dept_id = d.dept_id
WHERE e.salary > 80000;

-- Add index and compare
CREATE INDEX idx_salary ON employees(salary);
EXPLAIN SELECT * FROM employees WHERE salary > 80000;

-- Show actual stats (MySQL 8.0+)
-- EXPLAIN ANALYZE SELECT e.name FROM employees e WHERE e.salary > 80000;
```

---

### Problem 30: Batch Updates

```sql
-- Update Salary in Batches (ป้องกัน Lock ทั้งตาราง)
-- Method: Update in chunks using WHERE with LIMIT

-- Step 1: Create a control table
CREATE TEMPORARY TABLE update_queue AS
SELECT emp_id, salary * 1.05 AS new_salary
FROM employees
WHERE dept_id = 1;  -- Engineering gets 5% raise

-- Step 2: Update in batch (ปกติจะใช้ Loop ใน Stored Procedure)
UPDATE employees e
JOIN update_queue uq ON e.emp_id = uq.emp_id
SET e.salary = uq.new_salary
WHERE e.dept_id = 1;

-- Verify
SELECT emp_id, name, salary FROM employees WHERE dept_id = 1;
```

---

## Problems 31-40: Data Quality and Validation

### Problem 31: Data Deduplication Strategy

```sql
-- หาและทำความสะอาด Duplicate Customers
INSERT INTO customers VALUES 
(9,'Somchai','Thailand','2020-02-01'),  -- Duplicate of customer 1
(10,'SOMCHAI','Thailand','2020-01-15'); -- Case different

-- Find duplicates by name similarity
SELECT 
    c1.customer_id AS id1,
    c1.name AS name1,
    c1.joined_date AS date1,
    c2.customer_id AS id2,
    c2.name AS name2,
    c2.joined_date AS date2,
    -- SOUNDEX-based matching (phonetic similarity)
    CASE WHEN UPPER(c1.name) = UPPER(c2.name) THEN 'EXACT MATCH'
         WHEN SOUNDEX(c1.name) = SOUNDEX(c2.name) THEN 'SOUNDEX MATCH'
         ELSE 'SIMILAR'
    END AS match_type
FROM customers c1
JOIN customers c2 ON c1.customer_id < c2.customer_id
    AND c1.country = c2.country
    AND (UPPER(c1.name) = UPPER(c2.name) 
         OR SOUNDEX(c1.name) = SOUNDEX(c2.name))
ORDER BY match_type, c1.customer_id;
```

---

### Problem 32: NULL Handling Patterns

```sql
-- Patterns สำหรับ NULL handling ที่ถูกต้อง
SELECT 
    e.emp_id,
    e.name,
    e.manager_id,
    -- COALESCE: first non-null value
    COALESCE(mgr.name, 'No Manager') AS manager_name,
    -- NULLIF: convert specific value to NULL
    NULLIF(e.dept_id, 0) AS dept_id_clean,
    -- IFNULL: 2-arg COALESCE
    IFNULL(e.manager_id, 0) AS manager_id_or_zero,
    -- IS NULL vs = NULL (= NULL never matches!)
    CASE WHEN e.manager_id IS NULL THEN 'Top Level' ELSE 'Has Manager' END AS hierarchy_level,
    -- NULL in aggregates (AVG ignores NULLs)
    -- COUNT(*) counts all rows; COUNT(col) excludes NULLs
    (SELECT COUNT(*) FROM employees e2 WHERE e2.dept_id = e.dept_id) AS dept_total_with_nulls,
    (SELECT COUNT(manager_id) FROM employees e2 WHERE e2.dept_id = e.dept_id) AS has_manager_count
FROM employees e
LEFT JOIN employees mgr ON e.manager_id = mgr.emp_id;
```

---

### Problem 33: Upsert Pattern (INSERT ... ON DUPLICATE KEY)

```sql
-- Upsert: Insert or Update
-- ใช้เมื่อต้องการ Insert ถ้าไม่มี, Update ถ้ามีอยู่แล้ว

-- First ensure unique constraint exists
-- ALTER TABLE customers ADD UNIQUE KEY uk_email (email);

-- Upsert pattern
INSERT INTO customers (customer_id, name, country, joined_date)
VALUES (1, 'Somchai Updated', 'Thailand', '2020-01-15')
ON DUPLICATE KEY UPDATE 
    name = VALUES(name),
    country = VALUES(country);

-- REPLACE INTO (DELETE + INSERT - loses data!)
-- Avoid this as it changes the primary key

-- Safer with INSERT IGNORE (skip on duplicate)
INSERT IGNORE INTO customers (customer_id, name, country, joined_date)
VALUES (1, 'This will be ignored', 'Thailand', '2020-01-15');
```

---

### Problem 34: Date/Time Edge Cases

```sql
-- Common Date/Time Problems and Solutions
SELECT 
    -- Last day of month
    LAST_DAY('2024-02-01') AS last_day_feb_2024,
    LAST_DAY('2024-03-01') AS last_day_mar,
    
    -- First day of month
    DATE_FORMAT(CURDATE(), '%Y-%m-01') AS first_day_this_month,
    DATE_FORMAT(DATE_ADD(CURDATE(), INTERVAL 1 MONTH), '%Y-%m-01') AS first_day_next_month,
    
    -- Start and end of week (Thai week starts Monday)
    DATE_ADD(CURDATE(), INTERVAL (1-DAYOFWEEK(CURDATE())) DAY) AS start_of_week_sun,
    DATE_SUB(CURDATE(), INTERVAL WEEKDAY(CURDATE()) DAY) AS start_of_week_mon,
    
    -- Age calculation (exact years)
    TIMESTAMPDIFF(YEAR, '1990-08-15', CURDATE()) AS age_years,
    
    -- Business days (approximate, excluding weekends)
    (DATEDIFF('2024-02-28', '2024-02-01') + 1) 
        - (FLOOR((DATEDIFF('2024-02-28', '2024-02-01') + DAYOFWEEK('2024-02-01')) / 7) * 2)
        - IF(DAYOFWEEK('2024-02-01') = 1, 1, 0)
    AS approx_business_days,
    
    -- Quarter
    QUARTER(CURDATE()) AS current_quarter,
    CONCAT('Q', QUARTER(CURDATE()), ' ', YEAR(CURDATE())) AS quarter_label;
```

---

### Problem 35: JSON in MySQL

```sql
-- Working with JSON data in MySQL 5.7+
ALTER TABLE employees ADD COLUMN skills JSON;

UPDATE employees SET skills = JSON_ARRAY('Python', 'SQL', 'Docker') WHERE emp_id = 2;
UPDATE employees SET skills = JSON_ARRAY('SQL', 'MySQL', 'PostgreSQL') WHERE emp_id = 3;
UPDATE employees SET skills = JSON_ARRAY('React', 'Node.js', 'SQL') WHERE emp_id = 11;

-- Query JSON
SELECT 
    emp_id,
    name,
    skills,
    JSON_LENGTH(skills) AS skill_count,
    JSON_EXTRACT(skills, '$[0]') AS first_skill,
    -- Check if skill exists
    JSON_CONTAINS(skills, '"SQL"') AS knows_sql,
    -- Extract as text (without quotes)
    skills->>'$[0]' AS first_skill_clean
FROM employees
WHERE JSON_CONTAINS(skills, '"SQL"') = 1;

-- JSON Aggregation
SELECT 
    dept_id,
    JSON_ARRAYAGG(name ORDER BY name) AS team_members,
    JSON_OBJECTAGG(CAST(emp_id AS CHAR), name) AS id_name_map
FROM employees
GROUP BY dept_id;
```

---

## Problems 36-45: Interview Favorites

### Problem 36: Nth Highest Salary (Classic)

```sql
-- N-th Highest Salary (n=3)
-- Method 1: Using DENSE_RANK
SELECT salary FROM (
    SELECT salary, DENSE_RANK() OVER (ORDER BY salary DESC) AS dr
    FROM employees
) ranked
WHERE dr = 3
LIMIT 1;

-- Method 2: Subquery
SELECT MAX(salary)
FROM employees
WHERE salary < (
    SELECT MAX(salary)
    FROM employees
    WHERE salary < (SELECT MAX(salary) FROM employees)
);

-- Method 3: LIMIT OFFSET
SELECT DISTINCT salary
FROM employees
ORDER BY salary DESC
LIMIT 1 OFFSET 2;  -- 0-based, so 2 = 3rd
```

---

### Problem 37: Employees Earning More Than Manager

```sql
SELECT 
    e.emp_id,
    e.name AS employee,
    e.salary AS emp_salary,
    mgr.name AS manager,
    mgr.salary AS mgr_salary,
    ROUND(e.salary - mgr.salary, 2) AS salary_difference
FROM employees e
JOIN employees mgr ON e.manager_id = mgr.emp_id
WHERE e.salary > mgr.salary
ORDER BY salary_difference DESC;
```

---

### Problem 38: Departments Without Employees

```sql
-- ใช้ LEFT JOIN + IS NULL (efficient)
SELECT d.dept_id, d.dept_name, d.location
FROM departments d
LEFT JOIN employees e ON d.dept_id = e.dept_id
WHERE e.emp_id IS NULL;

-- Alternative: NOT EXISTS
SELECT dept_id, dept_name
FROM departments d
WHERE NOT EXISTS (
    SELECT 1 FROM employees e WHERE e.dept_id = d.dept_id
);

-- Alternative: NOT IN (careful with NULLs)
SELECT dept_id, dept_name
FROM departments
WHERE dept_id NOT IN (
    SELECT DISTINCT dept_id FROM employees WHERE dept_id IS NOT NULL
);
```

---

### Problem 39: Find Customers Who Bought All Products

```sql
-- หา Customers ที่ซื้อสินค้าครบทุก Category
SELECT 
    c.customer_id,
    c.name,
    COUNT(DISTINCT p.category) AS categories_bought,
    (SELECT COUNT(DISTINCT category) FROM products) AS total_categories,
    GROUP_CONCAT(DISTINCT p.category ORDER BY p.category) AS bought_categories
FROM customers c
JOIN orders o ON c.customer_id = o.customer_id AND o.status = 'completed'
JOIN products p ON o.product_id = p.product_id
GROUP BY c.customer_id, c.name
HAVING COUNT(DISTINCT p.category) = (SELECT COUNT(DISTINCT category) FROM products);
```

---

### Problem 40: Swap Salary (Classic Facebook Question)

```sql
-- Swap: Male salary becomes Female and vice versa
-- Original problem: update in-place without temp table

-- Assume genders in employees
SELECT 
    emp_id,
    name,
    salary,
    CASE 
        WHEN emp_id % 2 = 0 THEN salary * 1.1  -- "Female employees" get +10%
        ELSE salary * 0.9   -- "Male employees" -10% (illustrative only)
    END AS adjusted_salary
FROM employees;

-- Classic "swap two values" without temp variable
UPDATE employees e1
JOIN employees e2 ON e1.emp_id = e2.emp_id + 1  -- Swap adjacent pairs
SET e1.salary = e2.salary, e2.salary = e1.salary
WHERE e1.emp_id % 2 = 0;  -- Only even IDs (pairs)
```

---

## Problems 41-50: Expert Level

### Problem 41: Recursive Category Tree

```sql
CREATE TEMPORARY TABLE category_tree (
    cat_id INT, parent_id INT, name VARCHAR(100)
);
INSERT INTO category_tree VALUES
(1,NULL,'Electronics'),
(2,1,'Computers'),(3,1,'Mobile'),
(4,2,'Laptops'),(5,2,'Desktops'),
(6,3,'Smartphones'),(7,3,'Tablets');

WITH RECURSIVE tree AS (
    SELECT cat_id, parent_id, name, 0 AS depth, 
           CAST(name AS CHAR(500)) AS path,
           CAST(cat_id AS CHAR(100)) AS id_path
    FROM category_tree WHERE parent_id IS NULL
    UNION ALL
    SELECT c.cat_id, c.parent_id, c.name, t.depth+1,
           CONCAT(t.path, ' > ', c.name),
           CONCAT(t.id_path, '/', c.cat_id)
    FROM category_tree c JOIN tree t ON c.parent_id = t.cat_id
)
SELECT cat_id, CONCAT(REPEAT('  ', depth), name) AS category, depth, path
FROM tree ORDER BY id_path;
```

---

### Problem 42: Sliding Window Fraud Detection

```sql
-- ตรวจ Fraud: 3+ transactions ใน 10 นาที จาก Customer เดียวกัน
WITH order_windows AS (
    SELECT 
        o1.order_id,
        o1.customer_id,
        o1.order_date,
        o1.amount,
        COUNT(o2.order_id) AS orders_in_window,
        SUM(o2.amount) AS amount_in_window
    FROM orders o1
    JOIN orders o2 ON o1.customer_id = o2.customer_id
        AND o2.order_date BETWEEN 
            DATE_SUB(o1.order_date, INTERVAL 1 DAY) 
            AND o1.order_date
        AND o2.order_id != o1.order_id
        AND o2.status = 'completed'
    WHERE o1.status = 'completed'
    GROUP BY o1.order_id, o1.customer_id, o1.order_date, o1.amount
)
SELECT 
    ow.*,
    c.name AS customer_name
FROM order_windows ow
JOIN customers c ON ow.customer_id = c.customer_id
WHERE ow.orders_in_window >= 2
ORDER BY ow.orders_in_window DESC;
```

---

### Problem 43: Generate Calendar

```sql
-- สร้าง Calendar ของปี 2024 ใน SQL
WITH RECURSIVE calendar AS (
    SELECT DATE('2024-01-01') AS dt
    UNION ALL
    SELECT DATE_ADD(dt, INTERVAL 1 DAY)
    FROM calendar
    WHERE dt < DATE('2024-12-31')
)
SELECT 
    dt AS date,
    DAYNAME(dt) AS day_name,
    WEEK(dt, 1) AS week_number,
    MONTH(dt) AS month_num,
    MONTHNAME(dt) AS month_name,
    QUARTER(dt) AS quarter,
    CASE WHEN DAYOFWEEK(dt) IN (1,7) THEN 'Weekend' ELSE 'Weekday' END AS day_type,
    -- Public holidays (Thailand 2024, approximate)
    CASE dt
        WHEN '2024-01-01' THEN 'New Year'
        WHEN '2024-04-06' THEN 'Chakri Day'
        WHEN '2024-04-13' THEN 'Songkran'
        WHEN '2024-04-15' THEN 'Songkran'
        WHEN '2024-05-01' THEN 'Labour Day'
        WHEN '2024-05-22' THEN 'Visakha Bucha'
        WHEN '2024-07-22' THEN 'Asalha Bucha'
        WHEN '2024-08-12' THEN 'Mother Day'
        WHEN '2024-10-23' THEN 'Chulalongkorn Day'
        WHEN '2024-12-05' THEN 'Father Day'
        WHEN '2024-12-10' THEN 'Constitution Day'
        WHEN '2024-12-31' THEN 'New Year Eve'
        ELSE NULL
    END AS holiday_name
FROM calendar
WHERE MONTH(dt) = 2  -- February only (shorten output)
ORDER BY dt;
```

---

### Problem 44: Dynamic SQL String Building

```sql
-- สร้าง Dynamic WHERE clause (ใช้ใน Stored Procedure)
DELIMITER //

CREATE PROCEDURE search_employees(
    IN p_dept_id INT,
    IN p_min_salary DECIMAL(10,2),
    IN p_job_title VARCHAR(100)
)
BEGIN
    -- MySQL doesn't have true dynamic SQL easily from SELECT,
    -- but here's the PREPARE/EXECUTE approach
    SET @sql = 'SELECT emp_id, name, salary, job_title FROM employees WHERE 1=1';
    
    IF p_dept_id IS NOT NULL THEN
        SET @sql = CONCAT(@sql, ' AND dept_id = ', p_dept_id);
    END IF;
    
    IF p_min_salary IS NOT NULL THEN
        SET @sql = CONCAT(@sql, ' AND salary >= ', p_min_salary);
    END IF;
    
    IF p_job_title IS NOT NULL AND p_job_title != '' THEN
        SET @sql = CONCAT(@sql, ' AND job_title LIKE ''%', p_job_title, '%''');
    END IF;
    
    SET @sql = CONCAT(@sql, ' ORDER BY salary DESC');
    
    PREPARE stmt FROM @sql;
    EXECUTE stmt;
    DEALLOCATE PREPARE stmt;
END //

DELIMITER ;

CALL search_employees(1, 80000, NULL);
```

---

### Problem 45: Materialized View Pattern

```sql
-- MySQL ไม่มี Materialized View แต่ทำได้ด้วย Table + Event/Trigger

CREATE TABLE mv_department_stats (
    dept_id         INT PRIMARY KEY,
    dept_name       VARCHAR(100),
    emp_count       INT,
    avg_salary      DECIMAL(10,2),
    max_salary      DECIMAL(10,2),
    min_salary      DECIMAL(10,2),
    total_salary    DECIMAL(12,2),
    last_refreshed  TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    FOREIGN KEY (dept_id) REFERENCES departments(dept_id)
);

-- Refresh procedure
DELIMITER //
CREATE PROCEDURE refresh_dept_stats()
BEGIN
    REPLACE INTO mv_department_stats (dept_id, dept_name, emp_count, avg_salary, max_salary, min_salary, total_salary)
    SELECT 
        d.dept_id,
        d.dept_name,
        COUNT(e.emp_id),
        ROUND(AVG(e.salary), 2),
        MAX(e.salary),
        MIN(e.salary),
        SUM(e.salary)
    FROM departments d
    LEFT JOIN employees e ON d.dept_id = e.dept_id
    GROUP BY d.dept_id, d.dept_name;
END //
DELIMITER ;

CALL refresh_dept_stats();
SELECT * FROM mv_department_stats;
```

---

### Problems 46-50: Final Boss Questions

### Problem 46: Time-Based Inventory Valuation

```sql
-- คำนวณมูลค่าสินค้าคงคลัง ณ วันที่ใดก็ได้ในอดีต
-- ใช้ Stock Moves เป็น Audit Trail
WITH moves AS (
    SELECT 
        product_id,
        SUM(CASE WHEN move_type = 'receipt' THEN quantity ELSE -quantity END) AS net_qty,
        SUM(CASE WHEN move_type = 'receipt' THEN quantity * unit_cost 
                 ELSE -quantity * unit_cost END) AS net_cost
    FROM (
        SELECT 'receipt' AS move_type, p.product_id, 
               100 AS quantity, p.standard_cost AS unit_cost
        FROM products p
        UNION ALL
        SELECT 'delivery', p.product_id, 30, p.standard_cost
        FROM products p WHERE p.product_id <= 3
    ) simulated_moves
    GROUP BY product_id
)
SELECT 
    p.product_id,
    p.name,
    ROUND(m.net_qty, 0) AS estimated_on_hand,
    ROUND(m.net_cost / NULLIF(m.net_qty, 0), 2) AS avg_unit_cost,
    ROUND(m.net_cost, 2) AS inventory_value
FROM products p
JOIN moves m ON p.product_id = m.product_id
ORDER BY m.net_cost DESC;
```

---

### Problem 47: Customer Journey Attribution

```sql
-- Multi-Touch Attribution: แต่ละ Touchpoint ได้ Credit เท่าไร
WITH customer_journey AS (
    SELECT 
        customer_id,
        activity,
        activity_date,
        ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY activity_date) AS touch_num,
        COUNT(*) OVER (PARTITION BY customer_id) AS total_touches,
        CASE activity WHEN 'purchase' THEN 1 ELSE 0 END AS is_conversion
    FROM user_activity
),
converted_customers AS (
    SELECT DISTINCT customer_id
    FROM user_activity WHERE activity = 'purchase'
),
attribution AS (
    SELECT 
        cj.customer_id,
        cj.activity AS channel,
        cj.touch_num,
        cj.total_touches,
        -- First Touch Attribution
        CASE WHEN cj.touch_num = 1 THEN 1.0 ELSE 0 END AS first_touch_credit,
        -- Last Touch Attribution
        CASE WHEN cj.touch_num = cj.total_touches - 1 THEN 1.0 ELSE 0 END AS last_touch_credit,
        -- Linear Attribution
        1.0 / (cj.total_touches - 1) AS linear_credit
    FROM customer_journey cj
    JOIN converted_customers cc ON cj.customer_id = cc.customer_id
    WHERE cj.is_conversion = 0  -- Touchpoints before conversion
)
SELECT 
    channel,
    COUNT(*) AS total_touches,
    ROUND(SUM(first_touch_credit), 2) AS first_touch_total,
    ROUND(SUM(last_touch_credit), 2) AS last_touch_total,
    ROUND(SUM(linear_credit), 2) AS linear_total
FROM attribution
GROUP BY channel
ORDER BY linear_total DESC;
```

---

### Problem 48: Recursive Bill of Materials

```sql
-- Bill of Materials: หา Components ทั้งหมดของ Product
CREATE TEMPORARY TABLE bom (
    component_id INT, parent_id INT, name VARCHAR(100), qty_per_parent DECIMAL(10,4), unit_cost DECIMAL(10,2)
);
INSERT INTO bom VALUES
(1,NULL,'Laptop',1,NULL),
(2,1,'Motherboard',1,8000),
(3,1,'Screen',1,5000),
(4,1,'Battery',1,2000),
(5,2,'CPU',1,12000),
(6,2,'RAM',2,1500),
(7,2,'SSD',1,3000);

WITH RECURSIVE bom_explosion AS (
    SELECT component_id, parent_id, name, qty_per_parent, unit_cost, 
           1 AS depth, qty_per_parent AS total_qty
    FROM bom WHERE parent_id IS NULL
    UNION ALL
    SELECT b.component_id, b.parent_id, b.name, b.qty_per_parent, b.unit_cost,
           be.depth + 1,
           be.total_qty * b.qty_per_parent
    FROM bom b JOIN bom_explosion be ON b.parent_id = be.component_id
)
SELECT 
    CONCAT(REPEAT('  ', depth-1), name) AS component,
    total_qty,
    unit_cost,
    ROUND(total_qty * COALESCE(unit_cost, 0), 2) AS extended_cost,
    depth AS level
FROM bom_explosion
ORDER BY depth, component_id;
```

---

### Problem 49: Weighted Average Calculation

```sql
-- Weighted Average Price (VWAP - Volume Weighted Average Price)
WITH product_orders AS (
    SELECT 
        p.product_id,
        p.name,
        p.category,
        o.amount,
        o.order_date,
        -- Assume quantity = 1 for this example; in real scenario, have quantity column
        1 AS quantity
    FROM products p
    JOIN orders o ON p.product_id = o.product_id AND o.status = 'completed'
),
vwap AS (
    SELECT 
        product_id,
        name,
        category,
        SUM(amount * quantity) AS total_value,
        SUM(quantity) AS total_quantity,
        -- VWAP
        ROUND(SUM(amount * quantity) / NULLIF(SUM(quantity), 0), 2) AS vwap,
        COUNT(*) AS trade_count,
        MIN(amount) AS min_price,
        MAX(amount) AS max_price,
        AVG(amount) AS simple_avg_price
    FROM product_orders
    GROUP BY product_id, name, category
)
SELECT 
    *,
    ROUND(vwap - simple_avg_price, 2) AS vwap_vs_simple_avg
FROM vwap
ORDER BY total_value DESC;
```

---

### Problem 50: Master Final Challenge - Full Analytics Pipeline

```sql
-- Full Analytics Query: Complete Sales Dashboard
WITH date_range AS (
    SELECT '2023-01-01' AS start_date, '2023-12-31' AS end_date
),
sales_base AS (
    SELECT 
        o.order_id,
        o.customer_id,
        o.product_id,
        o.order_date,
        o.amount,
        p.category,
        p.name AS product_name,
        c.country,
        DATE_FORMAT(o.order_date, '%Y-%m') AS month
    FROM orders o
    JOIN products p ON o.product_id = p.product_id
    JOIN customers c ON o.customer_id = c.customer_id
    CROSS JOIN date_range dr
    WHERE o.status = 'completed'
      AND o.order_date BETWEEN dr.start_date AND dr.end_date
),
monthly_kpis AS (
    SELECT 
        month,
        COUNT(DISTINCT order_id) AS orders,
        COUNT(DISTINCT customer_id) AS customers,
        ROUND(SUM(amount), 2) AS revenue,
        ROUND(AVG(amount), 2) AS aov,
        SUM(amount) / COUNT(DISTINCT customer_id) AS revenue_per_customer
    FROM sales_base
    GROUP BY month
),
category_kpis AS (
    SELECT 
        category,
        COUNT(DISTINCT order_id) AS orders,
        ROUND(SUM(amount), 2) AS revenue,
        ROUND(SUM(amount) * 100.0 / SUM(SUM(amount)) OVER (), 2) AS revenue_share_pct,
        RANK() OVER (ORDER BY SUM(amount) DESC) AS revenue_rank
    FROM sales_base
    GROUP BY category
),
customer_kpis AS (
    SELECT 
        country,
        COUNT(DISTINCT customer_id) AS customers,
        ROUND(SUM(amount), 2) AS revenue,
        ROUND(AVG(amount), 2) AS avg_order
    FROM sales_base
    GROUP BY country
)
-- Final output
SELECT 
    'Monthly KPIs' AS section, month AS dimension,
    CONCAT('Orders: ', orders, ' | Revenue: ', FORMAT(revenue, 0), ' | AOV: ', FORMAT(aov, 0)) AS metrics
FROM monthly_kpis
UNION ALL
SELECT 'Category Revenue', category, 
    CONCAT('Revenue: ', FORMAT(revenue, 0), ' (', revenue_share_pct, '%) | Rank: ', revenue_rank)
FROM category_kpis
UNION ALL
SELECT 'Country Stats', country, 
    CONCAT('Customers: ', customers, ' | Revenue: ', FORMAT(revenue, 0))
FROM customer_kpis
ORDER BY section, dimension;
```

---

## สรุป Tips สำหรับ SQL Interview

```
1. ALWAYS clarify requirements ก่อนเขียน Query
   - ถาม: "Table มี index ที่ไหน?" 
   - ถาม: "ต้องการ exact result หรือ approximate?"
   - ถาม: "มี NULL values ไหม? ควร handle อย่างไร?"

2. Start simple, then optimize
   - เขียน Query ที่ถูกต้องก่อน
   - จากนั้น optimize ทีละขั้น
   - อธิบาย Trade-offs

3. Window Functions vs Subqueries vs CTEs
   - Window Functions: readable, good for ranking/running totals
   - CTEs: readable, reusable, debugging-friendly
   - Subqueries: sometimes faster, sometimes not

4. Index Strategy
   - Composite index: Column ใน WHERE, JOIN, ORDER BY
   - Covering index: เมื่อ SELECT columns เป็น subset ของ index
   - Avoid functions on indexed columns in WHERE

5. NULL handling
   - NULL != NULL (use IS NULL / IS NOT NULL)
   - NULL in NOT IN → always no results!
   - COALESCE/NULLIF/IFNULL

6. Performance red flags
   - SELECT * in production
   - Functions on indexed columns (WHERE YEAR(date) = 2024)
   - N+1 queries (solve with JOIN or IN)
   - Missing indexes on FK columns
   - OFFSET for deep pagination (use keyset)
```

---

## แบบฝึกหัด (Challenge Exercises)

1. **Median vs Mean**: เขียน Query แสดงความแตกต่างระหว่าง Median และ Mean Salary ของแต่ละแผนก และอธิบายว่าทำไมถึงต่างกัน

2. **Time Zone Conversion**: เขียน Query แปลง Timestamps จาก UTC เป็น Bangkok Time (UTC+7) และ Tokyo Time (UTC+9) พร้อมกัน

3. **Longest Common Sequence**: หา Sequence ของ Products ที่ลูกค้าซื้อในลำดับที่เหมือนกันมากที่สุด (เช่น Laptop → Mouse → Keyboard)

4. **Complex Permissions**: ออกแบบ RBAC (Role-Based Access Control) และเขียน Query ตรวจสอบว่า User มีสิทธิ์ทำ Action ใดๆ หรือไม่ (รวม Inherited permissions)

5. **Approximations**: เขียน Query ประมาณค่า HyperLogLog (Distinct Count) และ Count-Min Sketch โดยไม่ใช้ exact COUNT DISTINCT

6. **Sharding Key Selection**: วิเคราะห์ Data Distribution ของ Orders Table และแนะนำ Sharding Key ที่เหมาะสม โดยคำนวณ Cardinality และ Hot Spots

7. **CQRS Pattern**: ออกแบบ Command (Write) และ Query (Read) Models สำหรับ Order System โดยแยก Schema สำหรับ OLTP และ Reporting

8. **Explain Query Cost**: ใช้ EXPLAIN ANALYZE อธิบาย Cost ของ Query ที่มี Nested Loop Join vs Hash Join และเมื่อไรควรใช้แต่ละแบบ

9. **Temporal Tables**: จำลอง Temporal Tables (ข้อมูลที่ Valid ณ เวลาต่างๆ) โดยใช้ valid_from/valid_to columns และเขียน Query ดึงข้อมูล ณ จุดเวลาใดก็ได้

10. **SQL Antipatterns**: ระบุและแก้ไข Antipatterns ใน Query ที่กำหนด (Entity-Attribute-Value, Polymorphic Associations, Implicit Columns)
