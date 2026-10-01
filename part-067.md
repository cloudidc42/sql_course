# Part 067: Table Statistics and the Query Planner

## Statistics และ Query Planner

---

## บทนำ

Query Planner คือส่วนที่ชาญฉลาดของ database ที่ตัดสินใจว่าจะ execute query อย่างไร โดยอ้างอิงจาก **statistics** ของข้อมูล ถ้า statistics ไม่ถูกต้องหรือล้าสมัย planner อาจเลือกแผนที่ไม่เหมาะสม ส่งผลให้ query ช้ากว่าที่ควรจะเป็น

---

## 1. How the Query Planner Uses Statistics

### Planner ทำงานอย่างไร?

```
Query → Parser → Analyzer → Rewriter → Planner → Executor

Planner รับ SQL ที่ถูก parse แล้ว แล้ว:
1. สร้าง "plan tree" ที่เป็นไปได้หลายๆ แบบ
2. ประมาณ cost ของแต่ละแบบโดยใช้ statistics
3. เลือกแบบที่มี cost ต่ำสุด
4. ส่งต่อให้ Executor
```

### ข้อมูลที่ Planner ใช้

```sql
-- Statistics ที่ planner ใช้ในการตัดสินใจ:
-- 1. จำนวนแถวในตาราง (n_live_tup)
-- 2. จำนวน distinct values ในแต่ละ column (n_distinct)
-- 3. Most common values (MCVs) และ frequencies
-- 4. Histogram boundaries (การกระจายของข้อมูล)
-- 5. Correlation ระหว่างค่าและ physical order

-- ดู statistics summary ของ PostgreSQL:
SELECT relname, reltuples, relpages
FROM pg_class
WHERE relname = 'employees';
```

---

## 2. pg_stats (PostgreSQL)

`pg_stats` เป็น view ที่แสดง column statistics ที่ planner ใช้

```sql
-- ดู statistics ทั้งหมดของ column
SELECT *
FROM pg_stats
WHERE tablename = 'employees'
AND attname = 'department_id';
```

### อธิบาย pg_stats columns สำคัญ

```sql
-- ดู statistics พร้อมคำอธิบาย
SELECT 
    attname AS column_name,
    
    -- null_frac: สัดส่วนของ NULL values (0.0 - 1.0)
    null_frac,
    
    -- avg_width: ความกว้างเฉลี่ยของค่า (bytes)
    avg_width,
    
    -- n_distinct: จำนวน distinct values
    -- ถ้า > 0: จำนวนจริง
    -- ถ้า < 0: fraction ของ total rows (เช่น -0.9 = 90% unique)
    n_distinct,
    
    -- correlation: ความสัมพันธ์ระหว่าง physical order และ logical order
    -- 1.0 = sorted perfectly (ดีสำหรับ index scan)
    -- 0.0 = random (Index Scan อาจช้า → prefer Seq Scan)
    -- -1.0 = reverse sorted
    correlation,
    
    -- most_common_vals: ค่าที่พบบ่อยที่สุด
    most_common_vals::text,
    
    -- most_common_freqs: ความถี่ของแต่ละ most common val
    most_common_freqs::text,
    
    -- histogram_bounds: ขอบเขตของ histogram buckets
    histogram_bounds::text

FROM pg_stats
WHERE tablename = 'orders'
ORDER BY attname;
```

### ตัวอย่างการอ่าน pg_stats

```sql
-- ตัวอย่างการอ่าน statistics สำหรับ column 'status' ในตาราง 'orders'
SELECT 
    attname,
    n_distinct,
    most_common_vals,
    most_common_freqs,
    null_frac
FROM pg_stats
WHERE tablename = 'orders' AND attname = 'status';

/*
ผลลัพธ์ตัวอย่าง:
attname  | n_distinct | most_common_vals                    | most_common_freqs
---------+------------+-------------------------------------+---------------------
status   | 5          | {completed,pending,cancelled,...}   | {0.73,0.15,0.08,...}

อ่านได้ว่า:
- status มี 5 ค่าที่ไม่ซ้ำ
- 'completed' มีความถี่ 73%
- 'pending' มีความถี่ 15%
- 'cancelled' มีความถี่ 8%
*/

-- Planner ใช้ข้อมูลนี้ประมาณว่า:
-- WHERE status = 'pending' → 15% ของแถว
-- WHERE status = 'cancelled' → 8% ของแถว
-- WHERE status = 'completed' → 73% ของแถว (Full Scan อาจดีกว่า!)
```

---

## 3. ANALYZE Command

```sql
-- อัพเดต statistics สำหรับ table ที่ระบุ
ANALYZE employees;

-- อัพเดต statistics สำหรับ columns ที่ระบุ
ANALYZE employees(department_id, salary, hire_date);

-- อัพเดต statistics ทั้ง schema
ANALYZE;

-- อัพเดต statistics พร้อมดู progress
ANALYZE VERBOSE employees;
/*
INFO:  analyzing "public.employees"
INFO:  "employees": scanned 1000 of 10000 pages, containing 100000 live rows 
       and 0 dead rows; 30000 rows in sample, 100000 estimated total rows
*/
```

### เมื่อใดควร ANALYZE

```sql
-- ตรวจสอบว่าตารางไหนที่ ANALYZE ล่าสุดเป็นเมื่อไหร่
SELECT 
    relname AS table_name,
    n_live_tup AS live_rows,
    n_dead_tup AS dead_rows,
    last_analyze,
    last_autoanalyze,
    CASE 
        WHEN last_analyze IS NULL AND last_autoanalyze IS NULL THEN 'NEVER'
        WHEN GREATEST(last_analyze, last_autoanalyze) < NOW() - INTERVAL '7 days' 
            THEN 'STALE (>7 days)'
        ELSE 'RECENT'
    END AS analyze_status,
    -- n_mod_since_analyze: rows modified since last analyze
    n_mod_since_analyze
FROM pg_stat_user_tables
WHERE schemaname = 'public'
ORDER BY n_mod_since_analyze DESC NULLS LAST;
```

---

## 4. Auto-analyze

PostgreSQL มี autovacuum daemon ที่รัน ANALYZE อัตโนมัติเมื่อข้อมูลเปลี่ยนแปลงมากพอ

```sql
-- ดู autovacuum settings
SHOW autovacuum;
SHOW autovacuum_analyze_scale_factor;
SHOW autovacuum_analyze_threshold;

-- Default trigger สำหรับ auto-analyze:
-- threshold + scale_factor * n_live_tup
-- = 50 + 0.2 * 100,000 = 20,050 rows modified

-- สำหรับตารางใหญ่ (100M rows):
-- = 50 + 0.2 * 100,000,000 = 20,000,050 rows ← ช้ามาก!
-- แก้โดย set scale_factor ต่ำลงสำหรับตารางใหญ่
```

```sql
-- ปรับ autovacuum settings สำหรับ large tables
ALTER TABLE large_events_table SET (
    autovacuum_analyze_scale_factor = 0.01,  -- 1% แทน 20%
    autovacuum_analyze_threshold = 1000       -- minimum 1000 rows
);

-- ตรวจสอบ table-level settings
SELECT 
    relname,
    reloptions
FROM pg_class
WHERE relname = 'large_events_table';
```

### Auto-analyze ใน MySQL

```sql
-- MySQL มี automatic statistics collection
-- ดูสถานะ:
SHOW VARIABLES LIKE 'innodb_stats_auto_recalc';
SHOW VARIABLES LIKE 'innodb_stats_persistent';

-- กำหนดจำนวน sample pages สำหรับ statistics:
SHOW VARIABLES LIKE 'innodb_stats_persistent_sample_pages';
-- Default = 20 pages (อาจน้อยเกินสำหรับตารางใหญ่)
ALTER TABLE large_table STATS_SAMPLE_PAGES = 200;

-- อัพเดต statistics:
ANALYZE TABLE employees;
ANALYZE TABLE employees UPDATE HISTOGRAM ON salary WITH 100 BUCKETS;  -- MySQL 8.0+
```

---

## 5. Statistics Target

Statistics target กำหนดความละเอียดของ statistics ที่เก็บ สูงกว่า = แม่นยำกว่า แต่ใช้ memory มากกว่าและ ANALYZE ช้ากว่า

```sql
-- ดู default statistics target
SHOW default_statistics_target;
-- Default = 100

-- เปลี่ยน statistics target สำหรับ column ที่สำคัญ
ALTER TABLE orders ALTER COLUMN customer_id SET STATISTICS 500;
ALTER TABLE orders ALTER COLUMN order_date SET STATISTICS 500;

-- รัน ANALYZE หลังเปลี่ยน statistics target
ANALYZE orders;

-- ดู custom statistics targets
SELECT 
    attname,
    attstattarget AS statistics_target
FROM pg_attribute
WHERE attrelid = 'orders'::regclass
AND attstattarget != -1  -- -1 = use default
ORDER BY attname;

-- Reset to default:
ALTER TABLE orders ALTER COLUMN customer_id SET STATISTICS -1;
```

### เมื่อใดควรเพิ่ม Statistics Target?

```sql
-- ตรวจสอบ estimated vs actual rows ที่ต่างกันมาก
EXPLAIN ANALYZE
SELECT * FROM orders WHERE customer_id = 1001;
/*
Index Scan ... (cost=... rows=5 ...)
              (actual time=... rows=500 loops=1)
-- estimated=5, actual=500 → ต่างกัน 100x! → ต้องเพิ่ม statistics target

แก้:
ALTER TABLE orders ALTER COLUMN customer_id SET STATISTICS 500;
ANALYZE orders;
*/
```

---

## 6. Column Statistics เชิงลึก

### Most Common Values (MCVs)

```sql
-- ดู MCVs สำหรับ column
SELECT 
    unnest(most_common_vals::text::text[]) AS value,
    unnest(most_common_freqs::float[]) AS frequency
FROM pg_stats
WHERE tablename = 'orders' AND attname = 'status'
ORDER BY frequency DESC;

/*
value     | frequency
----------+----------
completed | 0.73
pending   | 0.15
cancelled | 0.08
failed    | 0.03
refunded  | 0.01
*/
```

### Histogram Bounds

```sql
-- ดู histogram สำหรับ numeric/date column
SELECT 
    unnest(histogram_bounds::text::text[]) AS bound,
    generate_series(1, array_length(histogram_bounds, 1)) AS bucket
FROM pg_stats
WHERE tablename = 'orders' AND attname = 'total_amount';

-- Histogram แสดงการกระจายของข้อมูล
-- Planner ใช้ histogram เพื่อประมาณว่า WHERE total_amount BETWEEN X AND Y มีกี่ rows
```

---

## 7. Multi-column Statistics (PostgreSQL 10+)

เมื่อ columns มี correlation กัน (เช่น city และ zip_code) planner อาจประมาณค่าผิดจากการ assume independence

```sql
-- ปัญหา: Planner ประมาณผิดเพราะ assume independence
EXPLAIN ANALYZE
SELECT * FROM addresses WHERE city = 'Bangkok' AND zip_code = '10200';
-- estimated rows = total * P(city=Bangkok) * P(zip=10200) ← ผิด!
-- จริงๆ แล้ว city กับ zip_code correlate กัน

-- แก้: สร้าง Multi-column Statistics
CREATE STATISTICS stat_address_city_zip ON city, zip_code 
FROM addresses;

-- รัน ANALYZE เพื่อ collect statistics
ANALYZE addresses;

-- ตรวจสอบ statistics ที่สร้าง
SELECT 
    stxname,
    stxkeys,
    stxkind,  -- d=distinct, f=ndistinct, m=most common, e=expr
    stxdndistinct,
    stxdmcv
FROM pg_statistic_ext
JOIN pg_statistic_ext_data ON pg_statistic_ext.oid = pg_statistic_ext_data.stxoid
WHERE stxrelid = 'addresses'::regclass;
```

```sql
-- ตัวอย่างเพิ่มเติม: Multi-column stats สำหรับ correlated columns
-- product_id และ category_id มี correlation สูง (product อยู่ใน 1 category)
CREATE STATISTICS stat_product_category 
ON product_id, category_id 
FROM order_items;

-- first_name และ last_name ใช้ร่วมกันบ่อย
CREATE STATISTICS stat_employee_name (mcv)
ON first_name, last_name 
FROM employees;

ANALYZE employees;
ANALYZE order_items;
```

---

## 8. Stale Statistics Problems

### ปัญหาที่เกิดจาก Statistics ล้าสมัย

```sql
-- Scenario 1: Bulk load หลังจาก ANALYZE
-- ก่อน load: 1,000 rows
-- ANALYZE: statistics บอก 1,000 rows
-- Bulk INSERT: 10,000,000 rows
-- Statistics: ยังบอก 1,000 rows → Planner คิดตารางเล็ก!

-- ผล: Planner เลือก Nested Loop แทน Hash Join
-- เพราะคิดว่าตารางเล็ก

-- ตรวจสอบ:
SELECT relname, reltuples, pg_stat_user_tables.n_live_tup
FROM pg_class
JOIN pg_stat_user_tables ON pg_class.relname = pg_stat_user_tables.relname
WHERE pg_class.relname = 'my_table';
-- reltuples: statistics บอก (อาจล้าสมัย)
-- n_live_tup: count จริง (real-time)
```

```sql
-- Scenario 2: Seasonal data patterns
-- มกราคม: 1M orders
-- ANALYZE: statistics ถูกต้อง
-- ธันวาคม: 10M orders (holiday season)
-- Statistics: ยังบอก 1M → ประมาณ rows ผิด!

-- Solution: ตั้ง autovacuum aggressive สำหรับ seasonal tables
ALTER TABLE orders SET (
    autovacuum_analyze_scale_factor = 0.05,  -- analyze ทุก 5%
    autovacuum_vacuum_scale_factor = 0.05
);
```

---

## 9. Forcing Plan Recompilation

### PostgreSQL

```sql
-- PostgreSQL cache execution plans สำหรับ prepared statements
-- ปัญหา: plan ถูก cache ตอน statistics ยังผิด

-- ดู cached plans
SELECT 
    query,
    calls,
    mean_exec_time,
    plans,  -- number of times plan was generated
    total_plan_time
FROM pg_stat_statements
WHERE query LIKE '%orders%'
ORDER BY mean_exec_time DESC;

-- Force re-planning สำหรับ specific prepared statement:
DEALLOCATE ALL;  -- ล้าง all prepared statements

-- หรือใช้ pg_prepared_statements:
SELECT name, statement, from_sql FROM pg_prepared_statements;
DEALLOCATE my_prepared_statement;
```

```sql
-- ใน PostgreSQL 12+: Plan cache invalidation อัตโนมัติ
-- เมื่อ statistics เปลี่ยนมาก plan จะถูก recompute

-- สำหรับ function/procedure:
-- ใช้ EXECUTE ใน PL/pgSQL เพื่อ force re-plan:
CREATE OR REPLACE FUNCTION get_orders(p_customer_id INT)
RETURNS TABLE(order_id INT, order_date DATE) AS $$
BEGIN
    RETURN QUERY EXECUTE 
        'SELECT order_id, order_date FROM orders WHERE customer_id = $1'
        USING p_customer_id;
    -- EXECUTE forces fresh plan every time
END;
$$ LANGUAGE plpgsql;
```

### MySQL

```sql
-- MySQL Query Cache (deprecated ใน MySQL 8.0)
-- ล้าง Query Cache:
RESET QUERY CACHE;

-- ล้าง InnoDB Buffer Pool statistics:
FLUSH STATUS;

-- Force re-optimize (MySQL 5.7+):
-- ไม่มีวิธีตรงๆ แต่สามารถ:
-- 1. ANALYZE TABLE เพื่ออัพเดต statistics
-- 2. DELETE FROM query_cache (MySQL 5.6-)
-- 3. Restart MySQL (สุดขีด)
```

---

## 10. Statistics Inspection Queries สมบูรณ์

```sql
-- Query 1: Overview of table statistics
SELECT 
    c.relname AS table_name,
    c.reltuples::bigint AS estimated_rows,
    c.relpages AS pages,
    pg_size_pretty(pg_relation_size(c.oid)) AS table_size,
    s.n_live_tup AS actual_live_rows,
    s.n_dead_tup AS dead_rows,
    s.last_vacuum,
    s.last_analyze,
    s.last_autoanalyze,
    ROUND((s.n_dead_tup::float / NULLIF(s.n_live_tup + s.n_dead_tup, 0) * 100)::numeric, 2) 
        AS dead_ratio_pct,
    s.n_mod_since_analyze AS mods_since_analyze
FROM pg_class c
JOIN pg_stat_user_tables s ON c.relname = s.relname
WHERE c.relkind = 'r'
AND s.schemaname = 'public'
ORDER BY s.n_mod_since_analyze DESC NULLS LAST;
```

```sql
-- Query 2: Column statistics with analysis
SELECT 
    attname AS column_name,
    null_frac,
    CASE 
        WHEN n_distinct < 0 THEN ROUND(ABS(n_distinct) * c.reltuples)::bigint
        ELSE n_distinct::bigint
    END AS estimated_distinct_values,
    avg_width AS avg_bytes,
    correlation,
    CASE 
        WHEN ABS(correlation) > 0.9 THEN 'Highly correlated (good for index scan)'
        WHEN ABS(correlation) > 0.5 THEN 'Moderately correlated'
        ELSE 'Low correlation (prefer seq scan or bitmap)'
    END AS correlation_note
FROM pg_stats s
JOIN pg_class c ON c.relname = s.tablename
WHERE s.tablename = 'orders'
ORDER BY attname;
```

```sql
-- Query 3: Find tables needing ANALYZE
SELECT 
    schemaname,
    relname AS table_name,
    n_live_tup,
    n_mod_since_analyze,
    ROUND(n_mod_since_analyze::float / NULLIF(n_live_tup, 0) * 100, 2) AS pct_modified,
    last_autoanalyze,
    CASE 
        WHEN n_mod_since_analyze > n_live_tup * 0.1 THEN 'NEEDS ANALYZE NOW'
        WHEN n_mod_since_analyze > n_live_tup * 0.05 THEN 'Consider ANALYZE'
        ELSE 'OK'
    END AS recommendation
FROM pg_stat_user_tables
WHERE n_live_tup > 10000  -- เฉพาะตารางที่มีข้อมูลพอสมควร
ORDER BY pct_modified DESC NULLS LAST
LIMIT 20;
```

```sql
-- Query 4: Check statistics accuracy (compare estimated vs actual)
-- รัน query แล้วดู estimated vs actual rows
EXPLAIN (FORMAT JSON, ANALYZE) 
SELECT * FROM orders WHERE customer_id = 1001;

-- Parse JSON output เพื่อดู accuracy:
WITH plan AS (
    SELECT jsonb_path_query(
        (EXPLAIN (FORMAT JSON, ANALYZE) SELECT * FROM orders WHERE customer_id = 1001)::jsonb,
        '$.**.{"Actual Rows": @."Actual Rows", "Plan Rows": @."Plan Rows"}'
    ) AS node
)
SELECT 
    (node->>'Actual Rows')::int AS actual_rows,
    (node->>'Plan Rows')::int AS estimated_rows,
    ROUND(ABS((node->>'Actual Rows')::float / NULLIF((node->>'Plan Rows')::float, 0) - 1) * 100, 2) 
        AS estimation_error_pct
FROM plan;
```

---

## 11. Statistics Update Examples

```sql
-- ตัวอย่างที่สมบูรณ์: การจัดการ statistics

-- 1. ดูสถานะก่อน
SELECT relname, reltuples, last_analyze
FROM pg_class JOIN pg_stat_user_tables ON relname = relname_in_pg_stat
WHERE relname = 'large_orders';

-- 2. Manual ANALYZE
ANALYZE large_orders;

-- 3. ปรับ statistics target สำหรับ important columns
ALTER TABLE large_orders ALTER COLUMN customer_id SET STATISTICS 500;
ALTER TABLE large_orders ALTER COLUMN product_id SET STATISTICS 500;
ALTER TABLE large_orders ALTER COLUMN order_date SET STATISTICS 300;

-- 4. ANALYZE อีกครั้งหลังเพิ่ม statistics target
ANALYZE large_orders;

-- 5. สร้าง multi-column statistics สำหรับ correlated columns
CREATE STATISTICS stat_orders_customer_date 
ON customer_id, order_date 
FROM large_orders;
ANALYZE large_orders;

-- 6. ตรวจสอบผล
EXPLAIN ANALYZE
SELECT * FROM large_orders 
WHERE customer_id = 1001 AND order_date >= '2024-01-01';
-- ดูว่า estimated rows ใกล้เคียง actual rows มากขึ้นหรือไม่
```

---

## 12. Statistics ใน MySQL

```sql
-- MySQL 8.0 Histogram Statistics
-- สร้าง histogram สำหรับ column
ANALYZE TABLE orders UPDATE HISTOGRAM ON customer_id WITH 100 BUCKETS;
ANALYZE TABLE orders UPDATE HISTOGRAM ON total_amount WITH 50 BUCKETS;

-- ดู histogram
SELECT * FROM information_schema.COLUMN_STATISTICS 
WHERE TABLE_NAME = 'orders';

-- ลบ histogram
ANALYZE TABLE orders DROP HISTOGRAM ON customer_id;

-- ดู table statistics
SELECT 
    TABLE_NAME,
    TABLE_ROWS,
    AVG_ROW_LENGTH,
    DATA_LENGTH,
    INDEX_LENGTH,
    DATA_FREE,
    AUTO_INCREMENT,
    UPDATE_TIME
FROM information_schema.TABLES
WHERE TABLE_SCHEMA = DATABASE()
ORDER BY DATA_LENGTH + INDEX_LENGTH DESC;
```

---

## แบบฝึกหัด (10 ข้อ)

**ข้อ 1:** อธิบายว่า `n_distinct = -0.8` ใน pg_stats หมายความว่าอะไร

**เฉลยข้อ 1:**
`n_distinct = -0.8` หมายถึง 80% ของแถวมีค่าที่ไม่ซ้ำกัน (negative value = fraction of rows) ดังนั้นถ้าตารางมี 1,000,000 แถว จะมีประมาณ 800,000 distinct values นั่นคือ cardinality สูงมาก เหมาะสำหรับ index

---

**ข้อ 2:** เขียน query เพื่อหา tables ที่ statistics มีอายุมากกว่า 3 วัน ใน PostgreSQL

**เฉลยข้อ 2:**
```sql
SELECT 
    schemaname,
    relname AS table_name,
    GREATEST(last_analyze, last_autoanalyze) AS last_stats_update,
    n_live_tup,
    n_mod_since_analyze
FROM pg_stat_user_tables
WHERE GREATEST(last_analyze, last_autoanalyze) < NOW() - INTERVAL '3 days'
   OR (last_analyze IS NULL AND last_autoanalyze IS NULL)
ORDER BY last_stats_update ASC NULLS FIRST;
```

---

**ข้อ 3:** correlation ใน pg_stats บ่งบอกอะไรเกี่ยวกับการเลือก Index Scan vs Sequential Scan?

**เฉลยข้อ 3:**
`correlation` บอกว่าข้อมูลในคอลัมน์เรียงสอดคล้องกับ physical order บน disk แค่ไหน:
- correlation ≈ 1.0: ข้อมูลเรียงสม่ำเสมอ → Index Scan ดีมาก (ข้อมูลอยู่ใกล้กัน)
- correlation ≈ 0: ข้อมูล random → Index Scan ต้องกระโดดไปมาบน disk → Bitmap Scan หรือ Seq Scan อาจดีกว่า
- Planner ใช้ correlation เพื่อปรับ cost estimate ของ Index Scan

---

**ข้อ 4:** เมื่อใดควรสร้าง Multi-column Statistics ใน PostgreSQL?

**เฉลยข้อ 4:**
ควรสร้างเมื่อ:
1. มี columns ที่มี functional dependency กัน (เช่น city → zip_code, product_id → category_id)
2. Query ที่ใช้ทั้งสอง columns พร้อมกันมี estimated rows ผิดมากจาก actual
3. Planner เลือก join type ผิดเพราะประมาณ rows ผิด

```sql
CREATE STATISTICS stat_name ON col1, col2 FROM table_name;
ANALYZE table_name;
```

---

**ข้อ 5:** สร้าง query เพื่อ monitor autovacuum progress สำหรับตาราง large_transactions

**เฉลยข้อ 5:**
```sql
-- ดู autovacuum progress แบบ real-time
SELECT 
    pid,
    phase,
    heap_blks_total,
    heap_blks_scanned,
    ROUND(heap_blks_scanned::float / NULLIF(heap_blks_total, 0) * 100, 2) AS pct_scanned,
    index_vacuum_count,
    num_dead_tuples
FROM pg_stat_progress_vacuum
WHERE relid = 'large_transactions'::regclass;

-- ดูว่า autovacuum กำลังทำงานหรือไม่:
SELECT * FROM pg_stat_activity WHERE query LIKE '%autovacuum%';
```

---

**ข้อ 6:** เปรียบเทียบ default_statistics_target = 100 และ 500 ส่งผลต่ออะไรบ้าง?

**เฉลยข้อ 6:**
- **100 (default)**: ANALYZE ตรวจ sample ขนาดเล็กกว่า → เร็วกว่า, แต่ MCVs/histograms มีน้อยกว่า → อาจประมาณ rows ผิดสำหรับข้อมูลที่ skewed
- **500**: ANALYZE ตรวจ sample ใหญ่กว่า → ช้ากว่า, MCVs/histograms ละเอียดกว่า → ประมาณแม่นยำกว่า สำหรับข้อมูลซับซ้อน

ควรเพิ่มสำหรับ columns ที่: (1) distribution skewed มาก (2) query ประมาณ rows ผิดบ่อย (3) เป็น join columns สำคัญ

---

**ข้อ 7:** MySQL Histogram ต่างจาก PostgreSQL Statistics อย่างไร?

**เฉลยข้อ 7:**
- **MySQL 8.0 Histograms**: ต้องสร้างด้วย `ANALYZE TABLE ... UPDATE HISTOGRAM` แบบ manual, เก็บใน `information_schema.COLUMN_STATISTICS`, รองรับ singleton values และ equi-height buckets
- **PostgreSQL pg_stats**: สร้างอัตโนมัติด้วย ANALYZE, มี MCVs แยกต่างหากจาก histogram, มี correlation, รองรับ multi-column statistics, สามารถ tune ด้วย statistics_target ต่อ column

---

**ข้อ 8:** Planner เลือก Seq Scan แทน Index Scan ทั้งที่มี index อยู่ อธิบายเหตุผล 3 ข้อ

**เฉลยข้อ 8:**
1. **Selectivity สูง**: Query ดึง > 20-30% ของตาราง → Seq Scan ใช้ sequential I/O ซึ่งเร็วกว่า random I/O ของ Index Scan
2. **Correlation ต่ำ**: ข้อมูล random distribution → Index Scan ต้องกระโดด I/O เยอะ → Seq Scan ดีกว่า
3. **Statistics ผิดพลาด**: Planner ประมาณว่ามี rows เยอะ (เพราะ statistics เก่า) → คิดว่าต้องดึงข้อมูลเยอะ → เลือก Seq Scan

---

**ข้อ 9:** สร้าง script ที่รัน ANALYZE สำหรับ tables ทุกตารางที่ modifications เกิน 10% ของ live rows

**เฉลยข้อ 9:**
```sql
DO $$
DECLARE
    r RECORD;
    v_sql TEXT;
BEGIN
    FOR r IN 
        SELECT schemaname, relname
        FROM pg_stat_user_tables
        WHERE n_live_tup > 10000
        AND n_mod_since_analyze > n_live_tup * 0.10
        ORDER BY n_mod_since_analyze DESC
    LOOP
        v_sql := format('ANALYZE %I.%I', r.schemaname, r.relname);
        RAISE NOTICE 'Running: %', v_sql;
        EXECUTE v_sql;
    END LOOP;
END $$;
```

---

**ข้อ 10:** อธิบายกรณีที่ Most Common Values (MCVs) ใน pg_stats ช่วย Planner ตัดสินใจได้ดีขึ้น

**เฉลยข้อ 10:**
MCVs ช่วย Planner ได้เมื่อ:
1. **Skewed Distribution**: `status = 'completed'` มี 80% ของ rows → Planner รู้ว่า Full Scan ดีกว่า Index Scan สำหรับค่านี้
2. **Selective Queries**: `status = 'cancelled'` มีแค่ 2% → Planner เลือก Index Scan
3. **JOIN Cardinality**: Planner ประมาณขนาด result set ของ join ได้แม่นยำกว่า
4. **NOT IN ที่มี MCVs**: Planner สามารถประมาณ "rows ที่ไม่ใช่ค่านี้" ได้ถูกต้อง

```sql
-- ดู MCVs และ frequencies
SELECT unnest(most_common_vals::text::text[]) AS val,
       unnest(most_common_freqs::float[]) AS freq
FROM pg_stats WHERE tablename='orders' AND attname='status'
ORDER BY freq DESC;
```

---

## สรุป

ใน Part 067 เราได้เรียนรู้:

1. **Query Planner** ทำงานอย่างไร และใช้ statistics อะไร
2. **pg_stats** - Column statistics ที่ planner อ้างอิง
3. **ANALYZE command** - อัพเดต statistics แบบ manual
4. **Auto-analyze** - การตั้งค่าและ tuning
5. **Statistics Target** - ปรับความละเอียดของ statistics
6. **MCVs และ Histograms** - ข้อมูลการกระจายของค่า
7. **Multi-column Statistics** - สำหรับ correlated columns
8. **Stale Statistics** - ปัญหาและวิธีแก้
9. **MySQL Histograms** - Manual histogram creation
10. **Monitoring queries** - ตรวจสอบสถานะ statistics

ใน Part 068 เราจะเรียนรู้เกี่ยวกับ Caching และ Buffer Strategies
