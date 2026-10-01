# Part 068: Caching and Buffer Strategies

## กลยุทธ์ Caching และ Buffer Management

---

## บทนำ

Database performance ขึ้นอยู่กับว่าข้อมูลที่ต้องการอยู่ใน memory (fast) หรือต้องอ่านจาก disk (slow) การเข้าใจและ tune cache/buffer อย่างถูกต้องสามารถเพิ่มประสิทธิภาพได้หลายเท่า

```
Memory Access: ~100 nanoseconds
SSD Access:    ~100 microseconds  (1,000x ช้ากว่า memory)
HDD Access:    ~10 milliseconds   (100,000x ช้ากว่า memory)
```

---

## 1. Buffer Pool / Shared Buffers

### PostgreSQL: shared_buffers

`shared_buffers` คือ cache หลักของ PostgreSQL ที่เก็บ database pages ใน memory

```
Memory Architecture (PostgreSQL):
┌──────────────────────────────────────────────────────┐
│                    RAM (32GB)                        │
│  ┌───────────────────┐  ┌──────────────────────────┐ │
│  │  shared_buffers   │  │    OS Page Cache         │ │
│  │     (8GB)         │  │       (remaining)        │ │
│  │  - table pages    │  │  - file system cache     │ │
│  │  - index pages    │  │  - network buffers       │ │
│  │  - TOAST data     │  │                          │ │
│  └───────────────────┘  └──────────────────────────┘ │
│  ┌───────────────────┐  ┌──────────────────────────┐ │
│  │   work_mem        │  │   maintenance_work_mem   │ │
│  │ (per-query sort,  │  │  (VACUUM, CREATE INDEX)  │ │
│  │  hash, etc.)      │  │                          │ │
│  └───────────────────┘  └──────────────────────────┘ │
└──────────────────────────────────────────────────────┘
```

```sql
-- ดู shared_buffers setting
SHOW shared_buffers;

-- Best practice: 25% ของ total RAM
-- Server 32GB RAM: shared_buffers = 8GB
-- Server 8GB RAM:  shared_buffers = 2GB

-- เปลี่ยน shared_buffers (ต้อง restart PostgreSQL)
-- ใน postgresql.conf:
-- shared_buffers = 8GB
```

### InnoDB Buffer Pool (MySQL)

```sql
-- ดู buffer pool size
SHOW VARIABLES LIKE 'innodb_buffer_pool_size';
SHOW VARIABLES LIKE 'innodb_buffer_pool_instances';

-- Best practice: 70-80% ของ total RAM สำหรับ dedicated MySQL server
-- Server 32GB RAM: innodb_buffer_pool_size = 24G
-- Server 8GB RAM:  innodb_buffer_pool_size = 6G

-- ดู buffer pool usage
SELECT 
    pool_id,
    pool_size,
    free_buffers,
    database_pages,
    ROUND(database_pages / pool_size * 100, 2) AS fill_pct,
    hit_rate
FROM information_schema.INNODB_BUFFER_POOL_STATS;
```

---

## 2. Page Cache

### PostgreSQL: Double Buffering

PostgreSQL ใช้ทั้ง shared_buffers และ OS page cache ทำให้เกิด "double buffering"

```
Data flow:
Disk → OS Page Cache → shared_buffers → PostgreSQL Process

เมื่อ PostgreSQL อ่านข้อมูล:
1. ตรวจ shared_buffers ก่อน (fastest)
2. ถ้าไม่พบ ตรวจ OS page cache (fast)
3. ถ้าไม่พบ อ่านจาก disk (slow)
```

```sql
-- ดูว่า shared_buffers ใหญ่พอหรือไม่
-- Cache hit ratio ควร > 95%
SELECT 
    sum(heap_blks_read) AS heap_reads,           -- disk reads
    sum(heap_blks_hit) AS heap_hits,             -- buffer cache hits
    ROUND(sum(heap_blks_hit)::numeric / 
          NULLIF(sum(heap_blks_hit) + sum(heap_blks_read), 0) * 100, 2) 
          AS cache_hit_ratio
FROM pg_statio_user_tables;

-- ดูต่อ table
SELECT 
    relname AS table_name,
    heap_blks_read,
    heap_blks_hit,
    ROUND(heap_blks_hit::numeric / NULLIF(heap_blks_hit + heap_blks_read, 0) * 100, 2) 
        AS cache_hit_ratio
FROM pg_statio_user_tables
WHERE heap_blks_hit + heap_blks_read > 0
ORDER BY heap_blks_read DESC
LIMIT 20;
```

---

## 3. pg_buffercache (PostgreSQL)

Extension ที่ช่วยตรวจสอบ shared_buffers ในระดับ page

```sql
-- ติดตั้ง extension
CREATE EXTENSION pg_buffercache;

-- ดูข้อมูลใน buffer cache
SELECT 
    c.relname,
    COUNT(*) AS buffers,
    pg_size_pretty(COUNT(*) * 8192) AS size_in_cache,
    ROUND(COUNT(*) * 8192::numeric / 
          (SELECT setting::bigint * 1024 * 1024 FROM pg_settings WHERE name = 'shared_buffers') * 100, 2) 
          AS pct_of_shared_buffers
FROM pg_buffercache bc
JOIN pg_class c ON c.relfilenode = bc.relfilenode
GROUP BY c.relname
ORDER BY buffers DESC
LIMIT 20;

-- ดูว่า relation ไหนครอง cache มากที่สุด
SELECT 
    n.nspname AS schema,
    c.relname AS table_or_index,
    c.relkind,
    COUNT(*) AS cached_pages,
    pg_size_pretty(COUNT(*) * current_setting('block_size')::bigint) AS cached_size
FROM pg_buffercache bc
JOIN pg_class c ON c.relfilenode = bc.relfilenode
JOIN pg_namespace n ON n.oid = c.relnamespace
WHERE bc.isdirty IS NOT NULL
GROUP BY n.nspname, c.relname, c.relkind
ORDER BY cached_pages DESC
LIMIT 20;
```

```sql
-- ดู buffer usage patterns
SELECT 
    c.relname,
    SUM(CASE WHEN bc.isdirty THEN 1 ELSE 0 END) AS dirty_buffers,
    SUM(CASE WHEN NOT bc.isdirty OR bc.isdirty IS NULL THEN 1 ELSE 0 END) AS clean_buffers,
    COUNT(*) AS total_buffers,
    ROUND(AVG(bc.usagecount), 2) AS avg_usage_count
FROM pg_buffercache bc
JOIN pg_class c ON c.relfilenode = bc.relfilenode
WHERE c.relname NOT LIKE 'pg_%'
GROUP BY c.relname
ORDER BY total_buffers DESC
LIMIT 20;
```

---

## 4. Cache Hit Ratio Queries

### PostgreSQL Cache Monitoring

```sql
-- Overall cache hit ratio (table data)
SELECT 
    'Table Cache Hit Ratio' AS metric,
    ROUND(sum(heap_blks_hit)::numeric / 
          NULLIF(sum(heap_blks_hit) + sum(heap_blks_read), 0) * 100, 2) AS ratio_pct
FROM pg_statio_user_tables

UNION ALL

-- Index cache hit ratio
SELECT 
    'Index Cache Hit Ratio' AS metric,
    ROUND(sum(idx_blks_hit)::numeric / 
          NULLIF(sum(idx_blks_hit) + sum(idx_blks_read), 0) * 100, 2) AS ratio_pct
FROM pg_statio_user_indexes

UNION ALL

-- Toast cache hit ratio  
SELECT 
    'Toast Cache Hit Ratio' AS metric,
    ROUND(sum(toast_blks_hit)::numeric / 
          NULLIF(sum(toast_blks_hit) + sum(toast_blks_read), 0) * 100, 2) AS ratio_pct
FROM pg_statio_user_tables
WHERE toast_blks_hit + toast_blks_read > 0;
```

```sql
-- Cache hit ratio ต่อ table - ตารางไหน cache miss สูง
SELECT 
    relname AS table_name,
    heap_blks_read AS disk_reads,
    heap_blks_hit AS cache_hits,
    ROUND(heap_blks_hit::numeric / 
          NULLIF(heap_blks_hit + heap_blks_read, 0) * 100, 2) AS hit_ratio_pct,
    CASE 
        WHEN heap_blks_hit + heap_blks_read = 0 THEN 'No activity'
        WHEN heap_blks_hit::numeric / NULLIF(heap_blks_hit + heap_blks_read, 0) > 0.99 THEN 'Excellent'
        WHEN heap_blks_hit::numeric / NULLIF(heap_blks_hit + heap_blks_read, 0) > 0.95 THEN 'Good'
        WHEN heap_blks_hit::numeric / NULLIF(heap_blks_hit + heap_blks_read, 0) > 0.90 THEN 'Acceptable'
        ELSE 'Poor - consider increasing shared_buffers'
    END AS assessment
FROM pg_statio_user_tables
WHERE heap_blks_hit + heap_blks_read > 0
ORDER BY disk_reads DESC
LIMIT 20;
```

### MySQL Cache Monitoring

```sql
-- InnoDB Buffer Pool Hit Rate
SELECT 
    ROUND((1 - (innodb_buffer_pool_reads / innodb_buffer_pool_read_requests)) * 100, 2) 
        AS buffer_pool_hit_rate_pct
FROM information_schema.INNODB_METRICS
WHERE NAME IN ('buffer_pool_reads', 'buffer_pool_read_requests');

-- หรือใช้ SHOW STATUS:
SHOW GLOBAL STATUS LIKE 'Innodb_buffer_pool_read%';
/*
Innodb_buffer_pool_read_ahead_rnd: 0
Innodb_buffer_pool_read_ahead: 12345
Innodb_buffer_pool_read_ahead_evicted: 0
Innodb_buffer_pool_read_requests: 98765432  ← total requests
Innodb_buffer_pool_reads: 45678            ← disk reads
*/

-- คำนวณ hit rate:
-- (read_requests - reads) / read_requests = hit rate
-- (98,765,432 - 45,678) / 98,765,432 = 99.95% ← ดีมาก

-- ดู InnoDB Buffer Pool status
SHOW ENGINE INNODB STATUS;
-- ดู section "BUFFER POOL AND MEMORY"
```

```sql
-- MySQL: ดู buffer pool pages by type
SELECT 
    PAGE_TYPE,
    COUNT(*) AS page_count,
    ROUND(COUNT(*) * 100.0 / SUM(COUNT(*)) OVER (), 2) AS pct
FROM information_schema.INNODB_BUFFER_PAGE
GROUP BY PAGE_TYPE
ORDER BY page_count DESC;
```

---

## 5. Result Caching

### Application-level Caching

```
┌───────────────────────────────────────────────────────┐
│  Application Cache Layers:                           │
│                                                       │
│  Request → [L1: In-memory app cache (ms)]            │
│          → [L2: Redis/Memcached cache (ms)]          │
│          → [L3: Database query result cache (ms)]    │
│          → [L4: Database buffer cache (ms)]          │
│          → [L5: OS page cache (ms)]                  │
│          → [L6: Disk read (100ms+)]                  │
└───────────────────────────────────────────────────────┘
```

### PostgreSQL: pgmemcache / pg_redis_pubsub

```sql
-- PostgreSQL ไม่มี built-in result cache (ต่างจาก MySQL)
-- แต่สามารถใช้ Materialized View แทน:

-- Materialized View เป็น "cached query result"
CREATE MATERIALIZED VIEW mv_daily_sales AS
SELECT 
    DATE_TRUNC('day', order_date) AS sale_day,
    SUM(total_amount) AS daily_revenue,
    COUNT(*) AS order_count
FROM orders
WHERE order_date >= CURRENT_DATE - INTERVAL '30 days'
GROUP BY 1
ORDER BY 1;

-- สร้าง index บน materialized view
CREATE INDEX idx_mv_daily_sales_day ON mv_daily_sales(sale_day);

-- Refresh คงที่ (ต้อง lock ชั่วคราว)
REFRESH MATERIALIZED VIEW mv_daily_sales;

-- Refresh แบบ concurrent (ไม่ lock - PostgreSQL 9.4+)
REFRESH MATERIALIZED VIEW CONCURRENTLY mv_daily_sales;
-- ต้องมี UNIQUE index ก่อนใช้ CONCURRENTLY

CREATE UNIQUE INDEX idx_mv_daily_sales_day_uniq ON mv_daily_sales(sale_day);
```

---

## 6. Warming the Cache

### Cache Warming หลัง Restart

```sql
-- PostgreSQL: สร้าง warm-up script
-- pg_prewarm extension ช่วย load pages เข้า shared_buffers

CREATE EXTENSION pg_prewarm;

-- Warm up ตาราง:
SELECT pg_prewarm('employees');           -- load ทั้งตาราง
SELECT pg_prewarm('idx_employees_dept');  -- load index
SELECT pg_prewarm('products', 'buffer', 'main');  -- ระบุ fork

-- ดูว่า pages ถูก load แล้ว
SELECT * FROM pg_buffercache 
WHERE relfilenode = 'employees'::regclass::oid
LIMIT 5;
```

```sql
-- Auto warm-up script หลัง restart
-- ใน postgresql.conf:
-- shared_preload_libraries = 'pg_prewarm'
-- pg_prewarm.autoprewarm = on
-- จะ warm-up โดยอัตโนมัติจาก saved state!

-- บันทึก current buffer state:
SELECT autoprewarm_dump_now();
-- บันทึก state ก่อน shutdown

-- หรือ manual warm-up:
DO $$
DECLARE
    r RECORD;
BEGIN
    FOR r IN 
        SELECT relname 
        FROM pg_stat_user_tables 
        ORDER BY seq_scan + idx_scan DESC 
        LIMIT 20  -- warm up top 20 tables
    LOOP
        PERFORM pg_prewarm(r.relname::regclass);
        RAISE NOTICE 'Warmed: %', r.relname;
    END LOOP;
END $$;
```

### MySQL: InnoDB Buffer Pool Dump and Restore

```sql
-- MySQL: บันทึก buffer pool state
SET GLOBAL innodb_buffer_pool_dump_now = ON;
-- บันทึกไว้ใน datadir/ib_buffer_pool

-- โหลด buffer pool หลัง restart:
SET GLOBAL innodb_buffer_pool_load_now = ON;

-- หรือ auto load (ใน my.cnf):
-- innodb_buffer_pool_dump_at_shutdown = ON
-- innodb_buffer_pool_load_at_startup = ON

-- ดู progress:
SHOW STATUS LIKE 'Innodb_buffer_pool_dump_status';
SHOW STATUS LIKE 'Innodb_buffer_pool_load_status';
```

---

## 7. InnoDB Buffer Pool Monitoring

```sql
-- MySQL: ดู buffer pool status แบบละเอียด
SHOW ENGINE INNODB STATUS\G

-- ดูใน output ส่วน "BUFFER POOL AND MEMORY":
/*
Buffer pool size   131072
Free buffers       1024
Database pages     129948
Old database pages 47951
Modified db pages  1234
Pending reads      0
Pending writes: LRU 0, flush list 0, single page 0
Pages made young 12345678, not young 23456789
Pages read 456789, created 123456, written 234567
Buffer pool hit rate 999 / 1000  ← 99.9%!
*/
```

```sql
-- MySQL 8.0: InnoDB Buffer Pool Statistics per pool
SELECT 
    pool_id,
    ROUND(pool_size * 16 / 1024, 2) AS pool_size_mb,
    ROUND(free_buffers * 16 / 1024, 2) AS free_mb,
    ROUND(database_pages * 16 / 1024, 2) AS data_mb,
    ROUND(old_database_pages * 16 / 1024, 2) AS old_pages_mb,
    ROUND(modified_database_pages * 16 / 1024, 2) AS dirty_mb,
    hit_rate AS hit_rate_per_1000,
    ROUND(hit_rate / 10.0, 1) AS hit_rate_pct,
    pages_made_young,
    pages_not_made_young
FROM information_schema.INNODB_BUFFER_POOL_STATS;
```

---

## 8. Read Replicas for Scaling Reads

### PostgreSQL Read Replicas

```sql
-- สร้าง streaming replication (ใน primary postgresql.conf):
-- wal_level = replica
-- max_wal_senders = 3
-- wal_keep_size = 1GB

-- สร้าง replica slot:
SELECT pg_create_physical_replication_slot('replica_1');

-- บน replica server (recovery.conf หรือ postgresql.conf):
-- primary_conninfo = 'host=primary_host port=5432 user=replicator'
-- primary_slot_name = 'replica_1'
-- hot_standby = on  -- อนุญาต SELECT บน replica

-- ตรวจสอบ replication status บน primary:
SELECT 
    client_addr,
    state,
    sent_lsn,
    write_lsn,
    flush_lsn,
    replay_lsn,
    write_lag,
    flush_lag,
    replay_lag
FROM pg_stat_replication;
```

```sql
-- Application connection routing:
-- Read queries → replica
-- Write queries → primary

-- ตัวอย่างใน psql (เชื่อมต่อ replica):
-- psql -h replica-host -U app_user -d mydb

-- บน replica: ตรวจสอบว่า read-only mode:
SELECT pg_is_in_recovery();  -- returns true บน replica

-- ดู lag ของ replica:
SELECT 
    now() - pg_last_xact_replay_timestamp() AS replication_lag;
```

### MySQL Read Replicas

```sql
-- MySQL: setup replication (ใน my.cnf ของ primary):
-- server-id = 1
-- log_bin = mysql-bin
-- binlog_format = ROW

-- บน replica (my.cnf):
-- server-id = 2
-- relay_log = mysql-relay-bin

-- Setup replica:
CHANGE REPLICATION SOURCE TO
    SOURCE_HOST='primary-host',
    SOURCE_USER='replicator',
    SOURCE_PASSWORD='password',
    SOURCE_AUTO_POSITION=1;
START REPLICA;

-- ตรวจสอบ replica status:
SHOW REPLICA STATUS\G
-- Replica_IO_Running: Yes
-- Replica_SQL_Running: Yes
-- Seconds_Behind_Source: 0  ← เวลา lag

-- ตรวจสอบ replication lag:
SELECT 
    SECONDS_BEHIND_SOURCE
FROM performance_schema.replication_connection_status;
```

---

## 9. Cache Performance Impact Examples

```sql
-- ตัวอย่างที่ 1: เปรียบเทียบ query เดิมก่อน/หลัง cache warm
-- ครั้งแรก (cold cache):
EXPLAIN (ANALYZE, BUFFERS) 
SELECT * FROM large_table WHERE created_at >= '2024-01-01';

/*
Seq Scan on large_table ...
  Buffers: shared hit=1234 read=45678  ← read มาก = disk I/O
  Planning Time: 0.5 ms
  Execution Time: 4567.89 ms  ← ช้า!
*/

-- ครั้งที่สอง (warm cache):
EXPLAIN (ANALYZE, BUFFERS) 
SELECT * FROM large_table WHERE created_at >= '2024-01-01';

/*
Seq Scan on large_table ...
  Buffers: shared hit=46912  ← ไม่มี read = all from cache!
  Planning Time: 0.4 ms
  Execution Time: 234.56 ms  ← เร็วกว่า ~20x!
*/
```

```sql
-- ตัวอย่างที่ 2: Buffers output ใน EXPLAIN
-- "shared hit=X" = X pages จาก shared_buffers (cache)
-- "shared read=X" = X pages จาก disk
-- "local hit=X" = X pages จาก temp buffer (กรณี temp tables)

-- วิธีคำนวณ buffer cache hit ratio สำหรับ specific query:
-- hit_ratio = shared_hit / (shared_hit + shared_read)
-- ถ้า 0.99 ขึ้นไป = ดีมาก
-- ถ้า < 0.90 = อาจต้องเพิ่ม shared_buffers
```

---

## 10. Cache Monitoring Dashboard Queries

```sql
-- Dashboard Query 1: Overall Cache Health
SELECT 
    'Overall Table Cache Hit Rate' AS metric,
    ROUND(sum(heap_blks_hit)::numeric / 
          NULLIF(sum(heap_blks_hit) + sum(heap_blks_read), 0) * 100, 2) || '%' AS value
FROM pg_statio_user_tables

UNION ALL
SELECT 
    'Overall Index Cache Hit Rate',
    ROUND(sum(idx_blks_hit)::numeric / 
          NULLIF(sum(idx_blks_hit) + sum(idx_blks_read), 0) * 100, 2) || '%'
FROM pg_statio_user_indexes

UNION ALL
SELECT 
    'Shared Buffers Allocated',
    pg_size_pretty(setting::bigint * 8192)
FROM pg_settings
WHERE name = 'shared_buffers'

UNION ALL
SELECT 
    'Database Cache Usage',
    pg_size_pretty(
        (SELECT count(*) FROM pg_buffercache WHERE relfilenode IS NOT NULL) * 8192
    );
```

```sql
-- Dashboard Query 2: Top cache consumers
SELECT 
    c.relname AS object_name,
    c.relkind,
    COUNT(*) AS cached_pages,
    pg_size_pretty(COUNT(*) * 8192) AS cached_size,
    ROUND(COUNT(*) * 100.0 / (SELECT count(*) FROM pg_buffercache), 2) AS pct_of_cache
FROM pg_buffercache bc
JOIN pg_class c ON c.relfilenode = bc.relfilenode
WHERE bc.isdirty IS NOT NULL
GROUP BY c.relname, c.relkind
ORDER BY cached_pages DESC
LIMIT 15;
```

```sql
-- Dashboard Query 3: Cache eviction rate monitoring
-- ดู bgwriter statistics (เขียน dirty pages กลับ disk)
SELECT 
    checkpoints_timed,
    checkpoints_req,
    checkpoint_write_time,
    checkpoint_sync_time,
    buffers_checkpoint,
    buffers_clean,
    buffers_backend,
    buffers_backend_fsync,
    maxwritten_clean,  -- เกิน = bgwriter ทำงานหนักเกิน
    buffers_alloc,     -- ทั้งหมดที่ allocate
    ROUND(buffers_backend::numeric / NULLIF(buffers_alloc, 0) * 100, 2) AS backend_write_pct
FROM pg_stat_bgwriter;

-- buffers_backend สูง = backend processes ต้องเขียน dirty pages เอง
-- = แสดงว่า bgwriter ไม่เร็วพอ
-- แก้: เพิ่ม bgwriter_lru_maxpages หรือ bgwriter_delay
```

---

## 11. work_mem Tuning

`work_mem` กำหนด memory ต่อ sort/hash operation

```sql
-- ดู work_mem ปัจจุบัน
SHOW work_mem;  -- Default = 4MB

-- ปัญหา: work_mem น้อยเกิน = External Sort (ใช้ disk)
EXPLAIN ANALYZE SELECT * FROM large_table ORDER BY created_at;
/*
Sort  (cost=...) 
  Sort Method: external merge  Disk: 45678kB  ← ใช้ disk!
*/

-- เพิ่ม work_mem สำหรับ session นี้:
SET work_mem = '64MB';

-- ลอง query อีกครั้ง:
EXPLAIN ANALYZE SELECT * FROM large_table ORDER BY created_at;
/*
Sort  (cost=...)
  Sort Method: quicksort  Memory: 34234kB  ← ใน memory!
  Execution Time: 234 ms  ← เร็วขึ้น!
*/

-- คำเตือน: work_mem คูณด้วย connections และ operations
-- 200 connections × 10 sort ops × 64MB = 128GB!!! 
-- ต้องระวัง OOM
```

---

## แบบฝึกหัด (10 ข้อ)

**ข้อ 1:** อธิบายความแตกต่างระหว่าง shared_buffers และ OS page cache ใน PostgreSQL

**เฉลยข้อ 1:**
- **shared_buffers**: Buffer ของ PostgreSQL เอง, เก็บ database pages ใน shared memory, ควบคุมโดย PostgreSQL, ต้อง restart เพื่อเปลี่ยนขนาด
- **OS page cache**: Buffer ของ OS, เก็บ file content รวมถึง database files, ควบคุมโดย OS kernel, จัดการโดยอัตโนมัติ

PostgreSQL มี "double buffering": data อยู่ทั้งใน shared_buffers และ OS page cache เป็น overhead แต่ OS cache ช่วยเมื่อ PostgreSQL flush dirty pages

---

**ข้อ 2:** เขียน query เพื่อตรวจสอบ cache hit ratio ของ top 10 tables ที่มี disk reads สูงที่สุด

**เฉลยข้อ 2:**
```sql
SELECT 
    relname AS table_name,
    heap_blks_read AS disk_reads,
    heap_blks_hit AS cache_hits,
    ROUND(heap_blks_hit::numeric / 
          NULLIF(heap_blks_hit + heap_blks_read, 0) * 100, 2) AS hit_ratio_pct
FROM pg_statio_user_tables
WHERE heap_blks_read > 0
ORDER BY disk_reads DESC
LIMIT 10;
```

---

**ข้อ 3:** แนะนำ shared_buffers สำหรับ PostgreSQL server ที่มี RAM 64GB ที่ใช้เป็น dedicated database server

**เฉลยข้อ 3:**
แนะนำ: shared_buffers = 16GB (25% ของ 64GB)
เหตุผล: 25% เป็น rule of thumb ที่ดีเพราะ:
- เหลือ 75% สำหรับ OS page cache (ซึ่งก็ cache database files ด้วย)
- เหลือ work_mem, maintenance_work_mem, wal_buffers
- เหลือสำหรับ connection overhead

สำหรับ workload หนัก: อาจเพิ่มถึง 40% (25.6GB) แต่ test ก่อน

---

**ข้อ 4:** อธิบาย "Sort Method: external merge Disk: 45MB" ในผล EXPLAIN ANALYZE และวิธีแก้ไข

**เฉลยข้อ 4:**
External merge sort หมายความว่า query ต้องการ sort data ที่ใหญ่เกินกว่า work_mem ทำให้ต้องใช้ disk เป็น temporary storage ซึ่งช้ากว่า in-memory sort มาก แก้โดย:
1. `SET work_mem = '256MB';` สำหรับ session นี้
2. `SET work_mem` ใน postgresql.conf เพิ่ม globally (ระวัง OOM)
3. สร้าง index บน ORDER BY column (ใช้ Index Scan แทน Sort)
4. ลดข้อมูลก่อน sort ด้วย WHERE clause

---

**ข้อ 5:** อธิบายความสำคัญของ Cache Warm-up และวิธีทำใน PostgreSQL

**เฉลยข้อ 5:**
หลัง server restart, shared_buffers จะว่างเปล่า → ทุก query ต้องอ่านจาก disk → ช้ามาก (cold start penalty) Cache Warm-up คือการ pre-load pages เข้า cache ก่อนที่ users จะใช้งาน:

```sql
-- วิธีที่ 1: pg_prewarm
CREATE EXTENSION pg_prewarm;
SELECT pg_prewarm('critical_table');
SELECT pg_prewarm('important_index');

-- วิธีที่ 2: Auto warm-up (postgresql.conf)
shared_preload_libraries = 'pg_prewarm'
pg_prewarm.autoprewarm = on

-- วิธีที่ 3: Manual scan
SELECT COUNT(*) FROM critical_table;  -- forces pages into cache
```

---

**ข้อ 6:** เมื่อใดควรใช้ Materialized View แทน Result Caching ระดับ Application?

**เฉลยข้อ 6:**
ใช้ Materialized View เมื่อ:
1. Query ซับซ้อนมากที่รันเร็วไม่ได้ (aggregations ใหญ่, joins หลายตาราง)
2. Data เปลี่ยนช้า (daily, hourly refresh เพียงพอ)
3. ต้องการ database-level consistency
4. ต้องการ query ด้วย SQL เพิ่มเติม (JOIN กับ materialized view)

ใช้ Application Cache เมื่อ:
1. ต้องการ ms-level latency
2. Cache invalidation complex (ต้องทำใน application logic)
3. ต้องการ share cache ระหว่าง multiple database instances

---

**ข้อ 7:** เขียน query เพื่อ monitor InnoDB Buffer Pool hit rate ใน MySQL และแปลผล

**เฉลยข้อ 7:**
```sql
SELECT 
    (1 - (
        (SELECT variable_value FROM performance_schema.global_status 
         WHERE variable_name = 'Innodb_buffer_pool_reads')::float
        /
        (SELECT variable_value FROM performance_schema.global_status 
         WHERE variable_name = 'Innodb_buffer_pool_read_requests')::float
    )) * 100 AS hit_rate_pct;

-- แปลผล:
-- > 99%: ดีมาก, buffer pool ใหญ่พอ
-- 95-99%: ดี, ยอมรับได้
-- 90-95%: ต้องพิจารณาเพิ่ม buffer pool
-- < 90%: ต้องเพิ่ม innodb_buffer_pool_size ด่วน
```

---

**ข้อ 8:** Read Replica ช่วยลด load บน primary database อย่างไร? มีข้อจำกัดอะไร?

**เฉลยข้อ 8:**
Read Replica รับ SELECT queries แทน primary:
- Primary: เฉพาะ INSERT/UPDATE/DELETE + สำคัญ SELECT
- Replica: รับ reporting queries, analytics, BI tools

ข้อจำกัด:
1. Replication lag: replica อาจมีข้อมูลล่าช้า (seconds-minutes)
2. Read-only: ไม่สามารถ write ได้
3. Eventual consistency: ไม่ใช่ real-time synchronization
4. Network bandwidth: binary log streaming ใช้ bandwidth
5. ต้องจัดการ connection routing ที่ application level

---

**ข้อ 9:** อธิบาย "Buffers: shared hit=X read=Y" ใน EXPLAIN (BUFFERS) output และวิธีใช้ปรับปรุง query

**เฉลยข้อ 9:**
- `shared hit=X`: X pages อ่านจาก shared_buffers (ไม่ใช้ disk) → เร็ว
- `shared read=Y`: Y pages อ่านจาก disk (cache miss) → ช้า
- Hit ratio = X / (X + Y) ควร > 99%

ถ้า read มาก:
1. เพิ่ม shared_buffers
2. Warm up cache ก่อน: `SELECT pg_prewarm('table')`
3. ปรับ index ให้ดึงข้อมูลน้อยลง
4. ตรวจว่า table ใหญ่เกินกว่า RAM หรือไม่

---

**ข้อ 10:** สร้าง script ที่ warm up top 5 tables ที่มี total I/O สูงที่สุด ใน PostgreSQL

**เฉลยข้อ 10:**
```sql
DO $$
DECLARE
    r RECORD;
    v_result BIGINT;
BEGIN
    RAISE NOTICE 'Starting cache warm-up for top tables...';
    
    FOR r IN 
        SELECT relname
        FROM pg_statio_user_tables
        ORDER BY heap_blks_hit + heap_blks_read DESC NULLS LAST
        LIMIT 5
    LOOP
        BEGIN
            v_result := pg_prewarm(r.relname::regclass);
            RAISE NOTICE 'Warmed table: % (% pages loaded)', r.relname, v_result;
        EXCEPTION WHEN OTHERS THEN
            RAISE NOTICE 'Could not warm: % - %', r.relname, SQLERRM;
        END;
    END LOOP;
    
    RAISE NOTICE 'Cache warm-up complete!';
END $$;
```

---

## สรุป

ใน Part 068 เราได้เรียนรู้:

1. **Buffer Pool/Shared Buffers** - Memory structure ของ PostgreSQL และ MySQL
2. **OS Page Cache** - Double buffering และ interaction กับ shared_buffers
3. **pg_buffercache** - ตรวจสอบ buffer cache ระดับ page
4. **Cache Hit Ratio** - Monitoring และ interpretation
5. **Result Caching** - Application cache vs Materialized Views
6. **Cache Warming** - pg_prewarm, InnoDB dump/restore
7. **InnoDB Buffer Pool** - MySQL buffer monitoring
8. **Read Replicas** - Scale reads โดย distribute load
9. **work_mem** - Tuning sort/hash operations
10. **Performance Impact** - EXPLAIN BUFFERS analysis

ใน Part 069 เราจะเรียนรู้เกี่ยวกับ Table Partitioning สำหรับ performance
