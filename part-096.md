# ส่วนที่ 96: Advanced CTEs and Window Function Patterns

## บทนำ

ในบทนี้เราจะรวม CTEs และ Window Functions เข้าด้วยกันเพื่อแก้ปัญหาที่ซับซ้อนในโลกจริง โดยเฉพาะปัญหา Gap-and-Island ที่พบบ่อยใน time series analysis, session analysis, และ consecutive event detection

---

## 96.1 Gap and Island Problem

"Gap and Island" คือปัญหาในการหา "เกาะ" (islands) ของแถวที่ต่อเนื่องกัน และ "ช่องว่าง" (gaps) ระหว่างเกาะเหล่านั้น

### ตัวอย่างที่ 1: Classic Gap and Island

```sql
-- สร้างข้อมูล: วันที่พนักงานเข้างาน
CREATE TABLE employee_attendance (
    emp_id      INT,
    work_date   DATE
);

INSERT INTO employee_attendance VALUES
(1, '2024-01-02'), (1, '2024-01-03'), (1, '2024-01-04'),
-- gap: 2024-01-05 (หยุด)
(1, '2024-01-08'), (1, '2024-01-09'),
-- gap: 2024-01-10 ถึง 01-12
(1, '2024-01-15'), (1, '2024-01-16'), (1, '2024-01-17'), (1, '2024-01-18'),
(2, '2024-01-02'), (2, '2024-01-03'),
(2, '2024-01-08'), (2, '2024-01-09'), (2, '2024-01-10');

-- วิธีที่ 1: ROW_NUMBER - DATE difference
WITH island_groups AS (
    SELECT 
        emp_id,
        work_date,
        work_date - (ROW_NUMBER() OVER (PARTITION BY emp_id ORDER BY work_date) || ' days')::INTERVAL AS grp
        -- ถ้า dates ต่อเนื่องกัน grp จะเท่ากัน
    FROM employee_attendance
)
SELECT 
    emp_id,
    MIN(work_date) AS island_start,
    MAX(work_date) AS island_end,
    COUNT(*) AS consecutive_days
FROM island_groups
GROUP BY emp_id, grp
ORDER BY emp_id, island_start;
```

### ตัวอย่างที่ 2: Gap Detection

```sql
-- หา gaps (วันที่ขาดงาน)
WITH attendance_with_next AS (
    SELECT 
        emp_id,
        work_date,
        LEAD(work_date) OVER (PARTITION BY emp_id ORDER BY work_date) AS next_work_date
    FROM employee_attendance
),
gaps AS (
    SELECT 
        emp_id,
        work_date AS gap_after,
        next_work_date AS next_work,
        next_work_date - work_date - 1 AS gap_days
    FROM attendance_with_next
    WHERE next_work_date - work_date > 1  -- มี gap
)
SELECT 
    emp_id,
    gap_after,
    next_work,
    gap_days,
    -- สร้าง list ของวันที่ขาด
    gap_after + 1 AS first_absent_day,
    next_work - 1 AS last_absent_day
FROM gaps
ORDER BY emp_id, gap_after;
```

### ตัวอย่างที่ 3: Island ด้วย LAG (วิธีที่ 2)

```sql
-- วิธีที่ 2: ใช้ LAG เพื่อสร้าง group markers
WITH change_markers AS (
    SELECT 
        emp_id,
        work_date,
        CASE 
            WHEN work_date - LAG(work_date) OVER (PARTITION BY emp_id ORDER BY work_date) = 1 
            THEN 0  -- ต่อเนื่อง
            ELSE 1  -- เริ่ม island ใหม่
        END AS new_island
    FROM employee_attendance
),
island_numbers AS (
    SELECT 
        emp_id,
        work_date,
        SUM(new_island) OVER (PARTITION BY emp_id ORDER BY work_date) AS island_id
    FROM change_markers
)
SELECT 
    emp_id,
    island_id,
    MIN(work_date) AS start_date,
    MAX(work_date) AS end_date,
    COUNT(*) AS days_in_island
FROM island_numbers
GROUP BY emp_id, island_id
ORDER BY emp_id, start_date;
```

---

## 96.2 Session Analysis (Clickstream)

### ตัวอย่างที่ 4: Web Session Detection

```sql
-- กำหนด session: gap > 30 นาที = session ใหม่
CREATE TABLE page_events (
    event_id    INT,
    user_id     INT,
    page        VARCHAR(100),
    event_time  TIMESTAMP
);

INSERT INTO page_events VALUES
(1,  101, '/home',      '2024-01-01 09:00:00'),
(2,  101, '/products',  '2024-01-01 09:05:00'),
(3,  101, '/cart',      '2024-01-01 09:10:00'),
(4,  101, '/checkout',  '2024-01-01 09:15:00'),
-- 45-minute gap: new session
(5,  101, '/home',      '2024-01-01 10:00:00'),
(6,  101, '/about',     '2024-01-01 10:03:00'),
(7,  102, '/home',      '2024-01-01 11:00:00'),
(8,  102, '/products',  '2024-01-01 11:02:00'),
(9,  102, '/home',      '2024-01-01 11:05:00');

-- ตรวจจับ session boundaries
WITH session_starts AS (
    SELECT 
        event_id,
        user_id,
        page,
        event_time,
        EXTRACT(EPOCH FROM 
            event_time - LAG(event_time) OVER (PARTITION BY user_id ORDER BY event_time)
        ) / 60 AS minutes_from_prev,
        CASE 
            WHEN event_time - LAG(event_time) OVER (PARTITION BY user_id ORDER BY event_time) > INTERVAL '30 minutes'
              OR LAG(event_time) OVER (PARTITION BY user_id ORDER BY event_time) IS NULL
            THEN 1
            ELSE 0
        END AS is_new_session
    FROM page_events
),
with_session_id AS (
    SELECT *,
           SUM(is_new_session) OVER (PARTITION BY user_id ORDER BY event_time) AS session_id
    FROM session_starts
)
SELECT 
    user_id,
    session_id,
    MIN(event_time) AS session_start,
    MAX(event_time) AS session_end,
    COUNT(*) AS pages_viewed,
    MAX(event_time) - MIN(event_time) AS session_duration,
    ARRAY_AGG(page ORDER BY event_time) AS page_sequence
FROM with_session_id
GROUP BY user_id, session_id
ORDER BY user_id, session_id;
```

### ตัวอย่างที่ 5: Session Attribution

```sql
-- วิเคราะห์ว่าแต่ละ session มาจากช่องทางไหน
WITH sessions AS (
    SELECT 
        user_id,
        session_id,
        MIN(event_time) AS session_start,
        MAX(event_time) AS session_end,
        COUNT(*) AS page_count,
        FIRST_VALUE(page) OVER (PARTITION BY user_id, session_id ORDER BY event_time) AS entry_page,
        LAST_VALUE(page) OVER (
            PARTITION BY user_id, session_id ORDER BY event_time
            ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
        ) AS exit_page
    FROM (
        SELECT *,
               SUM(is_new_session) OVER (PARTITION BY user_id ORDER BY event_time) AS session_id
        FROM session_starts
    ) s
    GROUP BY user_id, session_id, page, event_time
)
SELECT DISTINCT
    user_id,
    session_id,
    session_start,
    session_end,
    page_count,
    entry_page,
    exit_page
FROM sessions
ORDER BY user_id, session_id;
```

---

## 96.3 Consecutive Days/Events Detection

### ตัวอย่างที่ 6: Longest Streak

```sql
-- หา longest streak ของการขายต่อเนื่อง
CREATE TABLE daily_sales (
    sales_date  DATE,
    units_sold  INT
);

INSERT INTO daily_sales VALUES
('2024-01-01', 5), ('2024-01-02', 8), ('2024-01-03', 3),
('2024-01-04', 0), ('2024-01-05', 0),  -- no sales
('2024-01-06', 7), ('2024-01-07', 4), ('2024-01-08', 9),
('2024-01-09', 11), ('2024-01-10', 2),
('2024-01-11', 0),  -- no sales
('2024-01-12', 6), ('2024-01-13', 8);

WITH has_sale AS (
    SELECT 
        sales_date,
        units_sold,
        CASE WHEN units_sold > 0 THEN 1 ELSE 0 END AS had_sale
    FROM daily_sales
),
island_groups AS (
    SELECT 
        sales_date,
        units_sold,
        had_sale,
        SUM(CASE WHEN had_sale = 0 THEN 1 ELSE 0 END) OVER (ORDER BY sales_date) AS grp
    FROM has_sale
),
streaks AS (
    SELECT 
        MIN(sales_date) AS streak_start,
        MAX(sales_date) AS streak_end,
        COUNT(*) AS streak_length,
        SUM(units_sold) AS total_sales_in_streak
    FROM island_groups
    WHERE had_sale = 1
    GROUP BY grp
)
SELECT 
    streak_start,
    streak_end,
    streak_length,
    total_sales_in_streak
FROM streaks
ORDER BY streak_length DESC;
```

### ตัวอย่างที่ 7: Consecutive Purchase Detection

```sql
-- หา customers ที่ซื้อสินค้าติดต่อกัน n เดือน
WITH monthly_purchases AS (
    SELECT 
        customer_id,
        DATE_TRUNC('month', purchase_date) AS purchase_month
    FROM customer_purchases
    GROUP BY customer_id, DATE_TRUNC('month', purchase_date)
),
with_row_num AS (
    SELECT 
        customer_id,
        purchase_month,
        ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY purchase_month) AS rn,
        -- ถ้า months ต่อเนื่องกัน: (month - rn months) จะเท่ากัน
        purchase_month - (ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY purchase_month) || ' months')::INTERVAL AS grp
    FROM monthly_purchases
),
streaks AS (
    SELECT 
        customer_id,
        MIN(purchase_month) AS streak_start,
        MAX(purchase_month) AS streak_end,
        COUNT(*) AS consecutive_months
    FROM with_row_num
    GROUP BY customer_id, grp
)
SELECT 
    customer_id,
    streak_start,
    streak_end,
    consecutive_months
FROM streaks
WHERE consecutive_months >= 3  -- ซื้อต่อเนื่อง 3 เดือนขึ้นไป
ORDER BY consecutive_months DESC;
```

---

## 96.4 Median Calculation

### ตัวอย่างที่ 8: Median ด้วย Window Functions

```sql
-- Median โดยไม่ใช้ aggregate function
WITH ranked AS (
    SELECT 
        department,
        salary,
        COUNT(*) OVER (PARTITION BY department) AS n,
        ROW_NUMBER() OVER (PARTITION BY department ORDER BY salary) AS rn
    FROM employees
)
SELECT 
    department,
    AVG(salary) AS median_salary
FROM ranked
WHERE rn IN (FLOOR((n + 1) / 2.0), CEIL((n + 1) / 2.0))
GROUP BY department;

-- หรือใช้ PERCENTILE_CONT (มาตรฐาน SQL)
SELECT 
    department,
    PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY salary) AS median_salary,
    PERCENTILE_DISC(0.5) WITHIN GROUP (ORDER BY salary) AS median_disc
FROM employees
GROUP BY department;
```

### ตัวอย่างที่ 9: Running Median

```sql
-- Running median (approximate)
WITH ordered AS (
    SELECT 
        year_month,
        revenue,
        ROW_NUMBER() OVER (ORDER BY year_month) AS rn,
        COUNT(*) OVER () AS total
    FROM monthly_sales
),
running_median AS (
    SELECT 
        o1.year_month,
        o1.revenue,
        o1.rn,
        AVG(o2.revenue) AS approx_running_median
    FROM ordered o1
    JOIN ordered o2 ON o2.rn <= o1.rn
    WHERE o2.rn IN (
        FLOOR((o1.rn + 1) / 2.0),
        CEIL((o1.rn + 1) / 2.0)
    )
    GROUP BY o1.year_month, o1.revenue, o1.rn
)
SELECT year_month, revenue, ROUND(approx_running_median, 2) AS running_median
FROM running_median
ORDER BY year_month;
```

---

## 96.5 Mode Calculation

### ตัวอย่างที่ 10: Most Frequent Value per Group

```sql
-- หา mode (ค่าที่พบบ่อยที่สุด) ของ product category ที่ลูกค้าซื้อบ่อยสุด
WITH purchase_counts AS (
    SELECT 
        customer_id,
        product_category,
        COUNT(*) AS purchase_count,
        RANK() OVER (PARTITION BY customer_id ORDER BY COUNT(*) DESC) AS rnk
    FROM customer_purchases
    GROUP BY customer_id, product_category
)
SELECT 
    customer_id,
    product_category AS favorite_category,
    purchase_count
FROM purchase_counts
WHERE rnk = 1;

-- Mode ด้วย MODE() aggregate (PostgreSQL)
SELECT 
    department,
    MODE() WITHIN GROUP (ORDER BY salary) AS modal_salary
FROM employees
GROUP BY department;
```

---

## 96.6 Cohort Analysis

### ตัวอย่างที่ 11: Cohort Revenue Matrix

```sql
-- สร้าง cohort matrix แบบสมบูรณ์
CREATE TABLE orders (
    order_id    INT PRIMARY KEY,
    customer_id INT,
    order_date  DATE,
    revenue     DECIMAL(12,2)
);

WITH 
-- กำหนด cohort month ของแต่ละ customer
customer_cohorts AS (
    SELECT 
        customer_id,
        DATE_TRUNC('month', MIN(order_date)) AS cohort_month
    FROM orders
    GROUP BY customer_id
),
-- รวม revenue แต่ละ customer-month
customer_monthly AS (
    SELECT 
        o.customer_id,
        DATE_TRUNC('month', o.order_date) AS order_month,
        SUM(o.revenue) AS monthly_revenue
    FROM orders o
    GROUP BY o.customer_id, DATE_TRUNC('month', o.order_date)
),
-- รวม cohort information
cohort_data AS (
    SELECT 
        cc.cohort_month,
        cm.order_month,
        EXTRACT(YEAR FROM AGE(cm.order_month, cc.cohort_month)) * 12 +
        EXTRACT(MONTH FROM AGE(cm.order_month, cc.cohort_month)) AS period_number,
        COUNT(DISTINCT cm.customer_id) AS active_customers,
        SUM(cm.monthly_revenue) AS cohort_revenue
    FROM customer_cohorts cc
    JOIN customer_monthly cm ON cc.customer_id = cm.customer_id
    WHERE cm.order_month >= cc.cohort_month
    GROUP BY cc.cohort_month, cm.order_month
),
-- เพิ่ม cohort size
cohort_sizes AS (
    SELECT cohort_month, COUNT(*) AS cohort_size
    FROM customer_cohorts
    GROUP BY cohort_month
)
SELECT 
    cd.cohort_month,
    cs.cohort_size,
    cd.period_number,
    cd.active_customers,
    cd.cohort_revenue,
    ROUND(cd.active_customers * 100.0 / cs.cohort_size, 1) AS retention_pct,
    ROUND(cd.cohort_revenue / cd.active_customers, 2) AS avg_revenue_per_active_customer
FROM cohort_data cd
JOIN cohort_sizes cs ON cd.cohort_month = cs.cohort_month
ORDER BY cd.cohort_month, cd.period_number;
```

### ตัวอย่างที่ 12: LTV by Cohort

```sql
-- Lifetime value แยกตาม acquisition cohort
WITH 
cohorts AS (
    SELECT 
        customer_id,
        DATE_TRUNC('month', MIN(order_date)) AS cohort_month
    FROM orders GROUP BY customer_id
),
cohort_revenue AS (
    SELECT 
        c.cohort_month,
        o.customer_id,
        SUM(o.revenue) AS lifetime_value
    FROM cohorts c
    JOIN orders o ON c.customer_id = o.customer_id
    GROUP BY c.cohort_month, o.customer_id
)
SELECT 
    cohort_month,
    COUNT(*) AS cohort_size,
    ROUND(AVG(lifetime_value), 2) AS avg_ltv,
    ROUND(PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY lifetime_value), 2) AS median_ltv,
    MAX(lifetime_value) AS max_ltv
FROM cohort_revenue
GROUP BY cohort_month
ORDER BY cohort_month;
```

---

## 96.7 Advanced Window Patterns

### ตัวอย่างที่ 13: Contiguous Range Finder

```sql
-- หา range ของตัวเลขต่อเนื่อง
CREATE TABLE order_ids_present (
    order_id INT
);

INSERT INTO order_ids_present VALUES
(1), (2), (3), (5), (6), (10), (11), (12), (13), (20);

WITH islands AS (
    SELECT 
        order_id,
        order_id - ROW_NUMBER() OVER (ORDER BY order_id) AS grp
    FROM order_ids_present
)
SELECT 
    MIN(order_id) AS range_start,
    MAX(order_id) AS range_end,
    COUNT(*) AS count_in_range
FROM islands
GROUP BY grp
ORDER BY range_start;
```

### ตัวอย่างที่ 14: Time Spent in State

```sql
-- คำนวณเวลาที่ใช้ในแต่ละ state
CREATE TABLE order_status_log (
    order_id    INT,
    status      VARCHAR(50),
    changed_at  TIMESTAMP
);

INSERT INTO order_status_log VALUES
(1, 'pending',    '2024-01-01 10:00'),
(1, 'processing', '2024-01-01 11:30'),
(1, 'shipped',    '2024-01-02 09:00'),
(1, 'delivered',  '2024-01-04 14:00'),
(2, 'pending',    '2024-01-01 12:00'),
(2, 'processing', '2024-01-01 12:30'),
(2, 'cancelled',  '2024-01-01 13:00');

SELECT 
    order_id,
    status,
    changed_at AS entered_at,
    LEAD(changed_at) OVER (PARTITION BY order_id ORDER BY changed_at) AS exited_at,
    EXTRACT(EPOCH FROM 
        LEAD(changed_at) OVER (PARTITION BY order_id ORDER BY changed_at) - changed_at
    ) / 3600 AS hours_in_status
FROM order_status_log
ORDER BY order_id, changed_at;
```

### ตัวอย่างที่ 15: Pattern Matching Sequence

```sql
-- ตรวจหา pattern: view → add_to_cart → purchase
WITH ordered_events AS (
    SELECT 
        user_id,
        action_type,
        action_time,
        ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY action_time) AS rn
    FROM user_actions
),
with_pattern AS (
    SELECT 
        e1.user_id,
        e1.action_time AS view_time,
        e2.action_time AS cart_time,
        e3.action_time AS purchase_time
    FROM ordered_events e1
    JOIN ordered_events e2 ON e1.user_id = e2.user_id 
        AND e1.rn < e2.rn
        AND e1.action_type = 'view_product'
        AND e2.action_type = 'add_to_cart'
    JOIN ordered_events e3 ON e2.user_id = e3.user_id 
        AND e2.rn < e3.rn
        AND e3.action_type = 'payment'
    WHERE NOT EXISTS (
        SELECT 1 FROM ordered_events e_between
        WHERE e_between.user_id = e1.user_id
          AND e_between.rn > e1.rn 
          AND e_between.rn < e2.rn
          AND e_between.action_type = 'view_product'
    )
)
SELECT 
    user_id,
    view_time,
    cart_time,
    purchase_time,
    EXTRACT(EPOCH FROM purchase_time - view_time) / 60 AS minutes_to_purchase
FROM with_pattern;
```

---

## 96.8 Median and Percentile ขั้นสูง

### ตัวอย่างที่ 16: Multiple Percentiles

```sql
-- Quartile breakdown ของ salary
SELECT 
    department,
    PERCENTILE_CONT(0.25) WITHIN GROUP (ORDER BY salary) AS p25,
    PERCENTILE_CONT(0.50) WITHIN GROUP (ORDER BY salary) AS median,
    PERCENTILE_CONT(0.75) WITHIN GROUP (ORDER BY salary) AS p75,
    PERCENTILE_CONT(0.90) WITHIN GROUP (ORDER BY salary) AS p90,
    PERCENTILE_CONT(0.95) WITHIN GROUP (ORDER BY salary) AS p95,
    MAX(salary) - MIN(salary) AS salary_range,
    PERCENTILE_CONT(0.75) WITHIN GROUP (ORDER BY salary) -
    PERCENTILE_CONT(0.25) WITHIN GROUP (ORDER BY salary) AS iqr
FROM employees
GROUP BY department;
```

### ตัวอย่างที่ 17: Percentile Rank ใน Sliding Window

```sql
-- Percentile rank ใน rolling 6-month window
WITH monthly_rev AS (
    SELECT 
        salesperson,
        DATE_TRUNC('month', sale_date) AS month,
        SUM(quantity * unit_price) AS revenue
    FROM sales
    GROUP BY salesperson, DATE_TRUNC('month', sale_date)
),
rolling_percentile AS (
    SELECT 
        m1.salesperson,
        m1.month,
        m1.revenue,
        -- Approximate percentile within rolling 6-month window
        COUNT(*) FILTER (WHERE m2.revenue <= m1.revenue AND m2.month BETWEEN m1.month - INTERVAL '5 months' AND m1.month) * 100.0 /
        COUNT(*) FILTER (WHERE m2.month BETWEEN m1.month - INTERVAL '5 months' AND m1.month) AS rolling_percentile
    FROM monthly_rev m1
    JOIN monthly_rev m2 ON m1.salesperson = m2.salesperson
    GROUP BY m1.salesperson, m1.month, m1.revenue
)
SELECT * FROM rolling_percentile ORDER BY salesperson, month;
```

---

## 96.9 Inventory and Supply Chain Patterns

### ตัวอย่างที่ 18: Running Inventory with Restock Detection

```sql
-- ติดตาม inventory พร้อมตรวจจับการ restock
WITH inv_with_prev AS (
    SELECT 
        product_id,
        move_date,
        quantity,
        move_type,
        SUM(quantity) OVER (
            PARTITION BY product_id
            ORDER BY move_date, movement_id
        ) AS stock_level,
        LAG(SUM(quantity) OVER (
            PARTITION BY product_id
            ORDER BY move_date, movement_id
        )) OVER (PARTITION BY product_id ORDER BY move_date) AS prev_stock_level
    FROM inventory_movements
)
SELECT 
    product_id,
    move_date,
    quantity,
    move_type,
    stock_level,
    CASE 
        WHEN quantity > 0 AND prev_stock_level < 20 THEN 'RESTOCK EVENT - was low'
        WHEN stock_level <= 10 THEN 'CRITICAL - REORDER REQUIRED'
        WHEN stock_level <= 20 THEN 'LOW STOCK'
        ELSE 'OK'
    END AS inventory_event
FROM inv_with_prev
ORDER BY product_id, move_date;
```

---

## 96.10 Complex Business Analytics

### ตัวอย่างที่ 19: RFM Analysis ด้วย Window Functions

```sql
-- RFM: Recency, Frequency, Monetary
WITH customer_rfm AS (
    SELECT 
        customer_id,
        MAX(purchase_date) AS last_purchase,
        COUNT(*) AS frequency,
        SUM(amount) AS monetary
    FROM customer_purchases
    GROUP BY customer_id
),
rfm_scores AS (
    SELECT 
        customer_id,
        last_purchase,
        CURRENT_DATE - last_purchase AS recency_days,
        frequency,
        monetary,
        -- Score 1-5: Recency (น้อย = ดี)
        NTILE(5) OVER (ORDER BY last_purchase DESC) AS r_score,
        -- Score 1-5: Frequency (มาก = ดี)
        NTILE(5) OVER (ORDER BY frequency) AS f_score,
        -- Score 1-5: Monetary (มาก = ดี)
        NTILE(5) OVER (ORDER BY monetary) AS m_score
    FROM customer_rfm
),
rfm_segments AS (
    SELECT *,
        r_score * 100 + f_score * 10 + m_score AS rfm_score,
        CASE 
            WHEN r_score >= 4 AND f_score >= 4 AND m_score >= 4 THEN 'Champions'
            WHEN r_score >= 3 AND f_score >= 3 THEN 'Loyal Customers'
            WHEN r_score >= 4 AND f_score <= 2 THEN 'Recent Customers'
            WHEN r_score >= 3 AND m_score >= 4 THEN 'Potential Loyalists'
            WHEN r_score <= 2 AND f_score >= 4 THEN 'At Risk'
            WHEN r_score <= 2 AND f_score <= 2 THEN 'Lost'
            ELSE 'Needs Attention'
        END AS segment
    FROM rfm_scores
)
SELECT 
    segment,
    COUNT(*) AS customer_count,
    ROUND(AVG(monetary), 2) AS avg_spend,
    ROUND(AVG(frequency), 1) AS avg_orders,
    ROUND(AVG(recency_days), 0) AS avg_days_since_purchase
FROM rfm_segments
GROUP BY segment
ORDER BY avg_spend DESC;
```

### ตัวอย่างที่ 20: Sales Attribution (First Touch / Last Touch)

```sql
-- Multi-touch attribution analysis
CREATE TABLE marketing_touchpoints (
    customer_id INT,
    channel     VARCHAR(50),
    touch_time  TIMESTAMP,
    converted   BOOLEAN
);

WITH touchpoints_ranked AS (
    SELECT 
        customer_id,
        channel,
        touch_time,
        converted,
        ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY touch_time) AS touch_order,
        ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY touch_time DESC) AS reverse_order,
        COUNT(*) OVER (PARTITION BY customer_id) AS total_touches
    FROM marketing_touchpoints
),
attribution AS (
    SELECT 
        customer_id,
        MAX(CASE WHEN touch_order = 1 THEN channel END) AS first_touch_channel,
        MAX(CASE WHEN reverse_order = 1 THEN channel END) AS last_touch_channel,
        MAX(total_touches) AS num_touches,
        MAX(CASE WHEN converted THEN 1 ELSE 0 END) AS did_convert
    FROM touchpoints_ranked
    GROUP BY customer_id
)
SELECT 
    first_touch_channel,
    last_touch_channel,
    COUNT(*) AS customers,
    SUM(did_convert) AS conversions,
    ROUND(SUM(did_convert) * 100.0 / COUNT(*), 2) AS conversion_rate
FROM attribution
GROUP BY first_touch_channel, last_touch_channel
ORDER BY conversions DESC;
```

---

## 96.11 Time Series Patterns

### ตัวอย่างที่ 21: Trend Detection Algorithm

```sql
-- ตรวจจับ trend ด้วยการเปรียบเทียบ moving averages
WITH short_ma AS (
    SELECT 
        year_month,
        revenue,
        AVG(revenue) OVER (ORDER BY year_month ROWS BETWEEN 2 PRECEDING AND CURRENT ROW) AS ma_3
    FROM monthly_sales
),
long_ma AS (
    SELECT 
        year_month,
        AVG(revenue) OVER (ORDER BY year_month ROWS BETWEEN 11 PRECEDING AND CURRENT ROW) AS ma_12
    FROM monthly_sales
),
crossovers AS (
    SELECT 
        s.year_month,
        s.revenue,
        s.ma_3,
        l.ma_12,
        -- Golden/Death cross
        CASE 
            WHEN s.ma_3 > l.ma_12 AND LAG(s.ma_3) OVER (ORDER BY s.year_month) <= LAG(l.ma_12) OVER (ORDER BY s.year_month) THEN 'GOLDEN CROSS (Buy Signal)'
            WHEN s.ma_3 < l.ma_12 AND LAG(s.ma_3) OVER (ORDER BY s.year_month) >= LAG(l.ma_12) OVER (ORDER BY s.year_month) THEN 'DEATH CROSS (Sell Signal)'
            WHEN s.ma_3 > l.ma_12 THEN 'Uptrend'
            ELSE 'Downtrend'
        END AS trend_signal
    FROM short_ma s
    JOIN long_ma l ON s.year_month = l.year_month
)
SELECT * FROM crossovers ORDER BY year_month;
```

---

## แบบฝึกหัดท้ายบท

### แบบฝึกหัดที่ 1
หา islands ของวันที่มียอดขาย และคำนวณยอดขายรวมในแต่ละ island

**คำตอบ:**
```sql
WITH has_sale AS (
    SELECT 
        sales_date,
        units_sold,
        CASE WHEN units_sold > 0 THEN 1 ELSE 0 END AS had_sale
    FROM daily_sales
),
island_groups AS (
    SELECT 
        sales_date,
        units_sold,
        had_sale,
        SUM(CASE WHEN had_sale = 0 THEN 1 ELSE 0 END) OVER (ORDER BY sales_date) AS island_id
    FROM has_sale
)
SELECT 
    island_id,
    MIN(sales_date) AS start_date,
    MAX(sales_date) AS end_date,
    COUNT(*) AS consecutive_days,
    SUM(units_sold) AS total_units
FROM island_groups
WHERE had_sale = 1
GROUP BY island_id
ORDER BY start_date;
```

### แบบฝึกหัดที่ 2
วิเคราะห์ sessions จาก page_events โดย session timeout = 20 นาที

**คำตอบ:**
```sql
WITH session_markers AS (
    SELECT 
        event_id,
        user_id,
        page,
        event_time,
        CASE 
            WHEN event_time - LAG(event_time) OVER (PARTITION BY user_id ORDER BY event_time) > INTERVAL '20 minutes'
              OR LAG(event_time) OVER (PARTITION BY user_id ORDER BY event_time) IS NULL
            THEN 1 ELSE 0
        END AS new_session
    FROM page_events
),
with_session_id AS (
    SELECT *,
           SUM(new_session) OVER (PARTITION BY user_id ORDER BY event_time) AS session_num
    FROM session_markers
)
SELECT 
    user_id,
    session_num,
    MIN(event_time) AS start,
    MAX(event_time) AS end_time,
    COUNT(*) AS pages,
    EXTRACT(EPOCH FROM MAX(event_time) - MIN(event_time)) / 60 AS duration_minutes
FROM with_session_id
GROUP BY user_id, session_num
ORDER BY user_id, session_num;
```

### แบบฝึกหัดที่ 3
หา customers ที่ซื้อสินค้าติดต่อกัน 3 เดือนหรือมากกว่า

**คำตอบ:**
```sql
WITH monthly AS (
    SELECT 
        customer_id,
        DATE_TRUNC('month', purchase_date) AS month
    FROM customer_purchases
    GROUP BY customer_id, DATE_TRUNC('month', purchase_date)
),
with_grp AS (
    SELECT 
        customer_id,
        month,
        month - (ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY month) || ' months')::INTERVAL AS grp
    FROM monthly
)
SELECT 
    customer_id,
    MIN(month) AS streak_start,
    MAX(month) AS streak_end,
    COUNT(*) AS consecutive_months
FROM with_grp
GROUP BY customer_id, grp
HAVING COUNT(*) >= 3
ORDER BY consecutive_months DESC;
```

### แบบฝึกหัดที่ 4
คำนวณ RFM scores และแบ่ง customers เป็น 5 segments

**คำตอบ:**
```sql
WITH rfm_base AS (
    SELECT 
        customer_id,
        CURRENT_DATE - MAX(purchase_date) AS recency,
        COUNT(*) AS frequency,
        SUM(amount) AS monetary
    FROM customer_purchases
    GROUP BY customer_id
)
SELECT 
    customer_id,
    recency,
    frequency,
    monetary,
    NTILE(5) OVER (ORDER BY recency ASC) AS r_score,
    NTILE(5) OVER (ORDER BY frequency DESC) AS f_score,
    NTILE(5) OVER (ORDER BY monetary DESC) AS m_score,
    NTILE(5) OVER (ORDER BY recency ASC) + 
    NTILE(5) OVER (ORDER BY frequency DESC) +
    NTILE(5) OVER (ORDER BY monetary DESC) AS rfm_total
FROM rfm_base
ORDER BY rfm_total DESC;
```

### แบบฝึกหัดที่ 5
สร้าง cohort retention matrix แสดงอัตรา retention รายเดือน

**คำตอบ:**
```sql
WITH first_order AS (
    SELECT customer_id, DATE_TRUNC('month', MIN(order_date)) AS cohort_month
    FROM orders GROUP BY customer_id
),
monthly_orders AS (
    SELECT 
        o.customer_id,
        f.cohort_month,
        DATE_TRUNC('month', o.order_date) AS order_month
    FROM orders o
    JOIN first_order f ON o.customer_id = f.customer_id
    GROUP BY o.customer_id, f.cohort_month, DATE_TRUNC('month', o.order_date)
),
cohort_counts AS (
    SELECT 
        cohort_month,
        EXTRACT(MONTH FROM AGE(order_month, cohort_month))::INT AS period,
        COUNT(DISTINCT customer_id) AS customers
    FROM monthly_orders
    GROUP BY cohort_month, period
),
cohort_size AS (
    SELECT cohort_month, COUNT(*) AS size FROM first_order GROUP BY cohort_month
)
SELECT 
    cc.cohort_month,
    cs.size,
    cc.period,
    cc.customers,
    ROUND(cc.customers * 100.0 / cs.size, 1) AS retention_pct
FROM cohort_counts cc
JOIN cohort_size cs ON cc.cohort_month = cs.cohort_month
ORDER BY cc.cohort_month, cc.period;
```

### แบบฝึกหัดที่ 6
วิเคราะห์ order status transitions และหา average time ในแต่ละ status

**คำตอบ:**
```sql
WITH status_times AS (
    SELECT 
        order_id,
        status,
        changed_at,
        LEAD(changed_at) OVER (PARTITION BY order_id ORDER BY changed_at) AS next_changed_at,
        EXTRACT(EPOCH FROM 
            LEAD(changed_at) OVER (PARTITION BY order_id ORDER BY changed_at) - changed_at
        ) / 3600 AS hours_in_status
    FROM order_status_log
    WHERE LEAD(changed_at) OVER (PARTITION BY order_id ORDER BY changed_at) IS NOT NULL
)
SELECT 
    status,
    COUNT(*) AS occurrences,
    ROUND(AVG(hours_in_status), 2) AS avg_hours,
    ROUND(MIN(hours_in_status), 2) AS min_hours,
    ROUND(MAX(hours_in_status), 2) AS max_hours,
    ROUND(PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY hours_in_status), 2) AS median_hours
FROM status_times
GROUP BY status
ORDER BY avg_hours DESC;
```

### แบบฝึกหัดที่ 7
หา gaps ในลำดับ order IDs และแสดงว่า gap มีขนาดเท่าไหร่

**คำตอบ:**
```sql
WITH order_sequence AS (
    SELECT 
        order_id,
        LEAD(order_id) OVER (ORDER BY order_id) AS next_id
    FROM orders
)
SELECT 
    order_id AS last_before_gap,
    next_id AS first_after_gap,
    next_id - order_id - 1 AS gap_size,
    order_id + 1 AS gap_start,
    next_id - 1 AS gap_end
FROM order_sequence
WHERE next_id - order_id > 1
ORDER BY order_id;
```

### แบบฝึกหัดที่ 8
วิเคราะห์ conversion funnel ด้วย session-level analysis

**คำตอบ:**
```sql
WITH session_events AS (
    SELECT 
        user_id,
        SUM(CASE WHEN action_type = 'login' THEN 1 ELSE 0 END) OVER (
            PARTITION BY user_id ORDER BY action_time
        ) AS session_id,
        action_type,
        action_time
    FROM user_actions
),
session_summary AS (
    SELECT 
        user_id,
        session_id,
        MAX(CASE WHEN action_type = 'view_product' THEN 1 ELSE 0 END) AS viewed,
        MAX(CASE WHEN action_type = 'add_to_cart' THEN 1 ELSE 0 END) AS added_to_cart,
        MAX(CASE WHEN action_type = 'checkout' THEN 1 ELSE 0 END) AS checked_out,
        MAX(CASE WHEN action_type = 'payment' THEN 1 ELSE 0 END) AS purchased
    FROM session_events
    GROUP BY user_id, session_id
)
SELECT 
    COUNT(*) AS total_sessions,
    SUM(viewed) AS sessions_with_view,
    SUM(added_to_cart) AS sessions_with_cart,
    SUM(checked_out) AS sessions_with_checkout,
    SUM(purchased) AS sessions_with_purchase,
    ROUND(SUM(added_to_cart) * 100.0 / NULLIF(SUM(viewed), 0), 2) AS view_to_cart_pct,
    ROUND(SUM(purchased) * 100.0 / NULLIF(SUM(viewed), 0), 2) AS overall_conv_pct
FROM session_summary;
```

### แบบฝึกหัดที่ 9
สร้าง running percentile rank ของ revenue ตลอดเวลา

**คำตอบ:**
```sql
WITH ranked AS (
    SELECT 
        year_month,
        revenue,
        ROW_NUMBER() OVER (ORDER BY year_month) AS rn,
        RANK() OVER (ORDER BY revenue) AS revenue_rank,
        COUNT(*) OVER () AS total_months
    FROM monthly_sales
),
running_pct AS (
    SELECT 
        r1.year_month,
        r1.revenue,
        r1.rn,
        COUNT(r2.year_month) AS months_with_less_revenue,
        r1.rn AS months_so_far,
        ROUND(
            COUNT(r2.year_month) * 100.0 / r1.rn,
        1) AS running_percentile
    FROM ranked r1
    JOIN ranked r2 ON r2.rn <= r1.rn AND r2.revenue < r1.revenue
    GROUP BY r1.year_month, r1.revenue, r1.rn
)
SELECT year_month, revenue, running_percentile
FROM running_pct
ORDER BY year_month;
```

### แบบฝึกหัดที่ 10
Implement sliding window anomaly detection สำหรับ transaction amounts

**คำตอบ:**
```sql
WITH txn_stats AS (
    SELECT 
        txn_id,
        account_id,
        txn_date,
        amount,
        AVG(ABS(amount)) OVER (
            PARTITION BY account_id
            ORDER BY txn_date, txn_id
            ROWS BETWEEN 10 PRECEDING AND 1 PRECEDING
        ) AS avg_10_prev,
        STDDEV(ABS(amount)) OVER (
            PARTITION BY account_id
            ORDER BY txn_date, txn_id
            ROWS BETWEEN 10 PRECEDING AND 1 PRECEDING
        ) AS std_10_prev,
        COUNT(*) OVER (
            PARTITION BY account_id
            ORDER BY txn_date, txn_id
            ROWS BETWEEN 10 PRECEDING AND 1 PRECEDING
        ) AS prev_count
    FROM bank_transactions
)
SELECT 
    txn_id,
    account_id,
    txn_date,
    amount,
    ROUND(avg_10_prev, 2) AS expected_amount,
    ROUND((ABS(amount) - avg_10_prev) / NULLIF(std_10_prev, 0), 2) AS z_score,
    CASE 
        WHEN prev_count >= 3 AND ABS(amount) > avg_10_prev + 3 * std_10_prev 
        THEN 'HIGH ANOMALY'
        WHEN prev_count >= 3 AND ABS(amount) > avg_10_prev + 2 * std_10_prev 
        THEN 'MODERATE ANOMALY'
        ELSE 'NORMAL'
    END AS anomaly_status
FROM txn_stats
ORDER BY account_id, txn_date;
```

---

## สรุปบทที่ 96

Advanced CTE และ Window Function Patterns:

### Gap and Island
- **วิธีที่ 1**: `ROW_NUMBER() - DATE` สำหรับ date sequences
- **วิธีที่ 2**: `LAG() + SUM()` สร้าง group markers
- ใช้สำหรับ: consecutive events, attendance, subscription periods

### Session Analysis
- ใช้ LAG + CASE เพื่อตรวจจับ session boundaries
- SUM ของ new_session markers สร้าง session_id

### Cohort Analysis
- กำหนด cohort_month ด้วย MIN(event_date)
- คำนวณ period_number ด้วย AGE/date difference
- FIRST_VALUE ดึง cohort size มาคำนวณ retention rate

ในบทถัดไปเราจะเรียนรู้เกี่ยวกับ JSON ใน SQL ซึ่งเป็นฟีเจอร์ที่ช่วยให้ database รองรับข้อมูลกึ่ง structured ได้อย่างยืดหยุ่น
