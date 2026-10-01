# ส่วนที่ 95: Window Functions ตอนที่ 3 - Aggregate Window Functions

## บทนำ

Aggregate Window Functions เอา aggregate functions คลาสสิก (SUM, AVG, MIN, MAX, COUNT) มาใช้งานในบริบทของ window เพื่อคำนวณค่าสะสม, ค่าเฉลี่ยเคลื่อนที่ (moving average), running totals, และอื่นๆ โดยไม่ทำให้จำนวนแถวลดลง

---

## 95.1 Running Total ด้วย SUM() OVER

### ตัวอย่างที่ 1: Running Total พื้นฐาน

```sql
-- สร้างข้อมูลธุรกรรมบัญชีธนาคาร
CREATE TABLE bank_transactions (
    txn_id      INT PRIMARY KEY,
    account_id  INT,
    txn_date    DATE,
    amount      DECIMAL(12,2),   -- บวก = เข้า, ลบ = ออก
    description VARCHAR(200)
);

INSERT INTO bank_transactions VALUES
(1,  1001, '2024-01-05', 50000,   'เงินเดือน'),
(2,  1001, '2024-01-08', -1500,   'ค่าน้ำค่าไฟ'),
(3,  1001, '2024-01-10', -3000,   'ค่าอาหาร'),
(4,  1001, '2024-01-15', -8000,   'ค่าเช่า'),
(5,  1001, '2024-01-20', 5000,    'รับเงินพิเศษ'),
(6,  1001, '2024-01-25', -2000,   'ค่าโทรศัพท์'),
(7,  1001, '2024-02-05', 50000,   'เงินเดือน'),
(8,  1001, '2024-02-10', -5000,   'ซื้อของ'),
(9,  1002, '2024-01-03', 100000,  'ฝากออมทรัพย์'),
(10, 1002, '2024-01-10', -20000,  'ถอนเงิน');

-- Running balance สะสม
SELECT 
    txn_id,
    account_id,
    txn_date,
    amount,
    description,
    SUM(amount) OVER (
        PARTITION BY account_id
        ORDER BY txn_date, txn_id
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    ) AS running_balance
FROM bank_transactions
ORDER BY account_id, txn_date, txn_id;
```

### ตัวอย่างที่ 2: Running Total ไม่ใช้ PARTITION (ทั้งตาราง)

```sql
-- Running total ของยอดขายทั้งบริษัท
SELECT 
    year_month,
    revenue,
    SUM(revenue) OVER (ORDER BY year_month) AS cumulative_revenue,
    ROUND(
        SUM(revenue) OVER (ORDER BY year_month) * 100.0 /
        SUM(revenue) OVER (),   -- SUM ทั้งตาราง
    2) AS pct_of_total_revenue
FROM monthly_sales
ORDER BY year_month;
```

### ตัวอย่างที่ 3: Running Total แบบ Year-to-Date

```sql
-- YTD revenue สะสมรายปี
SELECT 
    year_month,
    EXTRACT(YEAR FROM year_month) AS yr,
    revenue,
    SUM(revenue) OVER (
        PARTITION BY EXTRACT(YEAR FROM year_month)
        ORDER BY year_month
    ) AS ytd_revenue
FROM monthly_sales
ORDER BY year_month;
```

---

## 95.2 Running Average

### ตัวอย่างที่ 4: Running Average พื้นฐาน

```sql
-- ค่าเฉลี่ยสะสมของยอดขาย
SELECT 
    year_month,
    revenue,
    ROUND(
        AVG(revenue) OVER (
            ORDER BY year_month
            ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
        ),
    2) AS running_avg_revenue,
    COUNT(*) OVER (
        ORDER BY year_month
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    ) AS months_counted
FROM monthly_sales
ORDER BY year_month;
```

### ตัวอย่างที่ 5: 3-Month Moving Average

```sql
-- Moving average ย้อนหลัง 3 เดือน
SELECT 
    year_month,
    revenue,
    ROUND(
        AVG(revenue) OVER (
            ORDER BY year_month
            ROWS BETWEEN 2 PRECEDING AND CURRENT ROW
        ),
    2) AS ma_3month,
    COUNT(*) OVER (
        ORDER BY year_month
        ROWS BETWEEN 2 PRECEDING AND CURRENT ROW
    ) AS periods_in_window  -- บอกว่า window มีกี่แถว
FROM monthly_sales
ORDER BY year_month;
```

---

## 95.3 Frame Specification - หัวใจของ Window Functions

### ความหมายของ ROWS, RANGE, GROUPS

```
ROWS:   นับแถวตามลำดับ (physical rows)
RANGE:  นับตามช่วงค่า (logical range) - ค่าเท่ากันอยู่ใน frame เดียวกัน
GROUPS: นับตามกลุ่มค่าที่เท่ากัน (PostgreSQL 11+)
```

### Frame boundaries

```
UNBOUNDED PRECEDING  = แถวแรกสุดของ partition
n PRECEDING          = n แถวก่อนหน้าแถวปัจจุบัน
CURRENT ROW          = แถวปัจจุบัน
n FOLLOWING          = n แถวหลังแถวปัจจุบัน
UNBOUNDED FOLLOWING  = แถวสุดท้ายของ partition
```

### ตัวอย่างที่ 6: ROWS vs RANGE เมื่อค่าซ้ำกัน

```sql
CREATE TABLE score_table (
    student VARCHAR(10),
    score   INT
);

INSERT INTO score_table VALUES
('A', 90), ('B', 90), ('C', 85), ('D', 80), ('E', 80);

SELECT 
    student,
    score,
    -- ROWS: นับแถวจริง
    SUM(score) OVER (
        ORDER BY score DESC
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    ) AS rows_sum,
    -- RANGE: รวมค่าเท่ากันไว้ใน same frame
    SUM(score) OVER (
        ORDER BY score DESC
        RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    ) AS range_sum
FROM score_table;
-- A: ROWS=90, RANGE=180 (เพราะ B ก็มี score=90)
-- B: ROWS=180, RANGE=180
-- C: ROWS=265, RANGE=265
-- D: ROWS=345, RANGE=505 (เพราะ E ก็มี score=80)
-- E: ROWS=425, RANGE=505
```

### ตัวอย่างที่ 7: Frame Specifications ต่างๆ

```sql
SELECT 
    year_month,
    revenue,
    -- Cumulative (unbounded start to current)
    SUM(revenue) OVER (ORDER BY year_month
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS cumulative,
    
    -- Rolling 3-month (2 before + current)
    AVG(revenue) OVER (ORDER BY year_month
        ROWS BETWEEN 2 PRECEDING AND CURRENT ROW) AS ma_3m,
    
    -- Centered 3-month (1 before, current, 1 after)
    AVG(revenue) OVER (ORDER BY year_month
        ROWS BETWEEN 1 PRECEDING AND 1 FOLLOWING) AS centered_3m,
    
    -- Full window total
    SUM(revenue) OVER (ORDER BY year_month
        ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING) AS total,
    
    -- Remaining (current to end)
    SUM(revenue) OVER (ORDER BY year_month
        ROWS BETWEEN CURRENT ROW AND UNBOUNDED FOLLOWING) AS remaining
FROM monthly_sales;
```

---

## 95.4 Sliding Window Calculations

### ตัวอย่างที่ 8: 7-Day Moving Average (Daily Data)

```sql
-- 7-day moving average สำหรับ daily metrics
SELECT 
    metric_date,
    page_views,
    ROUND(AVG(page_views) OVER (
        ORDER BY metric_date
        ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
    ), 0) AS ma_7day,
    -- Centered 7-day MA (3 before, current, 3 after)
    ROUND(AVG(page_views) OVER (
        ORDER BY metric_date
        ROWS BETWEEN 3 PRECEDING AND 3 FOLLOWING
    ), 0) AS centered_ma_7day
FROM daily_metrics
ORDER BY metric_date;
```

### ตัวอย่างที่ 9: Moving MIN/MAX (Bollinger-like bands)

```sql
-- Rolling min/max สำหรับ price range analysis
SELECT 
    ticker,
    trade_date,
    close_price,
    MIN(close_price) OVER (
        PARTITION BY ticker ORDER BY trade_date
        ROWS BETWEEN 4 PRECEDING AND CURRENT ROW
    ) AS rolling_5d_low,
    MAX(close_price) OVER (
        PARTITION BY ticker ORDER BY trade_date
        ROWS BETWEEN 4 PRECEDING AND CURRENT ROW
    ) AS rolling_5d_high,
    AVG(close_price) OVER (
        PARTITION BY ticker ORDER BY trade_date
        ROWS BETWEEN 4 PRECEDING AND CURRENT ROW
    ) AS ma_5d,
    -- กำหนดว่าราคาอยู่ใกล้ high หรือ low
    CASE 
        WHEN close_price = MAX(close_price) OVER (
            PARTITION BY ticker ORDER BY trade_date
            ROWS BETWEEN 4 PRECEDING AND CURRENT ROW
        ) THEN 'AT 5D HIGH'
        WHEN close_price = MIN(close_price) OVER (
            PARTITION BY ticker ORDER BY trade_date
            ROWS BETWEEN 4 PRECEDING AND CURRENT ROW
        ) THEN 'AT 5D LOW'
        ELSE 'IN RANGE'
    END AS price_position
FROM stock_prices
ORDER BY ticker, trade_date;
```

### ตัวอย่างที่ 10: Moving COUNT (Activity Window)

```sql
-- นับ events ใน 7 วันที่ผ่านมา (RANGE-based)
SELECT 
    metric_date,
    page_views,
    COUNT(*) OVER (
        ORDER BY metric_date
        RANGE BETWEEN '7 days' PRECEDING AND CURRENT ROW
    ) AS days_in_window,
    SUM(page_views) OVER (
        ORDER BY metric_date
        RANGE BETWEEN '7 days' PRECEDING AND CURRENT ROW
    ) AS views_last_7days
FROM daily_metrics
ORDER BY metric_date;
```

---

## 95.5 Cumulative Distribution

### ตัวอย่างที่ 11: Cumulative Sum และ Percentage

```sql
-- Cumulative revenue percentage (Pareto analysis)
WITH product_revenue AS (
    SELECT 
        product_name,
        SUM(quantity * unit_price) AS total_revenue
    FROM sales s
    JOIN products p ON s.product_id = p.product_id  -- สมมติมีตาราง products
    GROUP BY product_name
    ORDER BY total_revenue DESC
)
SELECT 
    product_name,
    total_revenue,
    SUM(total_revenue) OVER (ORDER BY total_revenue DESC) AS cumulative_revenue,
    ROUND(
        SUM(total_revenue) OVER (ORDER BY total_revenue DESC) * 100.0 /
        SUM(total_revenue) OVER (),
    2) AS cumulative_pct,
    ROUND(total_revenue * 100.0 / SUM(total_revenue) OVER (), 2) AS individual_pct
FROM product_revenue;
```

### ตัวอย่างที่ 12: Pareto 80/20 Analysis

```sql
WITH product_rev AS (
    SELECT 
        product_id,
        SUM(units_sold) AS total_units
    FROM product_sales
    GROUP BY product_id
),
with_cumulative AS (
    SELECT 
        product_id,
        total_units,
        SUM(total_units) OVER (ORDER BY total_units DESC) AS cumulative_units,
        SUM(total_units) OVER () AS grand_total
    FROM product_rev
)
SELECT 
    product_id,
    total_units,
    ROUND(cumulative_units * 100.0 / grand_total, 2) AS cumulative_pct,
    CASE 
        WHEN cumulative_units * 100.0 / grand_total <= 80 THEN 'Top 80% (A-items)'
        WHEN cumulative_units * 100.0 / grand_total <= 95 THEN 'Next 15% (B-items)'
        ELSE 'Bottom 5% (C-items)'
    END AS abc_category
FROM with_cumulative
ORDER BY total_units DESC;
```

---

## 95.6 Partitioned Running Totals

### ตัวอย่างที่ 13: Running Total แยก Department

```sql
-- Running total ของเงินเดือนสะสมในแต่ละแผนก
SELECT 
    department,
    emp_id,
    name,
    salary,
    SUM(salary) OVER (
        PARTITION BY department
        ORDER BY emp_id
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    ) AS dept_running_total,
    AVG(salary) OVER (
        PARTITION BY department
        ORDER BY emp_id
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    ) AS dept_running_avg,
    COUNT(*) OVER (
        PARTITION BY department
        ORDER BY emp_id
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    ) AS dept_running_count
FROM employees
ORDER BY department, emp_id;
```

### ตัวอย่างที่ 14: Partitioned Cumulative Revenue

```sql
-- Cumulative revenue แยกตาม product
SELECT 
    product_id,
    month,
    revenue,
    SUM(revenue) OVER (
        PARTITION BY product_id
        ORDER BY month
    ) AS product_ytd_revenue,
    MAX(revenue) OVER (
        PARTITION BY product_id
    ) AS product_best_month,
    MIN(revenue) OVER (
        PARTITION BY product_id
    ) AS product_worst_month
FROM product_monthly_sales
ORDER BY product_id, month;
```

---

## 95.7 Percentage of Total

### ตัวอย่างที่ 15: % ของ Total แต่ละ Partition

```sql
-- % ของยอดขายในแผนกและในบริษัท
SELECT 
    name,
    department,
    salary,
    -- % ของแผนก
    ROUND(salary * 100.0 / SUM(salary) OVER (PARTITION BY department), 2) AS pct_of_dept,
    -- % ของบริษัท
    ROUND(salary * 100.0 / SUM(salary) OVER (), 2) AS pct_of_company,
    -- เงินเดือน dept รวม
    SUM(salary) OVER (PARTITION BY department) AS dept_total_salary
FROM employees
ORDER BY department, salary DESC;
```

### ตัวอย่างที่ 16: Revenue Share Analysis

```sql
WITH regional_monthly AS (
    SELECT 
        region,
        DATE_TRUNC('month', sale_date) AS month,
        SUM(quantity * unit_price) AS revenue
    FROM sales
    GROUP BY region, DATE_TRUNC('month', sale_date)
)
SELECT 
    region,
    month,
    revenue,
    -- % ของ month นั้น
    ROUND(revenue * 100.0 / SUM(revenue) OVER (PARTITION BY month), 2) AS pct_of_month,
    -- % ของ region นั้น
    ROUND(revenue * 100.0 / SUM(revenue) OVER (PARTITION BY region), 2) AS pct_of_region,
    -- cumulative ใน region
    SUM(revenue) OVER (PARTITION BY region ORDER BY month) AS region_cumulative
FROM regional_monthly
ORDER BY month, region;
```

---

## 95.8 Moving Statistics

### ตัวอย่างที่ 17: Moving Standard Deviation

```sql
-- Moving standard deviation สำหรับ volatility analysis
SELECT 
    trade_date,
    close_price,
    ROUND(AVG(close_price) OVER (
        ORDER BY trade_date ROWS BETWEEN 9 PRECEDING AND CURRENT ROW
    ), 2) AS ma_10d,
    ROUND(STDDEV(close_price) OVER (
        ORDER BY trade_date ROWS BETWEEN 9 PRECEDING AND CURRENT ROW
    ), 2) AS std_10d,
    -- Bollinger Bands
    ROUND(AVG(close_price) OVER (
        ORDER BY trade_date ROWS BETWEEN 19 PRECEDING AND CURRENT ROW
    ) + 2 * STDDEV(close_price) OVER (
        ORDER BY trade_date ROWS BETWEEN 19 PRECEDING AND CURRENT ROW
    ), 2) AS upper_band,
    ROUND(AVG(close_price) OVER (
        ORDER BY trade_date ROWS BETWEEN 19 PRECEDING AND CURRENT ROW
    ) - 2 * STDDEV(close_price) OVER (
        ORDER BY trade_date ROWS BETWEEN 19 PRECEDING AND CURRENT ROW
    ), 2) AS lower_band
FROM stock_prices
WHERE ticker = 'ADVANC'
ORDER BY trade_date;
```

### ตัวอย่างที่ 18: Exponential Moving Average (EMA) approximation

```sql
-- EMA ใน SQL ต้องใช้ recursive CTE
WITH RECURSIVE ema_calc AS (
    -- Seed: ค่าแรก
    SELECT 
        trade_date,
        close_price,
        close_price AS ema,
        ROW_NUMBER() OVER (ORDER BY trade_date) AS rn
    FROM stock_prices
    WHERE ticker = 'ADVANC'
      AND trade_date = (SELECT MIN(trade_date) FROM stock_prices WHERE ticker = 'ADVANC')
    
    UNION ALL
    
    SELECT 
        s.trade_date,
        s.close_price,
        -- EMA formula: α * price + (1 - α) * prev_ema, α = 2/(n+1) โดย n = 5
        0.333 * s.close_price + 0.667 * e.ema,
        e.rn + 1
    FROM stock_prices s
    JOIN ema_calc e ON s.trade_date > e.trade_date
    WHERE s.ticker = 'ADVANC'
      AND s.trade_date = (
          SELECT MIN(trade_date) FROM stock_prices 
          WHERE ticker = 'ADVANC' AND trade_date > e.trade_date
      )
)
SELECT trade_date, close_price, ROUND(ema, 2) AS ema_5period
FROM ema_calc
ORDER BY trade_date;
```

---

## 95.9 Sliding Window ขั้นสูง

### ตัวอย่างที่ 19: RANGE-based Window (Time-based)

```sql
-- PostgreSQL: ใช้ interval ใน RANGE
SELECT 
    metric_date,
    page_views,
    -- ค่าเฉลี่ย 7 วันที่ผ่านมา (based on date value ไม่ใช่ row count)
    ROUND(AVG(page_views) OVER (
        ORDER BY metric_date
        RANGE BETWEEN INTERVAL '6 days' PRECEDING AND CURRENT ROW
    ), 2) AS avg_last_7d_range,
    -- ค่าเฉลี่ย 7 rows ที่แล้ว (based on row count)
    ROUND(AVG(page_views) OVER (
        ORDER BY metric_date
        ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
    ), 2) AS avg_last_7d_rows
FROM daily_metrics
ORDER BY metric_date;
-- แตกต่างกันเมื่อมีวันที่หายไป (gaps in data)
```

### ตัวอย่างที่ 20: GROUPS Frame (PostgreSQL 11+)

```sql
-- GROUPS: นับกลุ่มของค่าเท่ากัน
SELECT 
    student,
    score,
    -- 2 GROUPS PRECEDING: รวม 3 กลุ่ม score ที่ต่ำกว่าหรือเท่ากัน
    SUM(score) OVER (
        ORDER BY score
        GROUPS BETWEEN 1 PRECEDING AND CURRENT ROW
    ) AS groups_sum
FROM score_table;
```

---

## 95.10 Comprehensive Analytics

### ตัวอย่างที่ 21: Sales Dashboard Metrics

```sql
-- Dashboard metrics ครบถ้วน
WITH daily_rev AS (
    SELECT 
        DATE_TRUNC('day', sale_date) AS dt,
        SUM(quantity * unit_price) AS daily_revenue
    FROM sales
    GROUP BY DATE_TRUNC('day', sale_date)
)
SELECT 
    dt AS date,
    daily_revenue,
    -- Running totals
    SUM(daily_revenue) OVER (ORDER BY dt) AS ytd_revenue,
    -- Moving averages
    ROUND(AVG(daily_revenue) OVER (ORDER BY dt ROWS BETWEEN 6 PRECEDING AND CURRENT ROW), 2) AS ma_7d,
    ROUND(AVG(daily_revenue) OVER (ORDER BY dt ROWS BETWEEN 29 PRECEDING AND CURRENT ROW), 2) AS ma_30d,
    -- Min/Max in window
    MIN(daily_revenue) OVER (ORDER BY dt ROWS BETWEEN 6 PRECEDING AND CURRENT ROW) AS min_7d,
    MAX(daily_revenue) OVER (ORDER BY dt ROWS BETWEEN 6 PRECEDING AND CURRENT ROW) AS max_7d,
    -- DoD
    ROUND(
        (daily_revenue - LAG(daily_revenue) OVER (ORDER BY dt)) * 100.0 /
        NULLIF(LAG(daily_revenue) OVER (ORDER BY dt), 0),
    1) AS dod_pct
FROM daily_rev
ORDER BY dt;
```

### ตัวอย่างที่ 22: Customer Spending Rolling Analysis

```sql
WITH customer_monthly_spend AS (
    SELECT 
        customer_id,
        DATE_TRUNC('month', purchase_date) AS month,
        SUM(amount) AS monthly_spend
    FROM customer_purchases
    GROUP BY customer_id, DATE_TRUNC('month', purchase_date)
)
SELECT 
    customer_id,
    month,
    monthly_spend,
    -- Cumulative spend
    SUM(monthly_spend) OVER (PARTITION BY customer_id ORDER BY month) AS lifetime_spend,
    -- 3-month rolling average
    ROUND(AVG(monthly_spend) OVER (
        PARTITION BY customer_id ORDER BY month
        ROWS BETWEEN 2 PRECEDING AND CURRENT ROW
    ), 2) AS rolling_3m_avg,
    -- % change vs rolling average
    ROUND(
        (monthly_spend - AVG(monthly_spend) OVER (
            PARTITION BY customer_id ORDER BY month
            ROWS BETWEEN 3 PRECEDING AND 1 PRECEDING
        )) * 100.0 / NULLIF(
            AVG(monthly_spend) OVER (
                PARTITION BY customer_id ORDER BY month
                ROWS BETWEEN 3 PRECEDING AND 1 PRECEDING
            ), 0),
    2) AS vs_prev_3m_avg_pct
FROM customer_monthly_spend
ORDER BY customer_id, month;
```

### ตัวอย่างที่ 23: Percentile Calculations

```sql
-- Percentile ของแต่ละ employee ในแผนก
SELECT 
    name,
    department,
    salary,
    -- Running percentile ตามจำนวนคนที่มีเงินเดือนน้อยกว่าหรือเท่ากัน
    ROUND(
        COUNT(*) FILTER (WHERE salary2.salary <= employees.salary) * 100.0 /
        COUNT(*) OVER (PARTITION BY department),
    1) AS dept_percentile
FROM employees
CROSS JOIN (SELECT salary FROM employees) salary2
WHERE employees.department = salary2.department  -- ต้อง match department
GROUP BY name, department, salary;

-- วิธีที่ถูกต้องกว่า ใช้ PERCENT_RANK
SELECT 
    name,
    department,
    salary,
    ROUND(PERCENT_RANK() OVER (PARTITION BY department ORDER BY salary) * 100, 1) AS dept_percentile,
    ROUND(CUME_DIST() OVER (PARTITION BY department ORDER BY salary) * 100, 1) AS dept_cume_dist
FROM employees
ORDER BY department, salary;
```

### ตัวอย่างที่ 24: Anomaly Detection ด้วย Moving Statistics

```sql
-- ตรวจจับ anomaly โดยใช้ moving average + standard deviation
WITH moving_stats AS (
    SELECT 
        metric_date,
        page_views,
        AVG(page_views) OVER (
            ORDER BY metric_date
            ROWS BETWEEN 6 PRECEDING AND 1 PRECEDING
        ) AS ma_prev_7d,
        STDDEV(page_views) OVER (
            ORDER BY metric_date
            ROWS BETWEEN 6 PRECEDING AND 1 PRECEDING
        ) AS std_prev_7d
    FROM daily_metrics
)
SELECT 
    metric_date,
    page_views,
    ROUND(ma_prev_7d, 0) AS expected_views,
    ROUND(std_prev_7d, 0) AS std_dev,
    ROUND((page_views - ma_prev_7d) / NULLIF(std_prev_7d, 0), 2) AS z_score,
    CASE 
        WHEN ABS(page_views - ma_prev_7d) > 2 * std_prev_7d THEN 'ANOMALY'
        WHEN ABS(page_views - ma_prev_7d) > 1.5 * std_prev_7d THEN 'WARNING'
        ELSE 'NORMAL'
    END AS status
FROM moving_stats
WHERE ma_prev_7d IS NOT NULL
ORDER BY metric_date;
```

### ตัวอย่างที่ 25: Financial Trend Analysis

```sql
-- วิเคราะห์ trend ทางการเงิน
WITH quarterly AS (
    SELECT 
        yr,
        qtr,
        revenue,
        cost,
        revenue - cost AS profit,
        ROUND((revenue - cost) * 100.0 / revenue, 2) AS margin
    FROM (
        SELECT 
            EXTRACT(YEAR FROM year_month) AS yr,
            EXTRACT(QUARTER FROM year_month) AS qtr,
            SUM(revenue) AS revenue,
            SUM(revenue) * 0.6 AS cost   -- สมมติ cost = 60% of revenue
        FROM monthly_sales
        GROUP BY yr, qtr
    ) q
)
SELECT 
    yr,
    qtr,
    revenue,
    profit,
    margin,
    -- Running total per year
    SUM(revenue) OVER (PARTITION BY yr ORDER BY qtr) AS ytd_revenue,
    -- QoQ growth
    ROUND(
        (revenue - LAG(revenue) OVER (ORDER BY yr, qtr)) * 100.0 /
        NULLIF(LAG(revenue) OVER (ORDER BY yr, qtr), 0),
    2) AS qoq_growth,
    -- YoY (same quarter last year)
    ROUND(
        (revenue - LAG(revenue, 4) OVER (ORDER BY yr, qtr)) * 100.0 /
        NULLIF(LAG(revenue, 4) OVER (ORDER BY yr, qtr), 0),
    2) AS yoy_growth,
    -- Trailing 4-quarter revenue
    SUM(revenue) OVER (ORDER BY yr, qtr ROWS BETWEEN 3 PRECEDING AND CURRENT ROW) AS ttm_revenue
FROM quarterly
ORDER BY yr, qtr;
```

---

## 95.11 ตัวอย่างขั้นสูง

### ตัวอย่างที่ 26: Weighted Moving Average

```sql
-- Weighted Moving Average: ค่าใหม่มี weight มากกว่า
-- WMA(3) = (3*current + 2*prev1 + 1*prev2) / 6
SELECT 
    year_month,
    revenue,
    ROUND(
        (3 * revenue + 
         2 * LAG(revenue, 1) OVER (ORDER BY year_month) +
         1 * LAG(revenue, 2) OVER (ORDER BY year_month)) /
        NULLIF(
            3 + 
            CASE WHEN LAG(revenue, 1) OVER (ORDER BY year_month) IS NOT NULL THEN 2 ELSE 0 END +
            CASE WHEN LAG(revenue, 2) OVER (ORDER BY year_month) IS NOT NULL THEN 1 ELSE 0 END,
        0),
    2) AS wma_3month
FROM monthly_sales
ORDER BY year_month;
```

### ตัวอย่างที่ 27: RANGE with UNBOUNDED

```sql
-- เปรียบเทียบทุก window boundary
SELECT 
    year_month,
    revenue,
    -- ทั้งหมดตั้งแต่ต้น
    SUM(revenue) OVER (ORDER BY year_month 
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS cum_total,
    -- เฉพาะ 3 เดือน centered
    AVG(revenue) OVER (ORDER BY year_month 
        ROWS BETWEEN 1 PRECEDING AND 1 FOLLOWING) AS centered_avg,
    -- ที่เหลือในอนาคต (ใช้ planning)
    SUM(revenue) OVER (ORDER BY year_month 
        ROWS BETWEEN CURRENT ROW AND UNBOUNDED FOLLOWING) AS remaining_total,
    -- Grand total
    SUM(revenue) OVER () AS grand_total
FROM monthly_sales;
```

### ตัวอย่างที่ 28: Revenue Forecast Simple

```sql
-- Simple linear forecast based on moving average trend
WITH with_ma AS (
    SELECT 
        year_month,
        revenue,
        AVG(revenue) OVER (ORDER BY year_month ROWS BETWEEN 2 PRECEDING AND CURRENT ROW) AS ma_3m
    FROM monthly_sales
),
with_trend AS (
    SELECT 
        year_month,
        revenue,
        ma_3m,
        ma_3m - LAG(ma_3m) OVER (ORDER BY year_month) AS ma_monthly_trend
    FROM with_ma
)
SELECT 
    year_month,
    revenue,
    ROUND(ma_3m, 2) AS moving_avg,
    ROUND(ma_monthly_trend, 2) AS trend,
    -- Simple forecast: current MA + trend
    ROUND(ma_3m + COALESCE(ma_monthly_trend, 0), 2) AS next_month_forecast
FROM with_trend
ORDER BY year_month;
```

### ตัวอย่างที่ 29: Cohort Retention with Window Functions

```sql
-- คำนวณ retention rate ด้วย window functions
WITH cohort_activity AS (
    SELECT 
        customer_id,
        DATE_TRUNC('month', MIN(purchase_date)) AS cohort_month,
        DATE_TRUNC('month', purchase_date) AS activity_month
    FROM customer_purchases
    GROUP BY customer_id, DATE_TRUNC('month', purchase_date)
),
cohort_counts AS (
    SELECT 
        cohort_month,
        activity_month,
        EXTRACT(MONTH FROM AGE(activity_month, cohort_month)) AS months_since_join,
        COUNT(DISTINCT customer_id) AS active_customers
    FROM cohort_activity
    GROUP BY cohort_month, activity_month
),
with_cohort_size AS (
    SELECT 
        cohort_month,
        activity_month,
        months_since_join,
        active_customers,
        FIRST_VALUE(active_customers) OVER (
            PARTITION BY cohort_month ORDER BY activity_month
        ) AS cohort_size
    FROM cohort_counts
)
SELECT 
    cohort_month,
    months_since_join,
    active_customers,
    cohort_size,
    ROUND(active_customers * 100.0 / cohort_size, 2) AS retention_rate
FROM with_cohort_size
ORDER BY cohort_month, months_since_join;
```

### ตัวอย่างที่ 30: Inventory Tracking

```sql
CREATE TABLE inventory_movements (
    movement_id INT PRIMARY KEY,
    product_id  INT,
    move_date   DATE,
    quantity    INT,       -- บวก = รับเข้า, ลบ = จ่ายออก
    move_type   VARCHAR(20)
);

INSERT INTO inventory_movements VALUES
(1, 1001, '2024-01-01',  100, 'initial_stock'),
(2, 1001, '2024-01-05',  -30, 'sale'),
(3, 1001, '2024-01-08',  -20, 'sale'),
(4, 1001, '2024-01-10',   50, 'restock'),
(5, 1001, '2024-01-15',  -40, 'sale'),
(6, 1001, '2024-01-18',  -15, 'sale'),
(7, 1001, '2024-01-22',  100, 'restock'),
(8, 1001, '2024-01-25',  -25, 'sale');

-- Running inventory balance
SELECT 
    movement_id,
    product_id,
    move_date,
    quantity,
    move_type,
    SUM(quantity) OVER (
        PARTITION BY product_id
        ORDER BY move_date, movement_id
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    ) AS stock_on_hand,
    -- Alert เมื่อ stock ต่ำ
    CASE 
        WHEN SUM(quantity) OVER (
            PARTITION BY product_id
            ORDER BY move_date, movement_id
            ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
        ) < 30 THEN 'LOW STOCK'
        ELSE 'OK'
    END AS stock_status
FROM inventory_movements
ORDER BY product_id, move_date, movement_id;
```

### ตัวอย่างที่ 31: Performance Against Benchmark

```sql
-- เปรียบเทียบ performance กับ moving benchmark
WITH sales_with_ma AS (
    SELECT 
        salesperson,
        DATE_TRUNC('month', sale_date) AS month,
        SUM(quantity * unit_price) AS monthly_sales
    FROM sales
    GROUP BY salesperson, DATE_TRUNC('month', sale_date)
),
with_team_avg AS (
    SELECT 
        salesperson,
        month,
        monthly_sales,
        AVG(monthly_sales) OVER (PARTITION BY month) AS team_avg_sales,
        RANK() OVER (PARTITION BY month ORDER BY monthly_sales DESC) AS monthly_rank
    FROM sales_with_ma
)
SELECT 
    salesperson,
    month,
    monthly_sales,
    ROUND(team_avg_sales, 2) AS team_avg,
    monthly_rank,
    ROUND((monthly_sales - team_avg_sales) * 100.0 / team_avg_sales, 2) AS vs_team_avg_pct,
    -- Cumulative rank
    SUM(CASE WHEN monthly_rank = 1 THEN 1 ELSE 0 END) OVER (
        PARTITION BY salesperson ORDER BY month
    ) AS times_ranked_first
FROM with_team_avg
ORDER BY month, monthly_rank;
```

### ตัวอย่างที่ 32: Transaction Volume Analysis

```sql
-- วิเคราะห์ volume ธุรกรรม
SELECT 
    txn_date,
    COUNT(*) AS daily_txn_count,
    SUM(ABS(amount)) AS daily_volume,
    
    -- 7-day moving metrics
    ROUND(AVG(COUNT(*)) OVER (
        ORDER BY txn_date ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
    ), 1) AS ma_7d_count,
    
    ROUND(AVG(SUM(ABS(amount))) OVER (
        ORDER BY txn_date ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
    ), 2) AS ma_7d_volume,
    
    -- Cumulative YTD
    SUM(COUNT(*)) OVER (
        ORDER BY txn_date
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    ) AS ytd_txn_count
FROM bank_transactions
GROUP BY txn_date
ORDER BY txn_date;
```

---

## 95.12 การใช้ FILTER กับ Window Functions

### ตัวอย่างที่ 33: COUNT และ SUM พร้อม FILTER

```sql
-- FILTER ใน window functions (PostgreSQL)
SELECT 
    name,
    department,
    salary,
    -- นับเฉพาะพนักงานที่เงินเดือน > 70000 ในแต่ละแผนก
    COUNT(*) FILTER (WHERE salary > 70000) OVER (PARTITION BY department) AS high_earners_in_dept,
    -- Sum ของเงินเดือนสูง
    SUM(salary) FILTER (WHERE salary > 70000) OVER (PARTITION BY department) AS high_salary_sum,
    -- % ที่เงินเดือนสูงกว่า 70000
    ROUND(
        COUNT(*) FILTER (WHERE salary > 70000) OVER (PARTITION BY department) * 100.0 /
        COUNT(*) OVER (PARTITION BY department),
    1) AS high_earner_pct
FROM employees
ORDER BY department, salary DESC;
```

### ตัวอย่างที่ 34: Conditional Running Total

```sql
-- Running total แยกตาม type
SELECT 
    txn_id,
    account_id,
    txn_date,
    amount,
    -- Running total เฉพาะ credit
    SUM(CASE WHEN amount > 0 THEN amount ELSE 0 END) OVER (
        PARTITION BY account_id
        ORDER BY txn_date, txn_id
    ) AS running_credits,
    -- Running total เฉพาะ debit
    SUM(CASE WHEN amount < 0 THEN ABS(amount) ELSE 0 END) OVER (
        PARTITION BY account_id
        ORDER BY txn_date, txn_id
    ) AS running_debits,
    -- Net balance
    SUM(amount) OVER (
        PARTITION BY account_id
        ORDER BY txn_date, txn_id
    ) AS running_net
FROM bank_transactions
ORDER BY account_id, txn_date, txn_id;
```

---

## 95.13 Performance Tips

### ตัวอย่างที่ 35: เหตุใดจึงควรใช้ CTE กับ Window Functions

```sql
-- ไม่ดี: คำนวณ window function ซ้ำหลายครั้ง
SELECT 
    name,
    salary,
    SUM(salary) OVER (PARTITION BY department) AS dept_total,
    salary / SUM(salary) OVER (PARTITION BY department) AS pct_of_dept,
    AVG(salary) OVER (PARTITION BY department) AS dept_avg,
    salary - AVG(salary) OVER (PARTITION BY department) AS vs_avg
FROM employees;

-- ดีกว่า: คำนวณครั้งเดียวใน CTE
WITH dept_window AS (
    SELECT 
        emp_id,
        name,
        department,
        salary,
        SUM(salary) OVER (PARTITION BY department) AS dept_total,
        AVG(salary) OVER (PARTITION BY department) AS dept_avg
    FROM employees
)
SELECT 
    name,
    department,
    salary,
    dept_total,
    ROUND(salary * 100.0 / dept_total, 2) AS pct_of_dept,
    dept_avg,
    ROUND(salary - dept_avg, 2) AS vs_avg
FROM dept_window
ORDER BY department, salary DESC;
```

### ตัวอย่างที่ 36: ลดการ Compute ซ้ำ

```sql
-- Optimal: ทำ GROUP BY ก่อน แล้วค่อย apply window
WITH monthly_agg AS (
    SELECT 
        DATE_TRUNC('month', sale_date) AS month,
        salesperson,
        SUM(quantity * unit_price) AS monthly_rev
    FROM sales
    GROUP BY DATE_TRUNC('month', sale_date), salesperson
)
SELECT 
    month,
    salesperson,
    monthly_rev,
    SUM(monthly_rev) OVER (PARTITION BY salesperson ORDER BY month) AS cumulative_rev,
    AVG(monthly_rev) OVER (PARTITION BY salesperson ORDER BY month ROWS BETWEEN 2 PRECEDING AND CURRENT ROW) AS ma_3m
FROM monthly_agg
ORDER BY salesperson, month;
```

---

## 95.14 ตัวอย่างเพิ่มเติม

### ตัวอย่างที่ 37: Count of Preceding Rows Matching Condition

```sql
-- นับว่ามีกี่เดือนที่ยอดขายเพิ่มขึ้นในช่วง 6 เดือนที่ผ่านมา
WITH monthly_change AS (
    SELECT 
        year_month,
        revenue,
        CASE 
            WHEN revenue > LAG(revenue) OVER (ORDER BY year_month) THEN 1 
            ELSE 0 
        END AS is_increase
    FROM monthly_sales
)
SELECT 
    year_month,
    revenue,
    is_increase,
    SUM(is_increase) OVER (
        ORDER BY year_month
        ROWS BETWEEN 5 PRECEDING AND CURRENT ROW
    ) AS positive_months_in_6m
FROM monthly_change
ORDER BY year_month;
```

### ตัวอย่างที่ 38: Average Excluding Current Row

```sql
-- ค่าเฉลี่ยของแผนกไม่รวมคนปัจจุบัน (peer average)
SELECT 
    name,
    department,
    salary,
    ROUND(
        (SUM(salary) OVER (PARTITION BY department) - salary) /
        NULLIF(COUNT(*) OVER (PARTITION BY department) - 1, 0),
    2) AS peer_avg_salary,
    salary - ROUND(
        (SUM(salary) OVER (PARTITION BY department) - salary) /
        NULLIF(COUNT(*) OVER (PARTITION BY department) - 1, 0),
    2) AS vs_peer_avg
FROM employees
ORDER BY department, salary DESC;
```

### ตัวอย่างที่ 39: Rolling Retention Rate

```sql
-- Rolling 90-day retention
WITH daily_active AS (
    SELECT 
        DATE_TRUNC('day', action_time) AS day,
        COUNT(DISTINCT user_id) AS dau
    FROM user_actions
    GROUP BY DATE_TRUNC('day', action_time)
)
SELECT 
    day,
    dau,
    -- 30-day rolling DAU average
    ROUND(AVG(dau) OVER (
        ORDER BY day ROWS BETWEEN 29 PRECEDING AND CURRENT ROW
    ), 0) AS mau_proxy,  -- Approximate Monthly Active Users
    -- Day 1 retention approximation
    ROUND(dau * 100.0 / MAX(dau) OVER (), 2) AS relative_to_peak
FROM daily_active
ORDER BY day;
```

### ตัวอย่างที่ 40: Comprehensive KPI Dashboard

```sql
-- KPI Dashboard ครบถ้วน
WITH daily_kpis AS (
    SELECT 
        metric_date,
        page_views,
        sessions,
        conversions,
        ROUND(conversions * 100.0 / NULLIF(sessions, 0), 2) AS conv_rate
    FROM daily_metrics
),
with_trends AS (
    SELECT 
        metric_date,
        page_views,
        sessions,
        conversions,
        conv_rate,
        -- 7-day moving averages
        ROUND(AVG(page_views) OVER (ORDER BY metric_date ROWS BETWEEN 6 PRECEDING AND CURRENT ROW), 0) AS views_ma7,
        ROUND(AVG(conv_rate) OVER (ORDER BY metric_date ROWS BETWEEN 6 PRECEDING AND CURRENT ROW), 2) AS conv_ma7,
        -- Cumulative totals
        SUM(page_views) OVER (ORDER BY metric_date) AS cum_views,
        SUM(conversions) OVER (ORDER BY metric_date) AS cum_conversions,
        -- vs Previous Week
        page_views - LAG(page_views, 7) OVER (ORDER BY metric_date) AS views_wow_change,
        -- Percentile in dataset
        ROUND(PERCENT_RANK() OVER (ORDER BY page_views) * 100, 1) AS views_percentile
    FROM daily_kpis
)
SELECT 
    metric_date,
    page_views,
    sessions,
    conversions,
    conv_rate,
    views_ma7,
    conv_ma7,
    cum_views,
    cum_conversions,
    views_wow_change,
    views_percentile,
    CASE 
        WHEN views_percentile >= 90 THEN 'Excellent Day'
        WHEN views_percentile >= 70 THEN 'Good Day'
        WHEN views_percentile >= 40 THEN 'Average Day'
        ELSE 'Below Average'
    END AS day_rating
FROM with_trends
ORDER BY metric_date;
```

---

## แบบฝึกหัดท้ายบท

### แบบฝึกหัดที่ 1
คำนวณ running balance ของบัญชีธนาคาร พร้อม alert เมื่อ balance ต่ำกว่า 10000

**คำตอบ:**
```sql
SELECT 
    txn_id,
    account_id,
    txn_date,
    amount,
    description,
    SUM(amount) OVER (
        PARTITION BY account_id
        ORDER BY txn_date, txn_id
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    ) AS balance,
    CASE 
        WHEN SUM(amount) OVER (
            PARTITION BY account_id
            ORDER BY txn_date, txn_id
            ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
        ) < 10000 THEN 'LOW BALANCE ALERT'
        ELSE 'OK'
    END AS alert
FROM bank_transactions
ORDER BY account_id, txn_date, txn_id;
```

### แบบฝึกหัดที่ 2
คำนวณ 3-month, 6-month, 12-month moving averages สำหรับ revenue

**คำตอบ:**
```sql
SELECT 
    year_month,
    revenue,
    ROUND(AVG(revenue) OVER (ORDER BY year_month ROWS BETWEEN 2 PRECEDING AND CURRENT ROW), 2) AS ma_3m,
    ROUND(AVG(revenue) OVER (ORDER BY year_month ROWS BETWEEN 5 PRECEDING AND CURRENT ROW), 2) AS ma_6m,
    ROUND(AVG(revenue) OVER (ORDER BY year_month ROWS BETWEEN 11 PRECEDING AND CURRENT ROW), 2) AS ma_12m
FROM monthly_sales
ORDER BY year_month;
```

### แบบฝึกหัดที่ 3
คำนวณ % ของยอดขายที่แต่ละสินค้า contribute ใน category และในบริษัท

**คำตอบ:**
```sql
SELECT 
    product_name,
    category,
    units_sold,
    ROUND(units_sold * 100.0 / SUM(units_sold) OVER (PARTITION BY category), 2) AS pct_in_category,
    ROUND(units_sold * 100.0 / SUM(units_sold) OVER (), 2) AS pct_of_total,
    SUM(units_sold) OVER (PARTITION BY category) AS category_total
FROM product_sales
ORDER BY category, units_sold DESC;
```

### แบบฝึกหัดที่ 4
Detect anomalies ใน daily page views โดยใช้ rolling mean และ standard deviation

**คำตอบ:**
```sql
SELECT 
    metric_date,
    page_views,
    ROUND(AVG(page_views) OVER (
        ORDER BY metric_date ROWS BETWEEN 6 PRECEDING AND 1 PRECEDING
    ), 0) AS rolling_mean,
    ROUND(STDDEV(page_views) OVER (
        ORDER BY metric_date ROWS BETWEEN 6 PRECEDING AND 1 PRECEDING
    ), 0) AS rolling_std,
    ROUND(
        (page_views - AVG(page_views) OVER (ORDER BY metric_date ROWS BETWEEN 6 PRECEDING AND 1 PRECEDING)) /
        NULLIF(STDDEV(page_views) OVER (ORDER BY metric_date ROWS BETWEEN 6 PRECEDING AND 1 PRECEDING), 0),
    2) AS z_score,
    CASE 
        WHEN ABS(page_views - AVG(page_views) OVER (ORDER BY metric_date ROWS BETWEEN 6 PRECEDING AND 1 PRECEDING))
             > 2 * STDDEV(page_views) OVER (ORDER BY metric_date ROWS BETWEEN 6 PRECEDING AND 1 PRECEDING)
        THEN 'ANOMALY'
        ELSE 'NORMAL'
    END AS status
FROM daily_metrics
ORDER BY metric_date;
```

### แบบฝึกหัดที่ 5
คำนวณ YTD revenue แยกตามปี และ % ของ full year target (สมมติ target = 2M ต่อปี)

**คำตอบ:**
```sql
SELECT 
    year_month,
    EXTRACT(YEAR FROM year_month) AS yr,
    revenue,
    SUM(revenue) OVER (
        PARTITION BY EXTRACT(YEAR FROM year_month)
        ORDER BY year_month
    ) AS ytd_revenue,
    ROUND(
        SUM(revenue) OVER (
            PARTITION BY EXTRACT(YEAR FROM year_month)
            ORDER BY year_month
        ) * 100.0 / 2000000,
    2) AS pct_of_annual_target
FROM monthly_sales
ORDER BY year_month;
```

### แบบฝึกหัดที่ 6
สร้าง Bollinger Bands (MA ± 2 standard deviations) สำหรับ stock price

**คำตอบ:**
```sql
SELECT 
    ticker,
    trade_date,
    close_price,
    ROUND(AVG(close_price) OVER (
        PARTITION BY ticker ORDER BY trade_date
        ROWS BETWEEN 19 PRECEDING AND CURRENT ROW
    ), 2) AS sma_20,
    ROUND(AVG(close_price) OVER (
        PARTITION BY ticker ORDER BY trade_date
        ROWS BETWEEN 19 PRECEDING AND CURRENT ROW
    ) + 2 * STDDEV(close_price) OVER (
        PARTITION BY ticker ORDER BY trade_date
        ROWS BETWEEN 19 PRECEDING AND CURRENT ROW
    ), 2) AS upper_band,
    ROUND(AVG(close_price) OVER (
        PARTITION BY ticker ORDER BY trade_date
        ROWS BETWEEN 19 PRECEDING AND CURRENT ROW
    ) - 2 * STDDEV(close_price) OVER (
        PARTITION BY ticker ORDER BY trade_date
        ROWS BETWEEN 19 PRECEDING AND CURRENT ROW
    ), 2) AS lower_band
FROM stock_prices
ORDER BY ticker, trade_date;
```

### แบบฝึกหัดที่ 7
คำนวณ cumulative % ของยอดขาย (Pareto curve) สำหรับแต่ละสินค้า

**คำตอบ:**
```sql
WITH product_totals AS (
    SELECT product_name, category, units_sold,
           SUM(units_sold) OVER () AS grand_total
    FROM product_sales
)
SELECT 
    product_name,
    category,
    units_sold,
    ROUND(units_sold * 100.0 / grand_total, 2) AS pct_of_total,
    ROUND(SUM(units_sold) OVER (ORDER BY units_sold DESC) * 100.0 / grand_total, 2) AS cumulative_pct,
    CASE 
        WHEN SUM(units_sold) OVER (ORDER BY units_sold DESC) * 100.0 / grand_total <= 80
        THEN 'A - Top 80%'
        WHEN SUM(units_sold) OVER (ORDER BY units_sold DESC) * 100.0 / grand_total <= 95
        THEN 'B - Next 15%'
        ELSE 'C - Bottom 5%'
    END AS abc_class
FROM product_totals
ORDER BY units_sold DESC;
```

### แบบฝึกหัดที่ 8
แสดง running count ของพนักงานที่ถูก hire ตามลำดับเวลา แยกตาม department

**คำตอบ:**
```sql
SELECT 
    hire_date,
    name,
    department,
    COUNT(*) OVER (ORDER BY hire_date, emp_id) AS company_headcount_at_hire,
    COUNT(*) OVER (PARTITION BY department ORDER BY hire_date, emp_id) AS dept_headcount_at_hire,
    SUM(salary) OVER (ORDER BY hire_date, emp_id) AS cumulative_salary_bill
FROM employees
ORDER BY hire_date, emp_id;
```

### แบบฝึกหัดที่ 9
คำนวณ moving sum ของ transaction สำหรับ fraud detection (transactions ใน 24 ชั่วโมง)

**คำตอบ:**
```sql
WITH txn_with_window AS (
    SELECT 
        txn_id,
        account_id,
        txn_date,
        amount,
        SUM(ABS(amount)) OVER (
            PARTITION BY account_id
            ORDER BY txn_date
            RANGE BETWEEN INTERVAL '1 day' PRECEDING AND CURRENT ROW
        ) AS rolling_24h_volume,
        COUNT(*) OVER (
            PARTITION BY account_id
            ORDER BY txn_date
            RANGE BETWEEN INTERVAL '1 day' PRECEDING AND CURRENT ROW
        ) AS txn_count_24h
    FROM bank_transactions
)
SELECT *,
    CASE 
        WHEN rolling_24h_volume > 50000 OR txn_count_24h > 10 
        THEN 'FRAUD ALERT'
        ELSE 'NORMAL'
    END AS fraud_flag
FROM txn_with_window
ORDER BY account_id, txn_date;
```

### แบบฝึกหัดที่ 10
สร้าง comprehensive inventory report: running stock, reorder alerts, และ average daily usage

**คำตอบ:**
```sql
WITH daily_usage AS (
    SELECT 
        product_id,
        move_date,
        quantity,
        move_type,
        SUM(quantity) OVER (
            PARTITION BY product_id
            ORDER BY move_date, movement_id
        ) AS running_stock,
        AVG(CASE WHEN move_type = 'sale' THEN ABS(quantity) ELSE NULL END) OVER (
            PARTITION BY product_id
            ORDER BY move_date
            ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
        ) AS avg_daily_usage_7d
    FROM inventory_movements
)
SELECT 
    product_id,
    move_date,
    quantity,
    move_type,
    running_stock,
    ROUND(avg_daily_usage_7d, 1) AS avg_daily_usage,
    CASE 
        WHEN avg_daily_usage_7d > 0 
        THEN ROUND(running_stock / avg_daily_usage_7d, 1)
        ELSE NULL
    END AS days_of_stock,
    CASE 
        WHEN running_stock < 20 THEN 'REORDER NOW'
        WHEN running_stock < 50 THEN 'REORDER SOON'
        ELSE 'ADEQUATE'
    END AS stock_status
FROM daily_usage
ORDER BY product_id, move_date;
```

---

## สรุปบทที่ 95

Aggregate Window Functions และ Frame Specifications:

### Frame Summary

| Frame | ความหมาย |
|-------|-----------|
| ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW | Running total (cumulative) |
| ROWS BETWEEN n PRECEDING AND CURRENT ROW | Rolling/Moving n-period |
| ROWS BETWEEN n PRECEDING AND n FOLLOWING | Centered window |
| ROWS BETWEEN CURRENT ROW AND UNBOUNDED FOLLOWING | Reverse cumulative |
| ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING | Entire partition |

### ข้อควรจำสำคัญ

- Default frame: `RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW`
- `RANGE` ใช้ value-based comparison, `ROWS` ใช้ physical row count
- `RANGE BETWEEN n PRECEDING AND CURRENT ROW` รองรับ interval สำหรับ date/timestamp ใน PostgreSQL
- Window functions ทำงานหลัง WHERE และ GROUP BY แต่ก่อน HAVING และ ORDER BY

ในบทถัดไปเราจะรวม CTEs และ Window Functions เข้าด้วยกันเพื่อแก้ปัญหา advanced analytics เช่น gap-and-island, sessions analysis, และ cohort analysis
