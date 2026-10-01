# ตอนที่ 108: SQL for Data Analysis and Reporting

## บทนำ

SQL เป็นเครื่องมือหลักสำหรับ Data Analysis และ Business Intelligence งานวิเคราะห์ข้อมูลส่วนใหญ่ทำได้ตรงในฐานข้อมูล โดยไม่ต้องส่งข้อมูลออกไปประมวลผลที่อื่น บทนี้ครอบคลุมตั้งแต่ Data Warehouse concepts ไปจนถึง advanced analytics patterns

---

## 1. Data Warehouse Concepts

### 1.1 Star Schema

```sql
-- Star Schema for E-commerce Data Warehouse

-- Dimension Tables
CREATE TABLE dim_date (
    date_id         INTEGER PRIMARY KEY,
    full_date       DATE NOT NULL,
    year            SMALLINT NOT NULL,
    quarter         SMALLINT NOT NULL,
    month           SMALLINT NOT NULL,
    month_name      VARCHAR(10) NOT NULL,
    week_of_year    SMALLINT NOT NULL,
    day_of_week     SMALLINT NOT NULL,
    day_name        VARCHAR(10) NOT NULL,
    is_weekend      BOOLEAN DEFAULT FALSE,
    is_holiday      BOOLEAN DEFAULT FALSE,
    fiscal_year     SMALLINT,
    fiscal_quarter  SMALLINT
);

CREATE TABLE dim_customer (
    customer_id     INTEGER PRIMARY KEY,
    customer_key    VARCHAR(50) UNIQUE NOT NULL,  -- natural key
    name            VARCHAR(100) NOT NULL,
    email           VARCHAR(200),
    city            VARCHAR(100),
    province        VARCHAR(100),
    country         VARCHAR(50) DEFAULT 'Thailand',
    segment         VARCHAR(50),    -- B2B, B2C, VIP
    tier            VARCHAR(20),    -- Bronze, Silver, Gold, Platinum
    acquired_date   DATE,
    -- SCD Type 2 fields
    effective_from  DATE NOT NULL,
    effective_to    DATE,
    is_current      BOOLEAN DEFAULT TRUE
);

CREATE TABLE dim_product (
    product_id      INTEGER PRIMARY KEY,
    product_key     VARCHAR(50) UNIQUE NOT NULL,
    sku             VARCHAR(50) NOT NULL,
    name            VARCHAR(200) NOT NULL,
    category_id     INTEGER,
    category        VARCHAR(100),
    subcategory     VARCHAR(100),
    brand           VARCHAR(100),
    unit_cost       DECIMAL(10,2),
    list_price      DECIMAL(10,2),
    is_active       BOOLEAN DEFAULT TRUE
);

CREATE TABLE dim_location (
    location_id     INTEGER PRIMARY KEY,
    city            VARCHAR(100),
    province        VARCHAR(100),
    region          VARCHAR(50),     -- North, South, Central, etc.
    country         VARCHAR(50),
    postal_code     VARCHAR(20),
    latitude        DECIMAL(9,6),
    longitude       DECIMAL(9,6)
);

-- Fact Table
CREATE TABLE fact_sales (
    sale_id         BIGINT PRIMARY KEY,
    date_id         INTEGER REFERENCES dim_date(date_id),
    customer_id     INTEGER REFERENCES dim_customer(customer_id),
    product_id      INTEGER REFERENCES dim_product(product_id),
    location_id     INTEGER REFERENCES dim_location(location_id),
    
    -- Degenerate dimensions
    order_id        VARCHAR(50),
    order_line      SMALLINT,
    
    -- Measures
    quantity        INTEGER NOT NULL,
    unit_price      DECIMAL(10,2) NOT NULL,
    unit_cost       DECIMAL(10,2) NOT NULL,
    discount_pct    DECIMAL(5,2) DEFAULT 0,
    
    -- Pre-calculated measures (for performance)
    gross_sales     DECIMAL(12,2) GENERATED ALWAYS AS (quantity * unit_price) STORED,
    discount_amount DECIMAL(12,2) GENERATED ALWAYS AS (quantity * unit_price * discount_pct / 100) STORED,
    net_sales       DECIMAL(12,2) GENERATED ALWAYS AS (quantity * unit_price * (1 - discount_pct/100)) STORED,
    cogs            DECIMAL(12,2) GENERATED ALWAYS AS (quantity * unit_cost) STORED,
    gross_profit    DECIMAL(12,2) GENERATED ALWAYS AS (quantity * (unit_price * (1 - discount_pct/100) - unit_cost)) STORED
);

-- Indexes สำหรับ analytics queries
CREATE INDEX idx_fact_sales_date ON fact_sales(date_id);
CREATE INDEX idx_fact_sales_customer ON fact_sales(customer_id);
CREATE INDEX idx_fact_sales_product ON fact_sales(product_id);
CREATE INDEX idx_fact_sales_date_product ON fact_sales(date_id, product_id);
CREATE INDEX idx_fact_sales_date_customer ON fact_sales(date_id, customer_id);
```

### 1.2 Snowflake Schema

```sql
-- Snowflake Schema: ขยาย dimension tables ออกไปอีก

CREATE TABLE dim_product_category (
    category_id     INTEGER PRIMARY KEY,
    category_name   VARCHAR(100),
    parent_category_id INTEGER REFERENCES dim_product_category(category_id),
    level           SMALLINT,
    path            TEXT   -- e.g., 'Electronics > Phones > Smartphones'
);

-- dim_product reference dim_product_category
ALTER TABLE dim_product 
ADD COLUMN category_key INTEGER REFERENCES dim_product_category(category_id);

-- Location hierarchy
CREATE TABLE dim_geography (
    geo_id          INTEGER PRIMARY KEY,
    postal_code     VARCHAR(20),
    subdistrict     VARCHAR(100),
    district        VARCHAR(100),
    province        VARCHAR(100),
    region          VARCHAR(50),
    country         VARCHAR(50)
);
```

---

## 2. ETL Patterns

### 2.1 Incremental Load

```sql
-- ETL: Incremental Load Pattern

-- Staging table (รับข้อมูลใหม่)
CREATE TABLE stg_orders (
    order_id        VARCHAR(50),
    customer_email  VARCHAR(200),
    product_sku     VARCHAR(50),
    quantity        INTEGER,
    unit_price      DECIMAL(10,2),
    order_date      TIMESTAMP,
    status          VARCHAR(20),
    loaded_at       TIMESTAMP DEFAULT NOW()
);

-- Watermark table (เก็บ last loaded timestamp)
CREATE TABLE etl_watermarks (
    source_table    VARCHAR(100) PRIMARY KEY,
    last_loaded_at  TIMESTAMP NOT NULL,
    rows_loaded     INTEGER,
    updated_at      TIMESTAMP DEFAULT NOW()
);

-- Incremental extraction
INSERT INTO stg_orders
SELECT order_id, customer_email, product_sku, quantity, unit_price, order_date, status
FROM source_orders
WHERE updated_at > (
    SELECT last_loaded_at 
    FROM etl_watermarks 
    WHERE source_table = 'orders'
);

-- Transform and Load with UPSERT
INSERT INTO fact_sales (date_id, customer_id, product_id, quantity, unit_price, unit_cost, order_id)
SELECT 
    d.date_id,
    c.customer_id,
    p.product_id,
    s.quantity,
    s.unit_price,
    p.unit_cost,
    s.order_id
FROM stg_orders s
JOIN dim_date d ON d.full_date = s.order_date::DATE
JOIN dim_customer c ON c.customer_key = s.customer_email AND c.is_current = TRUE
JOIN dim_product p ON p.product_key = s.product_sku
ON CONFLICT (order_id, order_line)
DO UPDATE SET
    quantity = EXCLUDED.quantity,
    unit_price = EXCLUDED.unit_price;
```

### 2.2 SCD Type 2 (Slowly Changing Dimensions)

```sql
-- SCD Type 2: เก็บประวัติ dimension

-- Procedure สำหรับ update customer dimension
CREATE OR REPLACE FUNCTION upsert_customer_scd2(
    p_customer_key VARCHAR(50),
    p_name VARCHAR(100),
    p_email VARCHAR(200),
    p_tier VARCHAR(20),
    p_segment VARCHAR(50)
)
RETURNS VOID AS $$
DECLARE
    v_current_record RECORD;
    v_has_changed BOOLEAN;
BEGIN
    -- ดึง current record
    SELECT * INTO v_current_record
    FROM dim_customer
    WHERE customer_key = p_customer_key
    AND is_current = TRUE;
    
    IF NOT FOUND THEN
        -- New customer
        INSERT INTO dim_customer (customer_key, name, email, tier, segment, effective_from)
        VALUES (p_customer_key, p_name, p_email, p_tier, p_segment, CURRENT_DATE);
    ELSE
        -- Check if anything changed
        v_has_changed := (
            v_current_record.name != p_name OR
            v_current_record.email != p_email OR
            v_current_record.tier != p_tier OR
            v_current_record.segment != p_segment
        );
        
        IF v_has_changed THEN
            -- Expire current record
            UPDATE dim_customer
            SET effective_to = CURRENT_DATE - 1,
                is_current = FALSE
            WHERE customer_id = v_current_record.customer_id;
            
            -- Insert new version
            INSERT INTO dim_customer (
                customer_key, name, email, tier, segment,
                effective_from, is_current
            )
            VALUES (
                p_customer_key, p_name, p_email, p_tier, p_segment,
                CURRENT_DATE, TRUE
            );
        END IF;
    END IF;
END;
$$ LANGUAGE plpgsql;
```

---

## 3. Data Cleaning SQL

### 3.1 Data Quality Checks

```sql
-- Data Quality Dashboard

-- 1. Null value audit
SELECT 
    column_name,
    COUNT(*) AS total_rows,
    COUNT(column_name) AS non_null_rows,
    COUNT(*) - COUNT(column_name) AS null_count,
    ROUND(100.0 * (COUNT(*) - COUNT(column_name)) / COUNT(*), 2) AS null_pct
FROM (
    SELECT 
        UNNEST(ARRAY['name', 'email', 'phone', 'address']) AS column_name,
        name, email, phone, address
    FROM customers
) t
GROUP BY column_name
ORDER BY null_pct DESC;

-- 2. Duplicate detection
SELECT 
    email,
    COUNT(*) AS occurrences,
    ARRAY_AGG(id ORDER BY created_at) AS customer_ids
FROM customers
GROUP BY email
HAVING COUNT(*) > 1
ORDER BY occurrences DESC;

-- 3. Outlier detection using IQR
WITH stats AS (
    SELECT 
        PERCENTILE_CONT(0.25) WITHIN GROUP (ORDER BY total_amount) AS q1,
        PERCENTILE_CONT(0.75) WITHIN GROUP (ORDER BY total_amount) AS q3,
        PERCENTILE_CONT(0.75) WITHIN GROUP (ORDER BY total_amount) -
        PERCENTILE_CONT(0.25) WITHIN GROUP (ORDER BY total_amount) AS iqr
    FROM orders
    WHERE status = 'completed'
),
bounds AS (
    SELECT 
        q1 - 1.5 * iqr AS lower_bound,
        q3 + 1.5 * iqr AS upper_bound
    FROM stats
)
SELECT 
    o.id,
    o.total_amount,
    CASE 
        WHEN o.total_amount < b.lower_bound THEN 'LOW_OUTLIER'
        WHEN o.total_amount > b.upper_bound THEN 'HIGH_OUTLIER'
    END AS outlier_type
FROM orders o, bounds b
WHERE o.total_amount < b.lower_bound OR o.total_amount > b.upper_bound
ORDER BY o.total_amount;
```

### 3.2 Data Transformation

```sql
-- Data Transformation Patterns

-- Standardize phone numbers
UPDATE customers
SET phone = REGEXP_REPLACE(
    REGEXP_REPLACE(phone, '[^0-9]', '', 'g'),  -- Remove non-digits
    '^66', '0'  -- Replace country code with 0
)
WHERE phone IS NOT NULL;

-- Normalize text data
UPDATE products
SET 
    name = INITCAP(TRIM(REGEXP_REPLACE(name, '\s+', ' ', 'g'))),
    sku = UPPER(TRIM(sku)),
    category = LOWER(TRIM(category));

-- Parse JSON fields
SELECT 
    id,
    metadata->>'source' AS source,
    metadata->>'campaign_id' AS campaign_id,
    (metadata->>'discount_applied')::BOOLEAN AS discount_applied
FROM orders
WHERE metadata IS NOT NULL;

-- Pivot data (crosstab)
SELECT 
    product_id,
    SUM(CASE WHEN EXTRACT(MONTH FROM order_date) = 1 THEN quantity ELSE 0 END) AS jan,
    SUM(CASE WHEN EXTRACT(MONTH FROM order_date) = 2 THEN quantity ELSE 0 END) AS feb,
    SUM(CASE WHEN EXTRACT(MONTH FROM order_date) = 3 THEN quantity ELSE 0 END) AS mar,
    SUM(CASE WHEN EXTRACT(MONTH FROM order_date) = 4 THEN quantity ELSE 0 END) AS apr,
    SUM(CASE WHEN EXTRACT(MONTH FROM order_date) = 5 THEN quantity ELSE 0 END) AS may,
    SUM(CASE WHEN EXTRACT(MONTH FROM order_date) = 6 THEN quantity ELSE 0 END) AS jun,
    SUM(quantity) AS total
FROM order_items oi
JOIN orders o ON oi.order_id = o.id
WHERE EXTRACT(YEAR FROM o.order_date) = 2024
AND o.status = 'completed'
GROUP BY product_id
ORDER BY total DESC;
```

---

## 4. Cohort Analysis

### 4.1 Acquisition Cohort

```sql
-- Customer Acquisition Cohort Analysis

-- Step 1: กำหนด cohort ของแต่ละลูกค้า (เดือนที่ซื้อครั้งแรก)
WITH first_purchase AS (
    SELECT 
        customer_id,
        DATE_TRUNC('month', MIN(created_at)) AS cohort_month
    FROM orders
    WHERE status = 'completed'
    GROUP BY customer_id
),

-- Step 2: ดู activity ในแต่ละเดือน
monthly_activity AS (
    SELECT 
        o.customer_id,
        DATE_TRUNC('month', o.created_at) AS activity_month,
        SUM(o.total_amount) AS monthly_revenue
    FROM orders o
    WHERE o.status = 'completed'
    GROUP BY o.customer_id, DATE_TRUNC('month', o.created_at)
),

-- Step 3: คำนวณ months since acquisition
cohort_data AS (
    SELECT 
        fp.cohort_month,
        EXTRACT(YEAR FROM AGE(ma.activity_month, fp.cohort_month)) * 12 +
        EXTRACT(MONTH FROM AGE(ma.activity_month, fp.cohort_month)) AS month_number,
        COUNT(DISTINCT ma.customer_id) AS active_customers,
        SUM(ma.monthly_revenue) AS revenue
    FROM first_purchase fp
    JOIN monthly_activity ma ON fp.customer_id = ma.customer_id
    GROUP BY fp.cohort_month, month_number
),

-- Step 4: หา cohort size
cohort_sizes AS (
    SELECT cohort_month, COUNT(*) AS cohort_size
    FROM first_purchase
    GROUP BY cohort_month
)

-- Final result: Retention matrix
SELECT 
    TO_CHAR(cd.cohort_month, 'YYYY-MM') AS cohort,
    cs.cohort_size,
    cd.month_number,
    cd.active_customers,
    ROUND(100.0 * cd.active_customers / cs.cohort_size, 1) AS retention_rate,
    cd.revenue,
    ROUND(cd.revenue / cd.active_customers, 2) AS avg_revenue_per_user
FROM cohort_data cd
JOIN cohort_sizes cs ON cd.cohort_month = cs.cohort_month
WHERE cd.cohort_month >= '2024-01-01'
AND cd.month_number <= 12
ORDER BY cd.cohort_month, cd.month_number;
```

---

## 5. Funnel Analysis

### 5.1 Conversion Funnel

```sql
-- E-commerce Conversion Funnel
-- Events: view_product -> add_to_cart -> checkout_start -> payment -> purchase

WITH funnel_steps AS (
    SELECT 
        user_id,
        session_id,
        -- ตรวจสอบว่าทำแต่ละ step ไหม
        MAX(CASE WHEN event_type = 'view_product' THEN 1 ELSE 0 END) AS did_view,
        MAX(CASE WHEN event_type = 'add_to_cart' THEN 1 ELSE 0 END) AS did_cart,
        MAX(CASE WHEN event_type = 'checkout_start' THEN 1 ELSE 0 END) AS did_checkout,
        MAX(CASE WHEN event_type = 'payment_started' THEN 1 ELSE 0 END) AS did_payment,
        MAX(CASE WHEN event_type = 'purchase_completed' THEN 1 ELSE 0 END) AS did_purchase
    FROM user_events
    WHERE event_date >= CURRENT_DATE - 30
    GROUP BY user_id, session_id
),

-- นับ sessions ที่ผ่านแต่ละ step
funnel_counts AS (
    SELECT
        SUM(did_view) AS step1_view,
        SUM(did_cart) AS step2_cart,
        SUM(did_checkout) AS step3_checkout,
        SUM(did_payment) AS step4_payment,
        SUM(did_purchase) AS step5_purchase
    FROM funnel_steps
    WHERE did_view = 1  -- เริ่มจาก view
)

SELECT
    step_name,
    users,
    ROUND(100.0 * users / LAG(users) OVER (ORDER BY step_order), 1) AS step_conversion_pct,
    ROUND(100.0 * users / FIRST_VALUE(users) OVER (ORDER BY step_order), 1) AS overall_conversion_pct
FROM (
    SELECT 1 AS step_order, 'View Product' AS step_name, step1_view AS users FROM funnel_counts
    UNION ALL
    SELECT 2, 'Add to Cart', step2_cart FROM funnel_counts
    UNION ALL
    SELECT 3, 'Checkout Start', step3_checkout FROM funnel_counts
    UNION ALL
    SELECT 4, 'Payment', step4_payment FROM funnel_counts
    UNION ALL
    SELECT 5, 'Purchase', step5_purchase FROM funnel_counts
) steps
ORDER BY step_order;
```

---

## 6. A/B Test Analysis

```sql
-- A/B Test Analysis

-- Experiment results table
CREATE TABLE ab_test_results (
    experiment_id   VARCHAR(50),
    user_id         INTEGER,
    variant         VARCHAR(10),  -- 'control' or 'treatment'
    assigned_at     TIMESTAMP,
    converted       BOOLEAN,
    revenue         DECIMAL(10,2)
);

-- A/B Test Summary
WITH variant_stats AS (
    SELECT 
        variant,
        COUNT(*) AS users,
        SUM(CASE WHEN converted THEN 1 ELSE 0 END) AS conversions,
        SUM(revenue) AS total_revenue,
        AVG(revenue) AS avg_revenue
    FROM ab_test_results
    WHERE experiment_id = 'homepage_v2'
    GROUP BY variant
),

-- Conversion rate with confidence interval (Wilson Score)
conversion_ci AS (
    SELECT 
        variant,
        users,
        conversions,
        ROUND(100.0 * conversions / users, 2) AS conversion_rate,
        -- Wilson score confidence interval (95%)
        ROUND(100.0 * (
            (conversions::FLOAT / users + 1.96^2 / (2*users) - 
             1.96 * SQRT((conversions::FLOAT/users * (1 - conversions::FLOAT/users) + 1.96^2/(4*users))/users)) /
            (1 + 1.96^2/users)
        ), 2) AS ci_lower,
        ROUND(100.0 * (
            (conversions::FLOAT / users + 1.96^2 / (2*users) + 
             1.96 * SQRT((conversions::FLOAT/users * (1 - conversions::FLOAT/users) + 1.96^2/(4*users))/users)) /
            (1 + 1.96^2/users)
        ), 2) AS ci_upper
    FROM variant_stats
)
SELECT 
    variant,
    users,
    conversions,
    conversion_rate,
    ci_lower || '% - ' || ci_upper || '%' AS confidence_interval_95,
    ROUND(avg_revenue, 2) AS avg_revenue_per_user
FROM conversion_ci
ORDER BY variant;
```

---

## 7. RFM Analysis

```sql
-- RFM Analysis (Recency, Frequency, Monetary)

WITH rfm_raw AS (
    SELECT 
        customer_id,
        MAX(order_date) AS last_purchase,
        COUNT(DISTINCT id) AS frequency,
        SUM(total_amount) AS monetary
    FROM orders
    WHERE status = 'completed'
    AND order_date >= CURRENT_DATE - INTERVAL '1 year'
    GROUP BY customer_id
),

-- Calculate RFM scores (1-5)
rfm_scores AS (
    SELECT 
        customer_id,
        CURRENT_DATE - last_purchase::DATE AS recency_days,
        frequency,
        monetary,
        -- Recency: ยิ่งซื้อล่าสุดยิ่งดี (score สูง)
        NTILE(5) OVER (ORDER BY last_purchase DESC) AS r_score,
        -- Frequency: ยิ่งซื้อบ่อยยิ่งดี
        NTILE(5) OVER (ORDER BY frequency ASC) AS f_score,
        -- Monetary: ยิ่งใช้จ่ายมากยิ่งดี
        NTILE(5) OVER (ORDER BY monetary ASC) AS m_score
    FROM rfm_raw
),

-- Segment ลูกค้า
rfm_segments AS (
    SELECT 
        customer_id,
        recency_days,
        frequency,
        monetary,
        r_score,
        f_score,
        m_score,
        r_score || f_score || m_score AS rfm_code,
        -- ถัวเฉลี่ย RFM
        ROUND((r_score + f_score + m_score) / 3.0, 1) AS rfm_avg,
        CASE 
            WHEN r_score >= 4 AND f_score >= 4 AND m_score >= 4 THEN 'Champions'
            WHEN r_score >= 4 AND f_score >= 3 THEN 'Loyal Customers'
            WHEN r_score >= 3 AND f_score >= 3 AND m_score >= 3 THEN 'Potential Loyalists'
            WHEN r_score = 5 THEN 'Recent Customers'
            WHEN r_score >= 3 AND m_score >= 3 THEN 'Promising'
            WHEN r_score >= 3 THEN 'Need Attention'
            WHEN r_score <= 2 AND f_score >= 4 THEN 'At Risk'
            WHEN r_score <= 2 AND f_score >= 2 THEN 'About To Sleep'
            WHEN r_score = 1 AND f_score >= 4 AND m_score >= 4 THEN 'Cant Lose Them'
            WHEN r_score = 1 AND f_score >= 2 THEN 'Hibernating'
            ELSE 'Lost'
        END AS segment
    FROM rfm_scores
)
SELECT 
    segment,
    COUNT(*) AS customer_count,
    ROUND(AVG(recency_days), 0) AS avg_recency_days,
    ROUND(AVG(frequency), 1) AS avg_frequency,
    ROUND(AVG(monetary), 2) AS avg_monetary,
    ROUND(SUM(monetary), 2) AS total_monetary,
    ROUND(100.0 * COUNT(*) / SUM(COUNT(*)) OVER (), 1) AS pct_of_customers
FROM rfm_segments
GROUP BY segment
ORDER BY total_monetary DESC;
```

---

## 8. Customer Lifetime Value (CLV)

```sql
-- CLV Analysis

-- Historical CLV
WITH customer_orders AS (
    SELECT 
        customer_id,
        COUNT(*) AS total_orders,
        SUM(total_amount) AS total_revenue,
        MIN(order_date) AS first_purchase,
        MAX(order_date) AS last_purchase,
        AVG(total_amount) AS avg_order_value,
        -- Purchase frequency per month
        COUNT(*) / GREATEST(
            EXTRACT(MONTH FROM AGE(MAX(order_date), MIN(order_date))) + 1, 
            1
        ) AS monthly_frequency
    FROM orders
    WHERE status = 'completed'
    GROUP BY customer_id
),

clv_calculation AS (
    SELECT 
        customer_id,
        total_orders,
        total_revenue,
        avg_order_value,
        monthly_frequency,
        first_purchase,
        last_purchase,
        -- Customer lifespan in months
        EXTRACT(MONTH FROM AGE(last_purchase, first_purchase)) + 1 AS lifespan_months,
        
        -- Historical CLV
        total_revenue AS historical_clv,
        
        -- Predicted CLV (12 months)
        avg_order_value * monthly_frequency * 12 AS predicted_clv_12m,
        
        -- Churn probability (simple recency-based)
        CASE 
            WHEN last_purchase >= CURRENT_DATE - INTERVAL '30 days' THEN 0.1
            WHEN last_purchase >= CURRENT_DATE - INTERVAL '60 days' THEN 0.3
            WHEN last_purchase >= CURRENT_DATE - INTERVAL '90 days' THEN 0.5
            WHEN last_purchase >= CURRENT_DATE - INTERVAL '180 days' THEN 0.7
            ELSE 0.9
        END AS churn_probability
    FROM customer_orders
)
SELECT 
    customer_id,
    total_orders,
    ROUND(total_revenue, 2) AS historical_clv,
    ROUND(predicted_clv_12m, 2) AS predicted_clv_12m,
    ROUND(predicted_clv_12m * (1 - churn_probability), 2) AS risk_adjusted_clv,
    churn_probability,
    CASE 
        WHEN predicted_clv_12m >= 50000 THEN 'Tier 1 (Platinum)'
        WHEN predicted_clv_12m >= 20000 THEN 'Tier 2 (Gold)'
        WHEN predicted_clv_12m >= 5000 THEN 'Tier 3 (Silver)'
        ELSE 'Tier 4 (Standard)'
    END AS customer_tier
FROM clv_calculation
ORDER BY risk_adjusted_clv DESC;
```

---

## 9. Churn Analysis

```sql
-- Customer Churn Analysis

-- Define churn: ไม่ซื้อใน 90 วัน
WITH customer_last_purchase AS (
    SELECT 
        customer_id,
        MAX(order_date) AS last_purchase_date,
        COUNT(*) AS total_orders,
        SUM(total_amount) AS lifetime_value
    FROM orders
    WHERE status = 'completed'
    GROUP BY customer_id
),

churn_analysis AS (
    SELECT 
        customer_id,
        last_purchase_date,
        total_orders,
        lifetime_value,
        CURRENT_DATE - last_purchase_date::DATE AS days_since_purchase,
        CASE 
            WHEN CURRENT_DATE - last_purchase_date::DATE > 90 THEN TRUE
            ELSE FALSE
        END AS is_churned
    FROM customer_last_purchase
),

-- Churn rate by acquisition month
cohort_churn AS (
    SELECT 
        DATE_TRUNC('month', c.acquired_date) AS cohort_month,
        COUNT(ca.customer_id) AS cohort_size,
        SUM(CASE WHEN ca.is_churned THEN 1 ELSE 0 END) AS churned,
        SUM(CASE WHEN NOT ca.is_churned THEN 1 ELSE 0 END) AS retained
    FROM customers c
    JOIN churn_analysis ca ON c.id = ca.customer_id
    GROUP BY DATE_TRUNC('month', c.acquired_date)
)
SELECT 
    TO_CHAR(cohort_month, 'YYYY-MM') AS cohort,
    cohort_size,
    churned,
    retained,
    ROUND(100.0 * churned / cohort_size, 1) AS churn_rate,
    ROUND(100.0 * retained / cohort_size, 1) AS retention_rate
FROM cohort_churn
WHERE cohort_month >= CURRENT_DATE - INTERVAL '1 year'
ORDER BY cohort_month;

-- Early churn signals
SELECT 
    ca.customer_id,
    ca.days_since_purchase,
    ca.total_orders,
    ca.lifetime_value,
    -- Early warning indicators
    CASE 
        WHEN ca.days_since_purchase BETWEEN 61 AND 90 THEN 'High Risk'
        WHEN ca.days_since_purchase BETWEEN 31 AND 60 THEN 'Medium Risk'
        WHEN ca.days_since_purchase BETWEEN 15 AND 30 THEN 'Low Risk'
        ELSE 'Active'
    END AS risk_level,
    -- Recommended action
    CASE 
        WHEN ca.days_since_purchase BETWEEN 61 AND 90 AND ca.lifetime_value > 10000 THEN 'Personal Outreach'
        WHEN ca.days_since_purchase BETWEEN 61 AND 90 THEN 'Win-back Campaign'
        WHEN ca.days_since_purchase BETWEEN 31 AND 60 THEN 'Re-engagement Email'
        WHEN ca.days_since_purchase BETWEEN 15 AND 30 THEN 'Loyalty Offer'
        ELSE 'Regular Newsletter'
    END AS recommended_action
FROM churn_analysis ca
WHERE NOT ca.is_churned
AND ca.days_since_purchase >= 15
ORDER BY ca.days_since_purchase DESC, ca.lifetime_value DESC;
```

---

## 10. Advanced Analytics Patterns

### 10.1 Moving Averages และ Trend Analysis

```sql
-- Rolling metrics
SELECT 
    order_date::DATE AS date,
    COUNT(*) AS daily_orders,
    SUM(total_amount) AS daily_revenue,
    
    -- 7-day moving average
    AVG(COUNT(*)) OVER (
        ORDER BY order_date::DATE 
        ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
    ) AS orders_7d_avg,
    
    -- 30-day moving average
    AVG(SUM(total_amount)) OVER (
        ORDER BY order_date::DATE 
        ROWS BETWEEN 29 PRECEDING AND CURRENT ROW
    ) AS revenue_30d_avg,
    
    -- Cumulative total
    SUM(SUM(total_amount)) OVER (
        ORDER BY order_date::DATE
    ) AS cumulative_revenue,
    
    -- Year-over-year comparison
    SUM(total_amount) / LAG(SUM(total_amount), 365) OVER (
        ORDER BY order_date::DATE
    ) - 1 AS yoy_growth
    
FROM orders
WHERE status = 'completed'
GROUP BY order_date::DATE
ORDER BY date;
```

### 10.2 Market Basket Analysis

```sql
-- Market Basket Analysis: สินค้าที่ซื้อร่วมกัน

WITH product_pairs AS (
    SELECT 
        a.product_id AS product_a,
        b.product_id AS product_b,
        COUNT(DISTINCT a.order_id) AS co_purchases
    FROM order_items a
    JOIN order_items b ON a.order_id = b.order_id AND a.product_id < b.product_id
    GROUP BY a.product_id, b.product_id
    HAVING COUNT(DISTINCT a.order_id) >= 10  -- minimum support
),

product_totals AS (
    SELECT product_id, COUNT(DISTINCT order_id) AS total_orders
    FROM order_items
    GROUP BY product_id
),

total_orders AS (SELECT COUNT(DISTINCT id) AS total FROM orders)

SELECT 
    pa.name AS product_a,
    pb.name AS product_b,
    pp.co_purchases AS support_count,
    ROUND(100.0 * pp.co_purchases / t.total, 2) AS support_pct,
    -- Confidence: P(B|A) = P(A and B) / P(A)
    ROUND(100.0 * pp.co_purchases / pta.total_orders, 2) AS confidence_a_to_b,
    ROUND(100.0 * pp.co_purchases / ptb.total_orders, 2) AS confidence_b_to_a,
    -- Lift: how much more likely than random
    ROUND(
        (pp.co_purchases::FLOAT / t.total) / 
        ((pta.total_orders::FLOAT / t.total) * (ptb.total_orders::FLOAT / t.total)),
        2
    ) AS lift
FROM product_pairs pp
JOIN products pa ON pp.product_a = pa.id
JOIN products pb ON pp.product_b = pb.id
JOIN product_totals pta ON pp.product_a = pta.product_id
JOIN product_totals ptb ON pp.product_b = ptb.product_id
CROSS JOIN total_orders t
ORDER BY lift DESC, support_count DESC
LIMIT 50;
```

---

## แบบฝึกหัด

### ข้อที่ 1: Star Schema ETL

**เฉลย:**
```sql
-- ETL procedure ครบวงจร
CREATE OR REPLACE PROCEDURE run_daily_etl()
LANGUAGE plpgsql AS $$
DECLARE
    v_last_loaded TIMESTAMP;
    v_rows_loaded INTEGER;
BEGIN
    -- Get watermark
    SELECT last_loaded_at INTO v_last_loaded
    FROM etl_watermarks WHERE source_table = 'orders';
    
    v_last_loaded := COALESCE(v_last_loaded, '2000-01-01');
    
    -- Load staging
    TRUNCATE stg_orders;
    INSERT INTO stg_orders
    SELECT * FROM source_orders WHERE updated_at > v_last_loaded;
    
    GET DIAGNOSTICS v_rows_loaded = ROW_COUNT;
    
    -- Transform and load fact
    INSERT INTO fact_sales (date_id, customer_id, product_id, quantity, unit_price, unit_cost, order_id, order_line)
    SELECT 
        d.date_id, c.customer_id, p.product_id,
        s.quantity, s.unit_price, p.unit_cost,
        s.order_id, ROW_NUMBER() OVER (PARTITION BY s.order_id ORDER BY s.product_sku)
    FROM stg_orders s
    JOIN dim_date d ON d.full_date = s.order_date::DATE
    JOIN dim_customer c ON c.customer_key = s.customer_email AND c.is_current = TRUE
    JOIN dim_product p ON p.product_key = s.product_sku
    ON CONFLICT (order_id, order_line) DO UPDATE SET
        quantity = EXCLUDED.quantity,
        unit_price = EXCLUDED.unit_price;
    
    -- Update watermark
    INSERT INTO etl_watermarks (source_table, last_loaded_at, rows_loaded)
    VALUES ('orders', NOW(), v_rows_loaded)
    ON CONFLICT (source_table) DO UPDATE SET
        last_loaded_at = NOW(),
        rows_loaded = v_rows_loaded,
        updated_at = NOW();
    
    RAISE NOTICE 'ETL completed: % rows loaded', v_rows_loaded;
END;
$$;
```

### ข้อที่ 2: Cohort Retention

**เฉลย:**
```sql
-- Cohort retention matrix
WITH cohorts AS (
    SELECT customer_id, DATE_TRUNC('month', MIN(order_date)) AS cohort_month
    FROM orders WHERE status = 'completed'
    GROUP BY customer_id
),
monthly_activity AS (
    SELECT DISTINCT customer_id, DATE_TRUNC('month', order_date) AS month
    FROM orders WHERE status = 'completed'
),
cohort_matrix AS (
    SELECT 
        c.cohort_month,
        EXTRACT(EPOCH FROM (ma.month - c.cohort_month)) / (30.44 * 86400) AS month_num,
        COUNT(DISTINCT c.customer_id) AS users
    FROM cohorts c
    JOIN monthly_activity ma ON c.customer_id = ma.customer_id
    GROUP BY c.cohort_month, month_num
),
cohort_sizes AS (
    SELECT cohort_month, users AS size
    FROM cohort_matrix WHERE month_num = 0
)
SELECT 
    TO_CHAR(cm.cohort_month, 'YYYY-MM') AS cohort,
    cs.size AS cohort_size,
    ROUND(cm.month_num::NUMERIC, 0) AS month,
    cm.users,
    ROUND(100.0 * cm.users / cs.size, 1) AS retention_rate
FROM cohort_matrix cm
JOIN cohort_sizes cs ON cm.cohort_month = cs.cohort_month
ORDER BY cm.cohort_month, cm.month_num;
```

### ข้อที่ 3-10 (เฉลย ย่อ)

```sql
-- ข้อที่ 3: RFM Segmentation
WITH rfm AS (
    SELECT customer_id,
           NTILE(5) OVER (ORDER BY MAX(order_date) DESC) AS r,
           NTILE(5) OVER (ORDER BY COUNT(*) ASC) AS f,
           NTILE(5) OVER (ORDER BY SUM(total_amount) ASC) AS m
    FROM orders WHERE status = 'completed'
    GROUP BY customer_id
)
SELECT *, r::TEXT || f::TEXT || m::TEXT AS rfm_score,
       CASE WHEN r >= 4 AND f >= 4 THEN 'Champions'
            WHEN r >= 3 AND f >= 3 THEN 'Loyal'
            WHEN r >= 4 THEN 'Recent'
            WHEN r <= 2 AND f >= 3 THEN 'At Risk'
            ELSE 'Others' END AS segment
FROM rfm;

-- ข้อที่ 4: Funnel Drop-off
WITH sessions AS (
    SELECT session_id,
           MAX(CASE WHEN event = 'view' THEN 1 END) AS did_view,
           MAX(CASE WHEN event = 'cart' THEN 1 END) AS did_cart,
           MAX(CASE WHEN event = 'checkout' THEN 1 END) AS did_checkout,
           MAX(CASE WHEN event = 'purchase' THEN 1 END) AS did_purchase
    FROM events GROUP BY session_id
)
SELECT
    SUM(did_view) AS views,
    SUM(did_cart) AS carts,
    SUM(did_checkout) AS checkouts,
    SUM(did_purchase) AS purchases,
    ROUND(100.0 * SUM(did_cart) / NULLIF(SUM(did_view), 0), 1) AS view_to_cart,
    ROUND(100.0 * SUM(did_purchase) / NULLIF(SUM(did_view), 0), 1) AS overall_cvr
FROM sessions;

-- ข้อที่ 5: Market Basket
SELECT pa.name, pb.name,
       COUNT(*) AS co_buys,
       ROUND(100.0 * COUNT(*) / pta.total, 1) AS confidence_pct
FROM order_items a
JOIN order_items b ON a.order_id = b.order_id AND a.product_id < b.product_id
JOIN products pa ON a.product_id = pa.id
JOIN products pb ON b.product_id = pb.id
JOIN (SELECT product_id, COUNT(DISTINCT order_id) AS total FROM order_items GROUP BY 1) pta ON a.product_id = pta.product_id
GROUP BY pa.name, pb.name, pta.total
HAVING COUNT(*) >= 5
ORDER BY co_buys DESC LIMIT 20;

-- ข้อที่ 6: Monthly YoY Revenue
SELECT 
    EXTRACT(YEAR FROM order_date) AS year,
    EXTRACT(MONTH FROM order_date) AS month,
    SUM(total_amount) AS revenue,
    LAG(SUM(total_amount)) OVER (ORDER BY EXTRACT(YEAR FROM order_date), EXTRACT(MONTH FROM order_date)) AS prev_year_revenue,
    ROUND(100.0 * (SUM(total_amount) / LAG(SUM(total_amount)) OVER (ORDER BY EXTRACT(YEAR FROM order_date), EXTRACT(MONTH FROM order_date)) - 1), 1) AS yoy_growth
FROM orders WHERE status = 'completed'
GROUP BY 1, 2 ORDER BY 1, 2;

-- ข้อที่ 7: Pareto Analysis (80/20)
WITH product_revenue AS (
    SELECT p.id, p.name,
           SUM(oi.quantity * oi.unit_price) AS revenue
    FROM products p JOIN order_items oi ON p.id = oi.product_id
    JOIN orders o ON oi.order_id = o.id WHERE o.status = 'completed'
    GROUP BY p.id, p.name
),
ranked AS (
    SELECT *, 
           SUM(revenue) OVER (ORDER BY revenue DESC) AS cumulative_revenue,
           SUM(revenue) OVER () AS total_revenue
    FROM product_revenue
)
SELECT *, 
       ROUND(100.0 * cumulative_revenue / total_revenue, 1) AS cumulative_pct,
       CASE WHEN 100.0 * cumulative_revenue / total_revenue <= 80 THEN 'Top 80%' ELSE 'Bottom 20%' END AS tier
FROM ranked ORDER BY revenue DESC;

-- ข้อที่ 8: CLV Prediction
SELECT customer_id,
       AVG(total_amount) AS aov,
       COUNT(*) / GREATEST(EXTRACT(MONTH FROM AGE(MAX(order_date), MIN(order_date))) + 1, 1) AS monthly_frequency,
       AVG(total_amount) * COUNT(*) / GREATEST(EXTRACT(MONTH FROM AGE(MAX(order_date), MIN(order_date))) + 1, 1) * 12 AS predicted_clv_12m
FROM orders WHERE status = 'completed'
GROUP BY customer_id HAVING COUNT(*) >= 2
ORDER BY predicted_clv_12m DESC;

-- ข้อที่ 9: Anomaly Detection
WITH daily_stats AS (
    SELECT DATE(order_date) AS d, SUM(total_amount) AS revenue,
           AVG(SUM(total_amount)) OVER (ORDER BY DATE(order_date) ROWS BETWEEN 29 PRECEDING AND CURRENT ROW) AS moving_avg,
           STDDEV(SUM(total_amount)) OVER (ORDER BY DATE(order_date) ROWS BETWEEN 29 PRECEDING AND CURRENT ROW) AS moving_std
    FROM orders WHERE status = 'completed'
    GROUP BY DATE(order_date)
)
SELECT d, revenue, moving_avg,
       (revenue - moving_avg) / NULLIF(moving_std, 0) AS z_score,
       CASE WHEN ABS((revenue - moving_avg) / NULLIF(moving_std, 0)) > 2 THEN 'ANOMALY' ELSE 'NORMAL' END AS status
FROM daily_stats ORDER BY d;

-- ข้อที่ 10: Executive Dashboard
SELECT 
    'Today' AS period,
    COUNT(*) FILTER (WHERE DATE(order_date) = CURRENT_DATE) AS orders,
    SUM(total_amount) FILTER (WHERE DATE(order_date) = CURRENT_DATE) AS revenue,
    COUNT(DISTINCT customer_id) FILTER (WHERE DATE(order_date) = CURRENT_DATE) AS customers
FROM orders WHERE status = 'completed'
UNION ALL
SELECT 'MTD', 
    COUNT(*) FILTER (WHERE DATE_TRUNC('month', order_date) = DATE_TRUNC('month', CURRENT_DATE)),
    SUM(total_amount) FILTER (WHERE DATE_TRUNC('month', order_date) = DATE_TRUNC('month', CURRENT_DATE)),
    COUNT(DISTINCT customer_id) FILTER (WHERE DATE_TRUNC('month', order_date) = DATE_TRUNC('month', CURRENT_DATE))
FROM orders WHERE status = 'completed'
UNION ALL
SELECT 'YTD',
    COUNT(*) FILTER (WHERE EXTRACT(YEAR FROM order_date) = EXTRACT(YEAR FROM CURRENT_DATE)),
    SUM(total_amount) FILTER (WHERE EXTRACT(YEAR FROM order_date) = EXTRACT(YEAR FROM CURRENT_DATE)),
    COUNT(DISTINCT customer_id) FILTER (WHERE EXTRACT(YEAR FROM order_date) = EXTRACT(YEAR FROM CURRENT_DATE))
FROM orders WHERE status = 'completed';
```

---

## สรุป

บทนี้ครอบคลุม SQL สำหรับ Data Analysis อย่างครบถ้วน:

1. **Data Warehouse** - Star/Snowflake Schema, Fact/Dimension tables
2. **ETL** - Incremental load, SCD Type 2, Data quality
3. **Data Cleaning** - Null audit, Duplicate detection, Outlier detection
4. **Cohort Analysis** - Retention matrix, Acquisition cohorts
5. **Funnel Analysis** - Conversion tracking, Drop-off points
6. **A/B Testing** - Statistical significance, Confidence intervals
7. **RFM Analysis** - Customer segmentation
8. **CLV** - Historical and predictive
9. **Churn Analysis** - Risk scoring, Early warning signals
10. **Advanced Analytics** - Moving averages, Market basket, Pareto

SQL ที่ดีสำหรับ analytics ต้องเข้าใจ business logic, ใช้ window functions อย่างชำนาญ, และสามารถสร้าง insights ที่ actionable ได้
