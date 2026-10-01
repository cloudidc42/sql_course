# Part 76: Deadlocks - Detection and Prevention

## บทนำ

**Deadlock** เกิดขึ้นเมื่อสอง (หรือมากกว่า) transactions ต่างรอให้อีกฝ่ายปล่อย lock ก่อน ทำให้ทั้งคู่ไม่สามารถดำเนินต่อได้ เหมือนกับ "ทางตัน" บนถนน

---

## 1. สาเหตุของ Deadlock

```
Deadlock คลาสสิก:

Transaction T1:              Transaction T2:
LOCK account A               LOCK account B
  (รอ B)...                    (รอ A)...
       ↘                    ↗
        DEADLOCK!
        
T1 ถือ A และรอ B
T2 ถือ B และรอ A
ไม่มีใครปล่อยก่อนได้!
```

```sql
-- สาธิต Deadlock
-- Session 1 (ใน terminal แรก):
BEGIN;
UPDATE accounts SET balance = balance - 100 WHERE account_id = 1;  -- Lock account 1
-- รอสักครู่ แล้วรัน:
UPDATE accounts SET balance = balance + 100 WHERE account_id = 2;  -- รอ account 2 จาก Session 2

-- Session 2 (ใน terminal สอง, รันพร้อมกับ Session 1):
BEGIN;
UPDATE accounts SET balance = balance - 100 WHERE account_id = 2;  -- Lock account 2
-- รัน immediately หลังจาก Session 1 รัน UPDATE account 1:
UPDATE accounts SET balance = balance + 100 WHERE account_id = 1;  -- รอ account 1 จาก Session 1

-- ผล: PostgreSQL จะ detect deadlock และ kill หนึ่งใน transactions
-- ERROR: deadlock detected
-- DETAIL: Process X waits for ShareLock on transaction Y; blocked by process Z.
--         Process Z waits for ShareLock on transaction X; blocked by process X.
-- HINT: See server log for query details.
```

---

## 2. Deadlock Detection Algorithms

### PostgreSQL Deadlock Detection

```sql
-- PostgreSQL detect deadlock โดย:
-- 1. รอจนถึง deadlock_timeout (default 1 วินาที)
-- 2. Build wait-for graph
-- 3. ค้นหา cycle ใน graph
-- 4. เลือก victim transaction (usually smaller/younger)
-- 5. Cancel victim transaction

-- ดูค่า deadlock_timeout
SHOW deadlock_timeout;  -- ค่าเริ่มต้น: 1s

-- ตั้งค่า deadlock_timeout (ใน postgresql.conf หรือ per-session)
SET deadlock_timeout = '500ms';  -- 0.5 วินาที
SET deadlock_timeout = '2s';     -- 2 วินาที

-- เหตุใด deadlock_timeout สำคัญ:
-- - สั้นเกินไป: overhead สูง (ตรวจบ่อยมาก)
-- - นานเกินไป: transactions รอนาน ก่อน detect

-- ดู deadlock statistics
SELECT 
    datname,
    deadlocks,
    xact_commit,
    xact_rollback
FROM pg_stat_database
WHERE datname = current_database();

-- Alert ถ้า deadlocks เพิ่มขึ้น
SELECT 
    datname,
    deadlocks,
    CASE 
        WHEN deadlocks > 100 THEN 'HIGH - investigate immediately'
        WHEN deadlocks > 10  THEN 'MEDIUM - monitor closely'
        ELSE 'LOW - normal'
    END AS alert_level
FROM pg_stat_database
WHERE datname NOT IN ('postgres', 'template0', 'template1');
```

---

## 3. Deadlock Error Messages

```sql
-- PostgreSQL Deadlock Error:
/*
ERROR: deadlock detected
DETAIL: Process 12345 waits for ShareLock on transaction 67890; blocked by process 54321.
        Process 54321 waits for ShareLock on transaction 11111; blocked by process 12345.
HINT: See server log for query details.
CONTEXT: while updating tuple (0,42) in relation "accounts"
*/

-- MySQL Deadlock Error:
/*
ERROR 1213 (40001): Deadlock found when trying to get lock; try restarting transaction
*/

-- SQL Server Deadlock Error:
/*
Msg 1205, Level 13, State 51, Line 1
Transaction (Process ID 53) was deadlocked on lock resources with another process 
and has been chosen as the deadlock victim. Rerun the transaction.
*/

-- จัดการ Deadlock Errors ใน PostgreSQL
CREATE OR REPLACE FUNCTION handle_deadlock_demo()
RETURNS TEXT AS $$
BEGIN
    BEGIN
        -- Operations ที่อาจเกิด deadlock
        UPDATE accounts SET balance = balance - 100 WHERE account_id = 1;
        UPDATE accounts SET balance = balance + 100 WHERE account_id = 2;
        RETURN 'Success';
    EXCEPTION
        WHEN deadlock_detected THEN
            -- SQLSTATE = '40P01' สำหรับ deadlock
            RAISE NOTICE 'Deadlock detected! SQLSTATE: %, Message: %', SQLSTATE, SQLERRM;
            RETURN 'Deadlock: retry needed';
    END;
END;
$$ LANGUAGE plpgsql;

-- SQLSTATE codes สำหรับ concurrency issues:
-- 40001 = serialization_failure
-- 40P01 = deadlock_detected
-- 55P03 = lock_not_available (เมื่อใช้ NOWAIT)
```

---

## 4. Deadlock Scenarios ที่พบบ่อย

### Scenario 1: Classic AB-BA Deadlock

```sql
-- Setup
CREATE TABLE accounts_dl (
    id       SERIAL PRIMARY KEY,
    name     VARCHAR(50),
    balance  DECIMAL(15,2)
);

INSERT INTO accounts_dl VALUES (1, 'Alice', 10000), (2, 'Bob', 5000);

/*
T1 Process:
BEGIN;
UPDATE accounts_dl SET balance = balance - 100 WHERE id = 1;  -- Lock id=1
-- ...ก่อนจะรัน UPDATE id=2...

T2 Process (run simultaneously):
BEGIN;
UPDATE accounts_dl SET balance = balance - 200 WHERE id = 2;  -- Lock id=2
UPDATE accounts_dl SET balance = balance + 200 WHERE id = 1;  -- Wait for id=1 → DEADLOCK!

T1 continues:
UPDATE accounts_dl SET balance = balance + 100 WHERE id = 2;  -- Wait for id=2 → DEADLOCK!
*/

-- Script สำหรับ simulate (ใช้ dblink หรือ pg_background)
-- เราจะสาธิต prevention แทน
```

### Scenario 2: Transaction with Locks ออก Order ต่างกัน

```sql
-- Pattern ที่เกิดบ่อยใน application code:
-- Function 1:
CREATE OR REPLACE FUNCTION transfer_v1(from_id INT, to_id INT, amount DECIMAL)
RETURNS VOID AS $$
BEGIN
    -- Lock from FIRST แล้ว to
    SELECT * FROM accounts_dl WHERE id = from_id FOR UPDATE;
    SELECT * FROM accounts_dl WHERE id = to_id FOR UPDATE;
    UPDATE accounts_dl SET balance = balance - amount WHERE id = from_id;
    UPDATE accounts_dl SET balance = balance + amount WHERE id = to_id;
END;
$$ LANGUAGE plpgsql;

-- Function 2 (กลับลำดับ):
CREATE OR REPLACE FUNCTION transfer_v2(from_id INT, to_id INT, amount DECIMAL)
RETURNS VOID AS $$
BEGIN
    -- Lock to FIRST แล้ว from (กลับลำดับ!)
    SELECT * FROM accounts_dl WHERE id = to_id FOR UPDATE;
    SELECT * FROM accounts_dl WHERE id = from_id FOR UPDATE;
    UPDATE accounts_dl SET balance = balance - amount WHERE id = from_id;
    UPDATE accounts_dl SET balance = balance + amount WHERE id = to_id;
END;
$$ LANGUAGE plpgsql;

-- ถ้า T1 เรียก transfer_v1(1, 2, ...) และ T2 เรียก transfer_v2(1, 2, ...)
-- → T1 lock 1, T2 lock 2, T1 รอ 2, T2 รอ 1 → DEADLOCK!
```

### Scenario 3: Implicit Locks ใน FK Checks

```sql
-- CREATE TABLE parent_dl (id INT PRIMARY KEY);
-- CREATE TABLE child_dl (id INT PRIMARY KEY, parent_id INT REFERENCES parent_dl(id));

-- T1: DELETE parent
-- BEGIN; DELETE FROM parent_dl WHERE id = 1; → Lock parent row

-- T2: INSERT child
-- BEGIN; INSERT INTO child_dl VALUES (1, 1); → Lock parent (shared) for FK check

-- T1: ตรวจสอบ child เพื่อตรวจ constraint
-- → ต้องการ lock บน child table

-- T2: ต้องการ exclusive lock บน parent → รอ T1

-- → DEADLOCK!

-- วิธีแก้: Lock in correct order, use CASCADE
CREATE TABLE parent_safe (
    id INT PRIMARY KEY
);

CREATE TABLE child_safe (
    id        INT PRIMARY KEY,
    parent_id INT REFERENCES parent_safe(id) ON DELETE CASCADE
);
-- CASCADE ลด lock contention
```

---

## 5. Deadlock Prevention Strategies

### Strategy 1: Consistent Lock Ordering

```sql
-- วิธีที่ดีที่สุด: Lock accounts เสมอตาม ID ที่เล็กกว่าก่อน
CREATE OR REPLACE FUNCTION transfer_safe(
    p_from_id INT,
    p_to_id   INT,
    p_amount  DECIMAL
) RETURNS TEXT AS $$
BEGIN
    -- Lock ตาม ID เสมอ (smaller ID first)
    IF p_from_id < p_to_id THEN
        PERFORM * FROM accounts_dl WHERE id = p_from_id FOR UPDATE;
        PERFORM * FROM accounts_dl WHERE id = p_to_id FOR UPDATE;
    ELSE
        PERFORM * FROM accounts_dl WHERE id = p_to_id FOR UPDATE;
        PERFORM * FROM accounts_dl WHERE id = p_from_id FOR UPDATE;
    END IF;
    
    UPDATE accounts_dl SET balance = balance - p_amount WHERE id = p_from_id;
    UPDATE accounts_dl SET balance = balance + p_amount WHERE id = p_to_id;
    
    RETURN 'Transfer completed';
END;
$$ LANGUAGE plpgsql;

-- ทดสอบ: ไม่ว่าจะ call ลำดับใด จะไม่เกิด deadlock
-- T1: transfer_safe(1, 2, 100)  → Lock 1, Lock 2
-- T2: transfer_safe(2, 1, 200)  → Lock 1 (wait), Lock 2 → ไม่ deadlock!
BEGIN;
SELECT transfer_safe(1, 2, 100);
COMMIT;
```

### Strategy 2: Use lock_timeout

```sql
-- ตั้งค่า lock_timeout เพื่อให้ fail fast แทนที่จะ deadlock
SET lock_timeout = '5s';

-- ใน configuration file:
-- lock_timeout = '10s'  -- Global default

-- ตั้งค่าสำหรับ role เฉพาะ:
ALTER ROLE api_user SET lock_timeout = '5s';
ALTER ROLE reporting_user SET lock_timeout = '30s';

-- ตัวอย่าง: Handle timeout gracefully
CREATE OR REPLACE FUNCTION transfer_with_timeout(
    p_from_id INT,
    p_to_id   INT,
    p_amount  DECIMAL,
    p_timeout_ms INT DEFAULT 5000
) RETURNS JSONB AS $$
BEGIN
    SET LOCAL lock_timeout = p_timeout_ms;
    
    BEGIN
        -- Lock in consistent order
        IF p_from_id < p_to_id THEN
            PERFORM * FROM accounts_dl WHERE id = p_from_id FOR UPDATE;
            PERFORM * FROM accounts_dl WHERE id = p_to_id FOR UPDATE;
        ELSE
            PERFORM * FROM accounts_dl WHERE id = p_to_id FOR UPDATE;
            PERFORM * FROM accounts_dl WHERE id = p_from_id FOR UPDATE;
        END IF;
        
        UPDATE accounts_dl SET balance = balance - p_amount WHERE id = p_from_id;
        UPDATE accounts_dl SET balance = balance + p_amount WHERE id = p_to_id;
        
        RETURN jsonb_build_object('success', true);
        
    EXCEPTION
        WHEN lock_not_available THEN
            RETURN jsonb_build_object(
                'success', false,
                'error', 'Lock timeout - system busy, retry later',
                'retry_after_ms', 1000
            );
        WHEN deadlock_detected THEN
            RETURN jsonb_build_object(
                'success', false,
                'error', 'Deadlock detected - retry immediately',
                'retry_after_ms', 100
            );
    END;
END;
$$ LANGUAGE plpgsql;

BEGIN;
SELECT transfer_with_timeout(1, 2, 100, 3000);
COMMIT;
```

### Strategy 3: Retry Logic

```sql
-- Retry pattern สำหรับ deadlock situations
CREATE OR REPLACE FUNCTION transfer_with_retry(
    p_from_id    INT,
    p_to_id      INT,
    p_amount     DECIMAL,
    p_max_retries INT DEFAULT 3,
    p_base_delay_ms INT DEFAULT 100
) RETURNS JSONB AS $$
DECLARE
    v_attempt   INT := 0;
    v_delay_ms  INT;
    v_result    JSONB;
BEGIN
    LOOP
        v_attempt := v_attempt + 1;
        
        BEGIN
            -- Consistent locking order
            IF p_from_id < p_to_id THEN
                PERFORM * FROM accounts_dl WHERE id = p_from_id FOR UPDATE;
                PERFORM * FROM accounts_dl WHERE id = p_to_id FOR UPDATE;
            ELSE
                PERFORM * FROM accounts_dl WHERE id = p_to_id FOR UPDATE;
                PERFORM * FROM accounts_dl WHERE id = p_from_id FOR UPDATE;
            END IF;
            
            UPDATE accounts_dl SET balance = balance - p_amount WHERE id = p_from_id;
            UPDATE accounts_dl SET balance = balance + p_amount WHERE id = p_to_id;
            
            RETURN jsonb_build_object('success', true, 'attempts', v_attempt);
            
        EXCEPTION
            WHEN deadlock_detected THEN
                IF v_attempt >= p_max_retries THEN
                    RETURN jsonb_build_object(
                        'success', false, 
                        'error', 'Max retries exceeded after deadlock',
                        'attempts', v_attempt
                    );
                END IF;
                
                -- Exponential backoff + jitter
                v_delay_ms := p_base_delay_ms * (2 ^ (v_attempt - 1)) + 
                              (random() * 50)::INT;
                
                RAISE NOTICE 'Deadlock on attempt %, waiting %ms before retry',
                              v_attempt, v_delay_ms;
                
                PERFORM pg_sleep(v_delay_ms / 1000.0);
                
            WHEN serialization_failure THEN
                -- Similar retry for serialization failures
                IF v_attempt >= p_max_retries THEN
                    RETURN jsonb_build_object(
                        'success', false,
                        'error', 'Serialization failure after max retries',
                        'attempts', v_attempt
                    );
                END IF;
                PERFORM pg_sleep(p_base_delay_ms / 1000.0 * v_attempt);
                
            WHEN OTHERS THEN
                RAISE;  -- ไม่ retry สำหรับ errors อื่น
        END;
    END LOOP;
END;
$$ LANGUAGE plpgsql;

BEGIN;
SELECT transfer_with_retry(1, 2, 100);
COMMIT;
```

### Strategy 4: Minimize Lock Hold Time

```sql
-- ลด scope ของ transaction เพื่อลด lock duration

-- ไม่ดี: transaction ยาวที่ hold lock นาน
BEGIN;
    SELECT * FROM products WHERE id = 1 FOR UPDATE;  -- Lock ทันที
    -- ทำการ validation ที่ใช้เวลา
    PERFORM long_validation_function();  -- ใช้เวลา 5 วินาที!
    UPDATE products SET stock = stock - 1 WHERE id = 1;
COMMIT;

-- ดี: ทำ validation ก่อน แล้ว lock แค่ช่วงที่จำเป็น
-- Step 1: Read without lock
SELECT stock FROM products WHERE id = 1;

-- Step 2: Run validation (outside transaction)
-- validate_product(product_data);

-- Step 3: Short transaction with lock
BEGIN;
    -- Lock และตรวจสอบอีกครั้ง (quick check)
    SELECT stock INTO v_stock FROM products WHERE id = 1 FOR UPDATE;
    IF v_stock < 1 THEN RAISE EXCEPTION 'Out of stock'; END IF;
    UPDATE products SET stock = stock - 1 WHERE id = 1;
COMMIT;

-- ตัวอย่าง: Optimized inventory deduction
CREATE OR REPLACE FUNCTION optimized_stock_deduct(
    p_product_id INT,
    p_quantity INT
) RETURNS BOOLEAN AS $$
DECLARE
    v_updated INT;
BEGIN
    -- Atomic UPDATE ที่ไม่ต้อง lock ก่อน read
    UPDATE products
    SET stock = stock - p_quantity
    WHERE product_id = p_product_id
      AND stock >= p_quantity;  -- Check condition ใน UPDATE เอง
    
    GET DIAGNOSTICS v_updated = ROW_COUNT;
    
    IF v_updated = 0 THEN
        RETURN FALSE;  -- ไม่มีสต็อกพอ
    END IF;
    
    RETURN TRUE;
END;
$$ LANGUAGE plpgsql;
-- วิธีนี้ atomic และลดโอกาส deadlock อย่างมาก
```

---

## 6. Deadlock Logs

```sql
-- ตั้งค่าให้ log deadlocks
-- postgresql.conf:
-- log_min_messages = notice
-- log_min_duration_statement = 0  (log ทุก statement)
-- log_lock_waits = on  (log เมื่อรอ lock นานกว่า deadlock_timeout)

-- ดู logs จาก PostgreSQL
-- Linux: tail -f /var/log/postgresql/postgresql-*.log

-- Sample deadlock log:
/*
2024-01-15 10:23:45 UTC [12345]: ERROR: deadlock detected
2024-01-15 10:23:45 UTC [12345]: DETAIL: Process 12345 waits for ShareLock on 
    transaction 67890; blocked by process 54321.
    Process 54321 waits for ShareLock on transaction 12345; blocked by process 12345.
2024-01-15 10:23:45 UTC [12345]: HINT: See server log for query details.
2024-01-15 10:23:45 UTC [12345]: CONTEXT: while updating tuple (0,42) in relation "accounts_dl"
2024-01-15 10:23:45 UTC [12345]: STATEMENT: UPDATE accounts_dl SET balance = balance + 100 
    WHERE id = 2
*/

-- Custom deadlock logging function
CREATE OR REPLACE FUNCTION log_deadlock(p_context TEXT DEFAULT '')
RETURNS VOID AS $$
BEGIN
    INSERT INTO deadlock_log (
        occurred_at,
        session_pid,
        context,
        query,
        lock_graph
    )
    SELECT 
        NOW(),
        pg_backend_pid(),
        p_context,
        (SELECT query FROM pg_stat_activity WHERE pid = pg_backend_pid()),
        (
            SELECT jsonb_agg(jsonb_build_object(
                'pid', pid,
                'locktype', locktype,
                'relation', CASE WHEN relation IS NOT NULL THEN relation::regclass::TEXT ELSE NULL END,
                'mode', mode,
                'granted', granted
            ))
            FROM pg_locks
            WHERE pid = pg_backend_pid()
        );
END;
$$ LANGUAGE plpgsql;

-- สร้าง deadlock log table
CREATE TABLE IF NOT EXISTS deadlock_log (
    id          SERIAL PRIMARY KEY,
    occurred_at TIMESTAMPTZ DEFAULT NOW(),
    session_pid INT,
    context     TEXT,
    query       TEXT,
    lock_graph  JSONB
);

-- ใช้ใน exception handler:
BEGIN
    -- risky operations
EXCEPTION
    WHEN deadlock_detected THEN
        PERFORM log_deadlock('transfer operation');
        RAISE;
END;
```

---

## 7. Application-Level Deadlock Handling

```sql
-- Pattern 1: Simple Retry
CREATE OR REPLACE FUNCTION execute_with_deadlock_retry(
    p_max_retries INT DEFAULT 5
) RETURNS VOID AS $$
DECLARE
    v_attempt INT := 0;
    v_delay INTERVAL;
BEGIN
    LOOP
        v_attempt := v_attempt + 1;
        
        BEGIN
            -- ทำ operation
            UPDATE accounts_dl SET balance = balance - 100 WHERE id = 1;
            UPDATE accounts_dl SET balance = balance + 100 WHERE id = 2;
            RETURN;  -- สำเร็จ → ออกจาก loop
            
        EXCEPTION WHEN deadlock_detected THEN
            IF v_attempt >= p_max_retries THEN
                RAISE EXCEPTION 'Deadlock after % retries', v_attempt;
            END IF;
            
            -- Jittered exponential backoff
            v_delay := (100 * (2 ^ (v_attempt - 1)) + (random() * 100)::INT) 
                       * INTERVAL '1 millisecond';
            
            RAISE NOTICE 'Deadlock on attempt %, waiting % before retry', v_attempt, v_delay;
            PERFORM pg_sleep(EXTRACT(EPOCH FROM v_delay));
        END;
    END LOOP;
END;
$$ LANGUAGE plpgsql;

-- Pattern 2: Deadlock-Safe Batch Processing
CREATE OR REPLACE FUNCTION process_transfers_batch(
    p_transfers JSONB[]  -- [{"from": 1, "to": 2, "amount": 100}, ...]
) RETURNS TABLE(
    transfer_index INT,
    success        BOOLEAN,
    message        TEXT,
    attempts       INT
) AS $$
DECLARE
    v_transfer  JSONB;
    v_idx       INT := 0;
    v_attempt   INT;
    v_done      BOOLEAN;
BEGIN
    FOREACH v_transfer IN ARRAY p_transfers
    LOOP
        v_idx := v_idx + 1;
        v_attempt := 0;
        v_done := FALSE;
        
        WHILE NOT v_done AND v_attempt < 3 LOOP
            v_attempt := v_attempt + 1;
            
            BEGIN
                PERFORM transfer_safe(
                    (v_transfer->>'from')::INT,
                    (v_transfer->>'to')::INT,
                    (v_transfer->>'amount')::DECIMAL
                );
                
                transfer_index := v_idx;
                success        := TRUE;
                message        := 'Success';
                attempts       := v_attempt;
                RETURN NEXT;
                v_done := TRUE;
                
            EXCEPTION
                WHEN deadlock_detected THEN
                    IF v_attempt < 3 THEN
                        PERFORM pg_sleep(0.05 * v_attempt);  -- Wait 50ms, 100ms
                    ELSE
                        transfer_index := v_idx;
                        success        := FALSE;
                        message        := 'Deadlock after 3 retries';
                        attempts       := v_attempt;
                        RETURN NEXT;
                        v_done := TRUE;
                    END IF;
                    
                WHEN OTHERS THEN
                    transfer_index := v_idx;
                    success        := FALSE;
                    message        := SQLERRM;
                    attempts       := v_attempt;
                    RETURN NEXT;
                    v_done := TRUE;
            END;
        END LOOP;
    END LOOP;
END;
$$ LANGUAGE plpgsql;
```

---

## 8. Deadlock Prevention Best Practices

```sql
-- Best Practice 1: Lock Ordering Convention
-- กำหนด convention ทั้ง codebase ให้ lock tables ตามลำดับ alphabetically
-- orders → order_items → products → inventory

-- Best Practice 2: Avoid Holding Locks During User Input
-- ไม่ดี:
BEGIN;
SELECT * FROM shopping_cart WHERE user_id = 1 FOR UPDATE;
-- รอ user กด checkout button...  (lock นาน 5 นาที!)
INSERT INTO orders SELECT * FROM shopping_cart WHERE user_id = 1;
COMMIT;

-- ดี:
SELECT * FROM shopping_cart WHERE user_id = 1;  -- Read ก่อน ไม่ต้อง lock
-- รอ user กด checkout
BEGIN;
-- Lock เฉพาะเมื่อต้องการ process
SELECT * FROM shopping_cart WHERE user_id = 1 FOR UPDATE;
INSERT INTO orders SELECT * FROM shopping_cart WHERE user_id = 1;
DELETE FROM shopping_cart WHERE user_id = 1;
COMMIT;

-- Best Practice 3: Use SELECT FOR UPDATE SKIP LOCKED สำหรับ Queue
-- ป้องกัน deadlock ใน queue processing
CREATE OR REPLACE FUNCTION process_next_batch()
RETURNS VOID AS $$
BEGIN
    WITH items AS (
        SELECT id FROM work_items
        WHERE status = 'pending'
        LIMIT 10
        FOR UPDATE SKIP LOCKED  -- ข้าม items ที่ถูก lock แล้ว
    )
    UPDATE work_items SET status = 'processing'
    FROM items
    WHERE work_items.id = items.id;
END;
$$ LANGUAGE plpgsql;

-- Best Practice 4: Reduce Transaction Scope
-- ทำ read operations ข้างนอก transaction ถ้าทำได้

-- Best Practice 5: Use Optimistic Locking ถ้า conflicts ไม่บ่อย
-- ดูใน Part 78

-- Best Practice 6: Monitor Regularly
CREATE OR REPLACE VIEW deadlock_monitor AS
SELECT 
    datname,
    deadlocks,
    pg_size_pretty(pg_database_size(datname)) AS db_size,
    stats_reset
FROM pg_stat_database
WHERE datname = current_database();

-- Alert เมื่อ deadlocks เพิ่มขึ้น
DO $$
DECLARE
    v_deadlocks BIGINT;
    v_prev_check TIMESTAMPTZ := NOW() - INTERVAL '1 hour';
BEGIN
    SELECT deadlocks INTO v_deadlocks
    FROM pg_stat_database
    WHERE datname = current_database();
    
    IF v_deadlocks > 10 THEN
        RAISE WARNING 'High deadlock count: % in this database', v_deadlocks;
    END IF;
END;
$$;
```

---

## 9. Deadlock Analysis และ Debugging

```sql
-- Query เพื่อหา potential deadlock scenarios
-- ค้นหา transactions ที่ hold locks และรอ locks

SELECT 
    a1.pid AS holding_pid,
    a1.usename AS holding_user,
    l1.relation::regclass AS holding_table,
    l1.mode AS held_lock,
    a2.pid AS waiting_pid,
    a2.usename AS waiting_user,
    l2.mode AS wanted_lock,
    now() - a2.query_start AS wait_duration,
    left(a2.query, 100) AS waiting_query
FROM pg_locks l1
JOIN pg_stat_activity a1 ON l1.pid = a1.pid AND l1.granted = TRUE
JOIN pg_locks l2 ON l2.relation = l1.relation AND l2.granted = FALSE
JOIN pg_stat_activity a2 ON l2.pid = a2.pid
WHERE l1.pid != l2.pid
  AND l1.locktype = 'relation'
ORDER BY wait_duration DESC;

-- Extended deadlock analysis - find cycles in wait-for graph
CREATE OR REPLACE FUNCTION analyze_deadlock_risk()
RETURNS TABLE(
    session_pid    INT,
    holding_lock   TEXT,
    waiting_for    TEXT,
    risk_level     TEXT
) AS $$
BEGIN
    RETURN QUERY
    WITH wait_for AS (
        SELECT 
            bl.pid AS blocked_pid,
            bk.pid AS blocking_pid,
            bl.relation::regclass::TEXT AS table_name,
            bl.mode AS needed_mode,
            bk.mode AS held_mode
        FROM pg_locks bl
        JOIN pg_locks bk ON bl.relation = bk.relation
                         AND bk.granted = TRUE
                         AND bk.pid != bl.pid
        WHERE NOT bl.granted
          AND bl.locktype = 'relation'
    ),
    potential_deadlocks AS (
        SELECT 
            wf1.blocked_pid,
            wf1.blocking_pid,
            wf1.table_name,
            wf2.table_name AS also_waiting_on
        FROM wait_for wf1
        JOIN wait_for wf2 ON wf1.blocking_pid = wf2.blocked_pid
                         AND wf2.blocking_pid = wf1.blocked_pid
    )
    SELECT 
        pd.blocked_pid::INT,
        pd.table_name,
        pd.also_waiting_on,
        'HIGH RISK - Circular wait detected'::TEXT
    FROM potential_deadlocks pd;
END;
$$ LANGUAGE plpgsql;

SELECT * FROM analyze_deadlock_risk();

-- Simulate safe environment สำหรับ testing
CREATE OR REPLACE FUNCTION create_deadlock_test_environment()
RETURNS VOID AS $$
BEGIN
    -- สร้าง tables สำหรับ test
    DROP TABLE IF EXISTS dl_test_a, dl_test_b;
    
    CREATE TABLE dl_test_a (id INT PRIMARY KEY, value INT);
    CREATE TABLE dl_test_b (id INT PRIMARY KEY, value INT);
    
    INSERT INTO dl_test_a VALUES (1, 100), (2, 200);
    INSERT INTO dl_test_b VALUES (1, 300), (2, 400);
    
    RAISE NOTICE 'Deadlock test environment ready';
    RAISE NOTICE 'To create deadlock:';
    RAISE NOTICE '  Session 1: BEGIN; UPDATE dl_test_a SET value=1 WHERE id=1; (wait)';
    RAISE NOTICE '  Session 2: BEGIN; UPDATE dl_test_b SET value=2 WHERE id=1; UPDATE dl_test_a SET value=3 WHERE id=1;';
    RAISE NOTICE '  Session 1: UPDATE dl_test_b SET value=4 WHERE id=1; (deadlock!)';
END;
$$ LANGUAGE plpgsql;
```

---

## 10. PostgreSQL-Specific Deadlock Settings

```sql
-- postgresql.conf settings สำหรับ deadlock management

-- 1. deadlock_timeout (ระยะเวลาก่อน detect deadlock)
-- ค่าเริ่มต้น: 1s
-- ต่ำเกินไป: overhead จาก frequent checks
-- สูงเกินไป: transactions รอนาน
SHOW deadlock_timeout;

-- ตั้งค่าชั่วคราว:
SET deadlock_timeout = '500ms';

-- 2. lock_timeout (รอ lock นานสุด)
SHOW lock_timeout;
SET lock_timeout = '10s';

-- 3. statement_timeout (รอ statement นานสุด)
SHOW statement_timeout;
SET statement_timeout = '60s';

-- 4. idle_in_transaction_session_timeout
-- ป้องกัน transactions ที่ hold locks โดยไม่ทำอะไร
SHOW idle_in_transaction_session_timeout;
SET idle_in_transaction_session_timeout = '5min';

-- 5. log_lock_waits (log เมื่อรอ lock นานกว่า deadlock_timeout)
SHOW log_lock_waits;
-- ตั้งใน postgresql.conf: log_lock_waits = on

-- ดู session-level settings
SELECT 
    name,
    setting,
    unit,
    source
FROM pg_settings
WHERE name IN (
    'deadlock_timeout',
    'lock_timeout',
    'statement_timeout',
    'idle_in_transaction_session_timeout',
    'log_lock_waits'
)
ORDER BY name;

-- ตั้งค่าสำหรับ database ทั้งหมด
ALTER DATABASE mydb SET deadlock_timeout = '500ms';
ALTER DATABASE mydb SET lock_timeout = '10s';
ALTER DATABASE mydb SET idle_in_transaction_session_timeout = '5min';

-- ตั้งค่าสำหรับ role เฉพาะ
ALTER ROLE api_role SET lock_timeout = '5s';
ALTER ROLE batch_role SET statement_timeout = '300s';
```

---

## 11. Real-World Deadlock Cases

### Case 1: E-Commerce Flash Sale

```sql
-- ปัญหา: หลาย users ซื้อสินค้าพร้อมกัน

-- วิธีที่เกิด deadlock:
-- T1: Lock product 101, ตรวจสอบ stock, Lock user cart
-- T2: Lock user cart (ของ user อื่น), Lock product 101 สำหรับ stock check
-- → T1 รอ cart ของ T2, T2 รอ product lock ของ T1 → DEADLOCK!

-- วิธีแก้: Consistent lock ordering
CREATE OR REPLACE FUNCTION flash_sale_purchase(
    p_user_id   INT,
    p_product_id INT,
    p_quantity  INT
) RETURNS JSONB AS $$
DECLARE
    v_available INT;
    v_cart_id   INT;
BEGIN
    -- Lock in CONSISTENT ORDER: products first, then carts
    -- (ไม่ใช่: cart first แล้ว products)
    
    -- Step 1: Lock product (always first)
    SELECT stock INTO v_available
    FROM flash_products
    WHERE product_id = p_product_id
    FOR UPDATE;
    
    IF v_available < p_quantity THEN
        RETURN jsonb_build_object('success', false, 'error', 'Sold out');
    END IF;
    
    -- Step 2: Lock user cart (always second)
    SELECT cart_id INTO v_cart_id
    FROM user_carts
    WHERE user_id = p_user_id
    FOR UPDATE;
    
    -- Step 3: Make updates
    UPDATE flash_products SET stock = stock - p_quantity WHERE product_id = p_product_id;
    INSERT INTO cart_items (cart_id, product_id, quantity) VALUES (v_cart_id, p_product_id, p_quantity);
    
    RETURN jsonb_build_object('success', true, 'remaining_stock', v_available - p_quantity);
END;
$$ LANGUAGE plpgsql;
```

### Case 2: Financial System

```sql
-- ปัญหา: Payment processor กับ Balance checker

-- Deadlock-Safe Payment:
CREATE OR REPLACE FUNCTION process_payment_safe(
    p_payer_id     INT,
    p_payee_id     INT,
    p_amount       DECIMAL,
    p_payment_type VARCHAR(50)
) RETURNS JSONB AS $$
DECLARE
    v_payer_balance DECIMAL;
    v_fee           DECIMAL;
    v_net_amount    DECIMAL;
BEGIN
    -- กำหนด fee ตาม type
    v_fee := CASE p_payment_type
        WHEN 'instant' THEN p_amount * 0.015
        WHEN 'standard' THEN p_amount * 0.005
        ELSE 0
    END;
    v_net_amount := p_amount - v_fee;
    
    -- LOCK IN CONSISTENT ORDER (เรียง ID เสมอ)
    IF p_payer_id < p_payee_id THEN
        SELECT balance INTO v_payer_balance 
        FROM financial_accounts WHERE user_id = p_payer_id FOR UPDATE;
        PERFORM * FROM financial_accounts WHERE user_id = p_payee_id FOR UPDATE;
    ELSE
        PERFORM * FROM financial_accounts WHERE user_id = p_payee_id FOR UPDATE;
        SELECT balance INTO v_payer_balance 
        FROM financial_accounts WHERE user_id = p_payer_id FOR UPDATE;
    END IF;
    
    -- Validation
    IF v_payer_balance < p_amount THEN
        RETURN jsonb_build_object('success', false, 'error', 'Insufficient balance');
    END IF;
    
    -- Transfer
    UPDATE financial_accounts SET balance = balance - p_amount WHERE user_id = p_payer_id;
    UPDATE financial_accounts SET balance = balance + v_net_amount WHERE user_id = p_payee_id;
    
    -- Record transaction
    INSERT INTO payment_history (payer_id, payee_id, amount, fee, payment_type, created_at)
    VALUES (p_payer_id, p_payee_id, p_amount, v_fee, p_payment_type, NOW());
    
    RETURN jsonb_build_object(
        'success', true,
        'amount', p_amount,
        'fee', v_fee,
        'net_received', v_net_amount
    );
END;
$$ LANGUAGE plpgsql;
```

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: Deadlock Scenario Recreation

```sql
-- คำตอบ: Setup สำหรับสาธิต deadlock
CREATE TABLE dl_exercise (
    id      SERIAL PRIMARY KEY,
    name    VARCHAR(50),
    value   INT DEFAULT 0
);

INSERT INTO dl_exercise VALUES (1, 'Resource A', 100), (2, 'Resource B', 200);

-- สร้าง script สำหรับ Session 1 (บันทึกไว้เป็น session1.sql):
/*
BEGIN;
UPDATE dl_exercise SET value = value + 10 WHERE id = 1;
SELECT pg_sleep(2);  -- รอ 2 วินาที
UPDATE dl_exercise SET value = value + 10 WHERE id = 2;
COMMIT;
*/

-- สร้าง script สำหรับ Session 2 (บันทึกไว้เป็น session2.sql):
/*
BEGIN;
UPDATE dl_exercise SET value = value + 20 WHERE id = 2;
SELECT pg_sleep(1);  -- รอ 1 วินาที
UPDATE dl_exercise SET value = value + 20 WHERE id = 1;
COMMIT;
*/

-- รัน: psql ... -f session1.sql & psql ... -f session2.sql
-- สังเกต: หนึ่งใน sessions จะได้รับ deadlock error
```

### แบบฝึกหัดที่ 2: Deadlock Prevention Function

```sql
-- คำตอบ
CREATE OR REPLACE FUNCTION deadlock_safe_update(
    p_ids  INT[],
    p_delta INT
) RETURNS INT AS $$
DECLARE
    v_sorted_ids INT[];
    v_id         INT;
BEGIN
    -- Sort IDs เพื่อ consistent lock order
    SELECT ARRAY_AGG(id ORDER BY id) INTO v_sorted_ids
    FROM UNNEST(p_ids) AS id;
    
    -- Lock in sorted order
    FOREACH v_id IN ARRAY v_sorted_ids
    LOOP
        PERFORM * FROM dl_exercise WHERE id = v_id FOR UPDATE;
    END LOOP;
    
    -- Update
    UPDATE dl_exercise SET value = value + p_delta WHERE id = ANY(v_sorted_ids);
    
    RETURN array_length(v_sorted_ids, 1);
END;
$$ LANGUAGE plpgsql;

-- Test: ไม่มี deadlock เพราะ lock order เหมือนกันเสมอ
BEGIN;
SELECT deadlock_safe_update(ARRAY[2, 1, 3], 10);  -- จะ lock ตามลำดับ 1, 2, 3
COMMIT;
```

### แบบฝึกหัดที่ 3: Deadlock Detection Query

```sql
-- คำตอบ
CREATE OR REPLACE VIEW potential_deadlocks AS
WITH waiting_sessions AS (
    SELECT 
        a.pid,
        a.usename,
        a.query,
        l.relation,
        l.mode
    FROM pg_stat_activity a
    JOIN pg_locks l ON l.pid = a.pid AND NOT l.granted
),
holding_sessions AS (
    SELECT 
        a.pid,
        a.usename,
        l.relation,
        l.mode
    FROM pg_stat_activity a
    JOIN pg_locks l ON l.pid = a.pid AND l.granted
)
SELECT 
    w.pid AS waiting_pid,
    w.usename AS waiting_user,
    h.pid AS holding_pid,
    h.usename AS holding_user,
    w.relation::regclass AS table_name,
    w.mode AS wanted_mode,
    h.mode AS held_mode
FROM waiting_sessions w
JOIN holding_sessions h ON w.relation = h.relation AND w.pid != h.pid
ORDER BY w.pid;

SELECT * FROM potential_deadlocks;
```

### แบบฝึกหัดที่ 4: Retry with Backoff

```sql
-- คำตอบ
CREATE OR REPLACE FUNCTION retry_on_deadlock(
    p_sql           TEXT,
    p_max_retries   INT DEFAULT 5,
    p_base_delay_ms INT DEFAULT 50
) RETURNS TEXT AS $$
DECLARE
    v_attempt INT := 0;
    v_delay   INT;
BEGIN
    LOOP
        v_attempt := v_attempt + 1;
        
        BEGIN
            EXECUTE p_sql;
            RETURN format('Success on attempt %s', v_attempt);
            
        EXCEPTION
            WHEN deadlock_detected THEN
                IF v_attempt >= p_max_retries THEN
                    RAISE EXCEPTION 'Failed after % retries due to deadlock', v_attempt;
                END IF;
                
                -- Jittered exponential backoff
                v_delay := p_base_delay_ms * (2 ^ (v_attempt - 1)) + (random() * p_base_delay_ms)::INT;
                RAISE NOTICE 'Deadlock attempt %, retrying in %ms', v_attempt, v_delay;
                PERFORM pg_sleep(v_delay / 1000.0);
                
            WHEN OTHERS THEN
                RAISE;
        END;
    END LOOP;
END;
$$ LANGUAGE plpgsql;
```

### แบบฝึกหัดที่ 5: Lock Statistics Report

```sql
-- คำตอบ
CREATE OR REPLACE FUNCTION generate_lock_report()
RETURNS TABLE(
    metric TEXT,
    value TEXT,
    status TEXT
) AS $$
DECLARE
    v_deadlocks BIGINT;
    v_waiting INT;
    v_long_waits INT;
BEGIN
    SELECT deadlocks INTO v_deadlocks FROM pg_stat_database 
    WHERE datname = current_database();
    
    SELECT COUNT(*) INTO v_waiting FROM pg_locks WHERE NOT granted;
    
    SELECT COUNT(*) INTO v_long_waits FROM pg_locks 
    WHERE NOT granted AND waitstart < NOW() - INTERVAL '30 seconds';
    
    metric := 'Total Deadlocks'; 
    value  := v_deadlocks::TEXT;
    status := CASE WHEN v_deadlocks > 100 THEN 'HIGH' WHEN v_deadlocks > 10 THEN 'MEDIUM' ELSE 'OK' END;
    RETURN NEXT;
    
    metric := 'Waiting Locks';
    value  := v_waiting::TEXT;
    status := CASE WHEN v_waiting > 20 THEN 'CRITICAL' WHEN v_waiting > 5 THEN 'WARNING' ELSE 'OK' END;
    RETURN NEXT;
    
    metric := 'Long Wait Locks (>30s)';
    value  := v_long_waits::TEXT;
    status := CASE WHEN v_long_waits > 0 THEN 'CRITICAL' ELSE 'OK' END;
    RETURN NEXT;
    
    metric := 'Deadlock Timeout';
    value  := current_setting('deadlock_timeout');
    status := 'INFO';
    RETURN NEXT;
END;
$$ LANGUAGE plpgsql;

SELECT * FROM generate_lock_report();
```

### แบบฝึกหัดที่ 6: Safe Inventory System

```sql
-- คำตอบ
CREATE TABLE safe_inventory (
    product_id  INT PRIMARY KEY,
    name        VARCHAR(200),
    stock       INT NOT NULL CHECK (stock >= 0),
    reserved    INT NOT NULL DEFAULT 0 CHECK (reserved >= 0),
    available   INT GENERATED ALWAYS AS (stock - reserved) STORED
);

CREATE OR REPLACE FUNCTION reserve_items(
    p_items     INT[],
    p_quantities INT[]
) RETURNS JSONB AS $$
DECLARE
    v_sorted_items   INT[];
    v_item           INT;
    v_idx            INT;
    v_available      INT;
    v_results        JSONB[] := '{}';
BEGIN
    -- Sort items untuk consistent lock order
    WITH sorted AS (
        SELECT UNNEST(p_items) AS item_id,
               UNNEST(p_quantities) AS qty,
               ROW_NUMBER() OVER (ORDER BY UNNEST(p_items)) AS rn
    )
    SELECT ARRAY_AGG(item_id ORDER BY item_id) INTO v_sorted_items FROM sorted;
    
    -- Lock all items in sorted order
    FOREACH v_item IN ARRAY v_sorted_items
    LOOP
        SELECT available INTO v_available
        FROM safe_inventory
        WHERE product_id = v_item
        FOR UPDATE;
        
        v_idx := ARRAY_POSITION(p_items, v_item);
        
        IF v_available < p_quantities[v_idx] THEN
            RAISE EXCEPTION 'Insufficient stock for product %', v_item;
        END IF;
    END LOOP;
    
    -- Apply reservations
    FOR v_idx IN 1..array_length(p_items, 1)
    LOOP
        UPDATE safe_inventory
        SET reserved = reserved + p_quantities[v_idx]
        WHERE product_id = p_items[v_idx];
        
        v_results := array_append(v_results,
            jsonb_build_object('product_id', p_items[v_idx], 'reserved', p_quantities[v_idx]));
    END LOOP;
    
    RETURN jsonb_build_object('success', true, 'reservations', to_jsonb(v_results));
END;
$$ LANGUAGE plpgsql;
```

### แบบฝึกหัดที่ 7: Deadlock Event Logger

```sql
-- คำตอบ
CREATE TABLE deadlock_events (
    event_id    SERIAL PRIMARY KEY,
    session_pid INT NOT NULL,
    db_name     TEXT DEFAULT current_database(),
    sql_state   CHAR(5),
    error_msg   TEXT,
    context     TEXT,
    lock_info   JSONB,
    occurred_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE OR REPLACE FUNCTION log_deadlock_event(
    p_context   TEXT DEFAULT NULL
) RETURNS INT AS $$
DECLARE
    v_event_id INT;
BEGIN
    INSERT INTO deadlock_events (
        session_pid, sql_state, error_msg, context, lock_info
    )
    VALUES (
        pg_backend_pid(),
        SQLSTATE,
        SQLERRM,
        p_context,
        (
            SELECT jsonb_agg(jsonb_build_object(
                'locktype', locktype,
                'relation', CASE WHEN relation IS NOT NULL 
                                 THEN relation::regclass::TEXT ELSE NULL END,
                'mode', mode,
                'granted', granted,
                'pid', pid
            ))
            FROM pg_locks
            WHERE pid IN (
                SELECT pid FROM pg_stat_activity WHERE state = 'active'
            )
        )
    )
    RETURNING event_id INTO v_event_id;
    
    RETURN v_event_id;
END;
$$ LANGUAGE plpgsql;

-- ใช้ใน exception handler:
-- EXCEPTION WHEN deadlock_detected THEN
--     PERFORM log_deadlock_event('transfer operation');
--     RAISE;
```

### แบบฝึกหัดที่ 8: Deadlock-Free Queue Processor

```sql
-- คำตอบ
CREATE TABLE processing_queue (
    task_id     SERIAL PRIMARY KEY,
    priority    INT DEFAULT 5,
    payload     JSONB,
    status      VARCHAR(20) DEFAULT 'pending',
    worker_pid  INT,
    created_at  TIMESTAMPTZ DEFAULT NOW(),
    started_at  TIMESTAMPTZ,
    completed_at TIMESTAMPTZ,
    attempts    INT DEFAULT 0,
    error_msg   TEXT
);

CREATE OR REPLACE FUNCTION process_queue_batch(
    p_worker_id INT,
    p_max_tasks INT DEFAULT 20
) RETURNS TABLE(
    task_id INT,
    status  TEXT,
    message TEXT
) AS $$
DECLARE
    r RECORD;
BEGIN
    -- SKIP LOCKED ensures no deadlocks in queue processing
    FOR r IN
        SELECT t.task_id, t.payload, t.attempts
        FROM processing_queue t
        WHERE t.status = 'pending'
          AND t.attempts < 3  -- Max 3 attempts
        ORDER BY t.priority DESC, t.created_at
        LIMIT p_max_tasks
        FOR UPDATE SKIP LOCKED
    LOOP
        -- Claim task
        UPDATE processing_queue SET
            status = 'processing',
            worker_pid = p_worker_id,
            started_at = NOW(),
            attempts = attempts + 1
        WHERE task_id = r.task_id;
        
        BEGIN
            -- Process task (simulated)
            PERFORM pg_sleep(0.001);  -- Simulated work
            
            UPDATE processing_queue SET
                status = 'completed',
                completed_at = NOW()
            WHERE task_id = r.task_id;
            
            task_id := r.task_id;
            status  := 'completed';
            message := 'Processed successfully';
            RETURN NEXT;
            
        EXCEPTION WHEN OTHERS THEN
            UPDATE processing_queue SET
                status = CASE WHEN r.attempts >= 2 THEN 'failed' ELSE 'pending' END,
                error_msg = SQLERRM,
                started_at = NULL
            WHERE task_id = r.task_id;
            
            task_id := r.task_id;
            status  := 'error';
            message := SQLERRM;
            RETURN NEXT;
        END;
    END LOOP;
END;
$$ LANGUAGE plpgsql;

BEGIN;
SELECT * FROM process_queue_batch(pg_backend_pid(), 5);
COMMIT;
```

### แบบฝึกหัดที่ 9: Deadlock Simulation and Prevention Test

```sql
-- คำตอบ
CREATE OR REPLACE FUNCTION test_deadlock_prevention()
RETURNS TABLE(
    test_name TEXT,
    result    TEXT,
    note      TEXT
) AS $$
DECLARE
    v_start TIMESTAMPTZ;
BEGIN
    -- Test 1: Consistent ordering
    test_name := 'Consistent Lock Ordering';
    v_start := clock_timestamp();
    
    BEGIN
        -- Simulate ordered locks
        SET LOCAL lock_timeout = '100ms';
        PERFORM * FROM dl_exercise WHERE id = 1 FOR UPDATE;
        PERFORM * FROM dl_exercise WHERE id = 2 FOR UPDATE;
        
        RELEASE SAVEPOINT undefined;  -- ไม่มี savepoint ที่นี่
    EXCEPTION
        WHEN undefined_object THEN NULL;
        WHEN lock_not_available THEN
            result := 'TIMEOUT (expected in contention)';
            note   := 'Lock ordering works but had contention';
            RETURN NEXT;
    END;
    
    result := 'PASS';
    note   := 'Consistent ordering prevents deadlock';
    RETURN NEXT;
    
    -- Test 2: Deadlock timeout setting
    test_name := 'Deadlock Timeout Config';
    result := current_setting('deadlock_timeout');
    note   := 'Lower = faster detection, Higher = less overhead';
    RETURN NEXT;
    
    -- Test 3: Advisory lock prevention
    test_name := 'Advisory Lock Prevention';
    BEGIN
        IF pg_try_advisory_xact_lock(99999) THEN
            result := 'PASS - exclusive access acquired';
            note   := 'Advisory locks can prevent deadlocks for coordination';
        ELSE
            result := 'OCCUPIED - lock already held';
            note   := 'Another session holds this advisory lock';
        END IF;
    END;
    RETURN NEXT;
END;
$$ LANGUAGE plpgsql;

BEGIN;
SELECT * FROM test_deadlock_prevention();
COMMIT;
```

### แบบฝึกหัดที่ 10: Production Deadlock Dashboard

```sql
-- คำตอบ
CREATE OR REPLACE VIEW deadlock_dashboard AS
WITH dl_stats AS (
    SELECT 
        deadlocks,
        xact_commit + xact_rollback AS total_transactions,
        stats_reset
    FROM pg_stat_database
    WHERE datname = current_database()
),
current_waits AS (
    SELECT 
        COUNT(*) AS waiting_count,
        MAX(EXTRACT(EPOCH FROM (NOW() - waitstart))) AS max_wait_seconds
    FROM pg_locks
    WHERE NOT granted
),
long_waits AS (
    SELECT COUNT(*) AS long_wait_count
    FROM pg_locks l
    JOIN pg_stat_activity a ON l.pid = a.pid
    WHERE NOT l.granted 
      AND l.waitstart < NOW() - INTERVAL '30 seconds'
)
SELECT
    s.deadlocks AS total_deadlocks_since_reset,
    ROUND(s.deadlocks::NUMERIC / NULLIF(s.total_transactions, 0) * 10000, 4) 
        AS deadlocks_per_10k_transactions,
    w.waiting_count AS currently_waiting,
    ROUND(w.max_wait_seconds, 2) AS max_wait_seconds,
    l.long_wait_count AS long_waits_over_30s,
    current_setting('deadlock_timeout') AS deadlock_timeout,
    current_setting('lock_timeout') AS lock_timeout,
    s.stats_reset AS stats_since
FROM dl_stats s
CROSS JOIN current_waits w
CROSS JOIN long_waits l;

SELECT * FROM deadlock_dashboard;

-- คำแนะนำ:
-- deadlocks_per_10k_transactions > 0.5 → ต้องการ investigation
-- max_wait_seconds > 30 → long-running blocker
-- long_waits_over_30s > 0 → critical issue
```

---

## สรุป

ใน Part นี้เราได้เรียนรู้:

1. **สาเหตุของ Deadlock**: Circular wait - T1 รอ T2, T2 รอ T1
2. **Deadlock Detection**: PostgreSQL ใช้ wait-for graph และ deadlock_timeout
3. **Error Messages**: SQLSTATE 40P01 สำหรับ deadlock
4. **Prevention Strategies**:
   - **Consistent Lock Ordering**: Lock resources ตามลำดับเดียวกันเสมอ
   - **lock_timeout**: Fail fast แทนรอ
   - **Retry Logic**: Exponential backoff กับ jitter
   - **Minimize Lock Hold Time**: ทำ transaction ให้สั้น
   - **SKIP LOCKED**: ป้องกัน deadlock ใน queue processing
5. **Monitoring**: Queries สำหรับ detect และ alert deadlocks
6. **Best Practices**: ป้องกัน deadlock ตั้งแต่ design ระบบ

ใน Part 77 เราจะเรียนรู้เรื่อง MVCC (Multi-Version Concurrency Control) ซึ่งเป็น mechanism สำคัญที่ PostgreSQL ใช้จัดการ concurrent access
