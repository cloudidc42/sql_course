# ส่วนที่ 94: Window Functions ตอนที่ 2 - Value Functions

## บทนำ

Value Functions ในกลุ่ม Window Functions ช่วยให้เราเข้าถึงค่าของแถวอื่นๆ ที่อยู่ใน "window" เดียวกัน โดยไม่ต้องทำ self-join ซึ่งทำให้การวิเคราะห์ time series, period-over-period comparisons, และ trend analysis ทำได้ง่ายมากขึ้น

---

## 94.1 LAG() - ค่าจากแถวก่อนหน้า

### ไวยากรณ์

```sql
LAG(expression [, offset [, default]])
OVER ([PARTITION BY ...] ORDER BY ...)
```

- `expression`: column หรือ expression ที่ต้องการ
- `offset`: จำนวนแถวที่จะ "ย้อนหลัง" (default = 1)
- `default`: ค่าที่ใช้เมื่อไม่มีแถวก่อนหน้า (default = NULL)

### ตัวอย่างที่ 1: LAG พื้นฐาน

```sql
-- สร้างข้อมูลยอดขายรายเดือน
CREATE TABLE monthly_sales (
    year_month  DATE,  -- '2024-01-01' แทน Jan 2024
    revenue     DECIMAL(12,2),
    orders      INT
);

INSERT INTO monthly_sales VALUES
('2023-01-01', 120000, 450),
('2023-02-01', 135000, 480),
('2023-03-01', 128000, 460),
('2023-04-01', 145000, 510),
('2023-05-01', 162000, 580),
('2023-06-01', 175000, 620),
('2023-07-01', 158000, 570),
('2023-08-01', 182000, 650),
('2023-09-01', 170000, 610),
('2023-10-01', 195000, 700),
('2023-11-01', 210000, 750),
('2023-12-01', 248000, 890),
('2024-01-01', 165000, 580),
('2024-02-01', 178000, 630);

-- เปรียบเทียบกับเดือนก่อน
SELECT 
    year_month,
    revenue,
    LAG(revenue) OVER (ORDER BY year_month) AS prev_month_revenue
FROM monthly_sales;
```

### ตัวอย่างที่ 2: LAG พร้อมคำนวณ MoM Growth

```sql
-- คำนวณ Month-over-Month growth
SELECT 
    year_month,
    revenue,
    LAG(revenue) OVER (ORDER BY year_month) AS prev_revenue,
    revenue - LAG(revenue) OVER (ORDER BY year_month) AS revenue_change,
    ROUND(
        (revenue - LAG(revenue) OVER (ORDER BY year_month)) * 100.0 /
        NULLIF(LAG(revenue) OVER (ORDER BY year_month), 0),
    2) AS mom_growth_pct
FROM monthly_sales
ORDER BY year_month;
```

### ตัวอย่างที่ 3: LAG กับ offset > 1 (Year-over-Year)

```sql
-- เปรียบเทียบกับปีที่แล้ว (12 เดือนก่อน)
SELECT 
    year_month,
    revenue,
    LAG(revenue, 12) OVER (ORDER BY year_month) AS same_month_last_year,
    ROUND(
        (revenue - LAG(revenue, 12) OVER (ORDER BY year_month)) * 100.0 /
        NULLIF(LAG(revenue, 12) OVER (ORDER BY year_month), 0),
    2) AS yoy_growth_pct
FROM monthly_sales
ORDER BY year_month;
```

### ตัวอย่างที่ 4: LAG พร้อม default value

```sql
-- ใช้ default value เมื่อไม่มีแถวก่อนหน้า
SELECT 
    year_month,
    revenue,
    LAG(revenue, 1, 0) OVER (ORDER BY year_month) AS prev_revenue,
    -- ใช้ 0 แทน NULL สำหรับเดือนแรก
    revenue - LAG(revenue, 1, revenue) OVER (ORDER BY year_month) AS change_from_prev
FROM monthly_sales;
```

### ตัวอย่างที่ 5: LAG กับ PARTITION BY

```sql
-- เปรียบเทียบกับเดือนก่อนหน้า แยกตามแต่ละสินค้า
CREATE TABLE product_monthly_sales (
    product_id  INT,
    month       DATE,
    units       INT,
    revenue     DECIMAL(12,2)
);

INSERT INTO product_monthly_sales VALUES
(1, '2024-01-01', 100, 50000),
(1, '2024-02-01', 120, 60000),
(1, '2024-03-01', 95,  47500),
(2, '2024-01-01', 200, 40000),
(2, '2024-02-01', 180, 36000),
(2, '2024-03-01', 220, 44000);

SELECT 
    product_id,
    month,
    revenue,
    LAG(revenue) OVER (PARTITION BY product_id ORDER BY month) AS prev_month,
    ROUND(
        (revenue - LAG(revenue) OVER (PARTITION BY product_id ORDER BY month)) * 100.0 /
        NULLIF(LAG(revenue) OVER (PARTITION BY product_id ORDER BY month), 0),
    2) AS mom_pct
FROM product_monthly_sales
ORDER BY product_id, month;
```

---

## 94.2 LEAD() - ค่าจากแถวถัดไป

### ตัวอย่างที่ 6: LEAD พื้นฐาน

```sql
-- มองไปข้างหน้า: ยอดขายเดือนถัดไป
SELECT 
    year_month,
    revenue,
    LEAD(revenue) OVER (ORDER BY year_month) AS next_month_revenue,
    LEAD(revenue) OVER (ORDER BY year_month) - revenue AS expected_change
FROM monthly_sales
ORDER BY year_month;
```

### ตัวอย่างที่ 7: LEAD สำหรับ Time-to-next-event

```sql
-- คำนวณเวลาระหว่าง events
CREATE TABLE user_sessions (
    session_id  INT,
    user_id     INT,
    start_time  TIMESTAMP,
    end_time    TIMESTAMP
);

INSERT INTO user_sessions VALUES
(1, 101, '2024-01-01 09:00', '2024-01-01 09:45'),
(2, 101, '2024-01-01 11:30', '2024-01-01 12:15'),
(3, 101, '2024-01-02 14:00', '2024-01-02 14:30'),
(4, 102, '2024-01-01 10:00', '2024-01-01 10:30'),
(5, 102, '2024-01-03 09:00', '2024-01-03 09:20');

-- เวลาระหว่าง sessions ของแต่ละ user
SELECT 
    user_id,
    session_id,
    start_time,
    end_time,
    LEAD(start_time) OVER (PARTITION BY user_id ORDER BY start_time) AS next_session_start,
    EXTRACT(EPOCH FROM 
        LEAD(start_time) OVER (PARTITION BY user_id ORDER BY start_time) - end_time
    ) / 3600 AS hours_until_next_session
FROM user_sessions
ORDER BY user_id, start_time;
```

### ตัวอย่างที่ 8: LAG + LEAD พร้อมกัน

```sql
-- ดู context รอบๆ แต่ละแถว
SELECT 
    year_month,
    LAG(revenue) OVER (ORDER BY year_month) AS prev_month,
    revenue AS current_month,
    LEAD(revenue) OVER (ORDER BY year_month) AS next_month,
    -- 3-month average centered
    (COALESCE(LAG(revenue) OVER (ORDER BY year_month), revenue) +
     revenue +
     COALESCE(LEAD(revenue) OVER (ORDER BY year_month), revenue)) / 3 AS centered_3m_avg
FROM monthly_sales;
```

---

## 94.3 FIRST_VALUE() และ LAST_VALUE()

### ตัวอย่างที่ 9: FIRST_VALUE พื้นฐาน

```sql
-- เปรียบเทียบแต่ละ record กับ record แรกสุด
SELECT 
    year_month,
    revenue,
    FIRST_VALUE(revenue) OVER (ORDER BY year_month) AS first_month_revenue,
    revenue - FIRST_VALUE(revenue) OVER (ORDER BY year_month) AS change_from_start,
    ROUND(
        (revenue - FIRST_VALUE(revenue) OVER (ORDER BY year_month)) * 100.0 /
        FIRST_VALUE(revenue) OVER (ORDER BY year_month),
    2) AS pct_change_from_start
FROM monthly_sales;
```

### ตัวอย่างที่ 10: FIRST_VALUE กับ PARTITION BY

```sql
-- เปรียบเทียบแต่ละสินค้ากับยอดขายเดือนแรก
SELECT 
    product_id,
    month,
    revenue,
    FIRST_VALUE(revenue) OVER (
        PARTITION BY product_id 
        ORDER BY month
    ) AS initial_revenue,
    ROUND(
        (revenue - FIRST_VALUE(revenue) OVER (PARTITION BY product_id ORDER BY month)) * 100.0 /
        FIRST_VALUE(revenue) OVER (PARTITION BY product_id ORDER BY month),
    2) AS growth_since_launch
FROM product_monthly_sales
ORDER BY product_id, month;
```

### ตัวอย่างที่ 11: LAST_VALUE - ปัญหา Default Frame

```sql
-- ⚠️ LAST_VALUE มี gotcha: default frame คือ RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
-- ทำให้ LAST_VALUE ให้ค่าของแถวปัจจุบัน ไม่ใช่แถวสุดท้ายของ partition!

-- ผิด: LAST_VALUE ไม่ได้ให้แถวสุดท้ายของ window
SELECT 
    year_month,
    revenue,
    LAST_VALUE(revenue) OVER (ORDER BY year_month) AS wrong_last_value
    -- ← นี่จะ return ค่าปัจจุบันเสมอ!
FROM monthly_sales;

-- ถูก: ต้องระบุ frame ให้ครอบคลุมถึงสุด
SELECT 
    year_month,
    revenue,
    LAST_VALUE(revenue) OVER (
        ORDER BY year_month
        ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
    ) AS correct_last_value
FROM monthly_sales;
```

### ตัวอย่างที่ 12: FIRST_VALUE และ LAST_VALUE ใน Partition

```sql
-- หาค่าต่ำสุดและสูงสุดของช่วงเวลา แต่ยังเห็นทุก record
SELECT 
    year_month,
    revenue,
    FIRST_VALUE(revenue) OVER (
        ORDER BY year_month
        ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
    ) AS very_first_month_revenue,
    LAST_VALUE(revenue) OVER (
        ORDER BY year_month
        ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
    ) AS very_last_month_revenue,
    FIRST_VALUE(year_month) OVER (
        ORDER BY revenue DESC
        ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
    ) AS month_of_peak_revenue
FROM monthly_sales;
```

---

## 94.4 NTH_VALUE() - ค่าที่ n ใน Window

### ตัวอย่างที่ 13: NTH_VALUE พื้นฐาน

```sql
-- ดึงค่าของแถวที่ 2 และ 3 ในแต่ละ partition
SELECT 
    department,
    name,
    salary,
    NTH_VALUE(salary, 1) OVER (
        PARTITION BY department ORDER BY salary DESC
        ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
    ) AS highest_salary,
    NTH_VALUE(salary, 2) OVER (
        PARTITION BY department ORDER BY salary DESC
        ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
    ) AS second_highest,
    NTH_VALUE(salary, 3) OVER (
        PARTITION BY department ORDER BY salary DESC
        ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
    ) AS third_highest
FROM employees;
```

---

## 94.5 IGNORE NULLS

### ตัวอย่างที่ 14: LAG/LEAD กับ IGNORE NULLS

```sql
-- ข้อมูลที่มีค่า NULL บางช่วง
CREATE TABLE sensor_readings (
    sensor_id   INT,
    read_time   TIMESTAMP,
    temperature DECIMAL(5,2)
);

INSERT INTO sensor_readings VALUES
(1, '2024-01-01 00:00', 25.5),
(1, '2024-01-01 01:00', NULL),   -- sensor error
(1, '2024-01-01 02:00', NULL),   -- sensor error
(1, '2024-01-01 03:00', 26.1),
(1, '2024-01-01 04:00', 25.8);

-- โดยปกติ LAG จะ return NULL สำหรับแถวที่มี gap
SELECT 
    read_time,
    temperature,
    LAG(temperature) OVER (PARTITION BY sensor_id ORDER BY read_time) AS prev_reading
FROM sensor_readings;

-- ด้วย IGNORE NULLS: ข้าม NULL ไปหาค่าก่อนหน้าที่ไม่ใช่ NULL
-- (PostgreSQL ยังไม่รองรับ IGNORE NULLS สำหรับ LAG ใน standard SQL)
-- ใช้ workaround แทน:
SELECT 
    read_time,
    temperature,
    LAST_VALUE(temperature IGNORE NULLS) OVER (
        PARTITION BY sensor_id
        ORDER BY read_time
        ROWS BETWEEN UNBOUNDED PRECEDING AND 1 PRECEDING
    ) AS last_non_null_reading
FROM sensor_readings;
-- Note: IGNORE NULLS รองรับใน Oracle, SQL Server 2022+, Snowflake, BigQuery
```

### ตัวอย่างที่ 15: Forward Fill (Fill NULL ด้วยค่าก่อนหน้า)

```sql
-- PostgreSQL workaround สำหรับ forward fill
SELECT 
    read_time,
    temperature,
    -- Fill NULL ด้วยค่าก่อนหน้าที่ไม่ใช่ NULL
    COALESCE(
        temperature,
        LAST_VALUE(temperature) OVER (
            PARTITION BY sensor_id, 
            -- Trick: สร้าง group ที่เปลี่ยนทุกครั้งที่พบค่าไม่ใช่ NULL
            SUM(CASE WHEN temperature IS NOT NULL THEN 1 ELSE 0 END) OVER (
                PARTITION BY sensor_id ORDER BY read_time
            )
            ORDER BY read_time
        )
    ) AS filled_temperature
FROM sensor_readings;
```

---

## 94.6 Period-over-Period Comparisons

### ตัวอย่างที่ 16: Day-over-Day

```sql
-- สร้างข้อมูลรายวัน
CREATE TABLE daily_metrics (
    metric_date DATE,
    page_views  INT,
    sessions    INT,
    conversions INT
);

INSERT INTO daily_metrics VALUES
('2024-01-01', 5000,  1200, 85),
('2024-01-02', 5500,  1350, 92),
('2024-01-03', 4800,  1100, 78),
('2024-01-04', 6200,  1500, 110),
('2024-01-05', 7800,  1900, 145),
('2024-01-06', 8200,  2100, 162),
('2024-01-07', 9500,  2400, 185),
('2024-01-08', 6100,  1480, 108),
('2024-01-09', 5800,  1420, 102),
('2024-01-10', 6400,  1560, 115);

-- Day-over-Day comparison
SELECT 
    metric_date,
    page_views,
    LAG(page_views) OVER (ORDER BY metric_date) AS prev_day_views,
    page_views - LAG(page_views) OVER (ORDER BY metric_date) AS daily_change,
    ROUND(
        (page_views - LAG(page_views) OVER (ORDER BY metric_date)) * 100.0 /
        NULLIF(LAG(page_views) OVER (ORDER BY metric_date), 0),
    1) AS dod_pct_change
FROM daily_metrics
ORDER BY metric_date;
```

### ตัวอย่างที่ 17: Week-over-Week

```sql
-- เปรียบเทียบกับ 7 วันก่อน
SELECT 
    metric_date,
    page_views,
    LAG(page_views, 7) OVER (ORDER BY metric_date) AS same_day_last_week,
    ROUND(
        (page_views - LAG(page_views, 7) OVER (ORDER BY metric_date)) * 100.0 /
        NULLIF(LAG(page_views, 7) OVER (ORDER BY metric_date), 0),
    1) AS wow_pct_change
FROM daily_metrics
WHERE LAG(page_views, 7) OVER (ORDER BY metric_date) IS NOT NULL;
```

### ตัวอย่างที่ 18: Month-over-Month ที่แม่นยำ

```sql
-- MoM comparison โดยนับเดือน
WITH monthly_data AS (
    SELECT 
        DATE_TRUNC('month', metric_date) AS month,
        SUM(page_views) AS monthly_views,
        SUM(sessions) AS monthly_sessions,
        SUM(conversions) AS monthly_conversions,
        ROUND(SUM(conversions) * 100.0 / NULLIF(SUM(sessions), 0), 2) AS conv_rate
    FROM daily_metrics
    GROUP BY DATE_TRUNC('month', metric_date)
)
SELECT 
    month,
    monthly_views,
    LAG(monthly_views) OVER (ORDER BY month) AS prev_month_views,
    ROUND(
        (monthly_views - LAG(monthly_views) OVER (ORDER BY month)) * 100.0 /
        NULLIF(LAG(monthly_views) OVER (ORDER BY month), 0),
    2) AS mom_views_change,
    conv_rate,
    LAG(conv_rate) OVER (ORDER BY month) AS prev_conv_rate,
    conv_rate - LAG(conv_rate) OVER (ORDER BY month) AS conv_rate_change
FROM monthly_data
ORDER BY month;
```

### ตัวอย่างที่ 19: Year-over-Year Analysis

```sql
-- YoY comparison
CREATE TABLE yearly_summary (
    year        INT,
    revenue     DECIMAL(15,2),
    cost        DECIMAL(15,2),
    customers   INT
);

INSERT INTO yearly_summary VALUES
(2019, 5000000,  3000000, 1200),
(2020, 4500000,  2700000, 1050),  -- COVID impact
(2021, 6200000,  3400000, 1450),
(2022, 8100000,  4200000, 1900),
(2023, 9800000,  4800000, 2300),
(2024, 11200000, 5200000, 2700);

SELECT 
    year,
    revenue,
    cost,
    revenue - cost AS profit,
    LAG(revenue) OVER (ORDER BY year) AS prev_year_revenue,
    ROUND((revenue - LAG(revenue) OVER (ORDER BY year)) * 100.0 /
          NULLIF(LAG(revenue) OVER (ORDER BY year), 0), 2) AS yoy_revenue_growth,
    ROUND((revenue - cost) * 100.0 / revenue, 2) AS profit_margin,
    ROUND(
        ((revenue - cost) - LAG(revenue - cost) OVER (ORDER BY year)) * 100.0 /
        NULLIF(LAG(revenue - cost) OVER (ORDER BY year), 0),
    2) AS yoy_profit_growth
FROM yearly_summary;
```

---

## 94.7 ตัวอย่างขั้นสูง

### ตัวอย่างที่ 20: Stock Price Analysis

```sql
-- วิเคราะห์ราคาหุ้น
CREATE TABLE stock_prices (
    ticker      VARCHAR(10),
    trade_date  DATE,
    open_price  DECIMAL(10,2),
    close_price DECIMAL(10,2),
    high_price  DECIMAL(10,2),
    low_price   DECIMAL(10,2),
    volume      BIGINT
);

INSERT INTO stock_prices VALUES
('ADVANC', '2024-01-02', 215.00, 218.50, 220.00, 213.00, 5000000),
('ADVANC', '2024-01-03', 218.50, 215.00, 219.00, 214.00, 4800000),
('ADVANC', '2024-01-04', 215.00, 222.00, 223.50, 214.50, 6200000),
('ADVANC', '2024-01-05', 222.00, 219.50, 223.00, 218.00, 5500000),
('ADVANC', '2024-01-08', 219.50, 225.00, 226.00, 219.00, 7100000),
('ADVANC', '2024-01-09', 225.00, 223.00, 226.50, 222.00, 5900000),
('ADVANC', '2024-01-10', 223.00, 228.00, 229.00, 222.50, 6800000);

SELECT 
    ticker,
    trade_date,
    close_price,
    LAG(close_price) OVER (PARTITION BY ticker ORDER BY trade_date) AS prev_close,
    ROUND(close_price - LAG(close_price) OVER (PARTITION BY ticker ORDER BY trade_date), 2) AS daily_change,
    ROUND(
        (close_price - LAG(close_price) OVER (PARTITION BY ticker ORDER BY trade_date)) * 100.0 /
        LAG(close_price) OVER (PARTITION BY ticker ORDER BY trade_date),
    2) AS pct_change,
    CASE 
        WHEN close_price > LAG(close_price) OVER (PARTITION BY ticker ORDER BY trade_date) THEN '▲'
        WHEN close_price < LAG(close_price) OVER (PARTITION BY ticker ORDER BY trade_date) THEN '▼'
        ELSE '→'
    END AS direction
FROM stock_prices
ORDER BY ticker, trade_date;
```

### ตัวอย่างที่ 21: Streak Detection

```sql
-- ตรวจสอบว่ายอดขายเพิ่มขึ้นกี่เดือนติดต่อกัน
WITH monthly_change AS (
    SELECT 
        year_month,
        revenue,
        LAG(revenue) OVER (ORDER BY year_month) AS prev_revenue,
        CASE 
            WHEN revenue > LAG(revenue) OVER (ORDER BY year_month) THEN 1
            ELSE 0
        END AS is_increase
    FROM monthly_sales
),
streak_groups AS (
    SELECT 
        year_month,
        revenue,
        is_increase,
        SUM(CASE WHEN is_increase = 0 THEN 1 ELSE 0 END) OVER (
            ORDER BY year_month
            ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
        ) AS streak_id
    FROM monthly_change
)
SELECT 
    year_month,
    revenue,
    is_increase,
    ROW_NUMBER() OVER (PARTITION BY streak_id ORDER BY year_month) - 
    CASE WHEN is_increase = 1 THEN 1 ELSE 0 END AS consecutive_increases
FROM streak_groups
ORDER BY year_month;
```

### ตัวอย่างที่ 22: Trend Direction

```sql
-- ระบุทิศทาง trend
SELECT 
    year_month,
    revenue,
    LAG(revenue, 1) OVER (ORDER BY year_month) AS prev_1m,
    LAG(revenue, 3) OVER (ORDER BY year_month) AS prev_3m,
    CASE 
        WHEN revenue > LAG(revenue, 1) OVER (ORDER BY year_month)
         AND LAG(revenue, 1) OVER (ORDER BY year_month) > LAG(revenue, 2) OVER (ORDER BY year_month)
        THEN 'Accelerating Growth'
        WHEN revenue > LAG(revenue, 1) OVER (ORDER BY year_month)
        THEN 'Growing'
        WHEN revenue < LAG(revenue, 1) OVER (ORDER BY year_month)
         AND LAG(revenue, 1) OVER (ORDER BY year_month) < LAG(revenue, 2) OVER (ORDER BY year_month)
        THEN 'Accelerating Decline'
        WHEN revenue < LAG(revenue, 1) OVER (ORDER BY year_month)
        THEN 'Declining'
        ELSE 'Stable'
    END AS trend
FROM monthly_sales
ORDER BY year_month;
```

### ตัวอย่างที่ 23: First and Last Purchase per Customer

```sql
-- วิเคราะห์ first/last purchase behavior
CREATE TABLE customer_purchases (
    purchase_id INT,
    customer_id INT,
    purchase_date DATE,
    amount DECIMAL(10,2),
    product_category VARCHAR(50)
);

-- Customer journey analysis
WITH customer_timeline AS (
    SELECT 
        customer_id,
        purchase_date,
        amount,
        product_category,
        FIRST_VALUE(purchase_date) OVER (
            PARTITION BY customer_id ORDER BY purchase_date
        ) AS first_purchase_date,
        LAST_VALUE(purchase_date) OVER (
            PARTITION BY customer_id ORDER BY purchase_date
            ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
        ) AS last_purchase_date,
        FIRST_VALUE(product_category) OVER (
            PARTITION BY customer_id ORDER BY purchase_date
        ) AS first_category,
        LAST_VALUE(product_category) OVER (
            PARTITION BY customer_id ORDER BY purchase_date
            ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
        ) AS last_category
    FROM customer_purchases
)
SELECT DISTINCT
    customer_id,
    first_purchase_date,
    last_purchase_date,
    first_category,
    last_category,
    last_purchase_date - first_purchase_date AS customer_lifetime_days
FROM customer_timeline
ORDER BY first_purchase_date;
```

### ตัวอย่างที่ 24: Peer Comparison

```sql
-- เปรียบเทียบแต่ละ record กับ 3 รายการก่อนหน้า
SELECT 
    year_month,
    revenue,
    -- Previous 3 months individually
    LAG(revenue, 1) OVER (ORDER BY year_month) AS lag_1m,
    LAG(revenue, 2) OVER (ORDER BY year_month) AS lag_2m,
    LAG(revenue, 3) OVER (ORDER BY year_month) AS lag_3m,
    -- Average of last 3 months
    ROUND(
        (COALESCE(LAG(revenue, 1) OVER (ORDER BY year_month), 0) +
         COALESCE(LAG(revenue, 2) OVER (ORDER BY year_month), 0) +
         COALESCE(LAG(revenue, 3) OVER (ORDER BY year_month), 0)) /
        CASE 
            WHEN LAG(revenue, 3) OVER (ORDER BY year_month) IS NOT NULL THEN 3
            WHEN LAG(revenue, 2) OVER (ORDER BY year_month) IS NOT NULL THEN 2
            WHEN LAG(revenue, 1) OVER (ORDER BY year_month) IS NOT NULL THEN 1
            ELSE 1
        END,
    2) AS avg_last_3m
FROM monthly_sales;
```

### ตัวอย่างที่ 25: Event Sequence Analysis

```sql
-- วิเคราะห์ sequence ของ events
CREATE TABLE user_actions (
    action_id   INT,
    user_id     INT,
    action_type VARCHAR(50),
    action_time TIMESTAMP
);

INSERT INTO user_actions VALUES
(1, 1001, 'login',       '2024-01-01 09:00:00'),
(2, 1001, 'view_product','2024-01-01 09:05:00'),
(3, 1001, 'add_to_cart', '2024-01-01 09:10:00'),
(4, 1001, 'checkout',    '2024-01-01 09:15:00'),
(5, 1001, 'payment',     '2024-01-01 09:20:00'),
(6, 1001, 'logout',      '2024-01-01 09:22:00'),
(7, 1002, 'login',       '2024-01-01 10:00:00'),
(8, 1002, 'view_product','2024-01-01 10:08:00'),
(9, 1002, 'logout',      '2024-01-01 10:09:00');  -- ออกโดยไม่ซื้อ

SELECT 
    user_id,
    action_id,
    action_type,
    action_time,
    LAG(action_type) OVER (PARTITION BY user_id ORDER BY action_time) AS prev_action,
    LEAD(action_type) OVER (PARTITION BY user_id ORDER BY action_time) AS next_action,
    EXTRACT(EPOCH FROM 
        action_time - LAG(action_time) OVER (PARTITION BY user_id ORDER BY action_time)
    ) / 60 AS minutes_since_prev_action
FROM user_actions
ORDER BY user_id, action_time;
```

### ตัวอย่างที่ 26: Sales Acceleration Detection

```sql
-- ตรวจหา acceleration ในยอดขาย
WITH growth_rates AS (
    SELECT 
        year_month,
        revenue,
        ROUND(
            (revenue - LAG(revenue) OVER (ORDER BY year_month)) * 100.0 /
            NULLIF(LAG(revenue) OVER (ORDER BY year_month), 0),
        2) AS growth_rate
    FROM monthly_sales
)
SELECT 
    year_month,
    revenue,
    growth_rate,
    LAG(growth_rate) OVER (ORDER BY year_month) AS prev_growth_rate,
    growth_rate - LAG(growth_rate) OVER (ORDER BY year_month) AS acceleration,
    CASE 
        WHEN growth_rate > LAG(growth_rate) OVER (ORDER BY year_month) THEN 'Accelerating'
        WHEN growth_rate < LAG(growth_rate) OVER (ORDER BY year_month) THEN 'Decelerating'
        ELSE 'Steady'
    END AS momentum
FROM growth_rates
ORDER BY year_month;
```

### ตัวอย่างที่ 27: Price Change History

```sql
-- ติดตามการเปลี่ยนแปลงราคา
CREATE TABLE price_history (
    product_id      INT,
    effective_date  DATE,
    price           DECIMAL(10,2)
);

INSERT INTO price_history VALUES
(1001, '2023-01-01', 1000),
(1001, '2023-04-01', 1050),
(1001, '2023-07-01', 980),
(1001, '2024-01-01', 1100),
(1002, '2023-01-01', 500),
(1002, '2023-06-01', 480),
(1002, '2024-01-01', 520);

SELECT 
    product_id,
    effective_date,
    price,
    LAG(price) OVER (PARTITION BY product_id ORDER BY effective_date) AS prev_price,
    price - LAG(price) OVER (PARTITION BY product_id ORDER BY effective_date) AS price_change,
    ROUND(
        (price - LAG(price) OVER (PARTITION BY product_id ORDER BY effective_date)) * 100.0 /
        NULLIF(LAG(price) OVER (PARTITION BY product_id ORDER BY effective_date), 0),
    2) AS pct_change,
    LEAD(effective_date) OVER (PARTITION BY product_id ORDER BY effective_date) AS next_change_date,
    LEAD(effective_date) OVER (PARTITION BY product_id ORDER BY effective_date) - effective_date AS days_at_this_price
FROM price_history
ORDER BY product_id, effective_date;
```

### ตัวอย่างที่ 28: Multi-metric Trend Report

```sql
-- Report หลาย metrics พร้อมกัน
SELECT 
    metric_date,
    page_views,
    sessions,
    conversions,
    ROUND(conversions * 100.0 / NULLIF(sessions, 0), 2) AS conversion_rate,
    
    -- DoD changes
    page_views - LAG(page_views) OVER (ORDER BY metric_date) AS views_dod,
    ROUND(
        (page_views - LAG(page_views) OVER (ORDER BY metric_date)) * 100.0 /
        NULLIF(LAG(page_views) OVER (ORDER BY metric_date), 0),
    1) AS views_dod_pct,
    
    -- WoW changes (7 days ago)
    page_views - LAG(page_views, 7) OVER (ORDER BY metric_date) AS views_wow,
    
    -- 7-day moving average
    ROUND(AVG(page_views) OVER (
        ORDER BY metric_date
        ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
    ), 0) AS views_7day_ma
FROM daily_metrics
ORDER BY metric_date;
```

---

## 94.8 Practical Business Scenarios

### ตัวอย่างที่ 29: Churn Prediction Signal

```sql
-- สัญญาณที่อาจนำไปสู่ churn
WITH customer_activity AS (
    SELECT 
        customer_id,
        DATE_TRUNC('month', purchase_date) AS month,
        SUM(amount) AS monthly_spend
    FROM customer_purchases
    GROUP BY customer_id, DATE_TRUNC('month', purchase_date)
),
spend_trend AS (
    SELECT 
        customer_id,
        month,
        monthly_spend,
        LAG(monthly_spend, 1) OVER (PARTITION BY customer_id ORDER BY month) AS prev_1m,
        LAG(monthly_spend, 2) OVER (PARTITION BY customer_id ORDER BY month) AS prev_2m,
        LAG(monthly_spend, 3) OVER (PARTITION BY customer_id ORDER BY month) AS prev_3m
    FROM customer_activity
)
SELECT 
    customer_id,
    month,
    monthly_spend,
    CASE 
        WHEN monthly_spend < prev_1m * 0.5 
         AND prev_1m < prev_2m * 0.5
        THEN 'HIGH CHURN RISK'
        WHEN monthly_spend < prev_1m * 0.7
        THEN 'MEDIUM CHURN RISK'
        WHEN monthly_spend IS NULL AND prev_1m IS NOT NULL
        THEN 'POSSIBLE CHURN - NO RECENT PURCHASE'
        ELSE 'NORMAL'
    END AS churn_signal
FROM spend_trend
WHERE churn_signal != 'NORMAL'
ORDER BY customer_id, month;
```

### ตัวอย่างที่ 30: Recovery After Decline

```sql
-- หาเดือนที่ยอดขาย "ฟื้นตัว" หลังจากลดลง
WITH changes AS (
    SELECT 
        year_month,
        revenue,
        LAG(revenue) OVER (ORDER BY year_month) AS prev_revenue,
        LAG(revenue, 2) OVER (ORDER BY year_month) AS prev_2m_revenue
    FROM monthly_sales
)
SELECT 
    year_month,
    revenue,
    prev_revenue,
    CASE 
        WHEN prev_revenue < prev_2m_revenue  -- เดือนก่อนลดลง
         AND revenue > prev_revenue          -- เดือนนี้เพิ่มขึ้น
        THEN 'RECOVERY'
        WHEN prev_revenue > prev_2m_revenue  -- เดือนก่อนเพิ่ม
         AND revenue < prev_revenue          -- เดือนนี้ลด
        THEN 'REVERSAL'
        ELSE NULL
    END AS inflection_point
FROM changes
WHERE year_month IS NOT NULL
ORDER BY year_month;
```

### ตัวอย่างที่ 31: Inventory Turnover Analysis

```sql
CREATE TABLE inventory_levels (
    product_id      INT,
    record_date     DATE,
    stock_quantity  INT,
    units_sold      INT
);

-- วิเคราะห์ inventory velocity
SELECT 
    product_id,
    record_date,
    stock_quantity,
    units_sold,
    -- How stock is moving
    stock_quantity - LAG(stock_quantity) OVER (
        PARTITION BY product_id ORDER BY record_date
    ) AS stock_change,
    -- Days of inventory remaining at current sell rate
    ROUND(stock_quantity * 1.0 / NULLIF(units_sold, 0), 1) AS days_remaining,
    -- Is velocity increasing or decreasing?
    units_sold - LAG(units_sold) OVER (
        PARTITION BY product_id ORDER BY record_date
    ) AS velocity_change
FROM inventory_levels
ORDER BY product_id, record_date;
```

### ตัวอย่างที่ 32: Goal Progress Tracking

```sql
CREATE TABLE sales_goals (
    salesperson VARCHAR(100),
    month       DATE,
    actual      DECIMAL(12,2),
    goal        DECIMAL(12,2)
);

INSERT INTO sales_goals VALUES
('สมชาย', '2024-01-01', 95000,  100000),
('สมชาย', '2024-02-01', 112000, 100000),
('สมชาย', '2024-03-01', 88000,  110000),
('สมชาย', '2024-04-01', 125000, 110000);

SELECT 
    salesperson,
    month,
    actual,
    goal,
    ROUND((actual / goal - 1) * 100, 2) AS vs_goal_pct,
    LAG(actual) OVER (PARTITION BY salesperson ORDER BY month) AS prev_actual,
    ROUND((actual - LAG(actual) OVER (PARTITION BY salesperson ORDER BY month)) * 100.0 /
          NULLIF(LAG(actual) OVER (PARTITION BY salesperson ORDER BY month), 0), 2) AS mom_growth,
    SUM(actual) OVER (PARTITION BY salesperson ORDER BY month) AS ytd_actual,
    SUM(goal) OVER (PARTITION BY salesperson ORDER BY month) AS ytd_goal
FROM sales_goals
ORDER BY salesperson, month;
```

---

## 94.9 การรวม Value Functions กับ CTEs

### ตัวอย่างที่ 33: Comprehensive Sales Analysis

```sql
WITH 
base_metrics AS (
    SELECT 
        year_month,
        revenue,
        orders,
        ROUND(revenue / NULLIF(orders, 0), 2) AS avg_order_value
    FROM monthly_sales
),
with_lags AS (
    SELECT 
        year_month,
        revenue,
        orders,
        avg_order_value,
        LAG(revenue)          OVER (ORDER BY year_month) AS prev_month_rev,
        LAG(revenue, 12)      OVER (ORDER BY year_month) AS same_month_prev_year,
        FIRST_VALUE(revenue)  OVER (ORDER BY year_month) AS first_month_rev
    FROM base_metrics
),
with_growth AS (
    SELECT *,
        ROUND((revenue - prev_month_rev) * 100.0 / NULLIF(prev_month_rev, 0), 2) AS mom_growth,
        ROUND((revenue - same_month_prev_year) * 100.0 / NULLIF(same_month_prev_year, 0), 2) AS yoy_growth,
        ROUND((revenue - first_month_rev) * 100.0 / first_month_rev, 2) AS growth_since_start
    FROM with_lags
)
SELECT 
    year_month,
    revenue,
    orders,
    avg_order_value,
    mom_growth,
    yoy_growth,
    growth_since_start
FROM with_growth
ORDER BY year_month;
```

### ตัวอย่างที่ 34: Moving Benchmark Comparison

```sql
-- เปรียบเทียบกับ moving benchmark (3 month avg)
WITH monthly_with_ma AS (
    SELECT 
        year_month,
        revenue,
        AVG(revenue) OVER (
            ORDER BY year_month
            ROWS BETWEEN 3 PRECEDING AND 1 PRECEDING
        ) AS prev_3m_avg
    FROM monthly_sales
)
SELECT 
    year_month,
    revenue,
    ROUND(prev_3m_avg, 2) AS benchmark_3m_avg,
    revenue - ROUND(prev_3m_avg, 2) AS vs_benchmark,
    CASE 
        WHEN revenue > prev_3m_avg * 1.1 THEN 'ABOVE TREND (+10%+)'
        WHEN revenue > prev_3m_avg THEN 'ABOVE TREND'
        WHEN revenue > prev_3m_avg * 0.9 THEN 'BELOW TREND'
        ELSE 'SIGNIFICANTLY BELOW TREND'
    END AS trend_status
FROM monthly_with_ma
WHERE prev_3m_avg IS NOT NULL
ORDER BY year_month;
```

### ตัวอย่างที่ 35: Cohort Revenue Tracking

```sql
-- ติดตาม revenue ของ customer cohort เทียบกับเดือนแรก
WITH cohort_monthly AS (
    SELECT 
        DATE_TRUNC('month', first_purchase) AS cohort_month,
        DATE_TRUNC('month', purchase_date) AS activity_month,
        SUM(amount) AS cohort_monthly_revenue
    FROM customer_purchases cp
    JOIN (
        SELECT customer_id, MIN(purchase_date) AS first_purchase
        FROM customer_purchases
        GROUP BY customer_id
    ) fp USING (customer_id)
    GROUP BY 1, 2
),
with_baseline AS (
    SELECT 
        cohort_month,
        activity_month,
        cohort_monthly_revenue,
        FIRST_VALUE(cohort_monthly_revenue) OVER (
            PARTITION BY cohort_month ORDER BY activity_month
        ) AS cohort_baseline_revenue
    FROM cohort_monthly
)
SELECT 
    cohort_month,
    activity_month,
    cohort_monthly_revenue,
    ROUND(cohort_monthly_revenue * 100.0 / cohort_baseline_revenue, 2) AS pct_of_baseline,
    EXTRACT(MONTH FROM AGE(activity_month, cohort_month)) AS months_since_acquisition
FROM with_baseline
ORDER BY cohort_month, activity_month;
```

---

## แบบฝึกหัดท้ายบท

### แบบฝึกหัดที่ 1
คำนวณ Day-over-Day และ Week-over-Week change สำหรับ page_views

**คำตอบ:**
```sql
SELECT 
    metric_date,
    page_views,
    page_views - LAG(page_views, 1) OVER (ORDER BY metric_date) AS dod_change,
    ROUND(
        (page_views - LAG(page_views, 1) OVER (ORDER BY metric_date)) * 100.0 /
        NULLIF(LAG(page_views, 1) OVER (ORDER BY metric_date), 0), 1
    ) AS dod_pct,
    page_views - LAG(page_views, 7) OVER (ORDER BY metric_date) AS wow_change,
    ROUND(
        (page_views - LAG(page_views, 7) OVER (ORDER BY metric_date)) * 100.0 /
        NULLIF(LAG(page_views, 7) OVER (ORDER BY metric_date), 0), 1
    ) AS wow_pct
FROM daily_metrics
ORDER BY metric_date;
```

### แบบฝึกหัดที่ 2
หาเดือนที่ยอดขาย "สูงสุด" และ "ต่ำสุด" สำหรับแต่ละปี โดยใช้ FIRST_VALUE และ LAST_VALUE

**คำตอบ:**
```sql
WITH monthly_with_year AS (
    SELECT 
        year_month,
        revenue,
        EXTRACT(YEAR FROM year_month) AS yr
    FROM monthly_sales
)
SELECT DISTINCT
    yr,
    FIRST_VALUE(year_month) OVER (PARTITION BY yr ORDER BY revenue DESC
        ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING) AS peak_month,
    MAX(revenue) OVER (PARTITION BY yr) AS peak_revenue,
    FIRST_VALUE(year_month) OVER (PARTITION BY yr ORDER BY revenue ASC
        ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING) AS lowest_month,
    MIN(revenue) OVER (PARTITION BY yr) AS lowest_revenue
FROM monthly_with_year
ORDER BY yr;
```

### แบบฝึกหัดที่ 3
คำนวณ YoY growth สำหรับแต่ละ product และระบุว่า growing หรือ declining

**คำตอบ:**
```sql
SELECT 
    product_id,
    month,
    revenue,
    LAG(revenue, 12) OVER (PARTITION BY product_id ORDER BY month) AS same_month_prev_year,
    ROUND(
        (revenue - LAG(revenue, 12) OVER (PARTITION BY product_id ORDER BY month)) * 100.0 /
        NULLIF(LAG(revenue, 12) OVER (PARTITION BY product_id ORDER BY month), 0),
    2) AS yoy_growth,
    CASE 
        WHEN revenue > LAG(revenue, 12) OVER (PARTITION BY product_id ORDER BY month) THEN 'GROWING'
        WHEN revenue < LAG(revenue, 12) OVER (PARTITION BY product_id ORDER BY month) THEN 'DECLINING'
        ELSE 'STABLE'
    END AS yoy_status
FROM product_monthly_sales
ORDER BY product_id, month;
```

### แบบฝึกหัดที่ 4
ใช้ LEAD เพื่อคำนวณว่าแต่ละ session ของ user ใช้เวลานานเท่าไหร่ก่อน session ถัดไป

**คำตอบ:**
```sql
SELECT 
    user_id,
    session_id,
    start_time,
    end_time,
    LEAD(start_time) OVER (PARTITION BY user_id ORDER BY start_time) AS next_session_start,
    CASE 
        WHEN LEAD(start_time) OVER (PARTITION BY user_id ORDER BY start_time) IS NOT NULL THEN
            EXTRACT(EPOCH FROM 
                LEAD(start_time) OVER (PARTITION BY user_id ORDER BY start_time) - end_time
            ) / 3600
        ELSE NULL
    END AS hours_to_next_session,
    EXTRACT(EPOCH FROM end_time - start_time) / 60 AS session_duration_minutes
FROM user_sessions
ORDER BY user_id, start_time;
```

### แบบฝึกหัดที่ 5
สร้าง comprehensive stock analysis report ด้วย LAG/LEAD

**คำตอบ:**
```sql
SELECT 
    ticker,
    trade_date,
    open_price,
    close_price,
    high_price,
    low_price,
    volume,
    close_price - open_price AS day_change,
    ROUND((close_price - open_price) * 100.0 / open_price, 2) AS intraday_pct,
    close_price - LAG(close_price) OVER (PARTITION BY ticker ORDER BY trade_date) AS vs_prev_close,
    LAG(close_price) OVER (PARTITION BY ticker ORDER BY trade_date) AS prev_close,
    LEAD(close_price) OVER (PARTITION BY ticker ORDER BY trade_date) AS next_close,
    CASE 
        WHEN close_price > LEAD(close_price) OVER (PARTITION BY ticker ORDER BY trade_date)
        THEN 'DOWN TOMORROW (predicted)'
        WHEN close_price < LEAD(close_price) OVER (PARTITION BY ticker ORDER BY trade_date)
        THEN 'UP TOMORROW (predicted)'
        ELSE 'UNKNOWN'
    END AS next_day_direction
FROM stock_prices
ORDER BY ticker, trade_date;
```

### แบบฝึกหัดที่ 6
วิเคราะห์ว่า sensor reading เพิ่มหรือลดเมื่อเทียบกับ 3 readings ที่แล้ว

**คำตอบ:**
```sql
SELECT 
    sensor_id,
    read_time,
    temperature,
    LAG(temperature, 1) OVER (PARTITION BY sensor_id ORDER BY read_time) AS prev_1,
    LAG(temperature, 2) OVER (PARTITION BY sensor_id ORDER BY read_time) AS prev_2,
    LAG(temperature, 3) OVER (PARTITION BY sensor_id ORDER BY read_time) AS prev_3,
    CASE 
        WHEN temperature > LAG(temperature, 1) OVER (PARTITION BY sensor_id ORDER BY read_time)
         AND LAG(temperature, 1) OVER (PARTITION BY sensor_id ORDER BY read_time) > 
             LAG(temperature, 2) OVER (PARTITION BY sensor_id ORDER BY read_time)
        THEN 'Consistently Rising'
        WHEN temperature < LAG(temperature, 1) OVER (PARTITION BY sensor_id ORDER BY read_time)
         AND LAG(temperature, 1) OVER (PARTITION BY sensor_id ORDER BY read_time) <
             LAG(temperature, 2) OVER (PARTITION BY sensor_id ORDER BY read_time)
        THEN 'Consistently Falling'
        ELSE 'Variable'
    END AS trend
FROM sensor_readings
WHERE temperature IS NOT NULL
ORDER BY sensor_id, read_time;
```

### แบบฝึกหัดที่ 7
คำนวณ customer's spend momentum: เปรียบเทียบ spending ใน 3 เดือนล่าสุดกับ 3 เดือนก่อนหน้า

**คำตอบ:**
```sql
WITH monthly_spend AS (
    SELECT 
        customer_id,
        DATE_TRUNC('month', purchase_date) AS month,
        SUM(amount) AS monthly_total
    FROM customer_purchases
    GROUP BY customer_id, DATE_TRUNC('month', purchase_date)
),
with_lags AS (
    SELECT 
        customer_id,
        month,
        monthly_total,
        LAG(monthly_total, 1) OVER (PARTITION BY customer_id ORDER BY month) AS m1,
        LAG(monthly_total, 2) OVER (PARTITION BY customer_id ORDER BY month) AS m2,
        LAG(monthly_total, 3) OVER (PARTITION BY customer_id ORDER BY month) AS m3,
        LAG(monthly_total, 4) OVER (PARTITION BY customer_id ORDER BY month) AS m4,
        LAG(monthly_total, 5) OVER (PARTITION BY customer_id ORDER BY month) AS m5,
        LAG(monthly_total, 6) OVER (PARTITION BY customer_id ORDER BY month) AS m6
    FROM monthly_spend
)
SELECT 
    customer_id,
    month,
    monthly_total,
    ROUND((COALESCE(m1,0) + COALESCE(m2,0) + COALESCE(monthly_total,0)) / 3, 2) AS recent_3m_avg,
    ROUND((COALESCE(m4,0) + COALESCE(m5,0) + COALESCE(m6,0)) / 3, 2) AS prev_3m_avg,
    CASE 
        WHEN (COALESCE(m1,0) + COALESCE(m2,0) + monthly_total) / 3 >
             (COALESCE(m4,0) + COALESCE(m5,0) + COALESCE(m6,0)) / 3 * 1.1 
        THEN 'ACCELERATING'
        WHEN (COALESCE(m1,0) + COALESCE(m2,0) + monthly_total) / 3 <
             (COALESCE(m4,0) + COALESCE(m5,0) + COALESCE(m6,0)) / 3 * 0.9 
        THEN 'DECELERATING'
        ELSE 'STABLE'
    END AS spend_momentum
FROM with_lags
WHERE m6 IS NOT NULL
ORDER BY customer_id, month;
```

### แบบฝึกหัดที่ 8
ใช้ NTH_VALUE เพื่อหาค่าสูงสุดอันดับ 1, 2, 3 ของเงินเดือนในแต่ละแผนก

**คำตอบ:**
```sql
SELECT DISTINCT
    department,
    NTH_VALUE(salary, 1) OVER (
        PARTITION BY department ORDER BY salary DESC
        ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
    ) AS top_1_salary,
    NTH_VALUE(salary, 2) OVER (
        PARTITION BY department ORDER BY salary DESC
        ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
    ) AS top_2_salary,
    NTH_VALUE(salary, 3) OVER (
        PARTITION BY department ORDER BY salary DESC
        ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
    ) AS top_3_salary
FROM employees
ORDER BY department;
```

### แบบฝึกหัดที่ 9
สร้าง price volatility report โดยดูว่า price change รายวันใหญ่แค่ไหนเทียบกับ 5 วันก่อน

**คำตอบ:**
```sql
WITH daily_changes AS (
    SELECT 
        ticker,
        trade_date,
        close_price,
        ABS(close_price - LAG(close_price) OVER (PARTITION BY ticker ORDER BY trade_date)) AS daily_abs_change
    FROM stock_prices
)
SELECT 
    ticker,
    trade_date,
    close_price,
    daily_abs_change,
    AVG(daily_abs_change) OVER (
        PARTITION BY ticker ORDER BY trade_date
        ROWS BETWEEN 5 PRECEDING AND 1 PRECEDING
    ) AS avg_5d_volatility,
    CASE 
        WHEN daily_abs_change > AVG(daily_abs_change) OVER (
            PARTITION BY ticker ORDER BY trade_date
            ROWS BETWEEN 5 PRECEDING AND 1 PRECEDING
        ) * 2 THEN 'HIGH VOLATILITY'
        ELSE 'NORMAL'
    END AS volatility_flag
FROM daily_changes
WHERE daily_abs_change IS NOT NULL
ORDER BY ticker, trade_date;
```

### แบบฝึกหัดที่ 10
วิเคราะห์ user churn signal โดยดูว่า session frequency ลดลงเรื่อยๆ หรือไม่

**คำตอบ:**
```sql
WITH session_gaps AS (
    SELECT 
        user_id,
        start_time,
        LEAD(start_time) OVER (PARTITION BY user_id ORDER BY start_time) AS next_session,
        EXTRACT(EPOCH FROM 
            LEAD(start_time) OVER (PARTITION BY user_id ORDER BY start_time) - start_time
        ) / 86400 AS days_to_next_session
    FROM user_sessions
),
gap_analysis AS (
    SELECT 
        user_id,
        start_time,
        days_to_next_session,
        LAG(days_to_next_session) OVER (PARTITION BY user_id ORDER BY start_time) AS prev_gap,
        LAG(days_to_next_session, 2) OVER (PARTITION BY user_id ORDER BY start_time) AS prev_prev_gap
    FROM session_gaps
    WHERE days_to_next_session IS NOT NULL
)
SELECT 
    user_id,
    start_time,
    days_to_next_session,
    prev_gap,
    CASE 
        WHEN days_to_next_session > prev_gap 
         AND prev_gap > prev_prev_gap THEN 'CHURN RISK - Increasing Gaps'
        WHEN days_to_next_session > prev_gap THEN 'WATCH - Gap Increasing'
        ELSE 'OK'
    END AS churn_signal
FROM gap_analysis
WHERE prev_prev_gap IS NOT NULL
ORDER BY user_id, start_time;
```

---

## สรุปบทที่ 94

Value Functions ช่วยให้ SQL วิเคราะห์ time series ได้อย่างทรงพลัง:

| Function | ใช้สำหรับ |
|----------|-----------|
| LAG(n)   | เปรียบเทียบกับ n periods ก่อนหน้า |
| LEAD(n)  | มองไปข้างหน้า n periods |
| FIRST_VALUE | เปรียบเทียบกับค่าแรกสุดใน window |
| LAST_VALUE | เปรียบเทียบกับค่าสุดท้ายใน window |
| NTH_VALUE | ดึงค่าที่ n ใน window |

**ข้อควรจำ:**
- LAST_VALUE ต้องระบุ `ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING` เสมอ
- IGNORE NULLS รองรับใน Oracle, SQL Server 2022+, Snowflake แต่ไม่ใช่ standard PostgreSQL
- LAG/LEAD พร้อม default value ป้องกัน NULL ได้

ในบทถัดไปเราจะเรียน Aggregate Window Functions ซึ่งใช้สำหรับ running totals, moving averages, และ frame specifications
