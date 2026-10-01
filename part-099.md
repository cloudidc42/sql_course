# ส่วนที่ 99: Analytical Functions และ OLAP Operations

## บทนำ

OLAP (Online Analytical Processing) คือเทคนิคการวิเคราะห์ข้อมูลหลายมิติ SQL มีฟังก์ชัน OLAP ที่ช่วยสร้าง reports, pivot tables, และการวิเคราะห์ขั้นสูง ได้แก่:
- **ROLLUP / CUBE / GROUPING SETS** — subtotals และ grand totals
- **PIVOT / UNPIVOT** — เปลี่ยน rows เป็น columns และกลับกัน
- **Percentile functions** — PERCENTILE_CONT, PERCENTILE_DISC
- **Statistical functions** — regression, correlation
- **Time-series analysis** — trend detection

---

## 99.1 ROLLUP - Hierarchical Subtotals

### ตัวอย่างที่ 1: ROLLUP พื้นฐาน

```sql
-- สร้างข้อมูลสำหรับตัวอย่าง
CREATE TABLE sales (
    sale_id    SERIAL PRIMARY KEY,
    year       INT,
    quarter    VARCHAR(2),
    month      INT,
    region     VARCHAR(50),
    product    VARCHAR(100),
    category   VARCHAR(50),
    amount     DECIMAL(12,2),
    quantity   INT
);

INSERT INTO sales (year, quarter, month, region, product, category, amount, quantity) VALUES
(2023, 'Q1', 1, 'North', 'Laptop Pro', 'Electronics', 45000, 3),
(2023, 'Q1', 1, 'North', 'Phone X', 'Electronics', 25000, 5),
(2023, 'Q1', 2, 'South', 'Laptop Pro', 'Electronics', 30000, 2),
(2023, 'Q1', 2, 'South', 'Desk Chair', 'Furniture', 12000, 4),
(2023, 'Q2', 4, 'North', 'Phone X', 'Electronics', 35000, 7),
(2023, 'Q2', 5, 'East', 'Laptop Pro', 'Electronics', 60000, 4),
(2023, 'Q2', 6, 'West', 'Desk Chair', 'Furniture', 18000, 6),
(2023, 'Q3', 7, 'North', 'Laptop Pro', 'Electronics', 50000, 3),
(2023, 'Q3', 8, 'South', 'Phone X', 'Electronics', 40000, 8),
(2023, 'Q4', 10, 'East', 'Desk Chair', 'Furniture', 22000, 7),
(2024, 'Q1', 1, 'North', 'Laptop Pro', 'Electronics', 55000, 4),
(2024, 'Q1', 2, 'South', 'Phone X', 'Electronics', 28000, 6),
(2024, 'Q2', 4, 'East', 'Laptop Pro', 'Electronics', 65000, 5),
(2024, 'Q3', 7, 'West', 'Desk Chair', 'Furniture', 20000, 7);

-- ROLLUP: สร้าง subtotals จาก specific → general
SELECT 
    COALESCE(year::TEXT, 'Grand Total') AS year,
    COALESCE(quarter, 'All Quarters') AS quarter,
    SUM(amount) AS total_amount,
    GROUPING(year) AS is_year_total,
    GROUPING(quarter) AS is_quarter_total
FROM sales
GROUP BY ROLLUP(year, quarter)
ORDER BY year NULLS LAST, quarter NULLS LAST;
```

### ตัวอย่างที่ 2: ROLLUP หลายระดับ

```sql
-- ROLLUP 3 ระดับ: region > category > product
SELECT 
    COALESCE(region, 'ALL REGIONS') AS region,
    COALESCE(category, 'ALL CATEGORIES') AS category,
    COALESCE(product, 'ALL PRODUCTS') AS product,
    SUM(amount) AS total_sales,
    SUM(quantity) AS total_qty,
    GROUPING(region, category, product) AS grouping_level
FROM sales
GROUP BY ROLLUP(region, category, product)
ORDER BY 
    GROUPING(region) DESC,
    region NULLS LAST,
    GROUPING(category) DESC,
    category NULLS LAST,
    GROUPING(product) DESC;
```

---

## 99.2 CUBE - All Combinations

### ตัวอย่างที่ 3: CUBE พื้นฐาน

```sql
-- CUBE: สร้าง subtotals ทุก combination
SELECT 
    COALESCE(year::TEXT, 'All Years') AS year,
    COALESCE(region, 'All Regions') AS region,
    COALESCE(category, 'All Categories') AS category,
    SUM(amount) AS total_amount,
    COUNT(*) AS transactions
FROM sales
GROUP BY CUBE(year, region, category)
ORDER BY 
    year NULLS LAST, 
    region NULLS LAST, 
    category NULLS LAST;
```

### ตัวอย่างที่ 4: GROUPING() function

```sql
-- GROUPING() ระบุว่า column อยู่ใน grouping หรือเปล่า
SELECT 
    CASE WHEN GROUPING(year) = 1 THEN 'All Years' ELSE year::TEXT END AS year,
    CASE WHEN GROUPING(region) = 1 THEN 'All Regions' ELSE region END AS region,
    SUM(amount) AS total,
    CASE 
        WHEN GROUPING(year) = 0 AND GROUPING(region) = 0 THEN 'Detail'
        WHEN GROUPING(year) = 0 AND GROUPING(region) = 1 THEN 'Year Subtotal'
        WHEN GROUPING(year) = 1 AND GROUPING(region) = 0 THEN 'Region Subtotal'
        ELSE 'Grand Total'
    END AS row_type
FROM sales
GROUP BY CUBE(year, region)
ORDER BY year NULLS LAST, region NULLS LAST;
```

---

## 99.3 GROUPING SETS - Custom Combinations

### ตัวอย่างที่ 5: GROUPING SETS

```sql
-- GROUPING SETS: ระบุ grouping combinations เอง
SELECT 
    year,
    quarter,
    region,
    SUM(amount) AS total
FROM sales
GROUP BY GROUPING SETS (
    (year, quarter),    -- by year-quarter
    (year, region),     -- by year-region
    (region),           -- by region only
    ()                  -- grand total
)
ORDER BY year NULLS LAST, quarter NULLS LAST, region NULLS LAST;
```

### ตัวอย่างที่ 6: GROUPING SETS vs ROLLUP vs CUBE

```sql
-- เหมือนกับ ROLLUP(a, b, c)
GROUP BY GROUPING SETS ((a,b,c), (a,b), (a), ())

-- เหมือนกับ CUBE(a, b)
GROUP BY GROUPING SETS ((a,b), (a), (b), ())

-- GROUPING SETS: เลือกเฉพาะที่ต้องการ
SELECT year, region, category, SUM(amount) AS total
FROM sales
GROUP BY GROUPING SETS (
    (year, region),
    (year, category),
    (year)
);
```

---

## 99.4 PIVOT - Rows to Columns

### ตัวอย่างที่ 7: Manual PIVOT ด้วย CASE WHEN

```sql
-- PIVOT: เปลี่ยน rows เป็น columns
-- รายงาน sales ตามปีและ quarter

SELECT 
    year,
    SUM(CASE WHEN quarter = 'Q1' THEN amount ELSE 0 END) AS q1_sales,
    SUM(CASE WHEN quarter = 'Q2' THEN amount ELSE 0 END) AS q2_sales,
    SUM(CASE WHEN quarter = 'Q3' THEN amount ELSE 0 END) AS q3_sales,
    SUM(CASE WHEN quarter = 'Q4' THEN amount ELSE 0 END) AS q4_sales,
    SUM(amount) AS total_sales
FROM sales
GROUP BY year
ORDER BY year;
```

### ตัวอย่างที่ 8: PIVOT แบบซับซ้อน

```sql
-- PIVOT: sales by region and category
SELECT 
    region,
    SUM(CASE WHEN category = 'Electronics' THEN amount ELSE 0 END) AS electronics,
    SUM(CASE WHEN category = 'Furniture' THEN amount ELSE 0 END) AS furniture,
    SUM(CASE WHEN category = 'Software' THEN amount ELSE 0 END) AS software,
    SUM(amount) AS total,
    ROUND(100.0 * SUM(CASE WHEN category = 'Electronics' THEN amount ELSE 0 END) / SUM(amount), 1) AS electronics_pct
FROM sales
GROUP BY region
ORDER BY total DESC;
```

### ตัวอย่างที่ 9: SQL Server PIVOT Syntax

```sql
-- SQL Server Native PIVOT
SELECT year, [Q1], [Q2], [Q3], [Q4]
FROM (
    SELECT year, quarter, amount
    FROM sales
) AS source_data
PIVOT (
    SUM(amount)
    FOR quarter IN ([Q1], [Q2], [Q3], [Q4])
) AS pivot_table
ORDER BY year;
```

### ตัวอย่างที่ 10: PostgreSQL Crosstab (tablefunc)

```sql
-- PostgreSQL crosstab extension
CREATE EXTENSION IF NOT EXISTS tablefunc;

-- Crosstab: ต้องระบุ column headers ล่วงหน้า
SELECT *
FROM crosstab(
    -- source query (sorted by row, category)
    'SELECT year::TEXT, quarter, SUM(amount)::NUMERIC
     FROM sales
     GROUP BY year, quarter
     ORDER BY year, quarter',
    -- category values
    'SELECT DISTINCT quarter FROM sales ORDER BY quarter'
) AS ct(year TEXT, "Q1" NUMERIC, "Q2" NUMERIC, "Q3" NUMERIC, "Q4" NUMERIC);
```

---

## 99.5 UNPIVOT - Columns to Rows

### ตัวอย่างที่ 11: Manual UNPIVOT ด้วย UNION ALL

```sql
-- สร้างตาราง wide format
CREATE TABLE quarterly_budget (
    year    INT,
    region  VARCHAR(50),
    q1_budget DECIMAL(12,2),
    q2_budget DECIMAL(12,2),
    q3_budget DECIMAL(12,2),
    q4_budget DECIMAL(12,2)
);

INSERT INTO quarterly_budget VALUES
(2024, 'North', 100000, 110000, 120000, 130000),
(2024, 'South', 80000, 85000, 90000, 95000),
(2024, 'East', 75000, 80000, 85000, 90000);

-- UNPIVOT: เปลี่ยน columns เป็น rows
SELECT year, region, 'Q1' AS quarter, q1_budget AS budget FROM quarterly_budget
UNION ALL
SELECT year, region, 'Q2', q2_budget FROM quarterly_budget
UNION ALL
SELECT year, region, 'Q3', q3_budget FROM quarterly_budget
UNION ALL
SELECT year, region, 'Q4', q4_budget FROM quarterly_budget
ORDER BY year, region, quarter;
```

### ตัวอย่างที่ 12: SQL Server UNPIVOT Syntax

```sql
-- SQL Server Native UNPIVOT
SELECT year, region, quarter, budget
FROM quarterly_budget
UNPIVOT (
    budget FOR quarter IN (q1_budget, q2_budget, q3_budget, q4_budget)
) AS unpvt;
```

---

## 99.6 PERCENTILE Functions

### ตัวอย่างที่ 13: PERCENTILE_CONT และ PERCENTILE_DISC

```sql
-- PERCENTILE_CONT: interpolated percentile (continuous)
-- PERCENTILE_DISC: actual value at percentile (discrete)

SELECT 
    category,
    PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY amount) AS median_cont,
    PERCENTILE_DISC(0.5) WITHIN GROUP (ORDER BY amount) AS median_disc,
    PERCENTILE_CONT(0.25) WITHIN GROUP (ORDER BY amount) AS q1,
    PERCENTILE_CONT(0.75) WITHIN GROUP (ORDER BY amount) AS q3,
    PERCENTILE_CONT(0.9) WITHIN GROUP (ORDER BY amount) AS p90,
    MIN(amount) AS min_val,
    MAX(amount) AS max_val,
    AVG(amount) AS mean_val
FROM sales
GROUP BY category
ORDER BY category;
```

### ตัวอย่างที่ 14: Percentile เป็น Window Function

```sql
-- PERCENT_RANK() และ CUME_DIST() เป็น window functions
SELECT 
    sale_id,
    region,
    amount,
    PERCENT_RANK() OVER (PARTITION BY region ORDER BY amount) AS percent_rank,
    CUME_DIST() OVER (PARTITION BY region ORDER BY amount) AS cume_dist,
    NTILE(4) OVER (PARTITION BY region ORDER BY amount) AS quartile
FROM sales
ORDER BY region, amount;
```

### ตัวอย่างที่ 15: Mode (Most Frequent Value)

```sql
-- MODE: ค่าที่ปรากฏบ่อยที่สุด (PostgreSQL 9.4+)
SELECT 
    region,
    MODE() WITHIN GROUP (ORDER BY product) AS most_common_product,
    MODE() WITHIN GROUP (ORDER BY quarter) AS peak_quarter
FROM sales
GROUP BY region;
```

---

## 99.7 Statistical Functions

### ตัวอย่างที่ 16: Correlation

```sql
-- CORR(): Pearson correlation coefficient
-- -1 = perfect negative, 0 = no correlation, 1 = perfect positive
SELECT 
    CORR(amount, quantity) AS amount_qty_correlation,
    REGR_R2(amount, quantity) AS r_squared
FROM sales;

-- Correlation by category
SELECT 
    category,
    CORR(amount, quantity) AS correlation,
    COUNT(*) AS sample_size
FROM sales
GROUP BY category
HAVING COUNT(*) >= 3;
```

### ตัวอย่างที่ 17: Linear Regression

```sql
-- Regression functions (PostgreSQL/SQL Standard)
SELECT 
    REGR_SLOPE(amount, quantity) AS slope,       -- y = slope*x + intercept
    REGR_INTERCEPT(amount, quantity) AS intercept,
    REGR_R2(amount, quantity) AS r_squared,       -- goodness of fit
    REGR_COUNT(amount, quantity) AS n,
    REGR_AVGX(amount, quantity) AS avg_x,
    REGR_AVGY(amount, quantity) AS avg_y,
    REGR_SXX(amount, quantity) AS sxx,
    REGR_SYY(amount, quantity) AS syy,
    REGR_SXY(amount, quantity) AS sxy
FROM sales
WHERE category = 'Electronics';
```

### ตัวอย่างที่ 18: Regression Prediction

```sql
-- ใช้ regression equation สำหรับ prediction
WITH regression AS (
    SELECT 
        REGR_SLOPE(amount, quantity) AS slope,
        REGR_INTERCEPT(amount, quantity) AS intercept
    FROM sales WHERE year = 2023
)
SELECT 
    s.sale_id,
    s.amount AS actual_amount,
    s.quantity,
    ROUND(r.slope * s.quantity + r.intercept, 2) AS predicted_amount,
    ROUND(s.amount - (r.slope * s.quantity + r.intercept), 2) AS residual
FROM sales s, regression r
WHERE s.year = 2024
ORDER BY ABS(s.amount - (r.slope * s.quantity + r.intercept)) DESC;
```

### ตัวอย่างที่ 19: Variance และ Standard Deviation

```sql
-- Statistical dispersion measures
SELECT 
    category,
    AVG(amount) AS mean,
    STDDEV(amount) AS std_dev,         -- sample std deviation
    STDDEV_POP(amount) AS std_dev_pop, -- population std deviation
    VARIANCE(amount) AS variance,
    VAR_POP(amount) AS var_pop,
    -- Coefficient of variation
    ROUND(STDDEV(amount) / NULLIF(AVG(amount), 0) * 100, 2) AS cv_pct
FROM sales
GROUP BY category;
```

---

## 99.8 Time-Series Analysis

### ตัวอย่างที่ 20: Trend Detection

```sql
-- Linear trend ใน time-series
WITH monthly_sales AS (
    SELECT 
        year,
        month,
        SUM(amount) AS monthly_total,
        ROW_NUMBER() OVER (ORDER BY year, month) AS period_num
    FROM sales
    GROUP BY year, month
)
SELECT 
    year,
    month,
    monthly_total,
    period_num,
    -- Trend line
    ROUND(
        REGR_SLOPE(monthly_total, period_num) OVER () * period_num +
        REGR_INTERCEPT(monthly_total, period_num) OVER (),
        2
    ) AS trend_value,
    -- Deviation from trend
    ROUND(
        monthly_total - (
            REGR_SLOPE(monthly_total, period_num) OVER () * period_num +
            REGR_INTERCEPT(monthly_total, period_num) OVER ()
        ),
        2
    ) AS deviation_from_trend
FROM monthly_sales
ORDER BY year, month;
```

### ตัวอย่างที่ 21: Year-over-Year Analysis

```sql
-- YoY comparison
WITH annual_summary AS (
    SELECT 
        year,
        quarter,
        SUM(amount) AS quarterly_sales
    FROM sales
    GROUP BY year, quarter
)
SELECT 
    curr.year,
    curr.quarter,
    curr.quarterly_sales,
    prev.quarterly_sales AS prev_year_sales,
    curr.quarterly_sales - prev.quarterly_sales AS yoy_change,
    ROUND(
        100.0 * (curr.quarterly_sales - prev.quarterly_sales) / 
        NULLIF(prev.quarterly_sales, 0),
        2
    ) AS yoy_pct_change
FROM annual_summary curr
LEFT JOIN annual_summary prev 
    ON curr.year = prev.year + 1 
    AND curr.quarter = prev.quarter
ORDER BY curr.year, curr.quarter;
```

### ตัวอย่างที่ 22: Seasonality Detection

```sql
-- Seasonality: เปรียบเทียบ quarter กับ avg ของทุก year
WITH quarterly_avg AS (
    SELECT 
        quarter,
        AVG(SUM(amount)) OVER (PARTITION BY quarter) AS avg_by_quarter,
        AVG(SUM(amount)) OVER () AS overall_avg
    FROM sales
    GROUP BY year, quarter
)
SELECT DISTINCT
    quarter,
    ROUND(avg_by_quarter, 2) AS avg_quarterly_sales,
    ROUND(overall_avg, 2) AS overall_avg,
    ROUND(100.0 * avg_by_quarter / NULLIF(overall_avg, 0), 2) AS seasonality_index
FROM quarterly_avg
ORDER BY quarter;
```

---

## 99.9 Advanced Analytics Patterns

### ตัวอย่างที่ 23: ABC Analysis (Pareto)

```sql
-- ABC Analysis: แบ่ง products ตามยอดขาย
-- A: top 20% products = 80% revenue
-- B: next 30% products = 15% revenue  
-- C: bottom 50% products = 5% revenue

WITH product_sales AS (
    SELECT 
        product,
        SUM(amount) AS total_sales
    FROM sales
    GROUP BY product
),
ranked AS (
    SELECT 
        product,
        total_sales,
        SUM(total_sales) OVER () AS grand_total,
        SUM(total_sales) OVER (ORDER BY total_sales DESC) AS running_total,
        SUM(total_sales) OVER (ORDER BY total_sales DESC) / 
            SUM(total_sales) OVER () AS cumulative_pct
    FROM product_sales
)
SELECT 
    product,
    total_sales,
    ROUND(100.0 * total_sales / grand_total, 2) AS revenue_pct,
    ROUND(100.0 * cumulative_pct, 2) AS cumulative_pct,
    CASE 
        WHEN cumulative_pct <= 0.8 THEN 'A'
        WHEN cumulative_pct <= 0.95 THEN 'B'
        ELSE 'C'
    END AS abc_class
FROM ranked
ORDER BY total_sales DESC;
```

### ตัวอย่างที่ 24: Market Basket Analysis (Item Associations)

```sql
-- สร้างข้อมูล orders
CREATE TABLE order_items (
    order_id   INT,
    product    VARCHAR(100)
);

INSERT INTO order_items VALUES
(1, 'Laptop'), (1, 'Mouse'), (1, 'Keyboard'),
(2, 'Phone'), (2, 'Case'), (2, 'Charger'),
(3, 'Laptop'), (3, 'Mouse'),
(4, 'Laptop'), (4, 'Keyboard'), (4, 'Monitor'),
(5, 'Phone'), (5, 'Charger'),
(6, 'Laptop'), (6, 'Mouse'), (6, 'Monitor');

-- Association rules: which products appear together
WITH order_count AS (
    SELECT COUNT(DISTINCT order_id) AS total_orders FROM order_items
),
item_support AS (
    SELECT product, COUNT(DISTINCT order_id) AS item_count
    FROM order_items
    GROUP BY product
),
item_pairs AS (
    SELECT 
        a.product AS item_a,
        b.product AS item_b,
        COUNT(DISTINCT a.order_id) AS pair_count
    FROM order_items a
    JOIN order_items b ON a.order_id = b.order_id AND a.product < b.product
    GROUP BY a.product, b.product
)
SELECT 
    p.item_a,
    p.item_b,
    p.pair_count,
    ROUND(100.0 * p.pair_count / oc.total_orders, 1) AS support_pct,
    ROUND(100.0 * p.pair_count / sa.item_count, 1) AS confidence_a_to_b_pct,
    ROUND(100.0 * p.pair_count / sb.item_count, 1) AS confidence_b_to_a_pct
FROM item_pairs p
JOIN item_support sa ON p.item_a = sa.product
JOIN item_support sb ON p.item_b = sb.product
CROSS JOIN order_count oc
WHERE p.pair_count >= 2
ORDER BY p.pair_count DESC;
```

### ตัวอย่างที่ 25: Funnel Analysis

```sql
-- สร้าง funnel data
CREATE TABLE user_events (
    user_id    INT,
    event_type VARCHAR(50),
    event_time TIMESTAMP
);

INSERT INTO user_events VALUES
(1, 'view_product', '2024-01-01 10:00'),
(1, 'add_to_cart', '2024-01-01 10:05'),
(1, 'checkout', '2024-01-01 10:10'),
(1, 'purchase', '2024-01-01 10:15'),
(2, 'view_product', '2024-01-01 11:00'),
(2, 'add_to_cart', '2024-01-01 11:10'),
(3, 'view_product', '2024-01-01 12:00'),
(3, 'add_to_cart', '2024-01-01 12:05'),
(3, 'checkout', '2024-01-01 12:20'),
(4, 'view_product', '2024-01-01 13:00'),
(5, 'view_product', '2024-01-01 14:00'),
(5, 'add_to_cart', '2024-01-01 14:10'),
(5, 'checkout', '2024-01-01 14:20'),
(5, 'purchase', '2024-01-01 14:25');

-- Funnel analysis
WITH funnel_steps AS (
    SELECT
        COUNT(DISTINCT CASE WHEN event_type = 'view_product' THEN user_id END) AS step1_view,
        COUNT(DISTINCT CASE WHEN event_type = 'add_to_cart' THEN user_id END) AS step2_cart,
        COUNT(DISTINCT CASE WHEN event_type = 'checkout' THEN user_id END) AS step3_checkout,
        COUNT(DISTINCT CASE WHEN event_type = 'purchase' THEN user_id END) AS step4_purchase
    FROM user_events
)
SELECT 
    'View Product' AS step, step1_view AS users,
    100.0 AS conversion_pct, 
    NULL AS drop_off_pct
FROM funnel_steps
UNION ALL
SELECT 'Add to Cart', step2_cart,
    ROUND(100.0 * step2_cart / NULLIF(step1_view, 0), 1),
    ROUND(100.0 * (step1_view - step2_cart) / NULLIF(step1_view, 0), 1)
FROM funnel_steps
UNION ALL
SELECT 'Checkout', step3_checkout,
    ROUND(100.0 * step3_checkout / NULLIF(step1_view, 0), 1),
    ROUND(100.0 * (step2_cart - step3_checkout) / NULLIF(step2_cart, 0), 1)
FROM funnel_steps
UNION ALL
SELECT 'Purchase', step4_purchase,
    ROUND(100.0 * step4_purchase / NULLIF(step1_view, 0), 1),
    ROUND(100.0 * (step3_checkout - step4_purchase) / NULLIF(step3_checkout, 0), 1)
FROM funnel_steps;
```

### ตัวอย่างที่ 26: Cohort Retention

```sql
-- Cohort retention analysis
CREATE TABLE user_activity (
    user_id         INT,
    activity_date   DATE
);

INSERT INTO user_activity VALUES
(1, '2024-01-05'), (1, '2024-02-10'), (1, '2024-03-15'),
(2, '2024-01-10'), (2, '2024-02-20'),
(3, '2024-01-15'), (3, '2024-03-20'), (3, '2024-04-25'),
(4, '2024-02-01'), (4, '2024-03-05'), (4, '2024-04-10'),
(5, '2024-02-15'), (5, '2024-03-20'),
(6, '2024-03-01'), (6, '2024-04-05');

WITH user_cohorts AS (
    SELECT 
        user_id,
        DATE_TRUNC('month', MIN(activity_date)) AS cohort_month
    FROM user_activity
    GROUP BY user_id
),
activity_months AS (
    SELECT 
        ua.user_id,
        uc.cohort_month,
        DATE_TRUNC('month', ua.activity_date) AS activity_month,
        EXTRACT(MONTH FROM AGE(
            DATE_TRUNC('month', ua.activity_date),
            uc.cohort_month
        )) AS months_since_cohort
    FROM user_activity ua
    JOIN user_cohorts uc ON ua.user_id = uc.user_id
)
SELECT 
    cohort_month,
    COUNT(DISTINCT CASE WHEN months_since_cohort = 0 THEN user_id END) AS month_0,
    COUNT(DISTINCT CASE WHEN months_since_cohort = 1 THEN user_id END) AS month_1,
    COUNT(DISTINCT CASE WHEN months_since_cohort = 2 THEN user_id END) AS month_2,
    COUNT(DISTINCT CASE WHEN months_since_cohort = 3 THEN user_id END) AS month_3,
    ROUND(100.0 * COUNT(DISTINCT CASE WHEN months_since_cohort = 1 THEN user_id END) /
        NULLIF(COUNT(DISTINCT CASE WHEN months_since_cohort = 0 THEN user_id END), 0), 0) AS m1_retention_pct,
    ROUND(100.0 * COUNT(DISTINCT CASE WHEN months_since_cohort = 2 THEN user_id END) /
        NULLIF(COUNT(DISTINCT CASE WHEN months_since_cohort = 0 THEN user_id END), 0), 0) AS m2_retention_pct
FROM activity_months
GROUP BY cohort_month
ORDER BY cohort_month;
```

### ตัวอย่างที่ 27: Dynamic PIVOT ด้วย plpgsql

```sql
-- Dynamic PIVOT: สร้าง query อัตโนมัติตาม distinct values
CREATE OR REPLACE FUNCTION dynamic_pivot(
    p_table TEXT,
    p_row_col TEXT,
    p_pivot_col TEXT,
    p_value_col TEXT,
    p_agg TEXT DEFAULT 'SUM'
)
RETURNS TEXT AS $$
DECLARE
    v_cols TEXT;
    v_query TEXT;
BEGIN
    -- หา distinct values สำหรับ pivot columns
    EXECUTE format(
        'SELECT string_agg(DISTINCT quote_literal(%I) || '' AS '' || quote_ident(%I::TEXT), '', ''),
         FROM %I',
        p_pivot_col, p_pivot_col, p_table
    ) INTO v_cols;
    
    -- สร้าง CASE WHEN สำหรับแต่ละ pivot value
    EXECUTE format(
        'SELECT string_agg(
            %L || ''(CASE WHEN %I = '' || quote_literal(val) || '' THEN %I END) AS '' || quote_ident(val::TEXT),
            '', ''
         )
         FROM (SELECT DISTINCT %I AS val FROM %I ORDER BY %I) t',
        p_agg, p_pivot_col, p_value_col,
        p_pivot_col, p_table, p_pivot_col
    ) INTO v_cols;
    
    v_query = format(
        'SELECT %I, %s FROM %I GROUP BY %I ORDER BY %I',
        p_row_col, v_cols, p_table, p_row_col, p_row_col
    );
    
    RETURN v_query;
END;
$$ LANGUAGE plpgsql;

-- ตรวจสอบ query ที่สร้าง
SELECT dynamic_pivot('sales', 'year', 'quarter', 'amount');
```

### ตัวอย่างที่ 28: Rolling Statistics

```sql
-- Rolling statistics สำหรับ time-series
WITH daily_sales AS (
    SELECT 
        year * 100 + month AS period,
        SUM(amount) AS daily_total
    FROM sales
    GROUP BY year, month
    ORDER BY year, month
),
with_stats AS (
    SELECT 
        period,
        daily_total,
        AVG(daily_total) OVER (
            ORDER BY period 
            ROWS BETWEEN 2 PRECEDING AND CURRENT ROW
        ) AS ma_3,
        STDDEV(daily_total) OVER (
            ORDER BY period 
            ROWS BETWEEN 2 PRECEDING AND CURRENT ROW
        ) AS rolling_stddev,
        MIN(daily_total) OVER (
            ORDER BY period 
            ROWS BETWEEN 2 PRECEDING AND CURRENT ROW
        ) AS rolling_min,
        MAX(daily_total) OVER (
            ORDER BY period 
            ROWS BETWEEN 2 PRECEDING AND CURRENT ROW
        ) AS rolling_max
    FROM daily_sales
)
SELECT 
    period,
    daily_total,
    ROUND(ma_3, 0) AS moving_avg_3,
    ROUND(rolling_stddev, 0) AS rolling_std,
    -- Bollinger-like bands
    ROUND(ma_3 - 2 * rolling_stddev, 0) AS lower_band,
    ROUND(ma_3 + 2 * rolling_stddev, 0) AS upper_band,
    CASE 
        WHEN daily_total > ma_3 + 2 * rolling_stddev THEN 'HIGH_ANOMALY'
        WHEN daily_total < ma_3 - 2 * rolling_stddev THEN 'LOW_ANOMALY'
        ELSE 'NORMAL'
    END AS signal
FROM with_stats;
```

### ตัวอย่างที่ 29: Hypothesis Testing in SQL

```sql
-- T-test approximation: เปรียบเทียบ sales ระหว่าง 2 groups
WITH group_stats AS (
    SELECT 
        region,
        AVG(amount) AS mean_val,
        STDDEV(amount) AS std_val,
        COUNT(*) AS n
    FROM sales
    WHERE region IN ('North', 'South')
    GROUP BY region
),
north AS (SELECT mean_val, std_val, n FROM group_stats WHERE region = 'North'),
south AS (SELECT mean_val, std_val, n FROM group_stats WHERE region = 'South')
SELECT 
    n.mean_val AS north_mean,
    s.mean_val AS south_mean,
    n.mean_val - s.mean_val AS mean_difference,
    n.std_val AS north_std,
    s.std_val AS south_std,
    -- Pooled standard error
    SQRT((n.std_val^2 / n.n) + (s.std_val^2 / s.n)) AS std_error,
    -- t-statistic
    ROUND(
        (n.mean_val - s.mean_val) / 
        NULLIF(SQRT((n.std_val^2 / n.n) + (s.std_val^2 / s.n)), 0),
        4
    ) AS t_statistic
    -- NOTE: ต้องใช้ t-distribution table แปลงเป็น p-value
FROM north n, south s;
```

### ตัวอย่างที่ 30: Weighted Moving Average

```sql
-- Weighted moving average: ข้อมูลล่าสุดมีน้ำหนักมากกว่า
WITH monthly_data AS (
    SELECT 
        year,
        month,
        SUM(amount) AS monthly_sales,
        ROW_NUMBER() OVER (ORDER BY year, month) AS rn
    FROM sales
    GROUP BY year, month
)
SELECT 
    year,
    month,
    monthly_sales,
    -- Exponential weighted: weight 3, 2, 1 for last 3 periods
    ROUND(
        (3 * monthly_sales + 
         2 * LAG(monthly_sales, 1) OVER (ORDER BY year, month) + 
         1 * LAG(monthly_sales, 2) OVER (ORDER BY year, month)) / 
        NULLIF(
            3 + 
            CASE WHEN LAG(monthly_sales, 1) OVER (ORDER BY year, month) IS NOT NULL THEN 2 ELSE 0 END +
            CASE WHEN LAG(monthly_sales, 2) OVER (ORDER BY year, month) IS NOT NULL THEN 1 ELSE 0 END,
            0
        ),
        2
    ) AS weighted_avg_3
FROM monthly_data
ORDER BY year, month;
```

---

## แบบฝึกหัดท้ายบท

### แบบฝึกหัดที่ 1
สร้างรายงาน ROLLUP ที่แสดง total sales ตาม year > region > category พร้อม subtotals

**คำตอบ:**
```sql
SELECT 
    CASE WHEN GROUPING(year) = 1 THEN 'TOTAL' ELSE year::TEXT END AS year,
    CASE WHEN GROUPING(region) = 1 THEN 'All Regions' ELSE region END AS region,
    CASE WHEN GROUPING(category) = 1 THEN 'All Categories' ELSE category END AS category,
    SUM(amount) AS total_sales,
    COUNT(*) AS transaction_count,
    ROUND(AVG(amount), 2) AS avg_transaction
FROM sales
GROUP BY ROLLUP(year, region, category)
ORDER BY 
    GROUPING(year), year,
    GROUPING(region), region,
    GROUPING(category), category;
```

### แบบฝึกหัดที่ 2
สร้าง PIVOT table แสดง total sales ต่อ product โดยมี year เป็น columns

**คำตอบ:**
```sql
SELECT 
    product,
    SUM(CASE WHEN year = 2023 THEN amount ELSE 0 END) AS "2023",
    SUM(CASE WHEN year = 2024 THEN amount ELSE 0 END) AS "2024",
    SUM(amount) AS total,
    ROUND(
        100.0 * (SUM(CASE WHEN year = 2024 THEN amount ELSE 0 END) -
                 SUM(CASE WHEN year = 2023 THEN amount ELSE 0 END)) /
        NULLIF(SUM(CASE WHEN year = 2023 THEN amount ELSE 0 END), 0),
        1
    ) AS yoy_growth_pct
FROM sales
GROUP BY product
ORDER BY total DESC;
```

### แบบฝึกหัดที่ 3
คำนวณ median, Q1, Q3, และ IQR สำหรับ sales amount ต่อ category

**คำตอบ:**
```sql
SELECT 
    category,
    ROUND(PERCENTILE_CONT(0.25) WITHIN GROUP (ORDER BY amount), 2) AS q1,
    ROUND(PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY amount), 2) AS median,
    ROUND(PERCENTILE_CONT(0.75) WITHIN GROUP (ORDER BY amount), 2) AS q3,
    ROUND(
        PERCENTILE_CONT(0.75) WITHIN GROUP (ORDER BY amount) -
        PERCENTILE_CONT(0.25) WITHIN GROUP (ORDER BY amount),
        2
    ) AS iqr,
    COUNT(*) AS n,
    ROUND(AVG(amount), 2) AS mean_val
FROM sales
GROUP BY category
ORDER BY category;
```

### แบบฝึกหัดที่ 4
สร้าง ABC analysis สำหรับ regions โดยใช้ cumulative sales percentage

**คำตอบ:**
```sql
WITH region_sales AS (
    SELECT 
        region,
        SUM(amount) AS total_sales
    FROM sales
    GROUP BY region
),
ranked AS (
    SELECT 
        region,
        total_sales,
        SUM(total_sales) OVER () AS grand_total,
        SUM(total_sales) OVER (ORDER BY total_sales DESC 
                               ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS running_total
    FROM region_sales
)
SELECT 
    region,
    total_sales,
    ROUND(100.0 * total_sales / grand_total, 2) AS pct_of_total,
    ROUND(100.0 * running_total / grand_total, 2) AS cumulative_pct,
    CASE 
        WHEN running_total / grand_total <= 0.8 THEN 'A - Major'
        WHEN running_total / grand_total <= 0.95 THEN 'B - Moderate'
        ELSE 'C - Minor'
    END AS abc_class
FROM ranked
ORDER BY total_sales DESC;
```

### แบบฝึกหัดที่ 5
สร้าง Year-over-Year growth report ตาม quarter และ region

**คำตอบ:**
```sql
WITH quarterly_regional AS (
    SELECT year, quarter, region, SUM(amount) AS sales
    FROM sales
    GROUP BY year, quarter, region
)
SELECT 
    curr.year,
    curr.quarter,
    curr.region,
    curr.sales AS current_sales,
    prev.sales AS prev_year_sales,
    curr.sales - COALESCE(prev.sales, 0) AS absolute_change,
    CASE 
        WHEN prev.sales IS NULL THEN NULL
        ELSE ROUND(100.0 * (curr.sales - prev.sales) / prev.sales, 2)
    END AS yoy_growth_pct
FROM quarterly_regional curr
LEFT JOIN quarterly_regional prev 
    ON curr.year = prev.year + 1 
    AND curr.quarter = prev.quarter 
    AND curr.region = prev.region
ORDER BY curr.year, curr.quarter, curr.region;
```

### แบบฝึกหัดที่ 6
คำนวณ Pearson correlation ระหว่าง sales amount และ quantity ต่อ category และแปลผล

**คำตอบ:**
```sql
SELECT 
    category,
    COUNT(*) AS n,
    ROUND(CORR(amount, quantity)::NUMERIC, 4) AS correlation,
    ROUND(REGR_SLOPE(amount, quantity)::NUMERIC, 2) AS slope,
    ROUND(REGR_INTERCEPT(amount, quantity)::NUMERIC, 2) AS intercept,
    ROUND(REGR_R2(amount, quantity)::NUMERIC, 4) AS r_squared,
    CASE 
        WHEN ABS(CORR(amount, quantity)) >= 0.9 THEN 'Very Strong'
        WHEN ABS(CORR(amount, quantity)) >= 0.7 THEN 'Strong'
        WHEN ABS(CORR(amount, quantity)) >= 0.5 THEN 'Moderate'
        WHEN ABS(CORR(amount, quantity)) >= 0.3 THEN 'Weak'
        ELSE 'Very Weak'
    END AS correlation_strength,
    CASE WHEN CORR(amount, quantity) > 0 THEN 'Positive' ELSE 'Negative' END AS direction
FROM sales
GROUP BY category
HAVING COUNT(*) >= 3;
```

### แบบฝึกหัดที่ 7
สร้าง CUBE report แสดง sales ทุก combination ของ year, region

**คำตอบ:**
```sql
SELECT 
    CASE WHEN GROUPING(year) = 1 THEN 'All Years' ELSE year::TEXT END AS year,
    CASE WHEN GROUPING(region) = 1 THEN 'All Regions' ELSE region END AS region,
    SUM(amount) AS total_sales,
    COUNT(*) AS transactions,
    ROUND(AVG(amount), 2) AS avg_sale,
    CASE 
        WHEN GROUPING(year) = 0 AND GROUPING(region) = 0 THEN 'Cell'
        WHEN GROUPING(year) = 0 AND GROUPING(region) = 1 THEN 'Year Total'
        WHEN GROUPING(year) = 1 AND GROUPING(region) = 0 THEN 'Region Total'
        ELSE 'Grand Total'
    END AS aggregation_level
FROM sales
GROUP BY CUBE(year, region)
ORDER BY year NULLS LAST, region NULLS LAST;
```

### แบบฝึกหัดที่ 8
สร้าง funnel analysis สำหรับ user_events และคำนวณ drop-off rate ในแต่ละขั้น

**คำตอบ:**
```sql
WITH funnel_data AS (
    SELECT
        COUNT(DISTINCT CASE WHEN event_type = 'view_product' THEN user_id END) AS views,
        COUNT(DISTINCT CASE WHEN event_type = 'add_to_cart' THEN user_id END) AS carts,
        COUNT(DISTINCT CASE WHEN event_type = 'checkout' THEN user_id END) AS checkouts,
        COUNT(DISTINCT CASE WHEN event_type = 'purchase' THEN user_id END) AS purchases
    FROM user_events
)
SELECT step_num, step_name, users,
       ROUND(100.0 * users / FIRST_VALUE(users) OVER (ORDER BY step_num), 1) AS overall_conversion,
       ROUND(100.0 * (LAG(users) OVER (ORDER BY step_num) - users) / 
             NULLIF(LAG(users) OVER (ORDER BY step_num), 0), 1) AS step_drop_off_pct
FROM (
    SELECT 1 AS step_num, 'View Product' AS step_name, views AS users FROM funnel_data
    UNION ALL SELECT 2, 'Add to Cart', carts FROM funnel_data
    UNION ALL SELECT 3, 'Checkout', checkouts FROM funnel_data
    UNION ALL SELECT 4, 'Purchase', purchases FROM funnel_data
) funnel
ORDER BY step_num;
```

### แบบฝึกหัดที่ 9
สร้าง seasonality analysis แสดง index ว่า quarter ไหนสูงกว่า/ต่ำกว่าค่าเฉลี่ย

**คำตอบ:**
```sql
WITH quarterly_totals AS (
    SELECT year, quarter, SUM(amount) AS quarterly_sales
    FROM sales
    GROUP BY year, quarter
),
quarterly_stats AS (
    SELECT 
        quarter,
        AVG(quarterly_sales) AS avg_quarterly,
        (SELECT AVG(quarterly_sales) FROM quarterly_totals) AS overall_avg
    FROM quarterly_totals
    GROUP BY quarter
)
SELECT 
    quarter,
    ROUND(avg_quarterly, 2) AS avg_sales,
    ROUND(overall_avg, 2) AS overall_avg,
    ROUND(100.0 * avg_quarterly / overall_avg, 1) AS seasonality_index,
    CASE 
        WHEN avg_quarterly > overall_avg * 1.1 THEN 'Peak Season (+10%)'
        WHEN avg_quarterly < overall_avg * 0.9 THEN 'Low Season (-10%)'
        ELSE 'Normal Season'
    END AS season_classification
FROM quarterly_stats
ORDER BY quarter;
```

### แบบฝึกหัดที่ 10
สร้าง GROUPING SETS report ที่แสดง sales ตาม (year, quarter), (region, category), และ grand total

**คำตอบ:**
```sql
SELECT 
    CASE WHEN GROUPING(year, quarter) = 0 THEN year::TEXT || ' ' || quarter ELSE NULL END AS time_period,
    CASE WHEN GROUPING(region, category) = 0 THEN region || ' / ' || category ELSE NULL END AS segment,
    year,
    quarter,
    region,
    category,
    SUM(amount) AS total_sales,
    COUNT(*) AS transactions,
    CASE 
        WHEN GROUPING(year, quarter) = 0 AND GROUPING(region, category) = 3 THEN 'By Time Period'
        WHEN GROUPING(year, quarter) = 3 AND GROUPING(region, category) = 0 THEN 'By Segment'
        ELSE 'Grand Total'
    END AS report_type
FROM sales
GROUP BY GROUPING SETS (
    (year, quarter),
    (region, category),
    ()
)
ORDER BY 
    GROUPING(year, quarter),
    year NULLS LAST, 
    quarter NULLS LAST,
    GROUPING(region, category),
    region NULLS LAST, 
    category NULLS LAST;
```

---

## สรุปบทที่ 99

### OLAP Functions สรุป

| Function | คำอธิบาย | ใช้เมื่อ |
|----------|-----------|---------|
| ROLLUP | Hierarchical subtotals | Drill-down reports |
| CUBE | All combinations | Cross-tab analysis |
| GROUPING SETS | Custom combinations | Flexible reporting |
| PIVOT (CASE WHEN) | Rows to columns | Comparison matrices |
| PERCENTILE_CONT | Interpolated percentile | Median, IQR |
| PERCENTILE_DISC | Actual value at percentile | Most common value |
| CORR | Pearson correlation | Relationship analysis |
| REGR_SLOPE | Regression coefficient | Trend, prediction |

### Performance Tips
1. ใช้ **GROUPING SETS** แทน UNION ALL multiple GROUP BY — ผ่านข้อมูลครั้งเดียว
2. **ROLLUP** เร็วกว่า **CUBE** เมื่อต้องการ hierarchical subtotals
3. ใช้ **CTE** เพื่อ share computation ระหว่าง multiple aggregation levels

ในบทถัดไป (บทสุดท้าย!) เราจะเรียนรู้เกี่ยวกับ Temporal Tables, Time-Travel Queries, และ Slowly Changing Dimensions
