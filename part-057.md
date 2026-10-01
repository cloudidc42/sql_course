# Part 057: Denormalization - When and How
# การ Denormalize - เมื่อใดและอย่างไร

---

## บทนำ: ทำไมต้อง Denormalize?

หลังจากเรียน Normalization มาหลาย Parts อาจดูขัดแย้งที่จะพูดถึง Denormalization แต่ความจริงคือ:

```
Normalization ดีสำหรับ:
✓ Data Integrity
✓ Avoiding Anomalies
✓ Write Operations

แต่บางครั้ง Normalized Schema ทำให้:
✗ Query ต้องใช้ JOIN หลายตาราง
✗ Aggregation ช้า (ต้องสแกนหลายตาราง)
✗ Report Generation ช้า
✗ Read Performance แย่

Denormalization คือการแลก:
Data Integrity ↔ Read Performance
```

---

## Read vs Write Performance Tradeoff

### Normalized Schema: ดีสำหรับ Write

```sql
-- Normalized: คลังข้อมูล E-Commerce
-- เขียนข้อมูล Order ใหม่: แค่ INSERT 2 ตาราง
INSERT INTO orders (customer_id, order_date, status)
VALUES (1001, CURRENT_DATE, 'pending');

INSERT INTO order_items (order_id, product_id, quantity, unit_price)
VALUES (LASTVAL(), 5001, 3, 150.00);

-- ✓ Fast Write: แค่ 2 INSERTs
-- ✓ No redundancy
```

```sql
-- แต่ Read ต้องใช้ JOIN ซับซ้อน
SELECT
    o.order_id,
    o.order_date,
    c.first_name || ' ' || c.last_name AS customer_name,
    c.email,
    SUM(oi.quantity * oi.unit_price) AS total_amount,
    STRING_AGG(p.name, ', ') AS products
FROM orders o
JOIN customers c ON o.customer_id = c.customer_id
JOIN order_items oi ON o.order_id = oi.order_id
JOIN products p ON oi.product_id = p.product_id
WHERE o.order_date >= CURRENT_DATE - INTERVAL '30 days'
GROUP BY o.order_id, o.order_date, c.first_name, c.last_name, c.email;
-- ช้าถ้า Orders มีหลาย million rows
```

### Denormalized Schema: ดีสำหรับ Read

```sql
-- Denormalized: Order Summary Table
CREATE TABLE order_summary (
    order_id      INTEGER PRIMARY KEY,
    order_date    DATE NOT NULL,
    customer_id   INTEGER,
    customer_name VARCHAR(200),  -- Denormalized!
    customer_email VARCHAR(255), -- Denormalized!
    total_amount  DECIMAL(10,2), -- Denormalized!
    item_count    INTEGER,       -- Denormalized!
    product_names TEXT           -- Denormalized!
);

-- Read เร็วมาก:
SELECT * FROM order_summary
WHERE order_date >= CURRENT_DATE - INTERVAL '30 days';
-- ไม่ต้อง JOIN เลย!

-- แต่ Write ซับซ้อนขึ้น:
-- ต้องอัปเดต order_summary ทุกครั้งที่:
-- - เพิ่ม/ลบ order item
-- - เปลี่ยนชื่อลูกค้า
-- - เปลี่ยนราคาสินค้า
```

---

## Pattern ที่ 1: Derived Columns (คอลัมน์คำนวณ)

### เมื่อใดควรเก็บ Derived Values?

```sql
-- ❌ ไม่เก็บ: คำนวณทุกครั้งใน Query (ช้าถ้าข้อมูลเยอะ)
SELECT
    order_id,
    SUM(quantity * unit_price) AS total
FROM order_items
GROUP BY order_id;

-- ✅ เก็บ Derived Value เมื่อ Query บ่อยมาก
ALTER TABLE orders ADD COLUMN total_amount DECIMAL(10,2);

-- อัปเดตเมื่อมีการเปลี่ยนแปลง:
CREATE OR REPLACE FUNCTION update_order_total()
RETURNS TRIGGER AS $$
BEGIN
    UPDATE orders
    SET total_amount = (
        SELECT SUM(quantity * unit_price)
        FROM order_items
        WHERE order_id = COALESCE(NEW.order_id, OLD.order_id)
    )
    WHERE order_id = COALESCE(NEW.order_id, OLD.order_id);
    RETURN COALESCE(NEW, OLD);
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_update_order_total
    AFTER INSERT OR UPDATE OR DELETE ON order_items
    FOR EACH ROW
    EXECUTE FUNCTION update_order_total();
```

### Pattern 1a: Age Precomputation

```sql
-- ปัญหา: คำนวณ age ทุกครั้งช้า (บน 10M rows)
SELECT *
FROM customers
WHERE EXTRACT(YEAR FROM AGE(birth_date)) BETWEEN 25 AND 35;
-- Full Table Scan!

-- ✅ แก้: เก็บ age_group ไว้เลย
ALTER TABLE customers
ADD COLUMN age_group VARCHAR(20);

-- อัปเดตด้วย Cron Job หรือ Scheduled Task รายวัน/รายเดือน
UPDATE customers
SET age_group = CASE
    WHEN birth_date IS NULL THEN 'unknown'
    WHEN EXTRACT(YEAR FROM AGE(birth_date)) < 18 THEN 'under_18'
    WHEN EXTRACT(YEAR FROM AGE(birth_date)) BETWEEN 18 AND 24 THEN '18-24'
    WHEN EXTRACT(YEAR FROM AGE(birth_date)) BETWEEN 25 AND 34 THEN '25-34'
    WHEN EXTRACT(YEAR FROM AGE(birth_date)) BETWEEN 35 AND 44 THEN '35-44'
    ELSE '45+'
END;

CREATE INDEX idx_customers_age_group ON customers(age_group);

-- Query เร็วมาก:
SELECT * FROM customers WHERE age_group = '25-34';
```

---

## Pattern ที่ 2: Pre-aggregated Data (ข้อมูล Aggregate ล่วงหน้า)

### Pattern 2a: Counter Columns

```sql
-- ❌ ช้า: นับทุกครั้ง
SELECT post_id, COUNT(*) AS like_count
FROM likes
WHERE post_id = 12345;

-- ✅ เก็บ Counter ใน Posts table
ALTER TABLE posts ADD COLUMN like_count INTEGER DEFAULT 0;
ALTER TABLE posts ADD COLUMN comment_count INTEGER DEFAULT 0;
ALTER TABLE posts ADD COLUMN share_count INTEGER DEFAULT 0;

-- Maintain ด้วย Trigger
CREATE OR REPLACE FUNCTION update_post_like_count()
RETURNS TRIGGER AS $$
BEGIN
    IF TG_OP = 'INSERT' THEN
        UPDATE posts SET like_count = like_count + 1 WHERE post_id = NEW.post_id;
    ELSIF TG_OP = 'DELETE' THEN
        UPDATE posts SET like_count = like_count - 1 WHERE post_id = OLD.post_id;
    END IF;
    RETURN COALESCE(NEW, OLD);
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_post_like_count
    AFTER INSERT OR DELETE ON likes
    FOR EACH ROW
    EXECUTE FUNCTION update_post_like_count();
```

### Pattern 2b: Daily/Monthly Aggregation Tables

```sql
-- สำหรับ Reports ที่ต้องการข้อมูลรายวัน

-- Normalized: ต้อง Aggregate ทุกครั้ง (ช้า!)
SELECT
    DATE_TRUNC('day', created_at) AS day,
    COUNT(*) AS order_count,
    SUM(total_amount) AS revenue
FROM orders
GROUP BY 1
ORDER BY 1;

-- ✅ Denormalized: Daily Summary Table
CREATE TABLE daily_order_summary (
    summary_date    DATE PRIMARY KEY,
    order_count     INTEGER NOT NULL DEFAULT 0,
    total_revenue   DECIMAL(15,2) NOT NULL DEFAULT 0,
    avg_order_value DECIMAL(10,2),
    new_customer_count INTEGER DEFAULT 0,
    returning_customer_count INTEGER DEFAULT 0,
    last_updated    TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Populate ด้วย Scheduled Job (ทุกคืน)
INSERT INTO daily_order_summary (summary_date, order_count, total_revenue, avg_order_value)
SELECT
    DATE_TRUNC('day', created_at)::DATE AS day,
    COUNT(*) AS order_count,
    SUM(total_amount) AS total_revenue,
    AVG(total_amount) AS avg_order_value
FROM orders
WHERE created_at::DATE = CURRENT_DATE - 1
GROUP BY 1
ON CONFLICT (summary_date) DO UPDATE
SET
    order_count = EXCLUDED.order_count,
    total_revenue = EXCLUDED.total_revenue,
    avg_order_value = EXCLUDED.avg_order_value,
    last_updated = CURRENT_TIMESTAMP;

-- Query เร็วมาก:
SELECT * FROM daily_order_summary
WHERE summary_date BETWEEN '2024-01-01' AND '2024-03-31';
```

---

## Pattern ที่ 3: Redundant Data (ข้อมูลซ้ำ)

### Pattern 3a: Snapshot Data

```sql
-- ข้อมูลที่ต้องการ "ณ เวลานั้น" ไม่เปลี่ยนแปลงตาม Current State

-- ❌ ปัญหา: ถ้าไม่เก็บ Snapshot ราคาของสินค้า
CREATE TABLE order_items_bad (
    order_id    INTEGER,
    product_id  INTEGER REFERENCES products,
    quantity    INTEGER
    -- ไม่มี unit_price! อ้างถึง products.price ตลอด
);

-- ปัญหา: ถ้าสินค้าเปลี่ยนราคา ยอดคำสั่งซื้อเก่าจะเปลี่ยนตามด้วย!

-- ✅ เก็บ Snapshot Price
CREATE TABLE order_items (
    order_id    INTEGER,
    product_id  INTEGER REFERENCES products,
    quantity    INTEGER NOT NULL,
    unit_price  DECIMAL(10,2) NOT NULL,  -- Snapshot ณ วันที่สั่งซื้อ
    product_name VARCHAR(200),           -- Optional: Snapshot ชื่อสินค้าด้วย
    PRIMARY KEY (order_id, product_id)
);
```

### Pattern 3b: Denormalized Foreign Data

```sql
-- สำหรับ Read-Heavy Tables เช่น Activity Logs
CREATE TABLE activity_logs (
    log_id       BIGSERIAL PRIMARY KEY,
    user_id      INTEGER NOT NULL,
    user_name    VARCHAR(100),        -- Denormalized!
    user_email   VARCHAR(255),        -- Denormalized!
    action       VARCHAR(100) NOT NULL,
    resource_type VARCHAR(50),
    resource_id  INTEGER,
    resource_name VARCHAR(200),       -- Denormalized!
    ip_address   INET,
    created_at   TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);
-- เหตุผล: ถ้า user ลบบัญชี ยังเห็น log ที่มีชื่อผู้ใช้
-- ไม่ต้อง JOIN users table ทุกครั้งที่ดู logs
```

---

## Pattern ที่ 4: Materialized Views

### พื้นฐาน Materialized View

```sql
-- Materialized View = View ที่เก็บผลลัพธ์จริง ไม่ใช่แค่ Query

-- สร้าง Materialized View สำหรับ Product Summary
CREATE MATERIALIZED VIEW product_sales_summary AS
SELECT
    p.product_id,
    p.name AS product_name,
    p.category_id,
    c.name AS category_name,
    COUNT(DISTINCT oi.order_id) AS total_orders,
    SUM(oi.quantity) AS total_units_sold,
    SUM(oi.quantity * oi.unit_price) AS total_revenue,
    AVG(oi.unit_price) AS avg_selling_price,
    MAX(o.order_date) AS last_order_date
FROM products p
LEFT JOIN categories c ON p.category_id = c.category_id
LEFT JOIN order_items oi ON p.product_id = oi.product_id
LEFT JOIN orders o ON oi.order_id = o.order_id
GROUP BY p.product_id, p.name, p.category_id, c.name
WITH DATA;

-- สร้าง Index บน Materialized View
CREATE INDEX idx_pss_category ON product_sales_summary(category_id);
CREATE INDEX idx_pss_revenue ON product_sales_summary(total_revenue DESC);

-- Query เร็วมาก!
SELECT *
FROM product_sales_summary
WHERE category_id = 5
ORDER BY total_revenue DESC
LIMIT 10;

-- Refresh ข้อมูล (อาจใช้เวลา)
REFRESH MATERIALIZED VIEW product_sales_summary;

-- Refresh แบบ Non-blocking (PostgreSQL 9.4+)
REFRESH MATERIALIZED VIEW CONCURRENTLY product_sales_summary;
-- ต้องมี UNIQUE Index ก่อน:
CREATE UNIQUE INDEX idx_pss_product ON product_sales_summary(product_id);
```

### Materialized View สำหรับ Analytics

```sql
-- Customer Analytics Materialized View
CREATE MATERIALIZED VIEW customer_analytics AS
SELECT
    c.customer_id,
    c.first_name,
    c.last_name,
    c.email,
    c.created_at AS member_since,
    COUNT(DISTINCT o.order_id) AS total_orders,
    SUM(o.total_amount) AS lifetime_value,
    AVG(o.total_amount) AS avg_order_value,
    MAX(o.created_at) AS last_order_date,
    MIN(o.created_at) AS first_order_date,
    CASE
        WHEN COUNT(DISTINCT o.order_id) = 0 THEN 'prospect'
        WHEN COUNT(DISTINCT o.order_id) = 1 THEN 'one-time'
        WHEN COUNT(DISTINCT o.order_id) BETWEEN 2 AND 5 THEN 'occasional'
        ELSE 'loyal'
    END AS customer_segment,
    EXTRACT(DAY FROM NOW() - MAX(o.created_at)) AS days_since_last_order
FROM customers c
LEFT JOIN orders o ON c.customer_id = o.customer_id AND o.status = 'delivered'
GROUP BY c.customer_id, c.first_name, c.last_name, c.email, c.created_at
WITH DATA;

CREATE UNIQUE INDEX idx_ca_customer ON customer_analytics(customer_id);
CREATE INDEX idx_ca_segment ON customer_analytics(customer_segment);
CREATE INDEX idx_ca_ltv ON customer_analytics(lifetime_value DESC);

-- Auto-refresh ด้วย Cron Job (ทุกชั่วโมง)
-- หรือใช้ pg_cron extension:
-- SELECT cron.schedule('refresh-customer-analytics', '0 * * * *',
--   'REFRESH MATERIALIZED VIEW CONCURRENTLY customer_analytics');
```

---

## Pattern ที่ 5: Wide Tables (Denormalized Reporting)

```sql
-- สำหรับ BI/Analytics Tools ที่ต้องการ Wide Table

-- Normalized Schema ต้องการ JOIN 6+ ตาราง:
-- orders + customers + products + categories + suppliers + addresses

-- ✅ Denormalized Wide Table สำหรับ Reporting
CREATE TABLE fact_sales (
    sale_id           BIGSERIAL PRIMARY KEY,
    
    -- Order Info
    order_id          INTEGER,
    order_date        DATE NOT NULL,
    order_month       CHAR(7),           -- '2024-01'
    order_quarter     CHAR(6),           -- '2024Q1'
    order_year        SMALLINT,
    
    -- Customer Info (Snapshot)
    customer_id       INTEGER,
    customer_name     VARCHAR(200),
    customer_email    VARCHAR(255),
    customer_city     VARCHAR(100),
    customer_segment  VARCHAR(20),
    
    -- Product Info (Snapshot)
    product_id        INTEGER,
    product_name      VARCHAR(200),
    product_sku       VARCHAR(50),
    category_id       INTEGER,
    category_name     VARCHAR(100),
    supplier_id       INTEGER,
    supplier_name     VARCHAR(200),
    
    -- Sales Metrics
    quantity          INTEGER NOT NULL,
    unit_price        DECIMAL(10,2) NOT NULL,
    line_total        DECIMAL(10,2) NOT NULL,
    
    -- Date Dimensions
    is_weekend        BOOLEAN,
    day_of_week       SMALLINT
);

-- Populate ด้วย ETL Process:
INSERT INTO fact_sales (
    order_id, order_date, order_month, order_year,
    customer_id, customer_name, customer_city,
    product_id, product_name, category_name,
    quantity, unit_price, line_total
)
SELECT
    o.order_id,
    o.created_at::DATE,
    TO_CHAR(o.created_at, 'YYYY-MM'),
    EXTRACT(YEAR FROM o.created_at)::SMALLINT,
    c.customer_id,
    c.first_name || ' ' || c.last_name,
    c.city,
    p.product_id, p.name,
    cat.name,
    oi.quantity,
    oi.unit_price,
    oi.quantity * oi.unit_price
FROM orders o
JOIN customers c ON o.customer_id = c.customer_id
JOIN order_items oi ON o.order_id = oi.order_id
JOIN products p ON oi.product_id = p.product_id
LEFT JOIN categories cat ON p.category_id = cat.category_id;
```

---

## Pattern ที่ 6: Hybrid Approach

### ใช้ JSONB สำหรับ Flexible Attributes

```sql
-- สำหรับข้อมูลที่มีโครงสร้างหลากหลาย
CREATE TABLE products (
    product_id    SERIAL PRIMARY KEY,
    name          VARCHAR(200) NOT NULL,
    price         DECIMAL(10,2) NOT NULL,
    category_id   INTEGER REFERENCES categories,
    -- Normalized core attributes ↑
    
    -- Denormalized: flexible attributes ↓
    attributes    JSONB DEFAULT '{}'
);

-- Electronics: {"brand": "Samsung", "warranty_years": 2, "voltage": "220V"}
-- Clothing: {"material": "cotton", "care": "machine wash cold"}
-- Food: {"ingredients": [...], "allergens": [...], "calories": 250}

INSERT INTO products (name, price, category_id, attributes) VALUES
('iPhone 15', 35000, 1, '{"brand":"Apple","storage":"128GB","ram":"6GB","color":["black","white"]}'),
('T-Shirt M', 350, 2, '{"brand":"Uniqlo","material":"100% cotton","size":"M","care":"machine wash"}');

-- Query JSONB:
SELECT name, attributes->>'brand' AS brand, price
FROM products
WHERE attributes->>'brand' = 'Apple'
AND (attributes->>'storage') = '128GB';

-- Index บน JSONB
CREATE INDEX idx_products_attributes ON products USING gin(attributes);
```

---

## Pattern ที่ 7: Star Schema (Data Warehouse)

```sql
-- Star Schema: Fact Table + Dimension Tables
-- Dimension Tables เป็น Denormalized (ไม่ต้อง Normalize)

-- Dimension: Date (pre-generated)
CREATE TABLE dim_date (
    date_key    INTEGER PRIMARY KEY,  -- YYYYMMDD
    full_date   DATE UNIQUE NOT NULL,
    year        SMALLINT NOT NULL,
    quarter     SMALLINT NOT NULL,
    month       SMALLINT NOT NULL,
    month_name  VARCHAR(10) NOT NULL,
    day_of_month SMALLINT NOT NULL,
    day_of_week SMALLINT NOT NULL,
    day_name    VARCHAR(10) NOT NULL,
    is_weekend  BOOLEAN NOT NULL,
    is_holiday  BOOLEAN DEFAULT FALSE
);

-- Dimension: Customer (Denormalized)
CREATE TABLE dim_customer (
    customer_key  SERIAL PRIMARY KEY,
    customer_id   INTEGER NOT NULL,
    name          VARCHAR(200),
    email         VARCHAR(255),
    city          VARCHAR(100),
    country       VARCHAR(50),
    segment       VARCHAR(30),
    -- SCD Type 2: เก็บ History
    effective_date DATE NOT NULL,
    expiry_date    DATE,
    is_current     BOOLEAN DEFAULT TRUE
);

-- Dimension: Product (Denormalized)
CREATE TABLE dim_product (
    product_key   SERIAL PRIMARY KEY,
    product_id    INTEGER NOT NULL,
    sku           VARCHAR(50),
    name          VARCHAR(200),
    category      VARCHAR(100),
    subcategory   VARCHAR(100),
    brand         VARCHAR(100),
    price_range   VARCHAR(20),  -- 'budget', 'mid', 'premium'
    is_current    BOOLEAN DEFAULT TRUE
);

-- Fact Table: Sales
CREATE TABLE fact_sales (
    sale_key      BIGSERIAL PRIMARY KEY,
    date_key      INTEGER REFERENCES dim_date,
    customer_key  INTEGER REFERENCES dim_customer,
    product_key   INTEGER REFERENCES dim_product,
    -- Measures
    quantity      INTEGER NOT NULL,
    unit_price    DECIMAL(10,2) NOT NULL,
    discount_amount DECIMAL(10,2) DEFAULT 0,
    gross_amount  DECIMAL(10,2) NOT NULL,
    net_amount    DECIMAL(10,2) NOT NULL,
    cost_amount   DECIMAL(10,2)
);

-- Analytics Queries เร็วมาก (ไม่ต้อง Normalize เพิ่ม)
SELECT
    dd.year,
    dd.month_name,
    dp.category,
    SUM(fs.net_amount) AS revenue,
    COUNT(DISTINCT fs.customer_key) AS unique_customers
FROM fact_sales fs
JOIN dim_date dd ON fs.date_key = dd.date_key
JOIN dim_product dp ON fs.product_key = dp.product_key
WHERE dd.year = 2024
GROUP BY dd.year, dd.month_name, dp.category
ORDER BY dd.month, dp.category;
```

---

## Pattern ที่ 8: Flattened Hierarchies

```sql
-- ปัญหา: Category Tree ที่ Deep มาก
-- Electronics > Phones > Smartphones > Android

-- ❌ Normalized (Self-referencing): ต้อง Recursive Query
CREATE TABLE categories_normalized (
    category_id INTEGER PRIMARY KEY,
    parent_id   INTEGER REFERENCES categories_normalized,
    name        VARCHAR(100) NOT NULL
);

-- Query ที่ซับซ้อน:
WITH RECURSIVE category_path AS (
    SELECT category_id, name, parent_id, name::TEXT AS path
    FROM categories_normalized
    WHERE parent_id IS NULL
    
    UNION ALL
    
    SELECT c.category_id, c.name, c.parent_id, cp.path || ' > ' || c.name
    FROM categories_normalized c
    JOIN category_path cp ON c.parent_id = cp.category_id
)
SELECT * FROM category_path WHERE category_id = 150;
```

```sql
-- ✅ Denormalized: Materialized Path
CREATE TABLE categories_flat (
    category_id  INTEGER PRIMARY KEY,
    name         VARCHAR(100) NOT NULL,
    parent_id    INTEGER,
    -- Denormalized path info:
    level        INTEGER NOT NULL DEFAULT 0,
    path         TEXT NOT NULL,          -- '/1/5/12/150'
    path_names   TEXT,                   -- 'Electronics > Phones > Smartphones > Android'
    root_id      INTEGER,                -- Level 0 ancestor
    l1_id        INTEGER,                -- Level 1 ancestor
    l2_id        INTEGER,                -- Level 2 ancestor
    l1_name      VARCHAR(100),           -- Denormalized!
    l2_name      VARCHAR(100)            -- Denormalized!
);

-- Query เร็วมาก:
-- ดู category path:
SELECT path_names FROM categories_flat WHERE category_id = 150;

-- ดู descendants:
SELECT * FROM categories_flat WHERE path LIKE '/1/5/%';

-- Filter by root category:
SELECT * FROM categories_flat WHERE root_id = 1;
```

---

## Pattern ที่ 9: Pre-computed Rankings

```sql
-- ปัญหา: ต้องการ Ranking/Percentile ที่คำนวณเร็ว
CREATE TABLE product_rankings (
    product_id          INTEGER PRIMARY KEY REFERENCES products,
    
    -- Pre-computed Rankings (อัปเดตรายวัน)
    rank_overall        INTEGER,         -- Rank by total sales
    rank_in_category    INTEGER,         -- Rank within category
    rank_this_week      INTEGER,         -- Rank by this week's sales
    rank_this_month     INTEGER,
    
    -- Pre-computed Metrics
    total_units_sold    INTEGER DEFAULT 0,
    weekly_units_sold   INTEGER DEFAULT 0,
    monthly_revenue     DECIMAL(15,2) DEFAULT 0,
    avg_rating          DECIMAL(3,2),
    review_count        INTEGER DEFAULT 0,
    
    -- Percentiles
    revenue_percentile  DECIMAL(5,2),   -- 0-100
    
    last_updated        TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- อัปเดตด้วย Stored Procedure (รายวัน)
CREATE OR REPLACE PROCEDURE refresh_product_rankings()
LANGUAGE plpgsql AS $$
BEGIN
    -- อัปเดต total units sold
    UPDATE product_rankings pr
    SET
        total_units_sold = stats.total_qty,
        monthly_revenue = stats.month_revenue,
        last_updated = CURRENT_TIMESTAMP
    FROM (
        SELECT
            oi.product_id,
            SUM(oi.quantity) AS total_qty,
            SUM(CASE WHEN o.created_at >= NOW() - INTERVAL '30 days'
                     THEN oi.quantity * oi.unit_price ELSE 0 END) AS month_revenue
        FROM order_items oi
        JOIN orders o ON oi.order_id = o.order_id
        WHERE o.status = 'delivered'
        GROUP BY oi.product_id
    ) stats
    WHERE pr.product_id = stats.product_id;

    -- อัปเดต Rankings
    UPDATE product_rankings pr
    SET rank_overall = ranked.rank
    FROM (
        SELECT product_id, ROW_NUMBER() OVER (ORDER BY total_units_sold DESC) AS rank
        FROM product_rankings
    ) ranked
    WHERE pr.product_id = ranked.product_id;
END;
$$;
```

---

## Pattern ที่ 10: Caching Frequently Accessed Data

```sql
-- สำหรับข้อมูลที่อ่านบ่อยมากแต่เปลี่ยนนาน

-- ตัวอย่าง: User Profile ที่รวม Stats ไว้แล้ว
CREATE TABLE user_profiles_cached (
    user_id         INTEGER PRIMARY KEY REFERENCES users,
    username        VARCHAR(50),
    display_name    VARCHAR(100),
    avatar_url      VARCHAR(500),
    bio             TEXT,
    
    -- Cached Stats (อัปเดตเป็นระยะ)
    post_count      INTEGER DEFAULT 0,
    follower_count  INTEGER DEFAULT 0,
    following_count INTEGER DEFAULT 0,
    like_count      INTEGER DEFAULT 0,
    
    -- Cache Metadata
    cache_updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Maintain ด้วย Triggers
CREATE OR REPLACE FUNCTION update_user_follower_count()
RETURNS TRIGGER AS $$
BEGIN
    IF TG_OP = 'INSERT' THEN
        UPDATE user_profiles_cached
        SET follower_count = follower_count + 1
        WHERE user_id = NEW.following_id;
        
        UPDATE user_profiles_cached
        SET following_count = following_count + 1
        WHERE user_id = NEW.follower_id;
    ELSIF TG_OP = 'DELETE' THEN
        UPDATE user_profiles_cached
        SET follower_count = follower_count - 1
        WHERE user_id = OLD.following_id;
        
        UPDATE user_profiles_cached
        SET following_count = following_count - 1
        WHERE user_id = OLD.follower_id;
    END IF;
    RETURN COALESCE(NEW, OLD);
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_user_follower_count
    AFTER INSERT OR DELETE ON follows
    FOR EACH ROW
    EXECUTE FUNCTION update_user_follower_count();
```

---

## Pattern ที่ 11: Lookup Table Denormalization

```sql
-- สำหรับ Reference Data ที่เปลี่ยนน้อย
-- แทนที่จะ JOIN ทุกครั้ง เก็บ Short Description ไว้เลย

-- ❌ ต้อง JOIN ทุกครั้ง
SELECT
    o.order_id,
    os.description AS status_label
FROM orders o
JOIN order_statuses os ON o.status_code = os.code;

-- ✅ เก็บ Status Description ไว้เลย (ถ้า status เปลี่ยนน้อยมาก)
CREATE TABLE orders (
    order_id      INTEGER PRIMARY KEY,
    status_code   VARCHAR(20) NOT NULL,
    status_label  VARCHAR(50),          -- Denormalized!
    -- อัปเดต status_label ทุกครั้งที่เปลี่ยน status_code
    created_at    TIMESTAMP
);
```

---

## Pattern ที่ 12: Inverted Index / Search Optimization

```sql
-- สำหรับ Full-Text Search ที่เร็ว
CREATE TABLE products (
    product_id    SERIAL PRIMARY KEY,
    name          VARCHAR(200) NOT NULL,
    description   TEXT,
    brand         VARCHAR(100),
    
    -- Denormalized Search Fields
    search_text   TSVECTOR,  -- Pre-computed search index
    tags          TEXT[]     -- Denormalized tags for quick filter
);

-- Auto-maintain search_text
CREATE OR REPLACE FUNCTION update_product_search()
RETURNS TRIGGER AS $$
BEGIN
    NEW.search_text = 
        SETWEIGHT(TO_TSVECTOR('english', COALESCE(NEW.name, '')), 'A') ||
        SETWEIGHT(TO_TSVECTOR('english', COALESCE(NEW.brand, '')), 'B') ||
        SETWEIGHT(TO_TSVECTOR('english', COALESCE(NEW.description, '')), 'C');
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_product_search
    BEFORE INSERT OR UPDATE ON products
    FOR EACH ROW
    EXECUTE FUNCTION update_product_search();

CREATE INDEX idx_products_search ON products USING gin(search_text);

-- Full-text search เร็วมาก
SELECT name, brand, ts_rank(search_text, query) AS relevance
FROM products, TO_TSQUERY('english', 'wireless & headphones') query
WHERE search_text @@ query
ORDER BY relevance DESC;
```

---

## Pattern ที่ 13: Partitioned Summary Tables

```sql
-- สำหรับข้อมูลขนาดใหญ่ที่ต้องการ Query ตามช่วงเวลา

CREATE TABLE sales_summary_monthly (
    year_month   CHAR(7) NOT NULL,   -- '2024-01'
    product_id   INTEGER NOT NULL,
    category_id  INTEGER,
    
    -- Pre-aggregated metrics
    order_count  INTEGER NOT NULL DEFAULT 0,
    units_sold   INTEGER NOT NULL DEFAULT 0,
    gross_revenue DECIMAL(15,2) NOT NULL DEFAULT 0,
    avg_price    DECIMAL(10,2),
    
    PRIMARY KEY (year_month, product_id)
);

-- กรอง Query เร็วมาก:
SELECT
    year_month,
    SUM(gross_revenue) AS monthly_revenue,
    SUM(units_sold) AS units
FROM sales_summary_monthly
WHERE year_month BETWEEN '2024-01' AND '2024-12'
AND category_id = 5
GROUP BY year_month
ORDER BY year_month;

-- Compare with Year-Over-Year:
SELECT
    SUBSTRING(year_month, 1, 4) AS year,
    LPAD(SUBSTRING(year_month, 6, 2), 2) AS month,
    SUM(gross_revenue) AS revenue
FROM sales_summary_monthly
WHERE year_month BETWEEN '2023-01' AND '2024-12'
GROUP BY year, month
ORDER BY year, month;
```

---

## Pattern ที่ 14: Flattened User Permissions

```sql
-- ปัญหา: Permission ที่ต้อง Traverse หลาย Level
-- User → Roles → Permissions → Resources

-- ❌ ต้องใช้ Recursive Query ที่ซับซ้อน
WITH RECURSIVE user_permissions AS (
    -- Complex recursive query
    ...
)

-- ✅ Flattened Permission Cache
CREATE TABLE user_permission_cache (
    user_id      INTEGER REFERENCES users,
    resource     VARCHAR(100) NOT NULL,
    action       VARCHAR(50) NOT NULL,
    granted_at   TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (user_id, resource, action)
);

-- Rebuild Cache เมื่อ Role/Permission เปลี่ยน
CREATE OR REPLACE PROCEDURE rebuild_user_permission_cache(p_user_id INTEGER)
LANGUAGE plpgsql AS $$
BEGIN
    DELETE FROM user_permission_cache WHERE user_id = p_user_id;
    
    INSERT INTO user_permission_cache (user_id, resource, action)
    SELECT DISTINCT p_user_id, p.resource, p.action
    FROM user_roles ur
    JOIN role_permissions rp ON ur.role_id = rp.role_id
    JOIN permissions p ON rp.permission_id = p.permission_id
    WHERE ur.user_id = p_user_id;
END;
$$;

-- Permission Check เร็วมาก:
SELECT EXISTS(
    SELECT 1 FROM user_permission_cache
    WHERE user_id = 1001
    AND resource = 'orders'
    AND action = 'create'
) AS has_permission;
```

---

## Pattern ที่ 15: Hybrid OLTP/OLAP Tables

```sql
-- บางระบบต้องการทั้ง Real-time Updates (OLTP) และ Fast Analytics (OLAP)

-- OLTP Table (Normalized):
CREATE TABLE orders (
    order_id     INTEGER PRIMARY KEY,
    customer_id  INTEGER REFERENCES customers,
    status       VARCHAR(20),
    created_at   TIMESTAMP
);

-- OLAP Materialized Aggregation (Denormalized):
CREATE TABLE order_analytics (
    analytics_date DATE PRIMARY KEY,
    total_orders   INTEGER,
    completed_orders INTEGER,
    cancelled_orders INTEGER,
    total_revenue  DECIMAL(15,2),
    avg_order_value DECIMAL(10,2),
    new_customers  INTEGER,
    returning_customers INTEGER
);

-- Strategy:
-- 1. Write ลงใน OLTP ตาราง (Normalized)
-- 2. Background Process สร้าง OLAP aggregations
-- 3. Dashboard/Report อ่านจาก OLAP ตาราง

-- Background Refresh:
INSERT INTO order_analytics
SELECT
    CURRENT_DATE - 1 AS analytics_date,
    COUNT(*) AS total_orders,
    COUNT(*) FILTER (WHERE status = 'completed') AS completed_orders,
    COUNT(*) FILTER (WHERE status = 'cancelled') AS cancelled_orders,
    COALESCE(SUM(total_amount) FILTER (WHERE status = 'completed'), 0),
    AVG(total_amount) FILTER (WHERE status = 'completed'),
    COUNT(DISTINCT customer_id) FILTER (WHERE customer_id NOT IN (
        SELECT DISTINCT customer_id FROM orders
        WHERE created_at < CURRENT_DATE - 1
    )),
    COUNT(DISTINCT customer_id) FILTER (WHERE customer_id IN (
        SELECT DISTINCT customer_id FROM orders
        WHERE created_at < CURRENT_DATE - 1
    ))
FROM orders
WHERE created_at::DATE = CURRENT_DATE - 1
ON CONFLICT (analytics_date) DO UPDATE
SET
    total_orders = EXCLUDED.total_orders,
    total_revenue = EXCLUDED.total_revenue,
    avg_order_value = EXCLUDED.avg_order_value;
```

---

## เมื่อใดควร Denormalize?

```
Denormalize เมื่อ:
1. Query Performance ช้าเกินรับได้ (ลอง Indexing ก่อน!)
2. Reporting/Analytics ที่รันบ่อย ต้องการผลใน < 1 วินาที
3. Read : Write ratio สูงมาก (เช่น 100:1)
4. ข้อมูลที่ Denormalize เปลี่ยนแปลงน้อย

อย่า Denormalize เมื่อ:
1. ยังไม่ได้วัด Performance จริง (อย่า Premature Optimization)
2. ข้อมูลเปลี่ยนบ่อยมาก (Write Heavy)
3. Data Integrity สำคัญกว่า Performance
4. ยังมีวิธีอื่นที่ดีกว่า (Indexing, Query Optimization, Caching)

กฎทอง: "Normalize First, Denormalize When Needed"
```

---

## แบบฝึกหัด (10 ข้อ)

### ข้อ 1
อธิบาย Trade-off ระหว่าง Normalized และ Denormalized Schema

**เฉลย**:
```
Normalized:
ข้อดี: Data Integrity, ไม่ซ้ำซ้อน, Update ง่าย
ข้อเสีย: ต้อง JOIN มาก, Query ซับซ้อน, Read ช้ากว่า

Denormalized:
ข้อดี: Read เร็ว, Query ง่าย, Report เร็ว
ข้อเสีย: Data Redundancy, Update ต้องแก้หลายที่, ข้อมูลอาจขัดแย้ง

Decision: ขึ้นอยู่กับ Use Case
- OLTP (Online Transaction Processing): Normalized
- OLAP (Online Analytical Processing): Denormalized
- Mixed: ใช้ทั้งสอง + Materialized Views
```

### ข้อ 2
สร้าง Materialized View สำหรับ Monthly Revenue Report

**เฉลย**:
```sql
CREATE MATERIALIZED VIEW monthly_revenue AS
SELECT
    DATE_TRUNC('month', o.created_at)::DATE AS month,
    COUNT(DISTINCT o.order_id) AS total_orders,
    COUNT(DISTINCT o.customer_id) AS unique_customers,
    SUM(oi.quantity * oi.unit_price) AS gross_revenue,
    SUM(o.discount_amount) AS total_discounts,
    SUM(o.total_amount) AS net_revenue,
    AVG(o.total_amount) AS avg_order_value
FROM orders o
JOIN order_items oi ON o.order_id = oi.order_id
WHERE o.status IN ('completed', 'delivered')
GROUP BY DATE_TRUNC('month', o.created_at)
WITH DATA;

CREATE UNIQUE INDEX idx_mr_month ON monthly_revenue(month);

-- Refresh เดือนละครั้ง
REFRESH MATERIALIZED VIEW CONCURRENTLY monthly_revenue;
```

### ข้อ 3
เพิ่ม Counter Columns สำหรับ Blog Posts (like_count, comment_count, view_count)

**เฉลย**:
```sql
ALTER TABLE posts
ADD COLUMN like_count INTEGER NOT NULL DEFAULT 0,
ADD COLUMN comment_count INTEGER NOT NULL DEFAULT 0,
ADD COLUMN view_count INTEGER NOT NULL DEFAULT 0;

-- Trigger for likes
CREATE OR REPLACE FUNCTION maintain_like_count()
RETURNS TRIGGER AS $$
BEGIN
    IF TG_OP = 'INSERT' THEN
        UPDATE posts SET like_count = like_count + 1 WHERE post_id = NEW.post_id;
    ELSIF TG_OP = 'DELETE' THEN
        UPDATE posts SET like_count = GREATEST(like_count - 1, 0) WHERE post_id = OLD.post_id;
    END IF;
    RETURN COALESCE(NEW, OLD);
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_like_count AFTER INSERT OR DELETE ON likes
FOR EACH ROW EXECUTE FUNCTION maintain_like_count();

-- Trigger for comments
CREATE OR REPLACE FUNCTION maintain_comment_count()
RETURNS TRIGGER AS $$
BEGIN
    IF TG_OP = 'INSERT' THEN
        UPDATE posts SET comment_count = comment_count + 1 WHERE post_id = NEW.post_id;
    ELSIF TG_OP = 'DELETE' THEN
        UPDATE posts SET comment_count = GREATEST(comment_count - 1, 0) WHERE post_id = OLD.post_id;
    END IF;
    RETURN COALESCE(NEW, OLD);
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_comment_count AFTER INSERT OR DELETE ON comments
FOR EACH ROW EXECUTE FUNCTION maintain_comment_count();

-- Re-sync counts (ป้องกัน drift)
UPDATE posts p
SET
    like_count = (SELECT COUNT(*) FROM likes WHERE post_id = p.post_id),
    comment_count = (SELECT COUNT(*) FROM comments WHERE post_id = p.post_id);
```

### ข้อ 4
อธิบายความแตกต่างระหว่าง Materialized View และ Regular View

**เฉลย**:
```
Regular View:
- เป็นแค่ "saved query"
- ทุกครั้งที่ Query จะ execute ใหม่
- ข้อมูล Up-to-date เสมอ
- ช้าถ้า underlying query ซับซ้อน
- ไม่กิน Storage

Materialized View:
- เก็บผลลัพธ์จริงในตาราง
- Query เร็วมาก (read from stored results)
- ข้อมูลอาจ Stale (ต้อง REFRESH)
- กิน Storage (เท่ากับผลลัพธ์)
- รองรับ Indexing
- PostgreSQL รองรับ REFRESH CONCURRENTLY

เหมาะสำหรับ:
Regular View: Query ง่ายหรือข้อมูล real-time สำคัญ
Materialized View: Complex aggregations, ข้อมูล Near real-time ได้
```

### ข้อ 5
ออกแบบ Daily Sales Summary Table สำหรับ Retail System

**เฉลย**:
```sql
CREATE TABLE daily_sales (
    sale_date     DATE NOT NULL,
    store_id      INTEGER NOT NULL,
    
    -- Transaction Metrics
    transaction_count INTEGER NOT NULL DEFAULT 0,
    unique_customers  INTEGER NOT NULL DEFAULT 0,
    new_customers     INTEGER NOT NULL DEFAULT 0,
    
    -- Sales Metrics
    gross_sales   DECIMAL(15,2) NOT NULL DEFAULT 0,
    discounts     DECIMAL(15,2) NOT NULL DEFAULT 0,
    net_sales     DECIMAL(15,2) NOT NULL DEFAULT 0,
    
    -- Product Metrics
    units_sold    INTEGER NOT NULL DEFAULT 0,
    items_per_transaction DECIMAL(6,2),
    
    -- Payment Breakdown
    cash_amount   DECIMAL(15,2) DEFAULT 0,
    card_amount   DECIMAL(15,2) DEFAULT 0,
    
    last_updated  TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (sale_date, store_id)
);

-- Refresh Procedure
CREATE OR REPLACE PROCEDURE refresh_daily_sales(p_date DATE)
LANGUAGE plpgsql AS $$
BEGIN
    INSERT INTO daily_sales (sale_date, store_id, transaction_count, net_sales, units_sold)
    SELECT
        p_date,
        store_id,
        COUNT(DISTINCT transaction_id),
        SUM(net_amount),
        SUM(quantity)
    FROM sales_transactions
    WHERE sale_date = p_date
    GROUP BY store_id
    ON CONFLICT (sale_date, store_id) DO UPDATE
    SET
        transaction_count = EXCLUDED.transaction_count,
        net_sales = EXCLUDED.net_sales,
        units_sold = EXCLUDED.units_sold,
        last_updated = CURRENT_TIMESTAMP;
END;
$$;
```

### ข้อ 6
เขียน Trigger เพื่อ Maintain Denormalized Data อย่างถูกต้อง

**เฉลย**:
```sql
-- ตัวอย่าง: Maintain order_summary ใน orders table
-- เมื่อ order_items เปลี่ยน

CREATE OR REPLACE FUNCTION sync_order_totals()
RETURNS TRIGGER AS $$
DECLARE
    v_order_id INTEGER;
    v_subtotal DECIMAL(10,2);
    v_item_count INTEGER;
BEGIN
    -- ระบุ order_id ที่เกี่ยวข้อง
    v_order_id := COALESCE(NEW.order_id, OLD.order_id);
    
    -- คำนวณค่าใหม่
    SELECT
        COALESCE(SUM(quantity * unit_price), 0),
        COALESCE(COUNT(*), 0)
    INTO v_subtotal, v_item_count
    FROM order_items
    WHERE order_id = v_order_id;
    
    -- อัปเดต orders table
    UPDATE orders
    SET
        subtotal = v_subtotal,
        total_amount = v_subtotal + shipping_fee - discount_amount,
        item_count = v_item_count,
        updated_at = CURRENT_TIMESTAMP
    WHERE order_id = v_order_id;
    
    RETURN COALESCE(NEW, OLD);
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_sync_order_totals
    AFTER INSERT OR UPDATE OR DELETE ON order_items
    FOR EACH ROW
    EXECUTE FUNCTION sync_order_totals();
```

### ข้อ 7
เปรียบเทียบประสิทธิภาพ Normalized vs Denormalized สำหรับ Product Catalog

**เฉลย**:
```sql
-- Normalized (ต้อง JOIN 4 ตาราง)
EXPLAIN ANALYZE
SELECT
    p.name,
    c.name AS category,
    s.name AS supplier,
    p.price,
    AVG(r.rating) AS avg_rating,
    COUNT(r.review_id) AS review_count
FROM products p
JOIN categories c ON p.category_id = c.category_id
JOIN suppliers s ON p.supplier_id = s.supplier_id
LEFT JOIN reviews r ON p.product_id = r.product_id
WHERE p.is_active = TRUE
GROUP BY p.product_id, p.name, c.name, s.name, p.price;
-- อาจใช้เวลา 500ms+ บน 100k products

-- Denormalized (single table)
CREATE MATERIALIZED VIEW product_catalog AS
SELECT
    p.product_id,
    p.name,
    c.name AS category,
    s.name AS supplier,
    p.price,
    COALESCE(r.avg_rating, 0) AS avg_rating,
    COALESCE(r.review_count, 0) AS review_count
FROM products p
JOIN categories c ON p.category_id = c.category_id
JOIN suppliers s ON p.supplier_id = s.supplier_id
LEFT JOIN (
    SELECT product_id, AVG(rating) AS avg_rating, COUNT(*) AS review_count
    FROM reviews GROUP BY product_id
) r ON p.product_id = r.product_id
WHERE p.is_active = TRUE
WITH DATA;

-- Query เร็วมาก:
EXPLAIN ANALYZE
SELECT * FROM product_catalog WHERE category = 'Electronics';
-- ใช้เวลา < 10ms (ขึ้นอยู่กับ Index)
```

### ข้อ 8
ออกแบบ Snapshot Strategy สำหรับ E-Commerce Order Items

**เฉลย**:
```sql
-- Order Items ควร Snapshot ข้อมูลอะไรบ้าง?

CREATE TABLE order_items (
    order_item_id   SERIAL PRIMARY KEY,
    order_id        INTEGER NOT NULL REFERENCES orders,
    product_id      INTEGER NOT NULL REFERENCES products,
    variant_id      INTEGER REFERENCES product_variants,
    
    -- SNAPSHOTS (ข้อมูล ณ วันที่สั่งซื้อ)
    product_name    VARCHAR(200) NOT NULL,   -- Snapshot ชื่อสินค้า
    product_sku     VARCHAR(50),             -- Snapshot SKU
    variant_options JSONB,                   -- {"color":"red","size":"L"}
    
    -- Pricing Snapshot
    list_price      DECIMAL(10,2) NOT NULL,  -- ราคาปกติ ณ วันที่สั่ง
    unit_price      DECIMAL(10,2) NOT NULL,  -- ราคาที่ลูกค้าจ่าย
    discount_amount DECIMAL(10,2) DEFAULT 0,
    
    -- Quantity
    quantity        INTEGER NOT NULL CHECK (quantity > 0),
    
    -- Computed (Snapshot)
    line_total      DECIMAL(10,2) NOT NULL,  -- unit_price * quantity
    
    -- Audit
    created_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- เหตุผลที่ต้อง Snapshot:
-- 1. ราคาสินค้าอาจเปลี่ยน → ต้องเก็บราคา ณ วันที่ซื้อ
-- 2. ชื่อสินค้าอาจเปลี่ยน → ต้องเก็บชื่อ ณ วันที่ซื้อ
-- 3. สินค้าอาจถูกลบ → ยังต้องดู Order History ได้
-- 4. Tax/Discount อาจเปลี่ยน → Snapshot ทั้งหมด
```

### ข้อ 9
สร้าง Star Schema อย่างง่ายสำหรับ Sales Analytics

**เฉลย**:
```sql
-- Dimension: Date
CREATE TABLE dim_date (
    date_key  INTEGER PRIMARY KEY,
    date      DATE UNIQUE,
    year      SMALLINT, quarter SMALLINT, month SMALLINT,
    month_name VARCHAR(10), week_of_year SMALLINT,
    day_of_month SMALLINT, day_name VARCHAR(10),
    is_weekend BOOLEAN, is_holiday BOOLEAN
);

-- Dimension: Customer
CREATE TABLE dim_customer (
    customer_key SERIAL PRIMARY KEY,
    customer_id  INTEGER NOT NULL,
    name         VARCHAR(200),
    city         VARCHAR(100),
    country      VARCHAR(50),
    segment      VARCHAR(30),
    is_current   BOOLEAN DEFAULT TRUE
);

-- Dimension: Product
CREATE TABLE dim_product (
    product_key SERIAL PRIMARY KEY,
    product_id  INTEGER NOT NULL,
    name        VARCHAR(200),
    brand       VARCHAR(100),
    category    VARCHAR(100),
    subcategory VARCHAR(100),
    is_current  BOOLEAN DEFAULT TRUE
);

-- Fact: Sales
CREATE TABLE fact_sales (
    sale_key      BIGSERIAL PRIMARY KEY,
    date_key      INTEGER REFERENCES dim_date,
    customer_key  INTEGER REFERENCES dim_customer,
    product_key   INTEGER REFERENCES dim_product,
    quantity      INTEGER NOT NULL,
    unit_price    DECIMAL(10,2) NOT NULL,
    net_amount    DECIMAL(10,2) NOT NULL,
    gross_margin  DECIMAL(10,2)
);

-- Indexes for fast aggregation
CREATE INDEX idx_fs_date ON fact_sales(date_key);
CREATE INDEX idx_fs_customer ON fact_sales(customer_key);
CREATE INDEX idx_fs_product ON fact_sales(product_key);
```

### ข้อ 10
ออกแบบกลยุทธ์ Denormalization สำหรับ News Website ที่มีบทความ 10 ล้าน

**เฉลย**:
```sql
-- Strategy: ใช้ Materialized Views + Counter Columns

-- 1. Normalized Core Tables:
CREATE TABLE articles (
    article_id   SERIAL PRIMARY KEY,
    title        VARCHAR(300) NOT NULL,
    content      TEXT NOT NULL,
    author_id    INTEGER REFERENCES authors,
    category_id  INTEGER REFERENCES categories,
    published_at TIMESTAMP,
    is_published BOOLEAN DEFAULT FALSE
);

-- 2. Counter Columns (อัปเดต Real-time):
ALTER TABLE articles
ADD COLUMN view_count   INTEGER DEFAULT 0,
ADD COLUMN share_count  INTEGER DEFAULT 0,
ADD COLUMN comment_count INTEGER DEFAULT 0,
ADD COLUMN like_count   INTEGER DEFAULT 0;

-- 3. Search Optimization:
ALTER TABLE articles
ADD COLUMN search_tsv TSVECTOR;

CREATE INDEX idx_articles_search ON articles USING gin(search_tsv);

-- 4. Materialized View สำหรับ Homepage:
CREATE MATERIALIZED VIEW trending_articles AS
SELECT
    a.article_id,
    a.title,
    au.name AS author_name,
    c.name AS category,
    a.published_at,
    a.view_count,
    a.like_count,
    a.comment_count,
    -- Trending Score
    (a.view_count * 1.0 + a.like_count * 5.0 + a.comment_count * 3.0) /
    GREATEST(EXTRACT(EPOCH FROM NOW() - a.published_at) / 3600, 1) AS trending_score
FROM articles a
JOIN authors au ON a.author_id = au.author_id
JOIN categories c ON a.category_id = c.category_id
WHERE a.is_published = TRUE
AND a.published_at > NOW() - INTERVAL '7 days'
WITH DATA;

REFRESH MATERIALIZED VIEW CONCURRENTLY trending_articles;
-- Refresh ทุก 15 นาที

-- 5. Category Stats (อัปเดตรายชั่วโมง):
CREATE MATERIALIZED VIEW category_stats AS
SELECT
    c.category_id,
    c.name,
    COUNT(*) AS article_count,
    SUM(a.view_count) AS total_views,
    MAX(a.published_at) AS last_article_date
FROM categories c
JOIN articles a ON c.category_id = a.category_id
WHERE a.is_published = TRUE
GROUP BY c.category_id, c.name
WITH DATA;
```

---

*จบ Part 057: Denormalization - When and How*

**ในส่วนต่อไป (Part 058)**: เราจะเรียนรู้เรื่อง Database Schema Design Patterns ซึ่งเป็น Patterns ที่ใช้บ่อยในการแก้ปัญหาทั่วไป
