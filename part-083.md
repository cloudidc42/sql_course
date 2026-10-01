# Part 083: Materialized Views (Materialized Views)

## บทนำ

Materialized Views (วิวส์แบบเก็บข้อมูลจริง) แตกต่างจาก Regular Views ตรงที่มันเก็บผลลัพธ์ของ query ไว้จริงๆ ในฐานข้อมูล ทำให้ query เร็วขึ้นมากสำหรับ aggregation ที่ซับซ้อน แต่ข้อเสียคือข้อมูลอาจไม่ Real-time

---

## 1. Regular View vs Materialized View

```sql
-- Regular View: ไม่เก็บข้อมูล, รัน query ทุกครั้ง
CREATE VIEW v_sales_summary AS
SELECT 
    YEAR(order_date) AS year,
    MONTH(order_date) AS month,
    COUNT(*) AS total_orders,
    SUM(total_amount) AS revenue
FROM orders
GROUP BY YEAR(order_date), MONTH(order_date);
-- ทุกครั้งที่ SELECT จาก View นี้ จะรัน query ใหม่

-- Materialized View: เก็บข้อมูลจริง, เร็วกว่า
CREATE MATERIALIZED VIEW mv_sales_summary AS
SELECT 
    YEAR(order_date) AS year,
    MONTH(order_date) AS month,
    COUNT(*) AS total_orders,
    SUM(total_amount) AS revenue
FROM orders
GROUP BY YEAR(order_date), MONTH(order_date);
-- ข้อมูลถูกเก็บไว้จริง SELECT จะเร็วมาก

-- เปรียบเทียบ:
-- Regular View: ข้อมูลล่าสุดเสมอ แต่ช้าถ้า query ซับซ้อน
-- Materialized View: เร็วมาก แต่ต้อง REFRESH เพื่อได้ข้อมูลล่าสุด
```

---

## 2. CREATE MATERIALIZED VIEW (PostgreSQL)

```sql
-- ตัวอย่างที่ 1: Materialized View พื้นฐาน (PostgreSQL)
CREATE MATERIALIZED VIEW mv_product_sales AS
SELECT 
    p.product_id,
    p.product_name,
    p.category_id,
    SUM(od.quantity) AS total_quantity_sold,
    SUM(od.quantity * od.unit_price) AS total_revenue,
    COUNT(DISTINCT od.order_id) AS total_orders,
    AVG(od.unit_price) AS avg_selling_price
FROM products p
JOIN order_details od ON p.product_id = od.product_id
JOIN orders o ON od.order_id = o.order_id
WHERE o.status = 'completed'
GROUP BY p.product_id, p.product_name, p.category_id;

-- ตัวอย่างที่ 2: Materialized View พร้อม Data ตอนสร้าง
CREATE MATERIALIZED VIEW mv_customer_stats
WITH DATA AS  -- เก็บข้อมูลทันที (default)
SELECT 
    c.customer_id,
    c.first_name,
    c.last_name,
    c.email,
    COUNT(o.order_id) AS total_orders,
    SUM(o.total_amount) AS lifetime_value,
    AVG(o.total_amount) AS avg_order_value,
    MAX(o.order_date) AS last_order_date,
    MIN(o.order_date) AS first_order_date
FROM customers c
LEFT JOIN orders o ON c.customer_id = o.customer_id
WHERE o.status = 'completed' OR o.status IS NULL
GROUP BY c.customer_id, c.first_name, c.last_name, c.email;

-- ตัวอย่างที่ 3: Materialized View ที่ยังไม่ดึงข้อมูล
CREATE MATERIALIZED VIEW mv_pending_stats
WITH NO DATA AS  -- ยังไม่ populate ข้อมูล
SELECT 
    status,
    COUNT(*) AS count,
    SUM(total_amount) AS total
FROM orders
GROUP BY status;
-- ต้อง REFRESH ก่อนใช้งาน
```

---

## 3. REFRESH MATERIALIZED VIEW

```sql
-- ตัวอย่างที่ 4: REFRESH พื้นฐาน
REFRESH MATERIALIZED VIEW mv_product_sales;
-- Lock table ระหว่าง refresh (ผู้ใช้ไม่สามารถ query ได้)

-- ตัวอย่างที่ 5: REFRESH CONCURRENTLY (ไม่ Lock table)
-- ต้องมี UNIQUE index ก่อน
CREATE UNIQUE INDEX idx_mv_product_sales_unique 
ON mv_product_sales(product_id);

REFRESH MATERIALIZED VIEW CONCURRENTLY mv_product_sales;
-- ผู้ใช้ยังคง query ได้ระหว่าง refresh

-- ตัวอย่างที่ 6: ตรวจสอบเวลาล่าสุดที่ refresh
SELECT 
    schemaname,
    matviewname,
    pg_size_pretty(pg_total_relation_size(schemaname || '.' || matviewname)) AS size,
    hasindexes,
    ispopulated
FROM pg_matviews
WHERE schemaname = 'public'
ORDER BY matviewname;

-- ตัวอย่างที่ 7: Refresh หลาย Materialized Views
REFRESH MATERIALIZED VIEW mv_product_sales;
REFRESH MATERIALIZED VIEW mv_customer_stats;
REFRESH MATERIALIZED VIEW mv_sales_summary;

-- ตัวอย่างที่ 8: Refresh แบบ Conditional (ตรวจสอบก่อนว่าข้อมูลเปลี่ยนหรือไม่)
CREATE TABLE mv_refresh_log (
    view_name VARCHAR(100),
    refreshed_at TIMESTAMP,
    duration_seconds NUMERIC
);

-- Procedure สำหรับ refresh พร้อม logging
CREATE OR REPLACE PROCEDURE refresh_mv_with_log(p_view_name TEXT)
LANGUAGE plpgsql
AS $$
DECLARE
    v_start TIMESTAMP;
    v_end TIMESTAMP;
BEGIN
    v_start := NOW();
    
    EXECUTE 'REFRESH MATERIALIZED VIEW CONCURRENTLY ' || p_view_name;
    
    v_end := NOW();
    
    INSERT INTO mv_refresh_log (view_name, refreshed_at, duration_seconds)
    VALUES (p_view_name, v_end, EXTRACT(EPOCH FROM (v_end - v_start)));
    
    RAISE NOTICE 'Refreshed % in % seconds', p_view_name, 
        EXTRACT(EPOCH FROM (v_end - v_start));
END;
$$;

CALL refresh_mv_with_log('mv_product_sales');
```

---

## 4. Indexes on Materialized Views

```sql
-- ตัวอย่างที่ 9: สร้าง Index บน Materialized View
CREATE MATERIALIZED VIEW mv_order_analytics AS
SELECT 
    o.order_id,
    o.order_date,
    o.customer_id,
    o.status,
    SUM(od.quantity * od.unit_price) AS order_total,
    COUNT(od.product_id) AS item_count
FROM orders o
JOIN order_details od ON o.order_id = od.order_id
GROUP BY o.order_id, o.order_date, o.customer_id, o.status;

-- สร้าง Index ต่างๆ
CREATE UNIQUE INDEX idx_mv_order_analytics_pk 
    ON mv_order_analytics(order_id);

CREATE INDEX idx_mv_order_analytics_date 
    ON mv_order_analytics(order_date);

CREATE INDEX idx_mv_order_analytics_customer 
    ON mv_order_analytics(customer_id);

CREATE INDEX idx_mv_order_analytics_status 
    ON mv_order_analytics(status);

-- Composite Index
CREATE INDEX idx_mv_order_analytics_date_status 
    ON mv_order_analytics(order_date, status);

-- ตัวอย่างที่ 10: Partial Index บน Materialized View
CREATE INDEX idx_mv_large_orders
ON mv_order_analytics(order_total)
WHERE order_total > 1000;

-- ตัวอย่างที่ 11: ใช้ Materialized View กับ Index
-- Query เร็วเพราะมี Index
SELECT * FROM mv_order_analytics 
WHERE customer_id = 100 
  AND order_date >= '2024-01-01';

-- ใช้ EXPLAIN เพื่อดู execution plan
EXPLAIN ANALYZE
SELECT * FROM mv_order_analytics 
WHERE status = 'pending'
ORDER BY order_date DESC
LIMIT 50;
```

---

## 5. Automatic Refresh Strategies

```sql
-- ตัวอย่างที่ 12: Refresh ด้วย pg_cron (PostgreSQL extension)
-- ติดตั้ง pg_cron ก่อน
CREATE EXTENSION pg_cron;

-- Refresh ทุกชั่วโมง
SELECT cron.schedule(
    'refresh-mv-hourly',
    '0 * * * *',  -- ทุกชั่วโมง
    'REFRESH MATERIALIZED VIEW CONCURRENTLY mv_product_sales'
);

-- Refresh ทุกคืนตอนเที่ยงคืน
SELECT cron.schedule(
    'refresh-mv-daily',
    '0 0 * * *',  -- ทุกวันเวลาเที่ยงคืน
    $$REFRESH MATERIALIZED VIEW mv_customer_stats;
      REFRESH MATERIALIZED VIEW mv_sales_summary;$$
);

-- ดูรายการ Scheduled Jobs
SELECT * FROM cron.job;

-- ยกเลิก Job
SELECT cron.unschedule('refresh-mv-hourly');

-- ตัวอย่างที่ 13: Refresh โดยใช้ Trigger อัตโนมัติ
-- สร้าง Function สำหรับ Refresh เมื่อมีการเปลี่ยนแปลงใน Source Table
CREATE OR REPLACE FUNCTION fn_queue_mv_refresh()
RETURNS TRIGGER
LANGUAGE plpgsql
AS $$
BEGIN
    -- เพิ่ม job ในคิวแทนที่จะ refresh ทันที (ป้องกัน refresh ถี่เกินไป)
    INSERT INTO mv_refresh_queue (view_name, queued_at)
    VALUES ('mv_product_sales', NOW())
    ON CONFLICT (view_name) DO UPDATE
    SET queued_at = EXCLUDED.queued_at;
    
    RETURN NEW;
END;
$$;

-- Attach Trigger
CREATE TRIGGER trg_queue_product_sales_refresh
AFTER INSERT OR UPDATE OR DELETE ON order_details
FOR EACH STATEMENT
EXECUTE FUNCTION fn_queue_mv_refresh();

-- ตัวอย่างที่ 14: Smart Refresh - refresh เฉพาะเมื่อข้อมูลเปลี่ยน
CREATE TABLE mv_source_checksum (
    view_name VARCHAR(100) PRIMARY KEY,
    last_checksum BIGINT,
    last_refresh TIMESTAMP
);

CREATE OR REPLACE PROCEDURE smart_refresh_mv()
LANGUAGE plpgsql
AS $$
DECLARE
    v_current_checksum BIGINT;
    v_stored_checksum BIGINT;
BEGIN
    -- คำนวณ checksum ของข้อมูลปัจจุบัน
    SELECT SUM(HASHTEXT(order_id::TEXT || total_amount::TEXT))
    INTO v_current_checksum
    FROM orders WHERE status = 'completed';
    
    -- ดึง checksum ที่เก็บไว้
    SELECT last_checksum INTO v_stored_checksum
    FROM mv_source_checksum
    WHERE view_name = 'mv_product_sales';
    
    -- Refresh เฉพาะเมื่อข้อมูลเปลี่ยน
    IF v_current_checksum != v_stored_checksum OR v_stored_checksum IS NULL THEN
        REFRESH MATERIALIZED VIEW CONCURRENTLY mv_product_sales;
        
        INSERT INTO mv_source_checksum (view_name, last_checksum, last_refresh)
        VALUES ('mv_product_sales', v_current_checksum, NOW())
        ON CONFLICT (view_name) DO UPDATE
        SET last_checksum = EXCLUDED.last_checksum,
            last_refresh = EXCLUDED.last_refresh;
            
        RAISE NOTICE 'Refreshed mv_product_sales - data changed';
    ELSE
        RAISE NOTICE 'Skipped refresh - no data changes';
    END IF;
END;
$$;
```

---

## 6. MySQL Workaround - Manual Tables with Events

MySQL ไม่มี Materialized Views ในตัว แต่สามารถจำลองได้ด้วยตารางปกติ + Events

```sql
-- ตัวอย่างที่ 15: สร้าง Pseudo-Materialized View ใน MySQL
-- สร้างตารางที่เก็บ aggregated data
CREATE TABLE mv_product_sales_cache (
    product_id INT PRIMARY KEY,
    product_name VARCHAR(200),
    category_id INT,
    total_quantity_sold INT DEFAULT 0,
    total_revenue DECIMAL(15,2) DEFAULT 0,
    total_orders INT DEFAULT 0,
    last_refreshed TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_category (category_id),
    INDEX idx_revenue (total_revenue)
);

-- Stored Procedure สำหรับ Refresh
DELIMITER //
CREATE PROCEDURE sp_refresh_mv_product_sales()
BEGIN
    -- ล้างข้อมูลเก่า
    TRUNCATE TABLE mv_product_sales_cache;
    
    -- Insert ข้อมูลใหม่
    INSERT INTO mv_product_sales_cache 
        (product_id, product_name, category_id, total_quantity_sold, total_revenue, total_orders)
    SELECT 
        p.product_id,
        p.product_name,
        p.category_id,
        SUM(od.quantity),
        SUM(od.quantity * od.unit_price),
        COUNT(DISTINCT od.order_id)
    FROM products p
    JOIN order_details od ON p.product_id = od.product_id
    JOIN orders o ON od.order_id = o.order_id
    WHERE o.status = 'completed'
    GROUP BY p.product_id, p.product_name, p.category_id;
    
    -- Update timestamp
    UPDATE mv_product_sales_cache SET last_refreshed = NOW();
END //
DELIMITER ;

-- ตัวอย่างที่ 16: MySQL Event Scheduler สำหรับ Auto Refresh
-- เปิด Event Scheduler
SET GLOBAL event_scheduler = ON;

-- สร้าง Event ที่ Refresh ทุกคืน
CREATE EVENT ev_refresh_product_sales
ON SCHEDULE EVERY 1 DAY
STARTS '2024-01-01 02:00:00'  -- เริ่มตี 2
DO
    CALL sp_refresh_mv_product_sales();

-- Event ที่ Refresh ทุกชั่วโมง
CREATE EVENT ev_refresh_hourly
ON SCHEDULE EVERY 1 HOUR
DO BEGIN
    CALL sp_refresh_mv_product_sales();
    CALL sp_refresh_mv_customer_stats();
END;

-- ดู Events ที่มีอยู่
SHOW EVENTS;
SHOW EVENTS FROM your_database;

-- ตัวอย่างที่ 17: MySQL - Incremental Refresh (ดีกว่า Full Refresh)
CREATE TABLE mv_product_sales_cache (
    product_id INT PRIMARY KEY,
    total_revenue DECIMAL(15,2),
    last_order_date TIMESTAMP,
    last_refreshed TIMESTAMP
);

DELIMITER //
CREATE PROCEDURE sp_incremental_refresh_sales()
BEGIN
    DECLARE v_last_refresh TIMESTAMP;
    
    -- ดูเวลา refresh ล่าสุด
    SELECT MIN(last_refreshed) INTO v_last_refresh FROM mv_product_sales_cache;
    SET v_last_refresh = COALESCE(v_last_refresh, '1970-01-01');
    
    -- อัพเดทเฉพาะ products ที่มี orders ใหม่หลังจาก refresh ล่าสุด
    INSERT INTO mv_product_sales_cache (product_id, total_revenue, last_order_date, last_refreshed)
    SELECT 
        od.product_id,
        SUM(od.quantity * od.unit_price),
        MAX(o.order_date),
        NOW()
    FROM order_details od
    JOIN orders o ON od.order_id = o.order_id
    WHERE o.order_date > v_last_refresh
      AND o.status = 'completed'
    GROUP BY od.product_id
    ON DUPLICATE KEY UPDATE
        total_revenue = total_revenue + VALUES(total_revenue),
        last_order_date = GREATEST(last_order_date, VALUES(last_order_date)),
        last_refreshed = VALUES(last_refreshed);
END //
DELIMITER ;
```

---

## 7. SQL Server Indexed Views

SQL Server เรียก Materialized Views ว่า "Indexed Views"

```sql
-- ตัวอย่างที่ 18: SQL Server Indexed View
-- ต้องใช้ WITH SCHEMABINDING
CREATE VIEW v_product_sales_indexed
WITH SCHEMABINDING
AS
SELECT 
    p.product_id,
    p.product_name,
    SUM(od.quantity) AS total_quantity,
    SUM(od.quantity * od.unit_price) AS total_revenue,
    COUNT_BIG(*) AS record_count  -- ต้องมี COUNT_BIG(*) สำหรับ indexed view
FROM dbo.products p
JOIN dbo.order_details od ON p.product_id = od.product_id
JOIN dbo.orders o ON od.order_id = o.order_id
WHERE o.status = 'completed'
GROUP BY p.product_id, p.product_name;

-- สร้าง Clustered Index เพื่อ "Materialize" View
CREATE UNIQUE CLUSTERED INDEX idx_v_product_sales
ON v_product_sales_indexed(product_id);

-- หลังจากนี้ SQL Server จะเก็บข้อมูลจริงและ auto-maintain

-- ตัวอย่างที่ 19: SQL Server Indexed View - Simple Example
CREATE VIEW v_order_counts
WITH SCHEMABINDING
AS
SELECT 
    customer_id,
    COUNT_BIG(*) AS order_count,
    SUM(total_amount) AS total_spent
FROM dbo.orders
WHERE status != 'cancelled'
GROUP BY customer_id;

CREATE UNIQUE CLUSTERED INDEX idx_v_order_counts
ON v_order_counts(customer_id);

-- ใช้งาน (SQL Server auto-ใช้ indexed view ถ้าเหมาะสม)
SELECT customer_id, order_count, total_spent
FROM v_order_counts
WHERE customer_id = 1001;

-- บังคับใช้ Indexed View
SELECT customer_id, order_count, total_spent
FROM v_order_counts WITH (NOEXPAND)  -- ใช้ indexed view เสมอ
WHERE customer_id = 1001;
```

---

## 8. Use Cases สำหรับ Materialized Views

```sql
-- ตัวอย่างที่ 20: Reporting Dashboard
CREATE MATERIALIZED VIEW mv_executive_dashboard AS
SELECT 
    -- Today's metrics
    (SELECT COUNT(*) FROM orders WHERE DATE(order_date) = CURRENT_DATE) AS orders_today,
    (SELECT SUM(total_amount) FROM orders WHERE DATE(order_date) = CURRENT_DATE AND status = 'completed') AS revenue_today,
    
    -- This month's metrics
    (SELECT COUNT(*) FROM orders WHERE DATE_TRUNC('month', order_date) = DATE_TRUNC('month', CURRENT_DATE)) AS orders_this_month,
    (SELECT SUM(total_amount) FROM orders WHERE DATE_TRUNC('month', order_date) = DATE_TRUNC('month', CURRENT_DATE) AND status = 'completed') AS revenue_this_month,
    
    -- YTD metrics
    (SELECT COUNT(*) FROM orders WHERE EXTRACT(YEAR FROM order_date) = EXTRACT(YEAR FROM CURRENT_DATE)) AS orders_ytd,
    (SELECT SUM(total_amount) FROM orders WHERE EXTRACT(YEAR FROM order_date) = EXTRACT(YEAR FROM CURRENT_DATE) AND status = 'completed') AS revenue_ytd,
    
    -- Active customers
    (SELECT COUNT(DISTINCT customer_id) FROM orders WHERE order_date >= CURRENT_DATE - INTERVAL '30 days') AS active_customers_30d,
    
    NOW() AS last_refreshed;

-- ตัวอย่างที่ 21: Geographic Sales Analysis
CREATE MATERIALIZED VIEW mv_sales_by_region AS
SELECT 
    c.country,
    c.city,
    COUNT(DISTINCT o.order_id) AS total_orders,
    COUNT(DISTINCT o.customer_id) AS unique_customers,
    SUM(o.total_amount) AS total_revenue,
    AVG(o.total_amount) AS avg_order_value,
    EXTRACT(YEAR FROM o.order_date) AS year,
    EXTRACT(QUARTER FROM o.order_date) AS quarter
FROM orders o
JOIN customers c ON o.customer_id = c.customer_id
WHERE o.status = 'completed'
GROUP BY c.country, c.city, 
         EXTRACT(YEAR FROM o.order_date),
         EXTRACT(QUARTER FROM o.order_date);

CREATE INDEX idx_mv_sales_region ON mv_sales_by_region(country, year, quarter);

-- ตัวอย่างที่ 22: Product Recommendation Cache
CREATE MATERIALIZED VIEW mv_frequently_bought_together AS
SELECT 
    od1.product_id AS product_a,
    od2.product_id AS product_b,
    COUNT(*) AS times_bought_together,
    COUNT(*)::FLOAT / (
        SELECT COUNT(DISTINCT order_id) FROM order_details WHERE product_id = od1.product_id
    ) AS co_occurrence_rate
FROM order_details od1
JOIN order_details od2 ON od1.order_id = od2.order_id 
    AND od1.product_id < od2.product_id
GROUP BY od1.product_id, od2.product_id
HAVING COUNT(*) >= 5
ORDER BY times_bought_together DESC;

CREATE INDEX idx_mv_fbt_product_a ON mv_frequently_bought_together(product_a);
CREATE INDEX idx_mv_fbt_product_b ON mv_frequently_bought_together(product_b);

-- ตัวอย่างที่ 23: Time Series Analysis Cache
CREATE MATERIALIZED VIEW mv_hourly_order_stats AS
SELECT 
    DATE_TRUNC('hour', order_date) AS hour_slot,
    COUNT(*) AS order_count,
    SUM(total_amount) AS revenue,
    AVG(total_amount) AS avg_order_value,
    COUNT(DISTINCT customer_id) AS unique_customers
FROM orders
WHERE status = 'completed'
GROUP BY DATE_TRUNC('hour', order_date)
ORDER BY hour_slot;

CREATE UNIQUE INDEX idx_mv_hourly_stats ON mv_hourly_order_stats(hour_slot);

-- ตัวอย่างที่ 24: Inventory Analytics
CREATE MATERIALIZED VIEW mv_inventory_analytics AS
SELECT 
    p.product_id,
    p.product_name,
    p.category_id,
    c.category_name,
    p.units_in_stock,
    p.reorder_level,
    p.unit_cost,
    p.units_in_stock * p.unit_cost AS inventory_value,
    COALESCE(sold.avg_daily_sales, 0) AS avg_daily_sales,
    CASE 
        WHEN COALESCE(sold.avg_daily_sales, 0) = 0 THEN NULL
        ELSE p.units_in_stock / sold.avg_daily_sales
    END AS days_of_stock,
    CASE 
        WHEN p.units_in_stock = 0 THEN 'Out of Stock'
        WHEN p.units_in_stock <= p.reorder_level THEN 'Reorder Now'
        WHEN COALESCE(sold.avg_daily_sales, 0) > 0 
             AND p.units_in_stock / sold.avg_daily_sales < 14 THEN 'Low Stock'
        ELSE 'Adequate'
    END AS stock_status
FROM products p
JOIN categories c ON p.category_id = c.category_id
LEFT JOIN (
    SELECT 
        od.product_id,
        SUM(od.quantity) / NULLIF(COUNT(DISTINCT DATE(o.order_date)), 0) AS avg_daily_sales
    FROM order_details od
    JOIN orders o ON od.order_id = o.order_id
    WHERE o.order_date >= CURRENT_DATE - INTERVAL '30 days'
    GROUP BY od.product_id
) sold ON p.product_id = sold.product_id
WHERE p.discontinued = 0;

-- ตัวอย่างที่ 25: Search Index Cache (Full-text search optimization)
CREATE MATERIALIZED VIEW mv_product_search_index AS
SELECT 
    p.product_id,
    p.product_name,
    c.category_name,
    s.company_name AS supplier_name,
    p.description,
    p.unit_price,
    p.units_in_stock > 0 AS in_stock,
    to_tsvector('english', 
        COALESCE(p.product_name, '') || ' ' ||
        COALESCE(c.category_name, '') || ' ' ||
        COALESCE(p.description, '')
    ) AS search_vector
FROM products p
JOIN categories c ON p.category_id = c.category_id
JOIN suppliers s ON p.supplier_id = s.supplier_id
WHERE p.discontinued = 0;

-- Full-text search index
CREATE INDEX idx_mv_product_search ON mv_product_search_index 
USING GIN(search_vector);

-- ใช้งาน
SELECT product_id, product_name, category_name
FROM mv_product_search_index
WHERE search_vector @@ plainto_tsquery('english', 'wireless keyboard');
```

---

## 9. Refresh Automation Strategies

```sql
-- ตัวอย่างที่ 26: Refresh ผ่าน Function ที่ตรวจสอบ dependency
CREATE OR REPLACE FUNCTION refresh_all_materialized_views()
RETURNS VOID
LANGUAGE plpgsql
AS $$
DECLARE
    v_view_record RECORD;
BEGIN
    FOR v_view_record IN 
        SELECT matviewname 
        FROM pg_matviews 
        WHERE schemaname = 'public'
        ORDER BY matviewname
    LOOP
        BEGIN
            EXECUTE 'REFRESH MATERIALIZED VIEW CONCURRENTLY public.' || v_view_record.matviewname;
            RAISE NOTICE 'Refreshed: %', v_view_record.matviewname;
        EXCEPTION WHEN OTHERS THEN
            RAISE WARNING 'Failed to refresh %: %', v_view_record.matviewname, SQLERRM;
        END;
    END LOOP;
END;
$$;

-- เรียกใช้
SELECT refresh_all_materialized_views();

-- ตัวอย่างที่ 27: Dependency-aware Refresh Order
-- กำหนดลำดับการ refresh (ถ้า view B ต้องการ view A)
CREATE TABLE mv_refresh_order (
    view_name VARCHAR(100) PRIMARY KEY,
    refresh_order INT,
    refresh_interval_minutes INT,
    last_refreshed TIMESTAMP,
    is_concurrent BOOLEAN DEFAULT TRUE
);

INSERT INTO mv_refresh_order VALUES
('mv_raw_orders', 1, 15, NULL, TRUE),
('mv_product_sales', 2, 30, NULL, TRUE),
('mv_customer_stats', 3, 60, NULL, TRUE),
('mv_executive_dashboard', 4, 60, NULL, FALSE);

CREATE OR REPLACE PROCEDURE refresh_mv_by_schedule()
LANGUAGE plpgsql
AS $$
DECLARE
    v_row RECORD;
    v_next_refresh TIMESTAMP;
BEGIN
    FOR v_row IN 
        SELECT * FROM mv_refresh_order
        ORDER BY refresh_order
    LOOP
        v_next_refresh := COALESCE(v_row.last_refreshed, '1970-01-01') 
                          + (v_row.refresh_interval_minutes || ' minutes')::INTERVAL;
        
        IF NOW() >= v_next_refresh THEN
            IF v_row.is_concurrent THEN
                EXECUTE 'REFRESH MATERIALIZED VIEW CONCURRENTLY ' || v_row.view_name;
            ELSE
                EXECUTE 'REFRESH MATERIALIZED VIEW ' || v_row.view_name;
            END IF;
            
            UPDATE mv_refresh_order 
            SET last_refreshed = NOW()
            WHERE view_name = v_row.view_name;
            
            RAISE NOTICE 'Refreshed: %', v_row.view_name;
        END IF;
    END LOOP;
END;
$$;
```

---

## 10. Dropping and Managing Materialized Views

```sql
-- ตัวอย่างที่ 28: DROP MATERIALIZED VIEW
DROP MATERIALIZED VIEW IF EXISTS mv_product_sales;

-- DROP พร้อม CASCADE (ถ้ามี object อื่นอ้างอิง)
DROP MATERIALIZED VIEW IF EXISTS mv_product_sales CASCADE;

-- ตัวอย่างที่ 29: ดูขนาดของ Materialized Views
SELECT 
    matviewname AS view_name,
    pg_size_pretty(pg_total_relation_size(matviewname::regclass)) AS total_size,
    pg_size_pretty(pg_relation_size(matviewname::regclass)) AS data_size,
    ispopulated AS has_data
FROM pg_matviews
WHERE schemaname = 'public'
ORDER BY pg_total_relation_size(matviewname::regclass) DESC;

-- ตัวอย่างที่ 30: Rebuild Materialized View (หลังจาก schema เปลี่ยน)
-- ต้อง DROP แล้ว CREATE ใหม่ ไม่มี ALTER MATERIALIZED VIEW
DROP MATERIALIZED VIEW mv_product_sales;

CREATE MATERIALIZED VIEW mv_product_sales AS
SELECT 
    p.product_id,
    p.product_name,
    p.category_id,
    p.new_column,  -- column ใหม่ที่เพิ่มเข้ามา
    SUM(od.quantity) AS total_quantity_sold
FROM products p
JOIN order_details od ON p.product_id = od.product_id
GROUP BY p.product_id, p.product_name, p.category_id, p.new_column;

-- สร้าง Index ใหม่ด้วย
CREATE UNIQUE INDEX idx_mv_product_sales ON mv_product_sales(product_id);
```

---

## แบบฝึกหัด (Exercises)

**ข้อ 1:** สร้าง Materialized View ชื่อ `mv_monthly_sales` ที่สรุปยอดขายรายเดือนพร้อม Index บน year และ month

**คำตอบข้อ 1:**
```sql
CREATE MATERIALIZED VIEW mv_monthly_sales AS
SELECT 
    EXTRACT(YEAR FROM order_date) AS year,
    EXTRACT(MONTH FROM order_date) AS month,
    COUNT(*) AS total_orders,
    SUM(total_amount) AS revenue,
    COUNT(DISTINCT customer_id) AS unique_customers
FROM orders
WHERE status = 'completed'
GROUP BY EXTRACT(YEAR FROM order_date), EXTRACT(MONTH FROM order_date);

CREATE UNIQUE INDEX idx_mv_monthly_sales ON mv_monthly_sales(year, month);
```

**ข้อ 2:** เขียน REFRESH MATERIALIZED VIEW CONCURRENTLY พร้อมอธิบายข้อกำหนด

**คำตอบข้อ 2:**
```sql
-- ต้องมี UNIQUE index ก่อน
CREATE UNIQUE INDEX idx_mv_monthly_sales ON mv_monthly_sales(year, month);

-- ถึงจะ CONCURRENTLY refresh ได้
REFRESH MATERIALIZED VIEW CONCURRENTLY mv_monthly_sales;
-- ข้อดี: ผู้ใช้ยังคง query ได้ระหว่าง refresh
-- ข้อกำหนด: ต้องมี UNIQUE index และ view ต้องมีข้อมูลแล้ว (WITH DATA)
```

**ข้อ 3:** จำลอง Materialized View ใน MySQL โดยใช้ Table + Event

**คำตอบข้อ 3:**
```sql
CREATE TABLE mv_top_products (
    product_id INT PRIMARY KEY,
    product_name VARCHAR(200),
    total_sold INT,
    last_refreshed TIMESTAMP
);

DELIMITER //
CREATE PROCEDURE sp_refresh_top_products()
BEGIN
    TRUNCATE mv_top_products;
    INSERT INTO mv_top_products (product_id, product_name, total_sold, last_refreshed)
    SELECT p.product_id, p.product_name, SUM(od.quantity), NOW()
    FROM products p JOIN order_details od ON p.product_id = od.product_id
    GROUP BY p.product_id ORDER BY SUM(od.quantity) DESC LIMIT 100;
END //
DELIMITER ;

CREATE EVENT ev_refresh_top_products
ON SCHEDULE EVERY 1 HOUR DO CALL sp_refresh_top_products();
```

**ข้อ 4:** สร้าง SQL Server Indexed View บน orders table

**คำตอบข้อ 4:**
```sql
CREATE VIEW v_customer_order_summary
WITH SCHEMABINDING AS
SELECT customer_id, COUNT_BIG(*) AS order_count, SUM(total_amount) AS total_spent
FROM dbo.orders WHERE status != 'cancelled'
GROUP BY customer_id;

CREATE UNIQUE CLUSTERED INDEX idx_customer_order_summary ON v_customer_order_summary(customer_id);
```

**ข้อ 5:** เขียน PostgreSQL procedure สำหรับ refresh Materialized View ทั้งหมดพร้อม logging

**คำตอบข้อ 5:**
```sql
CREATE OR REPLACE PROCEDURE refresh_all_mv_with_log()
LANGUAGE plpgsql AS $$
DECLARE v_view RECORD; v_start TIMESTAMP;
BEGIN
    FOR v_view IN SELECT matviewname FROM pg_matviews WHERE schemaname = 'public' LOOP
        v_start := NOW();
        BEGIN
            EXECUTE 'REFRESH MATERIALIZED VIEW CONCURRENTLY public.' || v_view.matviewname;
            INSERT INTO mv_refresh_log VALUES (v_view.matviewname, NOW(), EXTRACT(EPOCH FROM NOW()-v_start));
        EXCEPTION WHEN OTHERS THEN
            RAISE WARNING 'Failed: % - %', v_view.matviewname, SQLERRM;
        END;
    END LOOP;
END; $$;
```

**ข้อ 6:** สร้าง Materialized View สำหรับ Product Search โดยใช้ Full-text search

**คำตอบข้อ 6:**
```sql
CREATE MATERIALIZED VIEW mv_product_search AS
SELECT p.product_id, p.product_name, p.description, c.category_name, p.unit_price,
       to_tsvector('english', p.product_name || ' ' || COALESCE(p.description,'') || ' ' || c.category_name) AS tsv
FROM products p JOIN categories c ON p.category_id = c.category_id
WHERE p.discontinued = 0;

CREATE INDEX idx_mv_product_search_tsv ON mv_product_search USING GIN(tsv);
CREATE UNIQUE INDEX idx_mv_product_search_id ON mv_product_search(product_id);
```

**ข้อ 7:** เขียน MySQL Event สำหรับ Archive ข้อมูล Order เก่ากว่า 1 ปีทุกสัปดาห์

**คำตอบข้อ 7:**
```sql
DELIMITER //
CREATE PROCEDURE sp_archive_old_orders()
BEGIN
    INSERT INTO orders_archive SELECT * FROM orders 
    WHERE order_date < DATE_SUB(NOW(), INTERVAL 1 YEAR) AND status IN ('completed', 'cancelled');
    DELETE FROM orders WHERE order_date < DATE_SUB(NOW(), INTERVAL 1 YEAR) AND status IN ('completed', 'cancelled');
END //
DELIMITER ;

CREATE EVENT ev_weekly_archive ON SCHEDULE EVERY 1 WEEK STARTS '2024-01-07 03:00:00'
DO CALL sp_archive_old_orders();
```

**ข้อ 8:** เขียน Query เพื่อดูขนาดและสถานะของ Materialized Views ทั้งหมด (PostgreSQL)

**คำตอบข้อ 8:**
```sql
SELECT matviewname, schemaname, ispopulated,
       pg_size_pretty(pg_total_relation_size(schemaname||'.'||matviewname)) AS size,
       (SELECT COUNT(*) FROM pg_indexes WHERE tablename = matviewname) AS index_count
FROM pg_matviews WHERE schemaname = 'public'
ORDER BY pg_total_relation_size(schemaname||'.'||matviewname) DESC;
```

**ข้อ 9:** สร้าง Materialized View สำหรับ Cohort Analysis

**คำตอบข้อ 9:**
```sql
CREATE MATERIALIZED VIEW mv_customer_cohort AS
SELECT 
    DATE_TRUNC('month', first_order.first_date) AS cohort_month,
    DATE_TRUNC('month', o.order_date) AS order_month,
    COUNT(DISTINCT o.customer_id) AS active_customers,
    EXTRACT(MONTH FROM AGE(DATE_TRUNC('month', o.order_date), DATE_TRUNC('month', first_order.first_date))) AS months_since_first
FROM orders o
JOIN (SELECT customer_id, MIN(order_date) AS first_date FROM orders GROUP BY customer_id) first_order
    ON o.customer_id = first_order.customer_id
GROUP BY DATE_TRUNC('month', first_order.first_date), DATE_TRUNC('month', o.order_date);

CREATE UNIQUE INDEX idx_mv_cohort ON mv_customer_cohort(cohort_month, order_month);
```

**ข้อ 10:** สร้าง Procedure ที่ Refresh Materialized View เฉพาะเมื่อข้อมูลใหม่ถูก insert

**คำตอบข้อ 10:**
```sql
CREATE TABLE mv_dirty_flags (view_name VARCHAR(100) PRIMARY KEY, is_dirty BOOLEAN DEFAULT FALSE);
INSERT INTO mv_dirty_flags VALUES ('mv_product_sales', FALSE);

CREATE OR REPLACE FUNCTION fn_mark_mv_dirty() RETURNS TRIGGER LANGUAGE plpgsql AS $$
BEGIN
    UPDATE mv_dirty_flags SET is_dirty = TRUE WHERE view_name = 'mv_product_sales';
    RETURN NEW;
END; $$;

CREATE TRIGGER trg_mark_dirty AFTER INSERT OR UPDATE OR DELETE ON order_details
FOR EACH STATEMENT EXECUTE FUNCTION fn_mark_mv_dirty();

CREATE OR REPLACE PROCEDURE conditional_refresh()
LANGUAGE plpgsql AS $$
BEGIN
    IF (SELECT is_dirty FROM mv_dirty_flags WHERE view_name = 'mv_product_sales') THEN
        REFRESH MATERIALIZED VIEW CONCURRENTLY mv_product_sales;
        UPDATE mv_dirty_flags SET is_dirty = FALSE WHERE view_name = 'mv_product_sales';
    END IF;
END; $$;
```

---

*จบ Part 083: Materialized Views*
