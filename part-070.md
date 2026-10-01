# Part 070: Performance Testing and Benchmarking

## การทดสอบและวัดประสิทธิภาพ SQL

---

## บทนำ

Performance Benchmarking คือการวัดประสิทธิภาพของ database อย่างเป็นระบบ เพื่อ:
- สร้าง **baseline** ก่อนและหลังการเปลี่ยนแปลง
- **Identify bottlenecks** - หาจุดที่ระบบช้า
- **Validate improvements** - ยืนยันว่าการ optimize ได้ผล
- **Capacity planning** - วางแผน infrastructure

---

## 1. pgbench (PostgreSQL)

pgbench เป็น built-in benchmarking tool ของ PostgreSQL

### พื้นฐาน pgbench

```bash
# สร้าง test data (scale factor 50 = ~5 million rows)
pgbench -i -s 50 -U postgres mydb

# รัน benchmark พื้นฐาน (TPC-B-like)
pgbench -U postgres -c 10 -j 2 -T 60 mydb
# -c 10: 10 concurrent clients
# -j 2: 2 worker threads
# -T 60: รัน 60 วินาที

# ผลลัพธ์:
# transaction type: <builtin: TPC-B (sort of)>
# scaling factor: 50
# query mode: simple
# number of clients: 10
# number of threads: 2
# duration: 60 s
# number of transactions actually processed: 45678
# latency average = 13.132 ms
# tps = 761.30 (including connections establishing)
# tps = 761.95 (excluding connections establishing)
```

### pgbench Script ที่กำหนดเอง

```bash
# สร้าง custom benchmark script
cat > /tmp/test_query.sql << 'EOF'
\set customer_id random(1, 100000)
SELECT 
    o.order_id,
    o.total_amount,
    c.name
FROM orders o
JOIN customers c ON o.customer_id = c.customer_id
WHERE o.customer_id = :customer_id
ORDER BY o.created_at DESC
LIMIT 10;
EOF

# รัน custom benchmark
pgbench -U postgres -c 20 -j 4 -T 120 -f /tmp/test_query.sql mydb
```

```bash
# Test read vs write mix
cat > /tmp/mixed_workload.sql << 'EOF'
\set customer_id random(1, 100000)
\set order_amount random(100, 50000)

-- 70% reads
SELECT * FROM orders WHERE customer_id = :customer_id LIMIT 5;

-- 20% updates
UPDATE customers SET last_order_at = NOW() WHERE customer_id = :customer_id;

-- 10% inserts (แยกเป็น transaction weight ด้วย)
INSERT INTO order_log (customer_id, action, amount, logged_at)
VALUES (:customer_id, 'view', :order_amount, NOW());
EOF

pgbench -U postgres -c 10 -j 2 -T 60 -f /tmp/mixed_workload.sql mydb
```

### pgbench Baseline Comparison

```bash
# ขั้นตอนวัด baseline ก่อน optimization:

# 1. Reset stats
psql -c "SELECT pg_stat_reset();"

# 2. Warm up cache
pgbench -U postgres -c 5 -T 30 mydb > /dev/null 2>&1

# 3. รัน baseline
pgbench -U postgres -c 10 -j 4 -T 120 mydb > /tmp/before_optimization.txt
cat /tmp/before_optimization.txt

# 4. Apply optimization (เพิ่ม index, etc.)

# 5. รัน benchmark อีกครั้ง
pgbench -U postgres -c 10 -j 4 -T 120 mydb > /tmp/after_optimization.txt
cat /tmp/after_optimization.txt

# 6. เปรียบเทียบ
echo "=== BEFORE ===" && grep "tps" /tmp/before_optimization.txt
echo "=== AFTER ===" && grep "tps" /tmp/after_optimization.txt
```

---

## 2. mysqlslap (MySQL)

mysqlslap คือ benchmarking tool ของ MySQL

```bash
# Auto-generate test (create table, insert, query)
mysqlslap --auto-generate-sql \
    --auto-generate-sql-load-type=mixed \
    --concurrency=10 \
    --iterations=3 \
    --number-of-queries=1000 \
    --user=root --password

# Custom query benchmark
mysqlslap --create-schema=mydb \
    --query="SELECT * FROM orders WHERE customer_id = FLOOR(1 + RAND() * 100000) LIMIT 10" \
    --concurrency=20 \
    --iterations=5 \
    --user=root --password

# ผลลัพธ์:
# Benchmark
#         Average number of seconds to run all queries: 2.345 sec
#         Minimum number of seconds to run all queries: 2.123 sec
#         Maximum number of seconds to run all queries: 2.567 sec
#         Number of clients running queries: 20
#         Average number of queries per client: 50
```

```bash
# เปรียบเทียบ with/without index
mysqlslap --create-schema=testdb \
    --query=/tmp/test_query.sql \
    --concurrency=1,5,10,20,50 \
    --iterations=3 \
    --user=root --password \
    --number-of-queries=500
# ทดสอบกับ concurrency หลายระดับ
```

---

## 3. Creating a Performance Testing Methodology

### ขั้นตอนการทดสอบที่เป็นระบบ

```
Performance Testing Methodology:

1. DEFINE GOALS
   - Target: response time < 100ms สำหรับ 95th percentile
   - Load: 1,000 concurrent users
   - Duration: sustainable under 1 hour load

2. ESTABLISH BASELINE
   - Current performance ก่อนเปลี่ยนแปลงใดๆ
   - Document: TPS, latency percentiles, resource usage

3. IDENTIFY WORKLOAD
   - Production query mix (ดูจาก pg_stat_statements)
   - Peak load patterns
   - Mix ratio: read/write ratio

4. SET UP TEST ENVIRONMENT
   - Same hardware spec หรือ proportion ที่รู้จัก
   - Same data volume (หรือ representative sample)
   - Isolated (ไม่มี other workloads)

5. RUN TESTS
   - Warm up: รัน 5-10 นาทีก่อนวัด
   - Measure: รัน 15-30 นาที
   - Repeat: อย่างน้อย 3 รอบ
   - Record: ทุก run

6. ANALYZE RESULTS
   - Compare against baseline
   - Look at percentiles (p50, p95, p99)
   - Check resource bottlenecks (CPU, IO, memory)

7. DOCUMENT AND ITERATE
   - บันทึกทุก optimization และผล
   - Verify improvements ใน production
```

---

## 4. Baseline Measurements

```sql
-- สร้าง baseline snapshot ของ performance metrics

-- PostgreSQL: บันทึก current stats
CREATE TABLE perf_baseline AS
SELECT 
    'table_stats' AS category,
    relname AS object_name,
    seq_scan,
    seq_tup_read,
    idx_scan,
    idx_tup_fetch,
    n_live_tup,
    NOW() AS snapshot_time
FROM pg_stat_user_tables
WHERE schemaname = 'public';

-- Index stats baseline
CREATE TABLE index_baseline AS
SELECT 
    schemaname,
    tablename,
    indexname,
    idx_scan,
    idx_tup_read,
    idx_tup_fetch,
    NOW() AS snapshot_time
FROM pg_stat_user_indexes;

-- หลัง benchmark รัน: เปรียบเทียบ
SELECT 
    a.relname,
    a.seq_scan AS before_seq_scan,
    b.seq_scan AS after_seq_scan,
    b.seq_scan - a.seq_scan AS delta_seq_scan,
    a.idx_scan AS before_idx_scan,
    b.idx_scan AS after_idx_scan,
    b.idx_scan - a.idx_scan AS delta_idx_scan
FROM perf_baseline a
JOIN pg_stat_user_tables b ON a.relname = b.relname
WHERE a.snapshot_time = (SELECT MAX(snapshot_time) FROM perf_baseline)
ORDER BY delta_seq_scan + delta_idx_scan DESC;
```

---

## 5. Before/After Comparison Script

```sql
-- สร้าง performance comparison framework

-- Step 1: สร้างตาราง benchmark_results
CREATE TABLE benchmark_results (
    run_id      SERIAL PRIMARY KEY,
    test_name   VARCHAR(100),
    query_text  TEXT,
    run_type    VARCHAR(20),  -- 'before', 'after', 'baseline'
    execution_time_ms FLOAT,
    rows_returned INT,
    planning_time_ms FLOAT,
    buffers_hit INT,
    buffers_read INT,
    created_at  TIMESTAMP DEFAULT NOW()
);

-- Step 2: Function เพื่อรัน query และบันทึกผล
CREATE OR REPLACE FUNCTION benchmark_query(
    p_test_name TEXT,
    p_query TEXT,
    p_run_type TEXT,
    p_iterations INT DEFAULT 5
) RETURNS TABLE(
    avg_time_ms FLOAT,
    min_time_ms FLOAT,
    max_time_ms FLOAT,
    rows_count INT
) AS $$
DECLARE
    v_start TIMESTAMP;
    v_end TIMESTAMP;
    v_elapsed FLOAT;
    v_rows INT;
    i INT;
BEGIN
    FOR i IN 1..p_iterations LOOP
        v_start := clock_timestamp();
        EXECUTE p_query;
        GET DIAGNOSTICS v_rows = ROW_COUNT;
        v_end := clock_timestamp();
        v_elapsed := EXTRACT(EPOCH FROM (v_end - v_start)) * 1000;
        
        INSERT INTO benchmark_results 
            (test_name, query_text, run_type, execution_time_ms, rows_returned)
        VALUES 
            (p_test_name, p_query, p_run_type, v_elapsed, v_rows);
    END LOOP;
    
    RETURN QUERY
    SELECT 
        ROUND(AVG(execution_time_ms)::numeric, 2)::float,
        ROUND(MIN(execution_time_ms)::numeric, 2)::float,
        ROUND(MAX(execution_time_ms)::numeric, 2)::float,
        MAX(rows_returned)
    FROM benchmark_results
    WHERE test_name = p_test_name AND run_type = p_run_type;
END;
$$ LANGUAGE plpgsql;

-- Step 3: รัน benchmark
SELECT * FROM benchmark_query(
    'order_lookup',
    'SELECT * FROM orders WHERE customer_id = 1001 ORDER BY created_at DESC LIMIT 10',
    'before',
    10
);

-- สร้าง index
CREATE INDEX idx_orders_cust_date ON orders(customer_id, created_at DESC);

-- รัน benchmark อีกครั้ง
SELECT * FROM benchmark_query(
    'order_lookup',
    'SELECT * FROM orders WHERE customer_id = 1001 ORDER BY created_at DESC LIMIT 10',
    'after',
    10
);

-- Step 4: ดูผลเปรียบเทียบ
SELECT 
    b.test_name,
    a.avg_time AS before_avg_ms,
    b_agg.avg_time AS after_avg_ms,
    ROUND((a.avg_time - b_agg.avg_time) / a.avg_time * 100, 2) AS improvement_pct
FROM 
    (SELECT test_name, ROUND(AVG(execution_time_ms)::numeric, 2) AS avg_time
     FROM benchmark_results WHERE run_type = 'before' GROUP BY test_name) a
JOIN 
    (SELECT test_name, ROUND(AVG(execution_time_ms)::numeric, 2) AS avg_time
     FROM benchmark_results WHERE run_type = 'after' GROUP BY test_name) b_agg
    ON a.test_name = b_agg.test_name
ORDER BY improvement_pct DESC;
```

---

## 6. pg_stat_statements - Performance Monitoring Views

```sql
-- ติดตั้ง extension
CREATE EXTENSION pg_stat_statements;

-- ดู top 10 slow queries (by total time)
SELECT 
    ROUND(total_exec_time::numeric, 2) AS total_time_ms,
    calls,
    ROUND(mean_exec_time::numeric, 2) AS avg_time_ms,
    ROUND(stddev_exec_time::numeric, 2) AS stddev_ms,
    ROUND((100 * total_exec_time / SUM(total_exec_time) OVER ())::numeric, 2) AS pct_total,
    rows,
    query
FROM pg_stat_statements
WHERE query NOT LIKE '%pg_stat%'  -- ไม่รวม meta queries
ORDER BY total_exec_time DESC
LIMIT 10;

-- Top queries by average execution time
SELECT 
    ROUND(mean_exec_time::numeric, 2) AS avg_ms,
    calls,
    ROUND(total_exec_time::numeric, 2) AS total_ms,
    rows / NULLIF(calls, 0) AS avg_rows,
    query
FROM pg_stat_statements
WHERE calls > 10  -- query ที่รันบ่อย
ORDER BY mean_exec_time DESC
LIMIT 20;

-- Queries with high variability (stddev สูง = บางครั้งเร็ว บางครั้งช้ามาก)
SELECT 
    ROUND(mean_exec_time::numeric, 2) AS avg_ms,
    ROUND(stddev_exec_time::numeric, 2) AS stddev_ms,
    ROUND((stddev_exec_time / NULLIF(mean_exec_time, 0))::numeric, 2) AS coefficient_of_variation,
    calls,
    query
FROM pg_stat_statements
WHERE calls > 100
AND mean_exec_time > 10
ORDER BY coefficient_of_variation DESC
LIMIT 20;
```

```sql
-- Reset statistics (หลัง optimization)
SELECT pg_stat_statements_reset();

-- ดู query ที่มี full table scans (rows มากแต่ rows returned น้อย)
SELECT 
    query,
    calls,
    ROUND(mean_exec_time::numeric, 2) AS avg_ms,
    rows / NULLIF(calls, 0) AS avg_rows_returned,
    ROUND(shared_blks_read / NULLIF(calls, 0)) AS avg_disk_reads_per_call
FROM pg_stat_statements
WHERE shared_blks_read / NULLIF(calls, 0) > 1000  -- disk reads สูง
ORDER BY shared_blks_read DESC
LIMIT 20;
```

---

## 7. sys.dm_exec_query_stats (SQL Server)

```sql
-- SQL Server: หา slow queries
SELECT TOP 20
    qs.execution_count,
    qs.total_elapsed_time / 1000 AS total_elapsed_ms,
    qs.total_elapsed_time / qs.execution_count / 1000 AS avg_elapsed_ms,
    qs.total_worker_time / 1000 AS total_cpu_ms,
    qs.total_logical_reads,
    qs.total_logical_writes,
    SUBSTRING(qt.text, (qs.statement_start_offset/2)+1,
        ((CASE qs.statement_end_offset
            WHEN -1 THEN DATALENGTH(qt.text)
            ELSE qs.statement_end_offset
          END - qs.statement_start_offset)/2)+1) AS query_text
FROM sys.dm_exec_query_stats qs
CROSS APPLY sys.dm_exec_sql_text(qs.sql_handle) qt
ORDER BY qs.total_elapsed_time DESC;
```

---

## 8. Slow Query Log

### PostgreSQL: log_min_duration_statement

```sql
-- PostgreSQL: เปิด slow query logging
-- ใน postgresql.conf:
-- log_min_duration_statement = 1000  -- log queries > 1 second
-- log_statement = 'none'  -- ไม่ log ทุก query
-- log_line_prefix = '%m [%p] %q%u@%d '

-- หรือ per-session:
SET log_min_duration_statement = 500;  -- log queries > 500ms

-- ดู slow queries จาก log
-- TAIL /var/log/postgresql/postgresql-*.log | grep "duration:"

-- ใช้ pgBadger เพื่อ parse log:
-- pgbadger /var/log/postgresql/*.log -o report.html
```

### MySQL Slow Query Log

```sql
-- MySQL: เปิด slow query log
SET GLOBAL slow_query_log = 1;
SET GLOBAL long_query_time = 1;  -- log queries > 1 second
SET GLOBAL log_queries_not_using_indexes = 1;
SET GLOBAL slow_query_log_file = '/var/log/mysql/slow.log';

-- ดูว่า enabled หรือยัง:
SHOW VARIABLES LIKE 'slow_query_log%';
SHOW VARIABLES LIKE 'long_query_time';

-- ใช้ mysqldumpslow เพื่อ summarize:
-- mysqldumpslow -s t -t 20 /var/log/mysql/slow.log
-- -s t: sort by time
-- -t 20: top 20 queries

-- ใช้ pt-query-digest (Percona):
-- pt-query-digest /var/log/mysql/slow.log > report.txt
```

---

## 9. Load Testing Queries

```sql
-- สร้าง load test scenarios

-- Scenario 1: Read-heavy workload (e-commerce product browsing)
-- 70% reads, 20% cache lookups, 10% inserts

-- Read query (Product listing):
SELECT p.product_id, p.name, p.price, c.category_name
FROM products p
JOIN categories c ON p.category_id = c.category_id
WHERE p.category_id = FLOOR(1 + RANDOM() * 100)
AND p.is_active = TRUE
ORDER BY p.popularity_score DESC
LIMIT 20;

-- Read query (Product detail):
SELECT p.*, 
    AVG(r.score)::numeric(3,1) AS avg_rating,
    COUNT(r.review_id) AS review_count
FROM products p
LEFT JOIN reviews r ON p.product_id = r.product_id
WHERE p.product_id = FLOOR(1 + RANDOM() * 100000)
GROUP BY p.product_id;

-- Write query (Order placement):
INSERT INTO orders (customer_id, total_amount, status, created_at)
VALUES (FLOOR(1 + RANDOM() * 100000), RANDOM() * 10000, 'pending', NOW());
```

```sql
-- Scenario 2: Analytics workload (Daily reporting)
-- สร้าง materialized view สำหรับ report บ่อยๆ

CREATE MATERIALIZED VIEW mv_daily_revenue AS
SELECT 
    DATE_TRUNC('day', o.created_at) AS day,
    c.country_code,
    p.category_id,
    COUNT(DISTINCT o.order_id) AS orders,
    SUM(oi.quantity * oi.unit_price) AS revenue
FROM orders o
JOIN customers c ON o.customer_id = c.customer_id
JOIN order_items oi ON o.order_id = oi.order_id
JOIN products p ON oi.product_id = p.product_id
WHERE o.created_at >= CURRENT_DATE - INTERVAL '90 days'
GROUP BY 1, 2, 3;

CREATE INDEX idx_mv_daily_revenue ON mv_daily_revenue(day, country_code, category_id);

-- Refresh ทุกชั่วโมง:
-- SELECT cron.schedule('refresh-mv', '0 * * * *', $$REFRESH MATERIALIZED VIEW CONCURRENTLY mv_daily_revenue$$);
```

---

## 10. Identifying Bottlenecks

```sql
-- Bottleneck Analysis Queries

-- 1. IO Bottleneck: Tables ที่มี disk reads มาก
SELECT 
    relname,
    heap_blks_read AS disk_reads,
    heap_blks_hit AS cache_hits,
    ROUND(heap_blks_hit::numeric / NULLIF(heap_blks_hit + heap_blks_read, 0) * 100, 2) AS hit_pct,
    idx_blks_read AS idx_disk_reads,
    CASE 
        WHEN heap_blks_hit::numeric / NULLIF(heap_blks_hit + heap_blks_read, 0) < 0.90
        THEN 'IO BOTTLENECK'
        ELSE 'OK'
    END AS status
FROM pg_statio_user_tables
WHERE heap_blks_read > 1000
ORDER BY disk_reads DESC
LIMIT 10;

-- 2. CPU Bottleneck: Queries ที่ใช้ CPU มาก
SELECT 
    query,
    calls,
    total_exec_time / 1000 AS total_sec,
    total_exec_time / calls AS avg_ms,
    total_exec_time / NULLIF((SELECT SUM(total_exec_time) FROM pg_stat_statements), 0) * 100 AS cpu_pct
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 10;

-- 3. Lock Bottleneck: Waiting locks
SELECT 
    pid,
    wait_event_type,
    wait_event,
    state,
    query_start,
    NOW() - query_start AS waiting_duration,
    LEFT(query, 100) AS query_snippet
FROM pg_stat_activity
WHERE wait_event IS NOT NULL
AND state = 'active'
ORDER BY waiting_duration DESC;

-- 4. Connection Bottleneck: Connection pool usage
SELECT 
    state,
    COUNT(*) AS connections,
    MAX(NOW() - state_change) AS longest_in_state
FROM pg_stat_activity
WHERE datname = current_database()
GROUP BY state;

-- Total connections vs max:
SELECT 
    COUNT(*) AS current_connections,
    (SELECT setting::int FROM pg_settings WHERE name = 'max_connections') AS max_connections,
    ROUND(COUNT(*)::numeric / 
          (SELECT setting::int FROM pg_settings WHERE name = 'max_connections') * 100, 2) AS usage_pct
FROM pg_stat_activity;
```

---

## 11. Performance Monitoring Views (MySQL)

```sql
-- MySQL Performance Schema queries

-- Top slow queries
SELECT 
    DIGEST_TEXT AS query_pattern,
    COUNT_STAR AS executions,
    ROUND(SUM_TIMER_WAIT / 1e12, 3) AS total_sec,
    ROUND(AVG_TIMER_WAIT / 1e9, 3) AS avg_ms,
    ROUND(MAX_TIMER_WAIT / 1e9, 3) AS max_ms,
    SUM_ROWS_EXAMINED AS total_rows_examined,
    SUM_ROWS_SENT AS total_rows_sent
FROM performance_schema.events_statements_summary_by_digest
WHERE DIGEST_TEXT IS NOT NULL
ORDER BY SUM_TIMER_WAIT DESC
LIMIT 20;

-- Table I/O statistics
SELECT 
    OBJECT_SCHEMA,
    OBJECT_NAME,
    COUNT_READ,
    COUNT_WRITE,
    COUNT_FETCH,
    COUNT_INSERT,
    COUNT_UPDATE,
    COUNT_DELETE,
    ROUND(SUM_TIMER_READ / 1e12, 3) AS read_sec,
    ROUND(SUM_TIMER_WRITE / 1e12, 3) AS write_sec
FROM performance_schema.table_io_waits_summary_by_table
WHERE OBJECT_SCHEMA NOT IN ('mysql', 'information_schema', 'performance_schema')
ORDER BY SUM_TIMER_READ + SUM_TIMER_WRITE DESC
LIMIT 20;

-- Index usage statistics
SELECT 
    OBJECT_SCHEMA,
    OBJECT_NAME AS table_name,
    INDEX_NAME,
    COUNT_FETCH,
    COUNT_INSERT,
    COUNT_UPDATE,
    COUNT_DELETE
FROM performance_schema.table_io_waits_summary_by_index_usage
WHERE OBJECT_SCHEMA = DATABASE()
AND INDEX_NAME IS NOT NULL
ORDER BY COUNT_FETCH + COUNT_INSERT + COUNT_UPDATE + COUNT_DELETE DESC
LIMIT 20;
```

---

## 12. Benchmarking Scripts สมบูรณ์

```bash
#!/bin/bash
# performance_test.sh - Complete Performance Testing Script

DB_NAME="mydb"
DB_USER="postgres"
RESULTS_DIR="/tmp/perf_results"
TIMESTAMP=$(date +%Y%m%d_%H%M%S)

mkdir -p $RESULTS_DIR

echo "=== Performance Test: $TIMESTAMP ===" | tee $RESULTS_DIR/report_$TIMESTAMP.txt

# 1. System info
echo "" | tee -a $RESULTS_DIR/report_$TIMESTAMP.txt
echo "--- System Info ---" | tee -a $RESULTS_DIR/report_$TIMESTAMP.txt
psql -U $DB_USER $DB_NAME -c "SHOW server_version;" | tee -a $RESULTS_DIR/report_$TIMESTAMP.txt
psql -U $DB_USER $DB_NAME -c "SHOW shared_buffers;" | tee -a $RESULTS_DIR/report_$TIMESTAMP.txt
psql -U $DB_USER $DB_NAME -c "SHOW max_connections;" | tee -a $RESULTS_DIR/report_$TIMESTAMP.txt

# 2. Reset stats
echo "" | tee -a $RESULTS_DIR/report_$TIMESTAMP.txt
echo "--- Resetting statistics ---" | tee -a $RESULTS_DIR/report_$TIMESTAMP.txt
psql -U $DB_USER $DB_NAME -c "SELECT pg_stat_reset();"
psql -U $DB_USER $DB_NAME -c "SELECT pg_stat_statements_reset();"

# 3. Warm up (ไม่นับผล)
echo "--- Warming up cache (30s) ---" | tee -a $RESULTS_DIR/report_$TIMESTAMP.txt
pgbench -U $DB_USER -c 5 -T 30 $DB_NAME > /dev/null 2>&1

# 4. Benchmark tests
echo "" | tee -a $RESULTS_DIR/report_$TIMESTAMP.txt
echo "--- Read-only Test (60s, 10 clients) ---" | tee -a $RESULTS_DIR/report_$TIMESTAMP.txt
pgbench -U $DB_USER -c 10 -j 4 -T 60 -S $DB_NAME | tee -a $RESULTS_DIR/report_$TIMESTAMP.txt

echo "" | tee -a $RESULTS_DIR/report_$TIMESTAMP.txt
echo "--- Mixed Test (60s, 10 clients) ---" | tee -a $RESULTS_DIR/report_$TIMESTAMP.txt
pgbench -U $DB_USER -c 10 -j 4 -T 60 $DB_NAME | tee -a $RESULTS_DIR/report_$TIMESTAMP.txt

# 5. Post-test analysis
echo "" | tee -a $RESULTS_DIR/report_$TIMESTAMP.txt
echo "--- Top 5 Slow Queries ---" | tee -a $RESULTS_DIR/report_$TIMESTAMP.txt
psql -U $DB_USER $DB_NAME -c "
SELECT 
    ROUND(mean_exec_time::numeric, 2) AS avg_ms,
    calls,
    LEFT(query, 80) AS query_snippet
FROM pg_stat_statements
ORDER BY mean_exec_time DESC
LIMIT 5;
" | tee -a $RESULTS_DIR/report_$TIMESTAMP.txt

echo "" | tee -a $RESULTS_DIR/report_$TIMESTAMP.txt
echo "--- Cache Hit Ratio ---" | tee -a $RESULTS_DIR/report_$TIMESTAMP.txt
psql -U $DB_USER $DB_NAME -c "
SELECT 
    ROUND(sum(heap_blks_hit)::numeric / 
          NULLIF(sum(heap_blks_hit) + sum(heap_blks_read), 0) * 100, 2) AS cache_hit_pct
FROM pg_statio_user_tables;
" | tee -a $RESULTS_DIR/report_$TIMESTAMP.txt

echo ""
echo "Report saved to: $RESULTS_DIR/report_$TIMESTAMP.txt"
```

---

## 13. Performance Checklist

```
Performance Testing Checklist:

PRE-TEST:
□ สร้าง baseline (ก่อนเปลี่ยนแปลงใดๆ)
□ ตรวจสอบ data volume (ควรใกล้เคียง production)
□ Warm up cache ก่อนวัดผล
□ Reset pg_stat_statements / slow query log
□ Isolate test environment (ไม่มี other workloads)
□ Document current indexes, config settings

DURING TEST:
□ Monitor CPU, memory, disk I/O ระหว่างทดสอบ
□ Log query execution times
□ Watch for lock waits
□ Monitor replication lag (ถ้ามี replica)
□ บันทึก concurrency level ที่ทดสอบ

POST-TEST ANALYSIS:
□ ดู top slow queries ใน pg_stat_statements
□ ตรวจสอบ cache hit ratio (> 95% คือดี)
□ ดู tables ที่มี high disk reads
□ ตรวจสอบ Index Hit vs Seq Scan ratio
□ ดู lock contention metrics
□ Compare before/after metrics

OPTIMIZATION VALIDATION:
□ รัน test ซ้ำอย่างน้อย 3 ครั้ง
□ ใช้ percentile (p95, p99) ไม่ใช่แค่ average
□ Test ที่ different load levels (1x, 5x, 10x)
□ Validate ใน staging ก่อน production
□ Monitor production หลัง deploy
```

---

## 14. pg_stat_activity Real-time Monitoring

```sql
-- Real-time monitoring dashboard
-- Query 1: Active queries right now
SELECT 
    pid,
    usename,
    application_name,
    state,
    ROUND(EXTRACT(EPOCH FROM (NOW() - query_start))::numeric, 2) AS duration_sec,
    wait_event_type || ':' || COALESCE(wait_event, '') AS wait_info,
    LEFT(query, 100) AS query
FROM pg_stat_activity
WHERE state != 'idle'
AND query NOT LIKE '%pg_stat_activity%'
ORDER BY duration_sec DESC;

-- Query 2: Long-running queries (> 30 seconds)
SELECT 
    pid,
    usename,
    NOW() - query_start AS duration,
    query
FROM pg_stat_activity
WHERE state = 'active'
AND query_start < NOW() - INTERVAL '30 seconds'
AND query NOT LIKE '%pg_stat_activity%'
ORDER BY query_start ASC;

-- Kill long-running query (ถ้าจำเป็น):
-- SELECT pg_terminate_backend(pid) WHERE pid = XXXX;

-- Query 3: Blocked queries
SELECT 
    blocked_locks.pid AS blocked_pid,
    blocked_activity.usename AS blocked_user,
    blocking_locks.pid AS blocking_pid,
    blocking_activity.usename AS blocking_user,
    blocked_activity.query AS blocked_statement,
    blocking_activity.query AS current_statement_in_blocking_process
FROM pg_catalog.pg_locks blocked_locks
JOIN pg_catalog.pg_stat_activity blocked_activity ON blocked_activity.pid = blocked_locks.pid
JOIN pg_catalog.pg_locks blocking_locks 
    ON blocking_locks.locktype = blocked_locks.locktype
    AND blocking_locks.DATABASE IS NOT DISTINCT FROM blocked_locks.DATABASE
    AND blocking_locks.relation IS NOT DISTINCT FROM blocked_locks.relation
    AND blocking_locks.page IS NOT DISTINCT FROM blocked_locks.page
    AND blocking_locks.tuple IS NOT DISTINCT FROM blocked_locks.tuple
    AND blocking_locks.virtualxid IS NOT DISTINCT FROM blocked_locks.virtualxid
    AND blocking_locks.transactionid IS NOT DISTINCT FROM blocked_locks.transactionid
    AND blocking_locks.classid IS NOT DISTINCT FROM blocked_locks.classid
    AND blocking_locks.objid IS NOT DISTINCT FROM blocked_locks.objid
    AND blocking_locks.objsubid IS NOT DISTINCT FROM blocked_locks.objsubid
    AND blocking_locks.pid != blocked_locks.pid
JOIN pg_catalog.pg_stat_activity blocking_activity ON blocking_activity.pid = blocking_locks.pid
WHERE NOT blocked_locks.GRANTED;
```

---

## แบบฝึกหัด (10 ข้อ)

**ข้อ 1:** อธิบายความแตกต่างระหว่าง Throughput (TPS) และ Latency ในการ benchmark database

**เฉลยข้อ 1:**
- **Throughput (TPS)**: จำนวน transactions ที่ทำได้ต่อวินาที ✓ สูงกว่า = ดีกว่า เหมาะสำหรับ: วัด capacity
- **Latency**: เวลาที่ใช้ต่อ request หนึ่งๆ ✓ ต่ำกว่า = ดีกว่า เหมาะสำหรับ: วัด user experience

ทั้งสองเชื่อมโยงกัน: เพิ่ม concurrency → TPS ขึ้น จนถึงจุดหนึ่ง latency จะเพิ่มเร็วมาก ควรหา "sweet spot" ที่ TPS สูงแต่ latency ยังยอมรับได้

---

**ข้อ 2:** เขียน pgbench custom script เพื่อทดสอบ mixed read/write workload

**เฉลยข้อ 2:**
```sql
-- /tmp/mixed.sql
\set customer_id random(1, 100000)
\set product_id random(1, 50000)
\set amount random(100, 50000)

-- 60% reads
SELECT o.order_id, o.total_amount, c.name
FROM orders o JOIN customers c ON o.customer_id = c.customer_id
WHERE o.customer_id = :customer_id LIMIT 5;

-- 30% reads
SELECT name, price FROM products WHERE product_id = :product_id;

-- 10% writes
INSERT INTO cart_events (customer_id, product_id, action, amount, event_time)
VALUES (:customer_id, :product_id, 'add_to_cart', :amount, NOW());
```
```bash
pgbench -U postgres -c 20 -j 4 -T 60 -f /tmp/mixed.sql mydb
```

---

**ข้อ 3:** pg_stat_statements มีประโยชน์อะไรในการ performance tuning?

**เฉลยข้อ 3:**
pg_stat_statements เก็บ statistics ของทุก query ที่รัน:
1. **หา slow queries**: `ORDER BY mean_exec_time DESC`
2. **หา queries ที่รันบ่อยและ aggregate time สูง**: `ORDER BY total_exec_time DESC`
3. **หา queries ที่ variable มาก**: เปรียบเทียบ mean vs stddev
4. **Monitor cache efficiency**: `shared_blks_read` สูง = ต้องการ index หรือเพิ่ม cache
5. **Identify regression**: compare before/after optimization

---

**ข้อ 4:** ทำไม "warm up" cache ก่อน benchmark จึงสำคัญ?

**เฉลยข้อ 4:**
หลัง restart หรือ reset stats, cache ว่างเปล่า (cold cache) → ทุก query ต้องอ่าน disk → ช้ากว่าปกติ ถ้าวัดผล cold cache จะได้ performance ที่ต่ำกว่าความเป็นจริงใน production ที่ cache warm อยู่แล้ว Warm-up คือ: รัน workload 5-10 นาทีโดยไม่นับผล เพื่อให้ cache เต็มก่อนแล้วค่อยวัด

---

**ข้อ 5:** เขียน query เพื่อหา index ที่ไม่ถูกใช้งาน (idx_scan = 0) แต่มีขนาดใหญ่

**เฉลยข้อ 5:**
```sql
SELECT 
    schemaname || '.' || tablename AS table,
    indexname,
    pg_size_pretty(pg_relation_size(indexrelid)) AS index_size,
    idx_scan AS scans,
    'DROP INDEX CONCURRENTLY ' || indexname || ';' AS drop_statement
FROM pg_stat_user_indexes
LEFT JOIN pg_index ON pg_stat_user_indexes.indexrelid = pg_index.indexrelid
WHERE idx_scan = 0
AND NOT indisprimary
AND NOT indisunique
AND pg_relation_size(indexrelid) > 10 * 1024 * 1024  -- > 10MB
ORDER BY pg_relation_size(indexrelid) DESC;
```

---

**ข้อ 6:** ความแตกต่างระหว่าง Average Latency และ P95/P99 Latency คืออะไร ทำไม P99 จึงสำคัญกว่า?

**เฉลยข้อ 6:**
- **Average**: ค่าเฉลี่ยของทุก request - ถูก pull down โดย fast requests
- **P95 (95th percentile)**: 95% ของ requests เร็วกว่าค่านี้ → เห็น tail latency
- **P99**: 99% เร็วกว่า → เห็น worst case ที่ users จะเจอ

P99 สำคัญกว่าเพราะ: ถ้า P99 = 10 seconds แปลว่า 1 ใน 100 users ต้องรอ 10 วินาที ซึ่ง users จะ complain! Average อาจแค่ 100ms แต่ซ่อน outliers ไว้

---

**ข้อ 7:** เขียน query เพื่อหา "locking problems" ใน PostgreSQL

**เฉลยข้อ 7:**
```sql
-- หา sessions ที่รอ lock
SELECT 
    waiting.pid AS waiting_pid,
    waiting.query AS waiting_query,
    blocking.pid AS blocking_pid,
    blocking.query AS blocking_query,
    NOW() - waiting.query_start AS wait_duration
FROM pg_stat_activity waiting
JOIN pg_stat_activity blocking 
    ON blocking.pid = ANY(pg_blocking_pids(waiting.pid))
WHERE waiting.wait_event_type = 'Lock'
ORDER BY wait_duration DESC;
```

---

**ข้อ 8:** ออกแบบ performance testing plan สำหรับ API ที่ query database หนัก

**เฉลยข้อ 8:**
```
1. Profile production: ดู pg_stat_statements → หา top 10 queries
2. สร้าง test data: ขนาดเท่า production (หรือ 10x สำหรับ stress test)
3. สร้าง benchmark scripts ตาม query patterns จาก production
4. Baseline measurement:
   - Single user: latency ของแต่ละ query
   - 10 concurrent users
   - 50 concurrent users  
   - 100 concurrent users (max expected)
5. Measure metrics: TPS, P50/P95/P99 latency, CPU%, Memory%, Disk IO
6. Apply optimization: add indexes, tune settings
7. Re-test: same scripts, same concurrency levels
8. Compare: TPS improvement %, latency reduction %
9. Deploy: monitor production ด้วย pg_stat_statements
```

---

**ข้อ 9:** MySQL slow query log บอกอะไรบ้าง และ parse อย่างไร?

**เฉลยข้อ 9:**
Slow query log บอก:
- Query text
- Execution time (Query_time)
- Lock time (Lock_time)
- Rows sent vs Rows examined
- Timestamp

Parse ด้วย:
```bash
# mysqldumpslow: built-in
mysqldumpslow -s t -t 20 /var/log/mysql/slow.log
# -s t: sort by time
# -t 20: top 20

# pt-query-digest: detailed analysis
pt-query-digest /var/log/mysql/slow.log --limit 20 > report.txt
# แสดง: query patterns, fingerprints, percentiles
```

---

**ข้อ 10:** สร้าง performance monitoring script ที่รันทุกชั่วโมงและ alert เมื่อ cache hit rate ต่ำกว่า 95%

**เฉลยข้อ 10:**
```sql
-- สร้าง function สำหรับ check และ alert
CREATE OR REPLACE FUNCTION check_cache_health()
RETURNS TEXT AS $$
DECLARE
    v_hit_rate FLOAT;
    v_alert TEXT := '';
BEGIN
    -- Check table cache hit rate
    SELECT 
        sum(heap_blks_hit)::float / NULLIF(sum(heap_blks_hit) + sum(heap_blks_read), 0) * 100
    INTO v_hit_rate
    FROM pg_statio_user_tables;
    
    IF v_hit_rate < 95 THEN
        v_alert := format('ALERT: Table cache hit rate is %.2f%% (threshold: 95%%)', v_hit_rate);
        -- ใส่ notification logic ที่นี่ (email, slack webhook, etc.)
        -- PERFORM pg_notify('performance_alerts', v_alert);
    END IF;
    
    RETURN COALESCE(v_alert, format('OK: Cache hit rate %.2f%%', v_hit_rate));
END;
$$ LANGUAGE plpgsql;

-- Schedule ด้วย pg_cron:
-- SELECT cron.schedule('check-cache', '0 * * * *', 'SELECT check_cache_health()');
```

```bash
#!/bin/bash
# check_performance.sh - รันทุกชั่วโมง
THRESHOLD=95
HIT_RATE=$(psql -U postgres mydb -t -c "
SELECT ROUND(sum(heap_blks_hit)::numeric / 
       NULLIF(sum(heap_blks_hit) + sum(heap_blks_read), 0) * 100, 2)
FROM pg_statio_user_tables;
")

if (( $(echo "$HIT_RATE < $THRESHOLD" | bc -l) )); then
    echo "ALERT: Cache hit rate $HIT_RATE% is below threshold $THRESHOLD%"
    # curl -X POST -H 'Content-type: application/json' \
    #   --data "{\"text\":\"DB Cache Alert: $HIT_RATE%\"}" \
    #   $SLACK_WEBHOOK_URL
fi
```

---

## สรุป

ใน Part 070 เราได้เรียนรู้:

1. **pgbench** - PostgreSQL benchmarking tool พร้อม custom scripts
2. **mysqlslap** - MySQL benchmarking tool
3. **Performance Testing Methodology** - กระบวนการทดสอบที่เป็นระบบ
4. **Baseline Measurements** - สร้างฐานเปรียบเทียบ
5. **Before/After Comparison** - Framework วัดผล optimization
6. **pg_stat_statements** - Monitor slow queries ใน PostgreSQL
7. **MySQL Performance Schema** - Monitor queries ใน MySQL
8. **Slow Query Log** - PostgreSQL และ MySQL
9. **Load Testing** - Scripts สำหรับ read/write workloads
10. **Bottleneck Identification** - IO, CPU, Locks, Connections
11. **Performance Checklist** - รายการตรวจสอบสมบูรณ์
12. **Real-time Monitoring** - pg_stat_activity

---

## สรุปภาพรวม Parts 061-070

### สิ่งที่ได้เรียนรู้ตลอด 10 Parts:

| Part | หัวข้อ | ความสำคัญ |
|------|--------|-----------|
| 061 | Understanding Indexes | B-tree, selectivity, cardinality |
| 062 | Creating & Managing Indexes | Syntax, maintenance, monitoring |
| 063 | Index Types | GIN, GiST, BRIN, Hash, Full-text |
| 064 | Composite Indexes | Leftmost prefix, covering indexes |
| 065 | Execution Plans | EXPLAIN ANALYZE, plan nodes |
| 066 | Query Optimization | Sargable predicates, rewrites |
| 067 | Statistics & Planner | ANALYZE, pg_stats, multi-column |
| 068 | Caching & Buffers | shared_buffers, cache hit ratio |
| 069 | Partitioning | Range, list, hash, maintenance |
| 070 | Benchmarking | pgbench, pg_stat_statements |

### Performance Optimization Golden Rules:

```
1. MEASURE FIRST: อย่า optimize โดยไม่มีข้อมูล
2. IDENTIFY BOTTLENECK: ค้นหาจุดที่ช้าจริงๆ
3. FIX ROOT CAUSE: แก้สาเหตุ ไม่ใช่ symptom
4. VERIFY IMPROVEMENT: วัดผลก่อนและหลัง
5. MONITOR CONTINUOUSLY: ปัญหาอาจกลับมา

Top optimization techniques:
✓ เพิ่ม missing indexes
✓ แก้ non-sargable queries
✓ สร้าง covering indexes
✓ Update stale statistics
✓ Tune shared_buffers/cache
✓ Partition large tables
✓ Use read replicas
```
