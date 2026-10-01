# Part 75: Locking Mechanisms

## บทนำ

**Locking** คือกลไกที่ฐานข้อมูลใช้ควบคุมการเข้าถึงข้อมูลพร้อมกัน เพื่อรักษา data integrity และป้องกัน conflicts ระหว่าง concurrent transactions

---

## 1. ประเภทของ Locks ตาม Granularity

### 1.1 Table-Level Locks

```sql
-- Lock ทั้งตาราง (coarse-grained)
-- ใช้สำหรับ DDL operations หรือเมื่อต้องการ exclusive access ทั้งตาราง

-- PostgreSQL: LOCK TABLE
BEGIN;
LOCK TABLE products IN SHARE MODE;      -- อนุญาต reads แต่ไม่ให้ writes
-- หรือ
LOCK TABLE products IN EXCLUSIVE MODE;  -- ไม่อนุญาตทั้ง reads และ writes

-- ประเภทของ Table Locks ใน PostgreSQL:
-- ACCESS SHARE         - สำหรับ SELECT (ต่ำสุด)
-- ROW SHARE            - สำหรับ SELECT FOR UPDATE/SHARE
-- ROW EXCLUSIVE        - สำหรับ UPDATE, DELETE, INSERT
-- SHARE UPDATE EXCLUSIVE - สำหรับ VACUUM, ANALYZE, INDEX CONCURRENTLY
-- SHARE                - สำหรับ CREATE INDEX (non-concurrent)
-- SHARE ROW EXCLUSIVE  - ใช้ได้ยากมาก
-- EXCLUSIVE            - ป้องกันทุกอย่างยกเว้น ACCESS SHARE
-- ACCESS EXCLUSIVE     - สำหรับ DROP, ALTER TABLE, TRUNCATE (สูงสุด)

-- Lock table สำหรับ bulk operations
BEGIN;
LOCK TABLE order_items IN EXCLUSIVE MODE;
-- ทำ bulk update โดยไม่ถูกรบกวน
UPDATE order_items SET price = price * 1.1;
COMMIT;

-- ดู table locks ปัจจุบัน
SELECT 
    c.relname AS table_name,
    l.mode AS lock_mode,
    l.granted,
    a.pid,
    a.usename,
    left(a.query, 80) AS query
FROM pg_locks l
JOIN pg_class c ON l.relation = c.oid
JOIN pg_stat_activity a ON l.pid = a.pid
WHERE c.relkind = 'r'  -- regular tables only
  AND c.relnamespace != 'pg_catalog'::regnamespace
ORDER BY c.relname;
```

### 1.2 Row-Level Locks

```sql
-- Lock ระดับ row (fine-grained)
-- ใช้สำหรับ UPDATE, DELETE, SELECT FOR UPDATE/SHARE

-- Row Locks ใน PostgreSQL:
-- FOR KEY SHARE  - อนุญาต updates ที่ไม่เปลี่ยน key columns
-- FOR SHARE      - อนุญาต SELECT FOR KEY SHARE เท่านั้น
-- FOR NO KEY UPDATE - เหมือน FOR UPDATE แต่อนุญาต FOR KEY SHARE
-- FOR UPDATE     - exclusive row lock (รุนแรงที่สุด)

-- ตัวอย่าง: Lock row สำหรับ update
BEGIN;
SELECT * FROM accounts 
WHERE account_id = 1 
FOR UPDATE;  -- Lock row นี้ ป้องกัน concurrent updates
-- ทำ update
UPDATE accounts SET balance = balance - 500 WHERE account_id = 1;
COMMIT;

-- ตัวอย่าง: Lock หลาย rows
BEGIN;
SELECT * FROM products 
WHERE category = 'electronics' AND stock > 0
FOR UPDATE;
-- Lock ทุก product ใน electronics category ที่มีสต็อก
UPDATE products SET stock = stock - 1 WHERE product_id = ANY(ARRAY[1,2,3]);
COMMIT;

-- Row lock ใน UPDATE/DELETE (implicit)
BEGIN;
UPDATE accounts SET balance = balance + 100 WHERE account_id = 1;
-- Row 1 ถูก lock โดยอัตโนมัติระหว่าง UPDATE
COMMIT;
```

### 1.3 Page-Level Locks

```sql
-- Page locks ใช้ภายใน PostgreSQL engine เอง
-- ไม่ค่อย expose ให้ user โดยตรง
-- แต่ปรากฏใน pg_locks ประเภท 'page'

-- ดู page locks
SELECT 
    locktype,
    relation::regclass AS table_name,
    page,
    tuple,
    pid,
    mode,
    granted
FROM pg_locks
WHERE locktype = 'page'
ORDER BY relation, page;

-- Page locks ถูกใช้สำหรับ:
-- 1. Sequence operations
-- 2. HOT (Heap Only Tuple) updates
-- 3. GIN index operations
```

---

## 2. Shared (READ) Locks vs Exclusive (WRITE) Locks

### Shared Locks

```sql
-- Shared Lock: หลาย transactions สามารถ hold ได้พร้อมกัน
-- Exclusive Lock: มีได้แค่หนึ่ง transaction เท่านั้น

-- Shared Lock ตัวอย่าง
BEGIN;
SELECT * FROM reports WHERE report_type = 'monthly' FOR SHARE;
-- หลาย transactions สามารถ SELECT FOR SHARE พร้อมกันได้
-- แต่ไม่มีใคร UPDATE/DELETE ได้จนกว่า COMMIT

-- SELECT FOR SHARE ใช้เมื่อ:
-- - ต้องการ prevent updates แต่อนุญาต other reads
-- - ใช้ใน referential integrity checks

-- ตัวอย่าง: ตรวจสอบ FK integrity
BEGIN;
SELECT id FROM parent_table WHERE id = 1 FOR SHARE;
-- ป้องกัน parent row จากการถูก DELETE ขณะที่เราตรวจสอบ
INSERT INTO child_table (parent_id, data) VALUES (1, 'child data');
COMMIT;
```

### Exclusive Locks

```sql
-- Exclusive Lock: ป้องกัน reads และ writes ทั้งหมด
BEGIN;
SELECT * FROM critical_table WHERE id = 1 FOR UPDATE;
-- ไม่มีใคร SELECT FOR UPDATE, UPDATE, หรือ DELETE row นี้ได้
-- (SELECT ปกติยังทำได้เพราะ MVCC)

-- ตัวอย่าง: Bank transfer ที่ปลอดภัย
BEGIN;
-- Lock ทั้งสอง accounts เพื่อป้องกัน concurrent modifications
SELECT account_id, balance FROM accounts 
WHERE account_id IN (1, 2)
ORDER BY account_id  -- สำคัญ! ต้อง lock ตามลำดับเดียวกันเสมอ
FOR UPDATE;

UPDATE accounts SET balance = balance - 500 WHERE account_id = 1;
UPDATE accounts SET balance = balance + 500 WHERE account_id = 2;
COMMIT;
```

---

## 3. Advisory Locks (PostgreSQL)

Advisory Locks เป็น lock ที่ Application กำหนดเองและ Database ไม่รู้ว่ามันหมายถึงอะไร

```sql
-- Advisory Locks มี 2 ประเภท:
-- Session-Level: อยู่จนกว่าจะ release หรือ session จบ
-- Transaction-Level: อยู่จนกว่า transaction จะจบ

-- Session-Level Advisory Locks
SELECT pg_advisory_lock(12345);       -- Exclusive lock
SELECT pg_advisory_lock_shared(12345); -- Shared lock
SELECT pg_advisory_unlock(12345);     -- Release exclusive lock
SELECT pg_advisory_unlock_shared(12345); -- Release shared lock

-- Transaction-Level Advisory Locks (auto-release เมื่อ transaction จบ)
BEGIN;
SELECT pg_advisory_xact_lock(12345);       -- Exclusive
SELECT pg_advisory_xact_lock_shared(12345); -- Shared
COMMIT;  -- Lock released automatically

-- Non-blocking version (returns FALSE ถ้าไม่สามารถ acquire ได้)
SELECT pg_try_advisory_lock(12345);        -- Returns true/false
SELECT pg_try_advisory_xact_lock(12345);   -- Transaction-level

-- ใช้ key สองตัว (hi, lo) สำหรับ namespace
SELECT pg_advisory_lock(1, 1001);  -- Lock สำหรับ object type 1, id 1001

-- Use cases:
-- 1. Prevent concurrent processing ของ job เดิม
-- 2. Distributed locking สำหรับ application-level coordination
-- 3. Prevent concurrent migrations

-- ตัวอย่าง: Process job ครั้งละหนึ่งเท่านั้น
CREATE OR REPLACE FUNCTION process_daily_report(p_report_date DATE)
RETURNS TEXT AS $$
DECLARE
    v_lock_key BIGINT;
BEGIN
    -- สร้าง unique key จาก report date
    v_lock_key := EXTRACT(EPOCH FROM p_report_date)::BIGINT;
    
    -- พยายาม acquire lock
    IF NOT pg_try_advisory_xact_lock(v_lock_key) THEN
        RETURN 'Report already being processed by another session';
    END IF;
    
    -- ตอนนี้เป็น exclusive access สำหรับ report นี้
    -- ไม่มี session อื่น process report วันเดียวกันได้
    
    -- ทำงาน report processing
    INSERT INTO reports (report_date, status, started_at)
    VALUES (p_report_date, 'processing', NOW())
    ON CONFLICT (report_date) DO UPDATE SET status = 'processing', started_at = NOW();
    
    -- ... process report ...
    
    UPDATE reports SET status = 'completed', completed_at = NOW()
    WHERE report_date = p_report_date;
    
    RETURN 'Report processed successfully';
    -- Lock released เมื่อ transaction commit/rollback
END;
$$ LANGUAGE plpgsql;

-- ดู advisory locks ปัจจุบัน
SELECT 
    locktype,
    objid AS lock_key,
    pid,
    mode,
    granted
FROM pg_locks
WHERE locktype = 'advisory'
ORDER BY lock_key;

-- ดูว่า advisory lock ถูก hold อยู่หรือไม่
SELECT pg_try_advisory_lock(12345) AS can_acquire;
-- ถ้า TRUE = ไม่มีใค lock อยู่
-- ถ้า FALSE = มีคน lock อยู่แล้ว (และเราได้ lock แล้วถ้า TRUE!)
-- ถ้า TRUE → ต้อง unlock ด้วย! (เราได้ lock ไปแล้ว)

-- Safe check ว่า lock ถูก held อยู่หรือไม่
SELECT EXISTS (
    SELECT 1 FROM pg_locks
    WHERE locktype = 'advisory'
      AND objid = 12345
      AND granted = TRUE
) AS is_locked;
```

---

## 4. Lock Modes ใน PostgreSQL โดยละเอียด

```sql
-- Lock Conflict Matrix
-- (Y = conflicts, N = ไม่ conflict)
/*
                   ACCESS  ROW    ROW    SHARE  SHARE  SHARE  EXCL   ACCESS
                   SHARE   SHARE  EXCL   UPDATE ROW    SHARE  USIVE  EXCL
                                  USIVE  EXCL   EXCL
ACCESS SHARE         N       N      N      N      N      N      N      Y
ROW SHARE            N       N      N      N      N      N      Y      Y
ROW EXCLUSIVE        N       N      N      N      N      Y      Y      Y
SHARE UPDATE EXC     N       N      N      N      Y      Y      Y      Y
SHARE                N       N      N      Y      Y      N      Y      Y
SHARE ROW EXCL       N       N      Y      Y      Y      Y      Y      Y
EXCLUSIVE            N       Y      Y      Y      Y      Y      Y      Y
ACCESS EXCLUSIVE     Y       Y      Y      Y      Y      Y      Y      Y
*/

-- ดูว่า command ใดใช้ lock mode ใด:
-- SELECT                → ACCESS SHARE
-- SELECT FOR SHARE      → ROW SHARE
-- INSERT, UPDATE, DELETE → ROW EXCLUSIVE
-- CREATE INDEX          → SHARE
-- VACUUM, ANALYZE       → SHARE UPDATE EXCLUSIVE
-- TRUNCATE, DROP, ALTER → ACCESS EXCLUSIVE

-- ทดสอบ lock conflicts
BEGIN;
LOCK TABLE orders IN SHARE MODE;
-- ตอนนี้ INSERT, UPDATE, DELETE ในตาราง orders จะ block
-- แต่ SELECT ยังทำได้

-- Check สถานะ lock
SELECT mode FROM pg_locks 
WHERE relation = 'orders'::regclass
  AND pid = pg_backend_pid();
COMMIT;

-- ตัวอย่าง: ป้องกัน concurrent schema changes
BEGIN;
LOCK TABLE users IN ACCESS SHARE MODE;
-- ป้องกัน ALTER TABLE ในขณะที่เราทำ report
SELECT * FROM users;
COMMIT;
```

---

## 5. SELECT FOR UPDATE

```sql
-- SELECT FOR UPDATE: Lock rows สำหรับ update
-- ป้องกัน concurrent transactions จากการ modify rows เดิม

-- ตัวอย่างที่ 1: Simple FOR UPDATE
BEGIN;
SELECT id, balance FROM accounts WHERE id = 1 FOR UPDATE;
-- Row 1 ถูก lock
-- Transaction อื่นที่พยายาม SELECT FOR UPDATE หรือ UPDATE จะ block

UPDATE accounts SET balance = balance - 100 WHERE id = 1;
COMMIT;

-- ตัวอย่างที่ 2: Lock หลาย rows ที่เกี่ยวข้อง
BEGIN;
WITH locked_orders AS (
    SELECT order_id FROM orders 
    WHERE status = 'pending' AND assigned_to IS NULL
    ORDER BY created_at
    LIMIT 10
    FOR UPDATE
)
UPDATE orders SET assigned_to = pg_backend_pid(), status = 'processing'
FROM locked_orders
WHERE orders.order_id = locked_orders.order_id;
COMMIT;

-- ตัวอย่างที่ 3: Conditional locking
BEGIN;
DECLARE
    v_stock INT;
BEGIN
    SELECT stock INTO v_stock
    FROM inventory
    WHERE product_id = 101
    FOR UPDATE;  -- Lock ทันที
    
    IF v_stock < 5 THEN
        RAISE EXCEPTION 'Insufficient stock: only % available', v_stock;
    END IF;
    
    UPDATE inventory SET stock = stock - 5 WHERE product_id = 101;
    INSERT INTO orders (product_id, quantity) VALUES (101, 5);
END;
COMMIT;

-- ตัวอย่างที่ 4: Lock ข้าม tables
BEGIN;
SELECT a.*, b.credit_limit 
FROM accounts a
JOIN credit_limits b ON a.customer_id = b.customer_id
WHERE a.account_id = 1
FOR UPDATE OF a;  -- Lock เฉพาะ table a

UPDATE accounts SET balance = balance - 100 WHERE account_id = 1;
COMMIT;
```

---

## 6. SELECT FOR SHARE

```sql
-- SELECT FOR SHARE: Lock rows สำหรับ read
-- ป้องกัน updates แต่อนุญาต other reads

-- ตัวอย่างที่ 1: ป้องกัน delete ของ referenced data
BEGIN;
-- Lock parent row เพื่อป้องกันการ delete ขณะ insert child
SELECT id FROM products WHERE id = 101 FOR SHARE;
-- ตอนนี้ product 101 จะไม่ถูก delete หรือ update
-- (transactions อื่นสามารถ SELECT ได้ แต่ไม่สามารถ FOR UPDATE ได้)

INSERT INTO order_items (product_id, quantity, price)
SELECT 101, 2, price FROM products WHERE id = 101;
COMMIT;

-- ตัวอย่างที่ 2: ใช้กับ multiple tables
BEGIN;
SELECT c.*, o.* 
FROM customers c
JOIN orders o ON c.customer_id = o.customer_id
WHERE c.customer_id = 1
FOR SHARE;
-- Lock ทั้ง customer และ order rows
-- ป้องกัน concurrent updates ในขณะที่เราดูข้อมูล

COMMIT;
```

---

## 7. NOWAIT และ SKIP LOCKED

```sql
-- NOWAIT: ถ้า lock ไม่พร้อม → error ทันที (ไม่รอ)
BEGIN;
SELECT * FROM accounts WHERE account_id = 1 FOR UPDATE NOWAIT;
-- ถ้า row นี้ถูก lock อยู่:
-- ERROR: could not obtain lock on row in relation "accounts"

-- ใช้ NOWAIT เมื่อ:
-- - ไม่ต้องการให้ application hang รอ lock
-- - ต้องการ fail fast แทน timeout

-- ตัวอย่าง: Try to process account, fail fast if locked
CREATE OR REPLACE FUNCTION try_process_account(p_account_id INT)
RETURNS TEXT AS $$
BEGIN
    BEGIN
        PERFORM * FROM accounts WHERE account_id = p_account_id FOR UPDATE NOWAIT;
        -- ทำงาน
        UPDATE accounts SET processed = TRUE WHERE account_id = p_account_id;
        RETURN 'Processed successfully';
    EXCEPTION
        WHEN lock_not_available THEN
            RETURN 'Account is being processed by another session';
    END;
END;
$$ LANGUAGE plpgsql;

-- SKIP LOCKED: ข้าม rows ที่ถูก lock อยู่
BEGIN;
SELECT * FROM job_queue
WHERE status = 'pending'
ORDER BY priority DESC, created_at
LIMIT 5
FOR UPDATE SKIP LOCKED;
-- ได้ 5 jobs ที่ไม่ถูก lock โดย transaction อื่น

UPDATE job_queue SET status = 'processing', worker_id = pg_backend_pid()
WHERE job_id = ANY(ARRAY[...]); -- IDs จาก SELECT ด้านบน
COMMIT;

-- SKIP LOCKED ดีมากสำหรับ:
-- 1. Queue processing ที่หลาย workers ทำงานพร้อมกัน
-- 2. Batch processing ที่ไม่ต้องการ block กัน
-- 3. Work stealing patterns

-- ตัวอย่าง: Worker Pool Pattern
CREATE OR REPLACE FUNCTION claim_jobs(
    p_worker_id INT,
    p_batch_size INT DEFAULT 10
) RETURNS TABLE(job_id INT, job_type TEXT, payload JSONB) AS $$
BEGIN
    RETURN QUERY
    WITH claimed AS (
        SELECT j.job_id
        FROM jobs j
        WHERE j.status = 'pending'
          AND (j.scheduled_at IS NULL OR j.scheduled_at <= NOW())
        ORDER BY j.priority DESC, j.created_at
        LIMIT p_batch_size
        FOR UPDATE SKIP LOCKED
    )
    UPDATE jobs SET 
        status = 'processing',
        worker_id = p_worker_id,
        started_at = NOW()
    FROM claimed
    WHERE jobs.job_id = claimed.job_id
    RETURNING jobs.job_id, jobs.job_type, jobs.payload;
END;
$$ LANGUAGE plpgsql;

-- หลาย workers สามารถเรียก claim_jobs พร้อมกันได้ โดยไม่ทับงานกัน
```

---

## 8. Lock Monitoring Queries

```sql
-- Query 1: ดู locks ทั้งหมด
SELECT 
    pid,
    locktype,
    CASE locktype
        WHEN 'relation'    THEN relation::regclass::TEXT
        WHEN 'transactionid' THEN transactionid::TEXT
        WHEN 'tuple'       THEN relation::regclass::TEXT || ':' || tuple::TEXT
        ELSE 'other'
    END AS object,
    mode,
    granted,
    waitstart
FROM pg_locks
ORDER BY granted, pid;

-- Query 2: ดู blocking relationships
WITH blocking AS (
    SELECT 
        blocked_locks.pid AS blocked_pid,
        blocked_activity.usename AS blocked_user,
        blocking_locks.pid AS blocking_pid,
        blocking_activity.usename AS blocking_user,
        blocked_activity.application_name,
        blocked_locks.relation::regclass AS locked_table,
        blocked_locks.locktype,
        now() - blocked_activity.xact_start AS blocked_duration,
        now() - blocking_activity.xact_start AS blocking_duration,
        blocked_activity.query AS blocked_query,
        blocking_activity.query AS blocking_query
    FROM pg_catalog.pg_locks blocked_locks
    JOIN pg_catalog.pg_stat_activity blocked_activity 
        ON blocked_activity.pid = blocked_locks.pid
    JOIN pg_catalog.pg_locks blocking_locks 
        ON blocking_locks.locktype = blocked_locks.locktype
        AND blocking_locks.relation IS NOT DISTINCT FROM blocked_locks.relation
        AND blocking_locks.page IS NOT DISTINCT FROM blocked_locks.page
        AND blocking_locks.tuple IS NOT DISTINCT FROM blocked_locks.tuple
        AND blocking_locks.transactionid IS NOT DISTINCT FROM blocked_locks.transactionid
        AND blocking_locks.classid IS NOT DISTINCT FROM blocked_locks.classid
        AND blocking_locks.objid IS NOT DISTINCT FROM blocked_locks.objid
        AND blocking_locks.objsubid IS NOT DISTINCT FROM blocked_locks.objsubid
        AND blocking_locks.pid != blocked_locks.pid
    JOIN pg_catalog.pg_stat_activity blocking_activity 
        ON blocking_activity.pid = blocking_locks.pid
    WHERE NOT blocked_locks.granted
)
SELECT 
    blocked_pid,
    blocked_user,
    blocking_pid,
    blocking_user,
    locked_table,
    locktype,
    blocked_duration AS how_long_waiting,
    left(blocked_query, 100) AS waiting_query,
    left(blocking_query, 100) AS blocker_query
FROM blocking
ORDER BY blocked_duration DESC;

-- Query 3: ดู lock wait chains (multi-level blocking)
WITH RECURSIVE lock_chain AS (
    -- Base case: sessions ที่ถูก block
    SELECT 
        w.pid AS blocked_pid,
        w.blocking_pids[1] AS blocking_pid,
        1 AS depth,
        ARRAY[w.pid] AS chain
    FROM pg_stat_activity a
    CROSS JOIN LATERAL (
        SELECT ARRAY_AGG(b.pid) AS blocking_pids
        FROM pg_locks bl
        JOIN pg_locks b ON bl.relation = b.relation
                        AND bl.locktype = b.locktype
                        AND b.granted = TRUE
                        AND b.pid != bl.pid
        WHERE bl.pid = a.pid AND NOT bl.granted
    ) w
    WHERE w.blocking_pids IS NOT NULL
    
    UNION ALL
    
    -- Recursive case: ติดตาม blocking chain
    SELECT 
        lc.blocking_pid AS blocked_pid,
        w.blocking_pids[1] AS blocking_pid,
        lc.depth + 1,
        lc.chain || lc.blocking_pid
    FROM lock_chain lc
    CROSS JOIN LATERAL (
        SELECT ARRAY_AGG(b.pid) AS blocking_pids
        FROM pg_locks bl
        JOIN pg_locks b ON bl.relation = b.relation
                        AND bl.locktype = b.locktype
                        AND b.granted = TRUE
                        AND b.pid != bl.pid
        WHERE bl.pid = lc.blocking_pid AND NOT bl.granted
    ) w
    WHERE w.blocking_pids IS NOT NULL
      AND NOT (lc.blocking_pid = ANY(lc.chain))  -- ป้องกัน cycles
      AND lc.depth < 10
)
SELECT 
    blocked_pid,
    blocking_pid,
    depth,
    chain::TEXT AS lock_chain
FROM lock_chain
ORDER BY depth DESC, blocked_pid;

-- Query 4: Summary of locks by table
SELECT 
    c.relname AS table_name,
    l.mode,
    COUNT(*) AS lock_count,
    COUNT(*) FILTER (WHERE NOT l.granted) AS waiting_count
FROM pg_locks l
JOIN pg_class c ON l.relation = c.oid
WHERE c.relkind = 'r'
  AND c.relnamespace != 'pg_catalog'::regnamespace
GROUP BY c.relname, l.mode
ORDER BY c.relname, l.mode;

-- Query 5: Lock statistics over time
SELECT 
    tablename,
    n_dead_tup AS dead_tuples_from_rollback,
    n_live_tup AS live_tuples,
    n_mod_since_analyze AS modifications_since_analyze,
    last_autovacuum,
    last_autoanalyze
FROM pg_stat_user_tables
WHERE n_dead_tup > 0
ORDER BY n_dead_tup DESC
LIMIT 20;

-- Query 6: Sessions waiting for locks
SELECT 
    a.pid,
    a.usename,
    a.application_name,
    a.state,
    a.wait_event_type,
    a.wait_event,
    now() - a.xact_start AS waiting_for,
    left(a.query, 150) AS query
FROM pg_stat_activity a
WHERE a.wait_event_type = 'Lock'
ORDER BY waiting_for DESC;

-- Query 7: Lock contentions per table
SELECT 
    schemaname,
    tablename,
    n_tup_upd AS updates,
    n_tup_del AS deletes,
    n_tup_ins AS inserts,
    n_dead_tup AS dead_tuples,
    ROUND(n_dead_tup::NUMERIC / NULLIF(n_live_tup + n_dead_tup, 0) * 100, 2) AS bloat_pct
FROM pg_stat_user_tables
ORDER BY n_dead_tup DESC
LIMIT 20;
```

---

## 9. Lock Timeouts

```sql
-- ป้องกัน transactions จากการรอ lock นานเกินไป

-- lock_timeout: ระยะเวลาสูงสุดในการรอ lock
SET lock_timeout = '5s';     -- 5 วินาที
SET lock_timeout = '1min';   -- 1 นาที
SET lock_timeout = 0;        -- ไม่มี timeout (ค่าเริ่มต้น)

-- statement_timeout: ระยะเวลาสูงสุดสำหรับ statement ทั้งหมด
SET statement_timeout = '30s';

-- deadlock_timeout: ระยะเวลาก่อน PostgreSQL ตรวจหา deadlock
SHOW deadlock_timeout;
SET deadlock_timeout = '1s';  -- ตรวจ deadlock ทุก 1 วินาที

-- ตัวอย่าง: ใช้ lock_timeout ป้องกัน queue blocking
BEGIN;
SET LOCAL lock_timeout = '2000';  -- 2 วินาที (milliseconds)
SELECT * FROM critical_records WHERE status = 'pending' FOR UPDATE;
-- ถ้า records ถูก lock อยู่ → ERROR หลัง 2 วินาที
COMMIT;

-- ตัวอย่าง: Retry pattern ด้วย lock_timeout
CREATE OR REPLACE FUNCTION process_with_timeout(
    p_record_id INT,
    p_max_wait_ms INT DEFAULT 5000
) RETURNS TEXT AS $$
BEGIN
    SET LOCAL lock_timeout = p_max_wait_ms;
    
    BEGIN
        SELECT * FROM work_queue WHERE id = p_record_id FOR UPDATE;
        UPDATE work_queue SET status = 'processing' WHERE id = p_record_id;
        -- ทำงาน
        UPDATE work_queue SET status = 'completed' WHERE id = p_record_id;
        RETURN 'Completed';
        
    EXCEPTION
        WHEN lock_not_available THEN
            RETURN format('Lock timeout after %sms, try again later', p_max_wait_ms);
    END;
END;
$$ LANGUAGE plpgsql;

-- ตั้งค่าใน postgresql.conf (global):
-- lock_timeout = '10s'
-- statement_timeout = '60s'
-- deadlock_timeout = '1s'

-- ตั้งค่าสำหรับ role เฉพาะ:
ALTER ROLE reporting_user SET statement_timeout = '120s';
ALTER ROLE api_user SET lock_timeout = '5s';
```

---

## 10. MySQL Lock Types

```sql
-- MySQL InnoDB Locks

-- 1. Record Locks (Row-level)
-- Locks ที่ index record เดี่ยว
BEGIN;
SELECT * FROM orders WHERE order_id = 1 FOR UPDATE;
-- Lock record ที่ order_id = 1
COMMIT;

-- 2. Gap Locks
-- Lock ช่องว่างระหว่าง records (ป้องกัน phantom reads)
BEGIN;
SELECT * FROM orders WHERE order_id BETWEEN 10 AND 20 FOR UPDATE;
-- Lock records 10-20 และ gap ระหว่างนั้น
-- ป้องกัน INSERT ใน range นี้จาก transaction อื่น
COMMIT;

-- 3. Next-Key Locks (Record + Gap)
-- เป็น default ใน MySQL REPEATABLE READ
BEGIN;
SELECT * FROM orders WHERE order_id > 100 FOR UPDATE;
-- Lock records > 100 และ gap ก่อนแต่ละ record
COMMIT;

-- 4. Insert Intention Locks
-- ใช้ก่อน INSERT เพื่อบอกว่าจะ insert ที่ gap นั้น

-- 5. Auto-Increment Locks
-- Lock สำหรับ auto-increment columns

-- ดู locks ใน MySQL
SELECT 
    engine_lock_id,
    engine_transaction_id,
    thread_id,
    event_id,
    object_name AS table_name,
    lock_type,
    lock_mode,
    lock_status,
    lock_data
FROM performance_schema.data_locks;

-- ดู lock waits
SELECT 
    requesting_engine_lock_id,
    requesting_engine_transaction_id,
    blocking_engine_lock_id,
    blocking_engine_transaction_id
FROM performance_schema.data_lock_waits;

-- ดู InnoDB transactions
SELECT 
    trx_id,
    trx_state,
    trx_started,
    trx_wait_started,
    trx_rows_locked,
    trx_rows_modified,
    trx_lock_structs
FROM information_schema.innodb_trx;
```

---

## 11. Lock Monitoring Scripts สำหรับ Production

```sql
-- Script 1: Lock Dashboard
CREATE OR REPLACE VIEW lock_dashboard AS
SELECT 
    'Active Locks' AS category,
    COUNT(*) AS count,
    NULL AS avg_wait_seconds
FROM pg_locks WHERE granted = TRUE

UNION ALL

SELECT 
    'Waiting Locks',
    COUNT(*),
    ROUND(AVG(EXTRACT(EPOCH FROM (NOW() - waitstart))), 2)
FROM pg_locks WHERE granted = FALSE

UNION ALL

SELECT 
    'Deadlock Prone Sessions',
    COUNT(DISTINCT pid),
    NULL
FROM pg_locks
WHERE NOT granted
  AND pid IN (
    SELECT pid FROM pg_locks WHERE granted AND pid != pg_backend_pid()
  )

UNION ALL

SELECT
    'Long Lock Waits (>30s)',
    COUNT(*),
    ROUND(MAX(EXTRACT(EPOCH FROM (NOW() - waitstart))), 2)
FROM pg_locks
WHERE NOT granted
  AND waitstart < NOW() - INTERVAL '30 seconds';

SELECT * FROM lock_dashboard;

-- Script 2: Lock Killer (ฉุกเฉิน)
CREATE OR REPLACE FUNCTION kill_blocking_queries(
    p_min_blocking_duration INTERVAL DEFAULT '5 minutes',
    p_dry_run BOOLEAN DEFAULT TRUE
) RETURNS TABLE(
    pid INT,
    usename TEXT,
    blocking_duration INTERVAL,
    blocked_count INT,
    action TEXT
) AS $$
BEGIN
    RETURN QUERY
    WITH blockers AS (
        SELECT 
            blocking_activity.pid,
            blocking_activity.usename,
            now() - blocking_activity.xact_start AS blocking_duration,
            COUNT(DISTINCT blocked_activity.pid) AS blocked_count
        FROM pg_locks blocked_locks
        JOIN pg_stat_activity blocked_activity ON blocked_activity.pid = blocked_locks.pid
        JOIN pg_locks blocking_locks ON 
            blocking_locks.relation = blocked_locks.relation
            AND blocking_locks.granted = TRUE
            AND blocking_locks.pid != blocked_locks.pid
        JOIN pg_stat_activity blocking_activity ON blocking_activity.pid = blocking_locks.pid
        WHERE NOT blocked_locks.granted
        GROUP BY blocking_activity.pid, blocking_activity.usename, blocking_activity.xact_start
    )
    SELECT 
        b.pid,
        b.usename::TEXT,
        b.blocking_duration,
        b.blocked_count::INT,
        CASE WHEN p_dry_run THEN 'Would terminate'
             WHEN pg_terminate_backend(b.pid) THEN 'Terminated'
             ELSE 'Failed to terminate'
        END AS action
    FROM blockers b
    WHERE b.blocking_duration > p_min_blocking_duration
    ORDER BY b.blocking_duration DESC;
END;
$$ LANGUAGE plpgsql;

-- ดูก่อน (dry run):
SELECT * FROM kill_blocking_queries('5 minutes', TRUE);

-- Kill จริง:
-- SELECT * FROM kill_blocking_queries('5 minutes', FALSE);

-- Script 3: Lock Alert System
CREATE OR REPLACE FUNCTION check_lock_health()
RETURNS TABLE(
    alert_level TEXT,
    message TEXT,
    count INT
) AS $$
BEGIN
    -- Check 1: Many waiting locks
    SELECT COUNT(*) INTO STRICT count FROM pg_locks WHERE NOT granted;
    IF count > 20 THEN
        alert_level := 'CRITICAL';
        message := 'High number of waiting locks: ' || count;
        RETURN NEXT;
    ELSIF count > 10 THEN
        alert_level := 'WARNING';
        message := 'Elevated waiting locks: ' || count;
        RETURN NEXT;
    END IF;
    
    -- Check 2: Long-waiting locks
    SELECT COUNT(*) INTO STRICT count 
    FROM pg_locks 
    WHERE NOT granted AND waitstart < NOW() - INTERVAL '1 minute';
    IF count > 0 THEN
        alert_level := 'CRITICAL';
        message := 'Locks waiting over 1 minute: ' || count;
        RETURN NEXT;
    END IF;
    
    -- Check 3: Deadlock count
    DECLARE v_deadlocks BIGINT;
    BEGIN
        SELECT deadlocks INTO v_deadlocks 
        FROM pg_stat_database WHERE datname = current_database();
        
        IF v_deadlocks > 0 THEN
            alert_level := 'WARNING';
            message := 'Deadlocks detected: ' || v_deadlocks;
            count := v_deadlocks::INT;
            RETURN NEXT;
        END IF;
    END;
    
    alert_level := 'OK';
    message := 'No lock issues detected';
    count := 0;
    RETURN NEXT;
END;
$$ LANGUAGE plpgsql;
```

---

## 12. Optimistic vs Pessimistic Locking Comparison

```sql
-- Pessimistic Locking: Lock ก่อนแก้ไข (ในบทนี้)
-- Optimistic Locking: ตรวจสอบหลังจากแก้ไข (Part 78)

-- Pessimistic example:
BEGIN;
SELECT version, data FROM records WHERE id = 1 FOR UPDATE;  -- Lock ทันที
UPDATE records SET data = 'new data' WHERE id = 1;
COMMIT;

-- เมื่อไรใช้ Pessimistic Locking:
-- 1. High contention: หลาย transactions แก้ไขข้อมูลเดิม
-- 2. Critical operations: banking, inventory
-- 3. Short operations: lock ไม่นาน
-- 4. เมื่อ retry cost สูง

-- เมื่อไรใช้ Optimistic Locking:
-- 1. Low contention: conflicts ไม่บ่อย
-- 2. Long-running reads: ไม่ต้องการ lock นาน
-- 3. Scale-out ที่ต้องการ throughput สูง
-- 4. เมื่อ retry cost ต่ำ

-- Hybrid approach:
CREATE OR REPLACE FUNCTION smart_update(
    p_id INT,
    p_new_data TEXT
) RETURNS TEXT AS $$
DECLARE
    v_current_version INT;
    v_updated_rows INT;
BEGIN
    -- ลอง optimistic first
    GET DIAGNOSTICS v_updated_rows = ROW_COUNT;
    
    UPDATE records 
    SET data = p_new_data, version = version + 1
    WHERE id = p_id AND version = (SELECT version FROM records WHERE id = p_id);
    
    GET DIAGNOSTICS v_updated_rows = ROW_COUNT;
    
    IF v_updated_rows = 1 THEN
        RETURN 'Updated with optimistic locking';
    END IF;
    
    -- Optimistic failed → ลอง pessimistic
    SELECT version INTO v_current_version FROM records WHERE id = p_id FOR UPDATE;
    
    UPDATE records SET data = p_new_data, version = version + 1 WHERE id = p_id;
    
    RETURN 'Updated with pessimistic locking (retry)';
END;
$$ LANGUAGE plpgsql;
```

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: Basic Lock Demonstration

```sql
-- คำตอบ
-- สร้าง table สำหรับทดสอบ
CREATE TABLE lock_demo (
    id      SERIAL PRIMARY KEY,
    value   INT NOT NULL,
    status  VARCHAR(20) DEFAULT 'active'
);

INSERT INTO lock_demo (value) SELECT generate_series(1, 10) * 100;

-- ทดสอบ FOR UPDATE
BEGIN;
SELECT id, value FROM lock_demo WHERE id = 1 FOR UPDATE;
-- Row 1 ถูก lock
-- Transaction อื่นจะ block ถ้าพยายาม UPDATE row นี้

SELECT 'Row 1 locked' AS message;
-- ยังไม่ commit → lock ยังอยู่

ROLLBACK;
```

### แบบฝึกหัดที่ 2: SKIP LOCKED Queue

```sql
-- คำตอบ
CREATE TABLE task_queue (
    task_id     SERIAL PRIMARY KEY,
    task_type   VARCHAR(50),
    status      VARCHAR(20) DEFAULT 'pending',
    payload     JSONB,
    created_at  TIMESTAMPTZ DEFAULT NOW(),
    started_at  TIMESTAMPTZ,
    completed_at TIMESTAMPTZ,
    worker_id   INT
);

INSERT INTO task_queue (task_type, payload)
SELECT 
    'send_email',
    jsonb_build_object('to', 'user' || generate_series(1, 20) || '@example.com')
FROM generate_series(1, 20);

-- Worker function ใช้ SKIP LOCKED
CREATE OR REPLACE FUNCTION claim_tasks(
    p_worker_id INT,
    p_batch_size INT DEFAULT 5
) RETURNS TABLE(task_id INT, task_type TEXT, payload JSONB) AS $$
BEGIN
    RETURN QUERY
    WITH claimed AS (
        SELECT t.task_id
        FROM task_queue t
        WHERE t.status = 'pending'
        ORDER BY t.created_at
        LIMIT p_batch_size
        FOR UPDATE SKIP LOCKED
    )
    UPDATE task_queue SET
        status = 'processing',
        worker_id = p_worker_id,
        started_at = NOW()
    FROM claimed
    WHERE task_queue.task_id = claimed.task_id
    RETURNING task_queue.task_id, task_queue.task_type, task_queue.payload;
END;
$$ LANGUAGE plpgsql;

-- ทดสอบกับ "หลาย workers"
BEGIN;
SELECT * FROM claim_tasks(1, 5);  -- Worker 1 claim 5 tasks
COMMIT;

BEGIN;
SELECT * FROM claim_tasks(2, 5);  -- Worker 2 claim 5 tasks ต่างๆ
COMMIT;
```

### แบบฝึกหัดที่ 3: Advisory Locks

```sql
-- คำตอบ
CREATE OR REPLACE FUNCTION run_exclusive_job(
    p_job_name TEXT
) RETURNS TEXT AS $$
DECLARE
    v_lock_key BIGINT;
BEGIN
    -- สร้าง consistent lock key จาก job name
    v_lock_key := hashtext(p_job_name);
    
    -- พยายาม acquire lock
    IF NOT pg_try_advisory_xact_lock(v_lock_key) THEN
        RETURN format('Job "%s" is already running by another session', p_job_name);
    END IF;
    
    RAISE NOTICE 'Starting exclusive job: %', p_job_name;
    
    -- จำลองงาน
    PERFORM pg_sleep(0.1);
    
    INSERT INTO job_executions (job_name, started_at, completed_at, status)
    VALUES (p_job_name, NOW() - INTERVAL '100 milliseconds', NOW(), 'success');
    
    RETURN format('Job "%s" completed successfully', p_job_name);
END;
$$ LANGUAGE plpgsql;

BEGIN;
SELECT run_exclusive_job('daily_report');
COMMIT;
```

### แบบฝึกหัดที่ 4: Lock Timeout Handling

```sql
-- คำตอบ
CREATE OR REPLACE FUNCTION safe_lock_with_timeout(
    p_table_name TEXT,
    p_record_id  INT,
    p_timeout_ms INT DEFAULT 3000
) RETURNS TEXT AS $$
BEGIN
    SET LOCAL lock_timeout = p_timeout_ms;
    
    BEGIN
        EXECUTE format(
            'SELECT 1 FROM %I WHERE id = $1 FOR UPDATE',
            p_table_name
        ) USING p_record_id;
        
        RETURN 'Lock acquired successfully';
        
    EXCEPTION
        WHEN lock_not_available THEN
            RETURN format('Could not acquire lock within %sms', p_timeout_ms);
        WHEN OTHERS THEN
            RAISE;
    END;
END;
$$ LANGUAGE plpgsql;
```

### แบบฝึกหัดที่ 5: Lock Monitoring Dashboard

```sql
-- คำตอบ
CREATE OR REPLACE FUNCTION get_lock_report()
RETURNS TABLE(
    category    TEXT,
    description TEXT,
    count       BIGINT,
    details     TEXT
) AS $$
BEGIN
    -- Total locks
    SELECT 'Total Active Locks', 'All granted locks in system', COUNT(*), NULL
    FROM pg_locks WHERE granted = TRUE
    INTO category, description, count, details;
    RETURN NEXT;
    
    -- Waiting locks
    SELECT 'Waiting Locks', 'Locks waiting to be granted', COUNT(*), 
           'Max wait: ' || COALESCE(MAX(EXTRACT(EPOCH FROM (NOW() - waitstart)))::TEXT || 's', 'N/A')
    FROM pg_locks WHERE granted = FALSE
    INTO category, description, count, details;
    RETURN NEXT;
    
    -- Locks per table
    FOR category, description, count, details IN
        SELECT 
            'Table: ' || c.relname,
            'Locks on table',
            COUNT(*),
            string_agg(DISTINCT l.mode, ', ')
        FROM pg_locks l
        JOIN pg_class c ON l.relation = c.oid
        WHERE c.relkind = 'r'
          AND c.relnamespace NOT IN (
              SELECT oid FROM pg_namespace WHERE nspname IN ('pg_catalog', 'information_schema')
          )
        GROUP BY c.relname
        ORDER BY COUNT(*) DESC
        LIMIT 10
    LOOP
        RETURN NEXT;
    END LOOP;
END;
$$ LANGUAGE plpgsql;
```

### แบบฝึกหัดที่ 6: NOWAIT Pattern

```sql
-- คำตอบ
CREATE OR REPLACE FUNCTION try_update_record(
    p_id    INT,
    p_value TEXT
) RETURNS JSONB AS $$
BEGIN
    BEGIN
        UPDATE my_table SET value = p_value, updated_at = NOW()
        WHERE id = p_id;
        
        IF NOT FOUND THEN
            RETURN jsonb_build_object('success', false, 'error', 'Record not found');
        END IF;
        
        RETURN jsonb_build_object('success', true, 'message', 'Updated successfully');
        
    EXCEPTION
        WHEN lock_not_available THEN
            RETURN jsonb_build_object(
                'success', false, 
                'error', 'Record is locked by another session',
                'retry_after', 1000
            );
    END;
END;
$$ LANGUAGE plpgsql;

-- ใช้ NOWAIT สำหรับ real-time API
BEGIN;
SET LOCAL lock_timeout = '100';  -- 100ms timeout
SELECT try_update_record(1, 'new_value');
COMMIT;
```

### แบบฝึกหัดที่ 7: Table Lock สำหรับ Maintenance

```sql
-- คำตอบ
CREATE OR REPLACE PROCEDURE maintenance_rebuild_index(
    p_table_name TEXT,
    p_index_name TEXT
) LANGUAGE plpgsql AS $$
BEGIN
    -- Lock table เพื่อป้องกัน concurrent access ระหว่าง maintenance
    EXECUTE format('LOCK TABLE %I IN SHARE MODE', p_table_name);
    
    RAISE NOTICE 'Table locked. Starting index rebuild...';
    
    -- Rebuild index
    EXECUTE format('REINDEX INDEX %I', p_index_name);
    
    RAISE NOTICE 'Index rebuilt successfully';
    
    -- Lock จะ released อัตโนมัติเมื่อ transaction จบ
END;
$$;
```

### แบบฝึกหัดที่ 8: Concurrent Queue Processing

```sql
-- คำตอบ
CREATE TABLE email_queue (
    id          SERIAL PRIMARY KEY,
    to_address  VARCHAR(255) NOT NULL,
    subject     VARCHAR(500),
    body        TEXT,
    status      VARCHAR(20) DEFAULT 'queued',
    attempts    INT DEFAULT 0,
    max_attempts INT DEFAULT 3,
    scheduled_at TIMESTAMPTZ DEFAULT NOW(),
    sent_at     TIMESTAMPTZ,
    error_msg   TEXT
);

CREATE OR REPLACE FUNCTION process_email_queue(
    p_worker_id INT DEFAULT 1,
    p_batch_size INT DEFAULT 10
) RETURNS TABLE(
    email_id INT,
    to_address TEXT,
    status TEXT
) AS $$
DECLARE
    r RECORD;
BEGIN
    FOR r IN
        SELECT eq.id, eq.to_address, eq.subject, eq.body
        FROM email_queue eq
        WHERE eq.status = 'queued'
          AND eq.attempts < eq.max_attempts
          AND eq.scheduled_at <= NOW()
        ORDER BY eq.scheduled_at
        LIMIT p_batch_size
        FOR UPDATE SKIP LOCKED
    LOOP
        BEGIN
            -- จำลองการส่ง email
            -- send_email(r.to_address, r.subject, r.body);
            
            UPDATE email_queue SET
                status = 'sent',
                sent_at = NOW(),
                attempts = attempts + 1
            WHERE id = r.id;
            
            email_id   := r.id;
            to_address := r.to_address;
            status     := 'sent';
            RETURN NEXT;
            
        EXCEPTION WHEN OTHERS THEN
            UPDATE email_queue SET
                attempts = attempts + 1,
                status = CASE 
                    WHEN attempts + 1 >= max_attempts THEN 'failed'
                    ELSE 'queued'
                END,
                error_msg = SQLERRM
            WHERE id = r.id;
            
            email_id   := r.id;
            to_address := r.to_address;
            status     := 'error: ' || SQLERRM;
            RETURN NEXT;
        END;
    END LOOP;
END;
$$ LANGUAGE plpgsql;

BEGIN;
SELECT * FROM process_email_queue(1, 5);
COMMIT;
```

### แบบฝึกหัดที่ 9: Lock Chain Detection

```sql
-- คำตอบ
CREATE OR REPLACE FUNCTION detect_lock_chains()
RETURNS TABLE(
    chain_length INT,
    chain_pids   TEXT,
    root_blocker INT,
    leaf_blocked INT,
    total_wait   INTERVAL
) AS $$
BEGIN
    RETURN QUERY
    WITH RECURSIVE chains AS (
        SELECT 
            blocked.pid AS blocked_pid,
            blocker.pid AS blocker_pid,
            1 AS depth,
            ARRAY[blocked.pid, blocker.pid] AS chain,
            now() - blocked.xact_start AS wait_time
        FROM pg_stat_activity blocked
        JOIN pg_locks bl ON bl.pid = blocked.pid AND NOT bl.granted
        JOIN pg_locks bk ON bk.relation = bl.relation AND bk.granted AND bk.pid != bl.pid
        JOIN pg_stat_activity blocker ON blocker.pid = bk.pid
        
        UNION ALL
        
        SELECT 
            c.blocked_pid,
            bk.pid,
            c.depth + 1,
            c.chain || bk.pid,
            c.wait_time + (now() - blocker2.xact_start)
        FROM chains c
        JOIN pg_locks bl ON bl.pid = c.blocker_pid AND NOT bl.granted
        JOIN pg_locks bk ON bk.relation = bl.relation AND bk.granted AND bk.pid != bl.pid
        JOIN pg_stat_activity blocker2 ON blocker2.pid = bk.pid
        WHERE NOT (bk.pid = ANY(c.chain))
          AND c.depth < 5
    )
    SELECT DISTINCT
        array_length(chain, 1) AS chain_length,
        array_to_string(chain, ' → ') AS chain_pids,
        chain[array_length(chain, 1)] AS root_blocker,
        chain[1] AS leaf_blocked,
        wait_time AS total_wait
    FROM chains
    ORDER BY chain_length DESC;
END;
$$ LANGUAGE plpgsql;

SELECT * FROM detect_lock_chains();
```

### แบบฝึกหัดที่ 10: Complete Lock Management System

```sql
-- คำตอบ
CREATE TABLE lock_incidents (
    incident_id  SERIAL PRIMARY KEY,
    detected_at  TIMESTAMPTZ DEFAULT NOW(),
    blocked_pids INT[],
    blocking_pid INT,
    duration     INTERVAL,
    resolved_at  TIMESTAMPTZ,
    resolution   TEXT
);

CREATE OR REPLACE PROCEDURE auto_resolve_lock_issues(
    p_threshold_seconds INT DEFAULT 60,
    p_auto_kill BOOLEAN DEFAULT FALSE
) LANGUAGE plpgsql AS $$
DECLARE
    r RECORD;
    v_incident_id INT;
BEGIN
    FOR r IN
        SELECT 
            ARRAY_AGG(DISTINCT blocked_activity.pid) AS blocked_pids,
            blocking_activity.pid AS blocking_pid,
            now() - blocking_activity.xact_start AS blocking_duration
        FROM pg_locks blocked_locks
        JOIN pg_stat_activity blocked_activity ON blocked_activity.pid = blocked_locks.pid
        JOIN pg_locks blocking_locks ON 
            blocking_locks.relation = blocked_locks.relation
            AND blocking_locks.granted
            AND blocking_locks.pid != blocked_locks.pid
        JOIN pg_stat_activity blocking_activity ON blocking_activity.pid = blocking_locks.pid
        WHERE NOT blocked_locks.granted
          AND blocking_activity.xact_start < now() - (p_threshold_seconds || ' seconds')::INTERVAL
        GROUP BY blocking_activity.pid, blocking_activity.xact_start
    LOOP
        -- บันทึก incident
        INSERT INTO lock_incidents (blocked_pids, blocking_pid, duration)
        VALUES (r.blocked_pids, r.blocking_pid, r.blocking_duration)
        RETURNING incident_id INTO v_incident_id;
        
        RAISE WARNING 'Lock incident %: PID % blocking % sessions for %',
            v_incident_id, r.blocking_pid, 
            array_length(r.blocked_pids, 1),
            r.blocking_duration;
        
        IF p_auto_kill THEN
            PERFORM pg_terminate_backend(r.blocking_pid);
            
            UPDATE lock_incidents 
            SET resolved_at = NOW(), resolution = 'Auto-terminated blocking session'
            WHERE incident_id = v_incident_id;
        END IF;
    END LOOP;
END;
$$;

-- ตรวจสอบทุก 5 นาที (ใน cron หรือ background job)
CALL auto_resolve_lock_issues(300, FALSE);  -- Dry run
-- CALL auto_resolve_lock_issues(300, TRUE);   -- Kill ถ้าจำเป็น
```

---

## สรุป

ใน Part นี้เราได้เรียนรู้:

1. **Lock Granularity**: Table locks, Row locks, Page locks
2. **Lock Types**: Shared (READ) และ Exclusive (WRITE) locks
3. **Advisory Locks**: Application-defined locks สำหรับ coordination
4. **Lock Modes ใน PostgreSQL**: ทั้ง 8 ระดับ ตั้งแต่ ACCESS SHARE จนถึง ACCESS EXCLUSIVE
5. **SELECT FOR UPDATE**: Lock rows สำหรับ update ป้องกัน concurrent modifications
6. **SELECT FOR SHARE**: Lock rows สำหรับ read protection
7. **NOWAIT**: Fail immediately ถ้า lock ไม่ available
8. **SKIP LOCKED**: ข้าม locked rows สำหรับ queue processing
9. **Lock Monitoring**: Queries สำหรับ monitor และ debug lock issues
10. **Lock Timeouts**: ป้องกัน indefinite waits

ใน Part 76 เราจะเรียนรู้เรื่อง Deadlocks - Detection and Prevention ซึ่งเป็นปัญหาที่เกิดจาก locking นี่เอง
