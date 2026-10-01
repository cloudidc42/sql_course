# Part 77: MVCC - Multi-Version Concurrency Control

## บทนำ

**MVCC (Multi-Version Concurrency Control)** คือ technique ที่ฐานข้อมูลใช้เพื่อให้หลาย transactions ทำงานพร้อมกันได้โดยไม่ต้องรอกัน แทนที่จะ lock data สำหรับ reads MVCC สร้าง **"version" หลายอัน** ของข้อมูล เพื่อให้แต่ละ transaction เห็น snapshot ที่สอดคล้องกัน

**หลักการ**: Readers ไม่ block Writers, Writers ไม่ block Readers

---

## 1. MVCC Concept และ How It Works

```sql
-- ก่อน MVCC (Lock-Based):
-- เมื่อ T1 กำลัง UPDATE → T2 ต้องรอจนกว่า T1 จะ COMMIT
-- → ประสิทธิภาพต่ำเมื่อมี concurrent users

-- MVCC Approach:
-- เมื่อ T1 UPDATE → สร้าง version ใหม่ของ row
-- T2 ยังคงเห็น version เก่า (snapshot) โดยไม่ต้องรอ
-- เมื่อ T1 COMMIT → version ใหม่กลายเป็น "current"

-- ลำดับเหตุการณ์:
/*
Time →

Row State:
v1: (id=1, balance=1000, xmin=100, xmax=0)

T1 (txid=200): BEGIN
T1: UPDATE accounts SET balance=2000 WHERE id=1

Row State:
v1: (id=1, balance=1000, xmin=100, xmax=200)  ← old version, marked for deletion
v2: (id=1, balance=2000, xmin=200, xmax=0)   ← new version

T2 (txid=201): SELECT balance FROM accounts WHERE id=1
→ T2 sees v1 (balance=1000) because T1 hasn't committed yet

T1: COMMIT

T2: SELECT balance FROM accounts WHERE id=1 (READ COMMITTED)
→ T2 sees v2 (balance=2000) because T1 committed

If T2 is REPEATABLE READ:
→ T2 still sees v1 (balance=1000) from its snapshot
*/

-- ดู system columns ที่ MVCC ใช้
SELECT 
    xmin,      -- Transaction ID ที่สร้าง row นี้
    xmax,      -- Transaction ID ที่ลบ/update row นี้ (0 = current)
    ctid,      -- Physical location: (page_number, tuple_number)
    cmin,      -- Command ID ที่สร้าง row นี้
    cmax,      -- Command ID ที่ลบ row นี้
    *
FROM accounts
WHERE account_id = 1;
```

---

## 2. xmin และ xmax System Columns

```sql
-- สร้างตารางสำหรับสาธิต
CREATE TABLE mvcc_demo (
    id      SERIAL PRIMARY KEY,
    value   TEXT,
    updated TIMESTAMPTZ DEFAULT NOW()
);

INSERT INTO mvcc_demo (value) VALUES ('initial');

-- ดู xmin หลัง INSERT
SELECT xmin, xmax, ctid, * FROM mvcc_demo;
-- xmin = transaction ID ของ INSERT
-- xmax = 0 (ยังไม่ถูก delete/update)

-- ทำ UPDATE และสังเกต
BEGIN;
    UPDATE mvcc_demo SET value = 'updated' WHERE id = 1;
    -- ดู xmin/xmax ระหว่าง transaction
    SELECT xmin, xmax, ctid, * FROM mvcc_demo;
    -- เห็น row ใหม่: xmin = current txid, xmax = 0
    -- row เก่า: xmax = current txid (marked for deletion)
COMMIT;

-- หลัง COMMIT
SELECT xmin, xmax, ctid, * FROM mvcc_demo;
-- เห็นเฉพาะ row ใหม่

-- ดู Transaction ID ปัจจุบัน
SELECT txid_current();
SELECT txid_current_snapshot();
-- txid_current_snapshot(): xmin:xmax:xip_list
-- xmin = oldest active transaction
-- xmax = next transaction ID to be assigned
-- xip_list = list of active transaction IDs

-- Row Visibility Rules:
-- Row ถูก "visible" ถ้า:
-- 1. xmin committed AND (xmax = 0 OR xmax not committed)
-- หรือ
-- 2. xmin = current transaction AND cmin < current command

-- ดูว่า row visible ได้อย่างไร (approximate check)
SELECT 
    id,
    value,
    xmin,
    xmax,
    txid_current() AS current_txid,
    txid_current_snapshot() AS current_snapshot,
    CASE 
        WHEN xmax = 0 THEN 'Currently visible'
        ELSE 'Marked for deletion by txn ' || xmax::TEXT
    END AS visibility_status
FROM mvcc_demo;
```

---

## 3. Transaction IDs และ Snapshots

```sql
-- PostgreSQL ใช้ 32-bit transaction IDs
-- ค่า special: 0=invalid, 1=bootstrap, 2=frozen

-- Transaction ID Wraparound:
-- เมื่อใช้ครบ ~2 billion transactions → wraparound
-- ทำให้ old rows ดูเหมือน "in the future" และถูกซ่อน

-- ดูสถานะ transaction ID ในฐานข้อมูล
SELECT 
    datname,
    age(datfrozenxid) AS txid_age,
    2147483648 - age(datfrozenxid) AS txids_before_wraparound
FROM pg_database
ORDER BY age(datfrozenxid) DESC;

-- Warning: ถ้า txid_age > 1.5 billion → ต้อง VACUUM เร่งด่วน!
-- ถ้า txid_age > 2 billion → database จะ refuse connections!

-- ดู tables ที่มีความเสี่ยง wraparound
SELECT 
    schemaname,
    tablename,
    age(relfrozenxid) AS table_txid_age,
    pg_size_pretty(pg_total_relation_size(schemaname || '.' || tablename)) AS size
FROM pg_stat_user_tables
JOIN pg_class ON relname = tablename
WHERE age(relfrozenxid) > 500000000  -- > 500M transactions
ORDER BY age(relfrozenxid) DESC;

-- Freeze old transactions (VACUUM FREEZE)
-- Freezing: ตั้ง xmin = 2 (frozen XID) เพื่อป้องกัน wraparound
VACUUM FREEZE mvcc_demo;  -- Freeze ทุก rows ใน table นั้น

-- ดูหลัง FREEZE
SELECT xmin, xmax, * FROM mvcc_demo;
-- xmin จะเป็น 2 (frozen) สำหรับ rows ที่ถูก freeze

-- autovacuum จัดการ freezing อัตโนมัติตาม:
-- vacuum_freeze_min_age (default 50M)
-- vacuum_freeze_table_age (default 150M)
SHOW vacuum_freeze_min_age;
SHOW vacuum_freeze_table_age;
```

---

## 4. Visibility Rules ใน Detail

```sql
-- Row visibility algorithm ใน PostgreSQL:
/*
A row is visible to transaction T if:

1. xmin is committed AND
   (xmax = 0 OR xmax is not committed at T's snapshot time)
   
2. OR xmin = T AND cmin < T's current command
   (ข้อมูลที่ T เพิ่งสร้างในระหว่าง transaction)

Snapshot model:
- xmin_snapshot = oldest active transaction ID at snapshot time
- xmax_snapshot = next transaction ID to be assigned
- xip_list = list of in-progress transaction IDs

Row's xmin is visible to snapshot if:
- xmin < xmin_snapshot (committed before snapshot) AND
  xmin NOT IN xip_list (was committed, not in-progress)
- OR xmin = T's own transaction ID
*/

-- สาธิตด้วย dblink (ถ้า enabled) หรือ pg_background
-- แสดงว่า snapshot ถูก isolated

-- ตัวอย่างง่าย: ดู snapshot behavior
DO $$
DECLARE
    v_snapshot pg_snapshot;
    v_xmin     BIGINT;
    v_xmax     BIGINT;
BEGIN
    v_snapshot := txid_current_snapshot();
    v_xmin := txid_snapshot_xmin(v_snapshot);
    v_xmax := txid_snapshot_xmax(v_snapshot);
    
    RAISE NOTICE 'Current snapshot: %', v_snapshot;
    RAISE NOTICE 'xmin (oldest active): %', v_xmin;
    RAISE NOTICE 'xmax (next to assign): %', v_xmax;
    RAISE NOTICE 'In-progress txids: %', txid_snapshot_xip(v_snapshot);
END;
$$;

-- Check ว่า transaction ใด visible ใน snapshot ปัจจุบัน
SELECT 
    txid_current() AS my_txid,
    txid_visible_in_snapshot(txid_current(), txid_current_snapshot()) AS am_i_visible;
-- FALSE เพราะ current transaction ยังไม่ committed

-- ดู committed transactions ใน range
SELECT 
    txid_status(txid_current() - 5) AS five_ago,
    txid_status(txid_current() - 1) AS one_ago,
    txid_status(txid_current())     AS current;
-- 'committed', 'committed', 'in progress'
```

---

## 5. Vacuum และ Dead Tuples

```sql
-- Dead Tuples: versions เก่าของ rows ที่ไม่มี transaction เห็นอีกต่อไป

-- เมื่อเกิด dead tuples:
-- 1. UPDATE → เพิ่ม new version, old version เป็น dead tuple
-- 2. DELETE → row กลายเป็น dead tuple
-- 3. ROLLBACK → new version กลายเป็น dead tuple

-- ดู dead tuples
SELECT 
    tablename,
    n_live_tup AS live_tuples,
    n_dead_tup AS dead_tuples,
    ROUND(n_dead_tup::NUMERIC / NULLIF(n_live_tup + n_dead_tup, 0) * 100, 2) AS dead_pct,
    last_vacuum,
    last_autovacuum,
    last_analyze
FROM pg_stat_user_tables
WHERE n_dead_tup > 0
ORDER BY dead_pct DESC;

-- สร้าง dead tuples
BEGIN;
UPDATE mvcc_demo SET value = 'v2' WHERE id = 1;
UPDATE mvcc_demo SET value = 'v3' WHERE id = 1;
UPDATE mvcc_demo SET value = 'v4' WHERE id = 1;
COMMIT;
-- ตอนนี้มี 3 dead tuples (v1, v2, v3) และ 1 live tuple (v4)

-- Check ก่อน VACUUM
SELECT n_live_tup, n_dead_tup FROM pg_stat_user_tables WHERE tablename = 'mvcc_demo';

-- VACUUM เก็บ dead tuples
VACUUM mvcc_demo;
-- หรือ VACUUM VERBOSE เพื่อดู details
VACUUM VERBOSE mvcc_demo;

-- Check หลัง VACUUM
SELECT n_live_tup, n_dead_tup FROM pg_stat_user_tables WHERE tablename = 'mvcc_demo';
-- n_dead_tup ควรลดลง

-- VACUUM FULL: เก็บ dead tuples AND compact disk space
-- (ต้องการ exclusive lock!)
-- VACUUM FULL mvcc_demo;  -- ระวัง: blocks ทุก operations ชั่วคราว!

-- Autovacuum configuration
SHOW autovacuum;                        -- on/off
SHOW autovacuum_vacuum_threshold;       -- จำนวน dead tuples ก่อน trigger (default 50)
SHOW autovacuum_vacuum_scale_factor;    -- % ของ live tuples (default 0.2 = 20%)
SHOW autovacuum_analyze_threshold;      -- ก่อน analyze
SHOW autovacuum_analyze_scale_factor;   -- % ก่อน analyze

-- Autovacuum จะ trigger เมื่อ dead tuples > threshold + scale_factor * live_tuples
-- เช่น: 50 + 0.2 * 1000 = 250 dead tuples → autovacuum จะทำงาน

-- ดู autovacuum ที่กำลังทำงาน
SELECT 
    pid,
    now() - xact_start AS duration,
    query
FROM pg_stat_activity
WHERE query LIKE 'autovacuum%'
ORDER BY duration DESC;

-- ตั้งค่า autovacuum per-table
ALTER TABLE hot_table SET (
    autovacuum_vacuum_scale_factor = 0.05,  -- Vacuum เมื่อ 5% dead
    autovacuum_vacuum_threshold = 10,
    autovacuum_analyze_scale_factor = 0.02
);

-- ดู table-specific autovacuum settings
SELECT 
    relname AS table_name,
    reloptions AS table_options
FROM pg_class
WHERE reloptions IS NOT NULL
  AND relkind = 'r'
ORDER BY relname;
```

---

## 6. HOT (Heap Only Tuple) Updates

```sql
-- HOT Update: Optimization ที่ลด overhead ของ MVCC

-- Regular UPDATE:
-- 1. สร้าง new version ใน new location (อาจ different page)
-- 2. Update ทุก indexes ให้ชี้ไป new version
-- → Overhead: อัปเดต indexes ด้วย

-- HOT Update (Heap Only Tuple):
-- 1. สร้าง new version ใน SAME page เป็น old version
-- 2. ไม่ต้องอัปเดต indexes! (indexes ยังชี้ไป old version แต่มี chain)
-- Condition: ต้องมี space ใน page เดิม AND ไม่เปลี่ยน indexed columns

-- ตรวจสอบว่า HOT Update ทำงานหรือไม่
SELECT 
    n_tup_upd AS total_updates,
    n_tup_hot_upd AS hot_updates,
    ROUND(n_tup_hot_upd::NUMERIC / NULLIF(n_tup_upd, 0) * 100, 2) AS hot_update_pct
FROM pg_stat_user_tables
WHERE tablename = 'mvcc_demo';

-- HOT updates สูง = ดีมาก (ประหยัด resources)
-- HOT updates ต่ำ = อาจมีปัญหา (table bloated, indexed columns changed บ่อย)

-- ปัจจัยที่ affect HOT updates:
-- 1. fillfactor: พื้นที่ว่างใน page สำหรับ updates
--    fillfactor=100 (default) = ไม่มีพื้นที่ว่าง → ไม่มี HOT
--    fillfactor=70 = 30% ว่าง → HOT ทำงานได้บ่อยกว่า

-- ตั้ง fillfactor สำหรับ high-update tables
ALTER TABLE high_update_table SET (fillfactor = 70);

-- ดู fillfactor
SELECT relname, reloptions 
FROM pg_class 
WHERE relname = 'high_update_table';

-- 2. ไม่เปลี่ยน indexed columns ใน UPDATE
-- Bad for HOT (เปลี่ยน indexed column):
UPDATE users SET email = 'new@email.com' WHERE id = 1;  -- email มี index
-- Good for HOT (เปลี่ยน non-indexed column):
UPDATE users SET last_login = NOW() WHERE id = 1;        -- last_login ไม่มี index

-- Demo HOT update
CREATE TABLE hot_demo (
    id       SERIAL PRIMARY KEY,
    value    TEXT,
    status   VARCHAR(20),
    updated  TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX ON hot_demo (value);  -- Index บน value
-- ไม่มี index บน status และ updated

ALTER TABLE hot_demo SET (fillfactor = 70);

INSERT INTO hot_demo (value, status) 
SELECT 'item_' || generate_series(1, 1000), 'active';

-- These updates should be HOT (ไม่เปลี่ยน indexed column)
UPDATE hot_demo SET status = 'updated', updated = NOW();

-- Check HOT ratio
SELECT n_tup_hot_upd, n_tup_upd,
       ROUND(n_tup_hot_upd::NUMERIC / n_tup_upd * 100, 2) AS hot_pct
FROM pg_stat_user_tables WHERE tablename = 'hot_demo';
-- ควรเห็น HOT% สูง
```

---

## 7. Write Amplification ใน MVCC

```sql
-- Write Amplification: การ write ข้อมูลมากกว่าที่จำเป็น
-- MVCC เพิ่ม write amplification เพราะสร้าง new versions แทนที่จะ update in-place

-- ตัวอย่าง: Update row 1,000 ครั้ง
-- → สร้าง 1,000 versions (1 live + 999 dead)
-- → Disk usage เพิ่มขึ้น จนกว่า VACUUM จะทำงาน

-- วัด write amplification
DO $$
DECLARE
    v_before_size BIGINT;
    v_after_size  BIGINT;
    v_writes INT := 1000;
BEGIN
    v_before_size := pg_total_relation_size('mvcc_demo');
    
    FOR i IN 1..v_writes LOOP
        UPDATE mvcc_demo SET value = 'iteration_' || i WHERE id = 1;
    END LOOP;
    
    v_after_size := pg_total_relation_size('mvcc_demo');
    
    RAISE NOTICE 'Before: % bytes, After: % bytes, Amplification: %.2fx',
                  v_before_size, v_after_size,
                  v_after_size::FLOAT / v_before_size;
END;
$$;

-- วิธีลด Write Amplification:
-- 1. ใช้ fillfactor ที่เหมาะสม
-- 2. VACUUM บ่อยขึ้น (autovacuum tuning)
-- 3. Batch updates แทน frequent small updates
-- 4. Update only changed columns

-- ตัวอย่าง: Batch update แทน individual updates
-- ไม่ดี: Update ทีละแถว 1,000 ครั้ง
-- FOR i IN 1..1000 LOOP
--     UPDATE counters SET value = value + 1 WHERE id = i;
-- END LOOP;

-- ดี: Update ทั้งหมดครั้งเดียว
UPDATE counters SET value = value + 1 WHERE id BETWEEN 1 AND 1000;
-- สร้าง dead tuples เท่ากัน แต่ overhead ต่ำกว่ามาก

-- Table Bloat Monitoring
SELECT 
    tablename,
    pg_size_pretty(pg_total_relation_size(schemaname || '.' || tablename)) AS total_size,
    pg_size_pretty(pg_relation_size(schemaname || '.' || tablename)) AS table_size,
    pg_size_pretty(pg_total_relation_size(schemaname || '.' || tablename) 
                   - pg_relation_size(schemaname || '.' || tablename)) AS index_size,
    n_live_tup,
    n_dead_tup,
    ROUND(n_dead_tup::NUMERIC / NULLIF(n_live_tup + n_dead_tup, 0) * 100, 2) AS bloat_pct
FROM pg_stat_user_tables
ORDER BY pg_total_relation_size(schemaname || '.' || tablename) DESC
LIMIT 20;
```

---

## 8. MVCC vs Lock-Based Concurrency

```sql
-- เปรียบเทียบ:

-- Lock-Based (Oracle traditional, SQL Server):
/*
READ:
  1. Acquire shared lock
  2. Read data
  3. Release lock
  
WRITE:
  1. Acquire exclusive lock
  2. Modify data in-place
  3. Release lock
  
ปัญหา: Readers block writers, Writers block readers
*/

-- MVCC (PostgreSQL, MySQL InnoDB, Oracle >=9i):
/*
READ (no lock needed):
  1. Note current snapshot
  2. Read appropriate version

WRITE:
  1. Acquire row lock
  2. Create new version
  3. Keep old version for concurrent readers
  
ข้อดี: Readers ไม่ block writers!
ข้อเสีย: Dead tuples สะสม → ต้องการ VACUUM
*/

-- Benchmark: Concurrent reads and writes
DO $$
DECLARE
    v_readers INT := 10;
    v_writers INT := 5;
    v_start TIMESTAMPTZ := clock_timestamp();
BEGIN
    -- PostgreSQL MVCC: readers และ writers ทำงานพร้อมกัน
    -- ใน Lock-Based: readers จะ block writers หรือ vice versa
    
    -- ใน PostgreSQL, การ SELECT ที่กำลังทำอยู่
    -- ไม่ block UPDATE ใน table เดียวกัน
    RAISE NOTICE 'MVCC allows % concurrent reads without blocking % concurrent writes',
                  v_readers, v_writers;
END;
$$;

-- ดู Read/Write patterns ใน production
SELECT 
    tablename,
    seq_scan AS full_table_scans,
    idx_scan AS index_scans,
    n_tup_ins AS inserts,
    n_tup_upd AS updates,
    n_tup_del AS deletes,
    n_tup_hot_upd AS hot_updates,
    ROUND(n_tup_hot_upd::NUMERIC / NULLIF(n_tup_upd, 0) * 100, 2) AS hot_update_rate
FROM pg_stat_user_tables
ORDER BY (n_tup_ins + n_tup_upd + n_tup_del) DESC
LIMIT 10;
```

---

## 9. MySQL InnoDB MVCC

```sql
-- MySQL InnoDB ใช้ MVCC เช่นกัน แต่ implementation ต่างกัน

-- MySQL ใช้:
-- 1. Undo Log: เก็บ previous versions ของ rows
-- 2. Read View: Snapshot ที่สร้างตอน transaction เริ่ม

-- ดู InnoDB MVCC stats (MySQL)
-- SHOW ENGINE INNODB STATUS\G

-- ดู undo log usage (MySQL)
/*
SELECT 
    name,
    subsystem,
    count,
    avg_count,
    min_count,
    max_count
FROM sys.metrics
WHERE name LIKE '%undo%';
*/

-- MySQL Read View:
-- READ COMMITTED: สร้าง new read view สำหรับแต่ละ read
-- REPEATABLE READ: สร้าง read view ครั้งเดียวตอน transaction เริ่ม

-- ดู InnoDB history list length (ความยาวของ undo log chain)
-- ถ้ายาวมาก → มี long-running transactions หรือ purge lag
/*
SELECT 
    variable_name,
    variable_value
FROM performance_schema.global_status
WHERE variable_name = 'Innodb_history_list_length';
*/

-- PostgreSQL equivalent สำหรับ comparison:
SELECT 
    pg_stat_activity.pid,
    now() - pg_stat_activity.xact_start AS txn_age,
    pg_stat_activity.state
FROM pg_stat_activity
WHERE pg_stat_activity.xact_start IS NOT NULL
ORDER BY txn_age DESC;
-- Long transactions → many old versions ต้องเก็บ (เหมือน long InnoDB history)
```

---

## 10. MVCC Internals - Page Structure

```sql
-- PostgreSQL ใช้ Heap files สำหรับ tables
-- แต่ละ Page มีขนาด 8KB (ค่าเริ่มต้น)
-- แต่ละ Page มี: header, tuple data, free space

-- ดู page/block information
SELECT 
    relpages AS total_pages,
    reltuples AS estimated_rows,
    relpages * 8 AS size_kb,
    ROUND(reltuples / NULLIF(relpages, 0), 0) AS rows_per_page
FROM pg_class
WHERE relname = 'accounts';

-- ดู heap bloat (ประมาณ)
-- ใช้ extension pgstattuple ถ้า available
-- CREATE EXTENSION IF NOT EXISTS pgstattuple;
-- SELECT * FROM pgstattuple('accounts');
-- ดู tuple_count, dead_tuple_count, free_space

-- ดู fragmentation
SELECT 
    schemaname,
    tablename,
    pg_size_pretty(pg_relation_size(schemaname || '.' || tablename)) AS table_size,
    pg_size_pretty(pg_total_relation_size(schemaname || '.' || tablename)) AS total_size
FROM pg_stat_user_tables
ORDER BY pg_relation_size(schemaname || '.' || tablename) DESC;

-- Page-level visibility map
-- PostgreSQL เก็บ visibility map: แต่ละ bit แทน 1 page
-- ถ้า bit = 1 → ทุก tuples ใน page นั้น visible ต่อทุก transactions
-- (ไม่ต้องตรวจ xmin/xmax ของแต่ละ row)
-- VACUUM อัปเดต visibility map → ทำให้ Sequential scans เร็วขึ้น

SELECT 
    relname AS table_name,
    pg_size_pretty(pg_relation_size(oid)) AS heap_size,
    pg_size_pretty(pg_relation_size(oid, 'vm')) AS vm_size,   -- Visibility Map
    pg_size_pretty(pg_relation_size(oid, 'fsm')) AS fsm_size  -- Free Space Map
FROM pg_class
WHERE relkind = 'r'
  AND relnamespace != (SELECT oid FROM pg_namespace WHERE nspname = 'pg_catalog')
ORDER BY pg_relation_size(oid) DESC
LIMIT 10;
```

---

## 11. VACUUM ใน Detail

```sql
-- VACUUM ทำอะไรบ้าง:
-- 1. Mark dead tuples ว่า reclaimable
-- 2. Update Free Space Map (FSM)
-- 3. Update Visibility Map
-- 4. Update statistics สำหรับ query planner
-- 5. Prevent transaction ID wraparound (VACUUM FREEZE)

-- ประเภทของ VACUUM:
-- VACUUM (regular): ทำแบบ concurrent กับ operations อื่น
-- VACUUM FULL: เขียน table ใหม่ทั้งหมด, ต้องการ exclusive lock
-- VACUUM ANALYZE: VACUUM + analyze statistics
-- VACUUM FREEZE: Force freeze ของ transactions เก่า
-- VACUUM VERBOSE: แสดง details ของการทำงาน

-- ดู VACUUM progress (PostgreSQL 9.6+)
SELECT 
    pid,
    phase,
    heap_blks_total,
    heap_blks_scanned,
    heap_blks_vacuumed,
    index_vacuum_count,
    num_dead_tuples
FROM pg_stat_progress_vacuum;

-- Manual VACUUM สำหรับ critical tables
VACUUM (VERBOSE, ANALYZE) accounts;
-- Output:
-- INFO: vacuuming "public.accounts"
-- INFO: scanned index "accounts_pkey" to remove 1000 row versions
-- INFO: "accounts": removed 1000 row versions in 50 pages
-- INFO: "accounts": found 0 removable, 5000 nonremovable row versions in 250 pages
-- ...

-- Autovacuum Tuning
-- Tuning per-table สำหรับ high-traffic tables
ALTER TABLE orders SET (
    autovacuum_vacuum_cost_delay = 2,       -- ms, lower = less IO throttling
    autovacuum_vacuum_cost_limit = 800,     -- higher = vacuum faster
    autovacuum_vacuum_scale_factor = 0.01,  -- trigger at 1% dead rows
    autovacuum_analyze_scale_factor = 0.005 -- analyze at 0.5% changes
);

-- Global autovacuum settings (postgresql.conf)
SHOW autovacuum_vacuum_cost_delay;
SHOW autovacuum_vacuum_cost_limit;
SHOW autovacuum_max_workers;

-- Force VACUUM on all user tables
DO $$
DECLARE
    r RECORD;
BEGIN
    FOR r IN 
        SELECT schemaname, tablename 
        FROM pg_stat_user_tables
        WHERE n_dead_tup > 1000
        ORDER BY n_dead_tup DESC
    LOOP
        RAISE NOTICE 'VACUUMing %.%', r.schemaname, r.tablename;
        EXECUTE format('VACUUM ANALYZE %I.%I', r.schemaname, r.tablename);
    END LOOP;
END;
$$;

-- VACUUM ANALYZE ทั้ง database
VACUUM ANALYZE;

-- ดูว่า autovacuum ทำงานได้ดีไหม
SELECT 
    schemaname,
    tablename,
    n_dead_tup,
    n_live_tup,
    ROUND(n_dead_tup::NUMERIC / NULLIF(n_live_tup + n_dead_tup, 0) * 100, 2) AS dead_pct,
    last_autovacuum,
    last_autoanalyze,
    autovacuum_count,
    CASE 
        WHEN last_autovacuum IS NULL THEN 'Never vacuumed!'
        WHEN now() - last_autovacuum > INTERVAL '1 day' THEN 'Overdue'
        ELSE 'Recent'
    END AS vacuum_health
FROM pg_stat_user_tables
WHERE n_dead_tup > 1000
ORDER BY dead_pct DESC;
```

---

## 12. MVCC และ Performance

```sql
-- Impact ของ MVCC ต่อ Performance:

-- 1. Read Performance (ดีมาก)
-- SELECT ไม่ต้องรอ locks → high concurrent reads
-- Snapshot isolation ไม่มี lock overhead สำหรับ reads

-- 2. Write Performance (ดีแต่มี overhead)
-- ต้องสร้าง new version (ไม่ใช่ in-place update)
-- Index ต้องอัปเดต (ยกเว้น HOT updates)
-- Dead tuples สะสม → ต้องการ VACUUM

-- 3. VACUUM Overhead
-- Regular VACUUM: ใช้ CPU/IO ขณะ vacuum
-- VACUUM FULL: ป้องกัน access ชั่วคราว

-- Performance queries:
-- ดู MVCC-related wait events
SELECT 
    wait_event_type,
    wait_event,
    COUNT(*) AS sessions_waiting
FROM pg_stat_activity
WHERE wait_event IS NOT NULL
  AND state = 'active'
GROUP BY wait_event_type, wait_event
ORDER BY sessions_waiting DESC;

-- Cache hit ratio (MVCC ทำงานดีบน cached data)
SELECT 
    ROUND(blks_hit::NUMERIC / NULLIF(blks_hit + blks_read, 0) * 100, 2) AS cache_hit_ratio
FROM pg_stat_database
WHERE datname = current_database();

-- ดู index effectiveness (HOT updates ช่วยลด index churn)
SELECT 
    indexrelname AS index_name,
    idx_scan AS scans,
    idx_tup_read AS tuples_read,
    idx_tup_fetch AS tuples_fetched,
    pg_size_pretty(pg_relation_size(indexrelid)) AS size
FROM pg_stat_user_indexes
ORDER BY idx_scan DESC;

-- Monitor MVCC health dashboard
CREATE OR REPLACE VIEW mvcc_health AS
SELECT 
    -- Table bloat
    (SELECT COUNT(*) FROM pg_stat_user_tables WHERE 
     n_dead_tup::NUMERIC / NULLIF(n_live_tup + n_dead_tup, 0) > 0.2) AS bloated_tables,
    
    -- Oldest transaction age
    (SELECT MAX(age(datfrozenxid)) FROM pg_database 
     WHERE datname NOT IN ('template0', 'template1')) AS max_db_age,
    
    -- Autovacuum pending
    (SELECT COUNT(*) FROM pg_stat_user_tables 
     WHERE n_dead_tup > 10000 AND 
     (last_autovacuum IS NULL OR last_autovacuum < NOW() - INTERVAL '1 hour')) AS pending_vacuum,
    
    -- Cache hit ratio
    (SELECT ROUND(blks_hit::NUMERIC / NULLIF(blks_hit + blks_read, 0) * 100, 2) 
     FROM pg_stat_database WHERE datname = current_database()) AS cache_hit_pct,
    
    -- Long transactions preventing vacuum
    (SELECT COUNT(*) FROM pg_stat_activity 
     WHERE xact_start < NOW() - INTERVAL '10 minutes' 
     AND state != 'idle') AS long_transactions;

SELECT * FROM mvcc_health;
```

---

## 13. Practical MVCC Scenarios

```sql
-- Scenario 1: High-concurrency counter update
-- ปัญหา: Counter ถูก update พร้อมกันโดยหลาย users

-- วิธีที่ไม่ดี (Race condition):
-- T1: SELECT value FROM counters WHERE id = 1;  → 100
-- T2: SELECT value FROM counters WHERE id = 1;  → 100
-- T1: UPDATE counters SET value = 101 WHERE id = 1;
-- T2: UPDATE counters SET value = 101 WHERE id = 1;  -- Lost update!
-- ผล: counter = 101 แทน 102

-- วิธีที่ดี (Atomic update, MVCC-friendly):
BEGIN;
UPDATE page_counters SET views = views + 1 WHERE page_id = 1;
COMMIT;
-- PostgreSQL UPDATE ทำ row lock อัตโนมัติ + atomic increment

-- Scenario 2: Reporting during high-traffic
-- MVCC ทำให้ report ได้ consistent snapshot โดยไม่ block traffic

BEGIN TRANSACTION ISOLATION LEVEL REPEATABLE READ;
    -- Report queries ทั้งหมดเห็น snapshot เดียวกัน
    SELECT COUNT(*) FROM orders WHERE status = 'completed';
    SELECT SUM(amount) FROM orders WHERE status = 'completed';
    SELECT AVG(amount) FROM orders WHERE status = 'completed';
COMMIT;

-- Scenario 3: Materialized views refresh
-- MVCC ทำให้ REFRESH MATERIALIZED VIEW CONCURRENTLY ทำงานได้
-- โดยไม่ lock table

CREATE MATERIALIZED VIEW sales_summary AS
SELECT 
    DATE_TRUNC('day', created_at) AS day,
    COUNT(*) AS order_count,
    SUM(amount) AS total_revenue
FROM orders
GROUP BY DATE_TRUNC('day', created_at);

-- ไม่ block reads!
REFRESH MATERIALIZED VIEW CONCURRENTLY sales_summary;

-- Scenario 4: Detecting stale data
CREATE OR REPLACE FUNCTION check_data_freshness()
RETURNS TABLE(
    table_name TEXT,
    last_vacuum TIMESTAMPTZ,
    last_analyze TIMESTAMPTZ,
    dead_tuple_ratio NUMERIC,
    recommendation TEXT
) AS $$
BEGIN
    RETURN QUERY
    SELECT 
        schemaname || '.' || tablename,
        last_autovacuum,
        last_autoanalyze,
        ROUND(n_dead_tup::NUMERIC / NULLIF(n_live_tup + n_dead_tup, 0) * 100, 2),
        CASE 
            WHEN n_dead_tup::NUMERIC / NULLIF(n_live_tup + n_dead_tup, 0) > 0.3 
                THEN 'VACUUM needed (>30% dead tuples)'
            WHEN last_autovacuum IS NULL 
                THEN 'Never vacuumed - check autovacuum'
            WHEN last_autovacuum < NOW() - INTERVAL '1 day' 
                THEN 'Not vacuumed in 24h - review settings'
            ELSE 'OK'
        END
    FROM pg_stat_user_tables
    ORDER BY n_dead_tup::NUMERIC / NULLIF(n_live_tup + n_dead_tup, 0) DESC NULLS LAST;
END;
$$ LANGUAGE plpgsql;

SELECT * FROM check_data_freshness();
```

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: MVCC Basic Observation

```sql
-- คำตอบ
CREATE TABLE mvcc_obs (
    id      SERIAL PRIMARY KEY,
    value   TEXT,
    counter INT DEFAULT 0
);

INSERT INTO mvcc_obs (value) VALUES ('initial');

-- ดู xmin ก่อนและหลัง update
SELECT xmin, xmax, ctid, * FROM mvcc_obs;

BEGIN;
UPDATE mvcc_obs SET value = 'updated', counter = counter + 1 WHERE id = 1;
-- ดูระหว่าง transaction
SELECT xmin, xmax, ctid, * FROM mvcc_obs;
COMMIT;

-- ดูหลัง commit
SELECT xmin, xmax, ctid, * FROM mvcc_obs;

-- Observation: ctid เปลี่ยนหลัง update (location ใหม่ใน heap)
```

### แบบฝึกหัดที่ 2: Dead Tuple Monitoring

```sql
-- คำตอบ
CREATE TABLE dead_tuple_test (
    id SERIAL PRIMARY KEY,
    value INT DEFAULT 0
);

INSERT INTO dead_tuple_test (value) SELECT generate_series(1, 10000);

-- Check initial state
SELECT n_live_tup, n_dead_tup FROM pg_stat_user_tables WHERE tablename = 'dead_tuple_test';

-- Create dead tuples
UPDATE dead_tuple_test SET value = value + 1;
UPDATE dead_tuple_test SET value = value + 1;
UPDATE dead_tuple_test SET value = value + 1;

-- Refresh stats and check
SELECT pg_stat_reset_single_table_counts('dead_tuple_test'::regclass);
ANALYZE dead_tuple_test;

SELECT tablename, n_live_tup, n_dead_tup,
       ROUND(n_dead_tup::NUMERIC / (n_live_tup + n_dead_tup + 0.001) * 100, 2) AS dead_pct
FROM pg_stat_user_tables WHERE tablename = 'dead_tuple_test';

-- VACUUM
VACUUM VERBOSE dead_tuple_test;

-- Check after VACUUM
ANALYZE dead_tuple_test;
SELECT tablename, n_live_tup, n_dead_tup
FROM pg_stat_user_tables WHERE tablename = 'dead_tuple_test';
```

### แบบฝึกหัดที่ 3: Transaction Snapshot Inspection

```sql
-- คำตอบ
DO $$
DECLARE
    v_snapshot pg_snapshot;
    v_xmin BIGINT;
    v_xmax BIGINT;
    v_xip  BIGINT[];
BEGIN
    v_snapshot := txid_current_snapshot();
    v_xmin := txid_snapshot_xmin(v_snapshot);
    v_xmax := txid_snapshot_xmax(v_snapshot);
    v_xip  := txid_snapshot_xip(v_snapshot);
    
    RAISE NOTICE '=== Transaction Snapshot Analysis ===';
    RAISE NOTICE 'Snapshot: %', v_snapshot;
    RAISE NOTICE 'My Transaction ID: %', txid_current();
    RAISE NOTICE 'Oldest Active TXN (xmin): %', v_xmin;
    RAISE NOTICE 'Next TXN ID (xmax): %', v_xmax;
    RAISE NOTICE 'In-Progress TXNs: %', v_xip;
    RAISE NOTICE 'Active Transaction Count: %', array_length(v_xip, 1);
    RAISE NOTICE '====================================';
END;
$$;
```

### แบบฝึกหัดที่ 4: HOT Update Analysis

```sql
-- คำตอบ
CREATE TABLE hot_analysis (
    id           SERIAL PRIMARY KEY,
    indexed_col  TEXT,
    normal_col   TEXT,
    counter      INT DEFAULT 0
);

CREATE INDEX idx_hot_indexed ON hot_analysis (indexed_col);

ALTER TABLE hot_analysis SET (fillfactor = 70);

INSERT INTO hot_analysis (indexed_col, normal_col)
SELECT 'val_' || i, 'data_' || i FROM generate_series(1, 1000) i;

-- Reset stats
SELECT pg_stat_reset_single_table_counts('hot_analysis'::regclass);

-- Update non-indexed column (should be HOT)
UPDATE hot_analysis SET counter = counter + 1, normal_col = 'updated';
ANALYZE hot_analysis;

SELECT 
    tablename,
    n_tup_upd AS total_updates,
    n_tup_hot_upd AS hot_updates,
    ROUND(n_tup_hot_upd::NUMERIC / NULLIF(n_tup_upd, 0) * 100, 2) AS hot_pct
FROM pg_stat_user_tables WHERE tablename = 'hot_analysis';

-- Reset and update indexed column (should NOT be HOT)
SELECT pg_stat_reset_single_table_counts('hot_analysis'::regclass);
UPDATE hot_analysis SET indexed_col = 'new_' || id;
ANALYZE hot_analysis;

SELECT 
    tablename,
    n_tup_upd AS total_updates,
    n_tup_hot_upd AS hot_updates,
    ROUND(n_tup_hot_upd::NUMERIC / NULLIF(n_tup_upd, 0) * 100, 2) AS hot_pct
FROM pg_stat_user_tables WHERE tablename = 'hot_analysis';
-- HOT% จะต่ำกว่าเพราะ indexed column เปลี่ยน
```

### แบบฝึกหัดที่ 5: VACUUM Tuning

```sql
-- คำตอบ
-- ตั้งค่า autovacuum สำหรับ high-traffic table
CREATE TABLE high_traffic (
    id      SERIAL PRIMARY KEY,
    session TEXT,
    data    JSONB,
    created TIMESTAMPTZ DEFAULT NOW()
);

-- Aggressive autovacuum settings
ALTER TABLE high_traffic SET (
    autovacuum_enabled = true,
    autovacuum_vacuum_threshold = 100,       -- trigger after 100 dead tuples
    autovacuum_vacuum_scale_factor = 0.01,   -- or 1% of table size
    autovacuum_vacuum_cost_delay = 2,        -- 2ms delay (less IO impact)
    autovacuum_vacuum_cost_limit = 1000,     -- higher limit = faster vacuum
    autovacuum_analyze_threshold = 50,
    autovacuum_analyze_scale_factor = 0.005
);

-- ดูการตั้งค่า
SELECT relname, reloptions 
FROM pg_class 
WHERE relname = 'high_traffic';

-- Monitor autovacuum effectiveness
SELECT 
    tablename,
    n_dead_tup,
    autovacuum_count,
    last_autovacuum,
    now() - last_autovacuum AS time_since_vacuum
FROM pg_stat_user_tables
WHERE tablename = 'high_traffic';
```

### แบบฝึกหัดที่ 6: Transaction ID Age Check

```sql
-- คำตอบ
CREATE OR REPLACE FUNCTION check_transaction_age_risk()
RETURNS TABLE(
    database_name TEXT,
    txid_age BIGINT,
    risk_level TEXT,
    action_needed TEXT
) AS $$
BEGIN
    RETURN QUERY
    SELECT 
        datname,
        age(datfrozenxid)::BIGINT,
        CASE 
            WHEN age(datfrozenxid) > 1500000000 THEN 'CRITICAL'
            WHEN age(datfrozenxid) > 750000000  THEN 'HIGH'
            WHEN age(datfrozenxid) > 250000000  THEN 'MEDIUM'
            ELSE 'LOW'
        END,
        CASE 
            WHEN age(datfrozenxid) > 1500000000 THEN 'VACUUM FREEZE immediately!'
            WHEN age(datfrozenxid) > 750000000  THEN 'Schedule VACUUM FREEZE soon'
            WHEN age(datfrozenxid) > 250000000  THEN 'Monitor closely'
            ELSE 'Normal'
        END
    FROM pg_database
    WHERE datname NOT IN ('template0', 'template1')
    ORDER BY age(datfrozenxid) DESC;
END;
$$ LANGUAGE plpgsql;

SELECT * FROM check_transaction_age_risk();
```

### แบบฝึกหัดที่ 7: MVCC Overhead Measurement

```sql
-- คำตอบ
CREATE TABLE mvcc_overhead_test (
    id    SERIAL PRIMARY KEY,
    value TEXT DEFAULT 'initial'
);

INSERT INTO mvcc_overhead_test (value) SELECT 'v0_' || generate_series(1, 10000);

-- Measure size before updates
SELECT pg_size_pretty(pg_relation_size('mvcc_overhead_test')) AS size_before;

-- Create many dead tuples (no VACUUM between updates)
UPDATE mvcc_overhead_test SET value = 'v1_' || id;
UPDATE mvcc_overhead_test SET value = 'v2_' || id;
UPDATE mvcc_overhead_test SET value = 'v3_' || id;

-- Measure bloat
SELECT 
    pg_size_pretty(pg_relation_size('mvcc_overhead_test')) AS current_size,
    (SELECT n_dead_tup FROM pg_stat_user_tables WHERE tablename = 'mvcc_overhead_test') AS dead_tuples,
    'Every UPDATE creates new version, old ones are dead tuples' AS explanation;

-- VACUUM and measure again
VACUUM mvcc_overhead_test;
ANALYZE mvcc_overhead_test;

SELECT 
    pg_size_pretty(pg_relation_size('mvcc_overhead_test')) AS size_after_vacuum,
    (SELECT n_dead_tup FROM pg_stat_user_tables WHERE tablename = 'mvcc_overhead_test') AS dead_after;
```

### แบบฝึกหัดที่ 8: MVCC Read Consistency Test

```sql
-- คำตอบ
-- ทดสอบว่า REPEATABLE READ ให้ snapshot consistent หรือไม่
DO $$
DECLARE
    v_count1 INT;
    v_count2 INT;
    v_sum1   DECIMAL;
    v_sum2   DECIMAL;
BEGIN
    SET LOCAL transaction_isolation = 'repeatable read';
    
    -- First read
    SELECT COUNT(*), SUM(value) INTO v_count1, v_sum1 FROM mvcc_overhead_test;
    
    -- Simulate delay (in real scenario, another session would update here)
    PERFORM pg_sleep(0.01);
    
    -- Second read (should be identical in REPEATABLE READ)
    SELECT COUNT(*), SUM(value::NUMERIC) INTO v_count2, v_sum2 FROM mvcc_overhead_test;
    
    RAISE NOTICE 'First read: count=%, sum=%', v_count1, v_sum1;
    RAISE NOTICE 'Second read: count=%, sum=%', v_count2, v_sum2;
    RAISE NOTICE 'Consistent: %', (v_count1 = v_count2);
END;
$$;
```

### แบบฝึกหัดที่ 9: MVCC Performance Dashboard

```sql
-- คำตอบ
CREATE OR REPLACE VIEW mvcc_performance_dashboard AS
SELECT 
    -- Table health
    (SELECT COUNT(*) FROM pg_stat_user_tables 
     WHERE n_dead_tup::NUMERIC / NULLIF(n_live_tup + n_dead_tup, 0) > 0.3) 
        AS tables_needing_vacuum,
    
    -- HOT update efficiency
    (SELECT ROUND(SUM(n_tup_hot_upd)::NUMERIC / NULLIF(SUM(n_tup_upd), 0) * 100, 2) 
     FROM pg_stat_user_tables) AS global_hot_update_pct,
    
    -- Cache efficiency
    (SELECT ROUND(blks_hit::NUMERIC / NULLIF(blks_hit + blks_read, 0) * 100, 2)
     FROM pg_stat_database WHERE datname = current_database()) AS buffer_cache_hit_pct,
    
    -- Oldest transaction
    (SELECT MAX(now() - xact_start) FROM pg_stat_activity 
     WHERE xact_start IS NOT NULL) AS oldest_transaction_age,
    
    -- Transaction ID age
    (SELECT MAX(age(datfrozenxid)) FROM pg_database 
     WHERE datname NOT IN ('template0', 'template1')) AS max_txid_age,
    
    -- Active autovacuums
    (SELECT COUNT(*) FROM pg_stat_activity WHERE query LIKE 'autovacuum%') AS active_autovacuums,
    
    -- Pending autovacuums
    (SELECT COUNT(*) FROM pg_stat_user_tables 
     WHERE n_dead_tup > autovacuum_vacuum_threshold + 
           autovacuum_vacuum_scale_factor * n_live_tup
     FROM (SELECT current_setting('autovacuum_vacuum_threshold')::INT AS autovacuum_vacuum_threshold,
                  current_setting('autovacuum_vacuum_scale_factor')::FLOAT AS autovacuum_vacuum_scale_factor) settings
    ) AS tables_due_for_vacuum;

SELECT * FROM mvcc_performance_dashboard;
```

### แบบฝึกหัดที่ 10: Complete MVCC Health Check

```sql
-- คำตอบ
CREATE OR REPLACE FUNCTION mvcc_health_check()
RETURNS TABLE(
    category     TEXT,
    metric       TEXT,
    value        TEXT,
    status       TEXT,
    recommendation TEXT
) AS $$
DECLARE
    v_cache_hit   NUMERIC;
    v_hot_pct     NUMERIC;
    v_max_age     BIGINT;
    v_bloated     INT;
    v_oldest_txn  INTERVAL;
BEGIN
    -- Cache hit ratio
    SELECT ROUND(blks_hit::NUMERIC / NULLIF(blks_hit + blks_read, 0) * 100, 2)
    INTO v_cache_hit
    FROM pg_stat_database WHERE datname = current_database();
    
    category := 'MVCC Efficiency';
    metric   := 'Buffer Cache Hit Rate';
    value    := v_cache_hit || '%';
    status   := CASE WHEN v_cache_hit > 95 THEN 'EXCELLENT'
                     WHEN v_cache_hit > 85 THEN 'GOOD'
                     WHEN v_cache_hit > 70 THEN 'FAIR'
                     ELSE 'POOR' END;
    recommendation := CASE WHEN v_cache_hit < 85 THEN 'Consider increasing shared_buffers' 
                           ELSE 'Cache performance is good' END;
    RETURN NEXT;
    
    -- HOT update percentage
    SELECT ROUND(SUM(n_tup_hot_upd)::NUMERIC / NULLIF(SUM(n_tup_upd), 0) * 100, 2)
    INTO v_hot_pct FROM pg_stat_user_tables;
    
    category := 'MVCC Efficiency';
    metric   := 'HOT Update Rate';
    value    := COALESCE(v_hot_pct::TEXT, '0') || '%';
    status   := CASE WHEN v_hot_pct > 70 THEN 'GOOD'
                     WHEN v_hot_pct > 30 THEN 'FAIR'
                     ELSE 'POOR' END;
    recommendation := CASE WHEN v_hot_pct < 50 
                           THEN 'Consider using fillfactor < 100 for UPDATE-heavy tables'
                           ELSE 'HOT update rate is acceptable' END;
    RETURN NEXT;
    
    -- Transaction ID age
    SELECT MAX(age(datfrozenxid)) INTO v_max_age
    FROM pg_database WHERE datname NOT IN ('template0', 'template1');
    
    category := 'MVCC Safety';
    metric   := 'Max Transaction ID Age';
    value    := v_max_age::TEXT;
    status   := CASE WHEN v_max_age > 1500000000 THEN 'CRITICAL'
                     WHEN v_max_age > 750000000  THEN 'WARNING'
                     WHEN v_max_age > 250000000  THEN 'CAUTION'
                     ELSE 'OK' END;
    recommendation := CASE 
        WHEN v_max_age > 1500000000 THEN 'Run VACUUM FREEZE IMMEDIATELY'
        WHEN v_max_age > 750000000  THEN 'Schedule VACUUM FREEZE soon'
        ELSE 'Transaction ID age is safe' END;
    RETURN NEXT;
    
    -- Bloated tables
    SELECT COUNT(*) INTO v_bloated
    FROM pg_stat_user_tables
    WHERE n_dead_tup::NUMERIC / NULLIF(n_live_tup + n_dead_tup, 0) > 0.2;
    
    category := 'MVCC Overhead';
    metric   := 'Tables with >20% Dead Tuples';
    value    := v_bloated::TEXT;
    status   := CASE WHEN v_bloated = 0 THEN 'GOOD'
                     WHEN v_bloated < 5 THEN 'FAIR'
                     ELSE 'POOR' END;
    recommendation := CASE WHEN v_bloated > 0 
                           THEN 'Run VACUUM ANALYZE on bloated tables'
                           ELSE 'Table bloat is normal' END;
    RETURN NEXT;
    
    -- Oldest active transaction
    SELECT MAX(now() - xact_start) INTO v_oldest_txn
    FROM pg_stat_activity WHERE xact_start IS NOT NULL AND state != 'idle';
    
    category := 'MVCC Safety';
    metric   := 'Oldest Active Transaction';
    value    := COALESCE(v_oldest_txn::TEXT, 'None');
    status   := CASE WHEN v_oldest_txn > INTERVAL '30 minutes' THEN 'WARNING'
                     WHEN v_oldest_txn > INTERVAL '5 minutes'  THEN 'CAUTION'
                     ELSE 'OK' END;
    recommendation := CASE 
        WHEN v_oldest_txn > INTERVAL '30 minutes' 
            THEN 'Long transactions prevent VACUUM from reclaiming dead tuples'
        ELSE 'Transaction duration is normal' END;
    RETURN NEXT;
END;
$$ LANGUAGE plpgsql;

SELECT * FROM mvcc_health_check();
```

---

## สรุป

MVCC เป็นหัวใจสำคัญของ PostgreSQL concurrency model:

1. **Core Concept**: สร้าง multiple versions แทน in-place updates → Readers ไม่ block Writers

2. **System Columns**:
   - `xmin`: Transaction ที่สร้าง row
   - `xmax`: Transaction ที่ delete/update row
   - `ctid`: Physical location ของ row

3. **Snapshot Isolation**: แต่ละ transaction เห็น consistent snapshot ตาม isolation level

4. **Dead Tuples**: Old versions หลัง UPDATE/DELETE → ต้องการ VACUUM เป็นระยะ

5. **HOT Updates**: Optimization ที่ช่วยลด index churn สำหรับ non-indexed column updates

6. **Transaction ID Wraparound**: ต้องระวัง! VACUUM FREEZE ป้องกัน catastrophic failure

7. **VACUUM**: กระบวนการสำคัญที่เก็บ dead tuples และอัปเดต visibility information

8. **MySQL InnoDB**: ใช้ MVCC เช่นกัน แต่ผ่าน undo log แทน multiple heap versions

ใน Part 78 เราจะเรียนรู้ Optimistic vs Pessimistic Locking ซึ่งเป็น design patterns ที่ใช้ MVCC principles
