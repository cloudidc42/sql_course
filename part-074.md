# Part 74: Isolation Levels

## บทนำ

**Isolation Level** คือระดับที่กำหนดว่า transaction หนึ่งสามารถ "เห็น" การเปลี่ยนแปลงจาก transaction อื่นที่กำลังทำงานพร้อมกันได้มากแค่ไหน การเลือก isolation level ที่เหมาะสมเป็นการ trade-off ระหว่าง **ความถูกต้องของข้อมูล** และ **ประสิทธิภาพ**

---

## 1. ปัญหา Concurrency ที่เกิดได้

### 1.1 Dirty Read

**Dirty Read** เกิดขึ้นเมื่อ transaction อ่านข้อมูลที่ transaction อื่นเปลี่ยนแปลงไปแล้วแต่ยังไม่ commit

```
ลำดับเวลา:
T1:  BEGIN
T1:  UPDATE accounts SET balance = 9999 WHERE id = 1;  -- ยังไม่ commit
       T2:  BEGIN
       T2:  SELECT balance FROM accounts WHERE id = 1;  -- อ่านได้ 9999!
       T2:  COMMIT
T1:  ROLLBACK  -- ค่ากลับเป็นเดิม แต่ T2 ได้ข้อมูลผิดไปแล้ว

ปัญหา: T2 ใช้ค่า 9999 ในการคำนวณ แต่ค่าจริงไม่เคยเป็น 9999 เลย
```

```sql
-- สาธิต Dirty Read ใน MySQL (READ UNCOMMITTED)
-- Session 1:
SET SESSION TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;
START TRANSACTION;
UPDATE accounts SET balance = 99999 WHERE account_id = 1;
-- ยังไม่ COMMIT

-- Session 2 (READ UNCOMMITTED):
SET SESSION TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;
SELECT balance FROM accounts WHERE account_id = 1;
-- อาจเห็น 99999 (dirty data!)

-- Session 1:
ROLLBACK;
-- ค่ากลับเป็นเดิม แต่ Session 2 ได้ข้อมูลผิดไปแล้ว

-- PostgreSQL จะไม่เกิด Dirty Read แม้จะตั้ง READ UNCOMMITTED
-- เพราะ PostgreSQL ไม่ implement READ UNCOMMITTED จริงๆ
```

### 1.2 Non-Repeatable Read

**Non-Repeatable Read** เกิดขึ้นเมื่อ transaction อ่านข้อมูลเดิมสองครั้งแล้วได้ค่าต่างกัน เพราะ transaction อื่น update/delete ระหว่างนั้น

```
ลำดับเวลา:
T1:  BEGIN
T1:  SELECT price FROM products WHERE id = 1;  -- ได้ 100
       T2:  BEGIN
       T2:  UPDATE products SET price = 200 WHERE id = 1;
       T2:  COMMIT
T1:  SELECT price FROM products WHERE id = 1;  -- ได้ 200 (เปลี่ยนไปแล้ว!)
T1:  COMMIT

ปัญหา: T1 ใช้ราคา 100 สำหรับการคำนวณในครั้งแรก
       แต่พอตรวจสอบอีกครั้งได้ 200 → ข้อมูลไม่สอดคล้อง
```

```sql
-- สาธิต Non-Repeatable Read ใน MySQL READ COMMITTED
-- Session 1:
SET SESSION TRANSACTION ISOLATION LEVEL READ COMMITTED;
START TRANSACTION;
SELECT price FROM products WHERE product_id = 101;  -- ได้ 100

-- Session 2:
START TRANSACTION;
UPDATE products SET price = 200 WHERE product_id = 101;
COMMIT;

-- Session 1: อ่านอีกครั้ง
SELECT price FROM products WHERE product_id = 101;  -- ได้ 200 แล้ว!

-- ใน REPEATABLE READ จะยังเห็น 100 เสมอในการอ่านครั้งที่สอง
```

### 1.3 Phantom Read

**Phantom Read** เกิดขึ้นเมื่อ transaction อ่านชุดของ rows สองครั้งแล้วได้จำนวน rows ต่างกัน เพราะ transaction อื่น insert/delete rows ระหว่างนั้น

```
ลำดับเวลา:
T1:  BEGIN
T1:  SELECT COUNT(*) FROM orders WHERE amount > 1000;  -- ได้ 5
       T2:  BEGIN
       T2:  INSERT INTO orders (amount) VALUES (5000);
       T2:  COMMIT
T1:  SELECT COUNT(*) FROM orders WHERE amount > 1000;  -- ได้ 6 (phantom row!)
T1:  COMMIT

ปัญหา: T1 เห็นแถวใหม่ที่ไม่มีตอนเริ่ม transaction
```

```sql
-- สาธิต Phantom Read
-- Session 1 (REPEATABLE READ):
SET SESSION TRANSACTION ISOLATION LEVEL REPEATABLE READ;
START TRANSACTION;
SELECT COUNT(*) FROM large_orders WHERE amount > 1000;  -- ได้ 5

-- Session 2:
START TRANSACTION;
INSERT INTO large_orders (customer_id, amount) VALUES (1, 5000);
COMMIT;

-- Session 1: COUNT อีกครั้ง
-- MySQL REPEATABLE READ: ยังได้ 5 (MVCC snapshot)
-- PostgreSQL REPEATABLE READ: ยังได้ 5 (MVCC snapshot)
-- MySQL/PostgreSQL READ COMMITTED: ได้ 6 (phantom!)
```

### 1.4 Lost Update

**Lost Update** เกิดขึ้นเมื่อสอง transactions อ่านค่าเดิม แก้ไข แล้ว write กลับ → transaction หนึ่งจะ "ทับ" การแก้ไขของอีก transaction

```
ลำดับเวลา:
T1:  READ balance = 1000
       T2:  READ balance = 1000
T1:  balance += 100  → balance = 1100
       T2:  balance += 200  → balance = 1200
T1:  WRITE balance = 1100
       T2:  WRITE balance = 1200  ← ทับ T1!

ผล: balance = 1200 แทนที่จะเป็น 1300 (สูญเสีย +100 ของ T1)
```

```sql
-- สาธิต Lost Update
-- Session 1:
BEGIN;
SELECT balance INTO v_balance FROM accounts WHERE id = 1;  -- v_balance = 1000
-- รอสักครู่ (Session 2 ทำงาน)
UPDATE accounts SET balance = v_balance + 100 WHERE id = 1;  -- SET 1100
COMMIT;

-- Session 2 (ทำก่อน Session 1 commit):
BEGIN;
SELECT balance INTO v_balance FROM accounts WHERE id = 1;  -- v_balance = 1000
UPDATE accounts SET balance = v_balance + 200 WHERE id = 1;  -- SET 1200
COMMIT;

-- ผล: balance = 1200 (ไม่ใช่ 1300!)
-- T1's +100 สูญหาย

-- วิธีแก้: ใช้ atomic update
BEGIN;
UPDATE accounts SET balance = balance + 100 WHERE id = 1;  -- ไม่ read ก่อน
COMMIT;
```

---

## 2. Isolation Levels ทั้ง 4 ระดับ

### ตารางเปรียบเทียบ

```
+--------------------+-------------+--------------------+---------------+
| Isolation Level    | Dirty Read  | Non-Repeatable    | Phantom Read  |
+--------------------+-------------+--------------------+---------------+
| READ UNCOMMITTED   | Possible    | Possible           | Possible      |
| READ COMMITTED     | Prevented   | Possible           | Possible      |
| REPEATABLE READ    | Prevented   | Prevented          | Possible*     |
| SERIALIZABLE       | Prevented   | Prevented          | Prevented     |
+--------------------+-------------+--------------------+---------------+
* MySQL/PostgreSQL ป้องกัน Phantom Read ใน REPEATABLE READ ด้วย MVCC
```

---

## 3. READ UNCOMMITTED

ระดับนี้อนุญาตให้อ่านข้อมูลที่ยังไม่ commit (**Dirty Reads**)

```sql
-- PostgreSQL: ตั้งค่า (แต่ทำงานเหมือน READ COMMITTED)
BEGIN TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;
    SELECT * FROM accounts;
COMMIT;

-- MySQL: ตั้งค่าจริง
SET SESSION TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;
START TRANSACTION;
    SELECT balance FROM accounts WHERE account_id = 1;
    -- อาจเห็นข้อมูลที่ยังไม่ commit จาก transaction อื่น!
COMMIT;

-- เมื่อไรที่ใช้ READ UNCOMMITTED?
-- - รายงานที่ approximate ได้ (เช่น ดู dashboard ข้อมูลคร่าวๆ)
-- - ระบบที่ความเร็วสำคัญกว่าความแม่นยำ
-- - Debug/monitoring ที่ต้องการดู uncommitted changes

-- ตัวอย่าง use case ที่ยอมรับได้:
-- ดู approximate row count สำหรับ large table
BEGIN TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;
    SELECT schemaname, tablename, n_live_tup
    FROM pg_stat_user_tables
    ORDER BY n_live_tup DESC;
COMMIT;
```

### Transaction Timeline Diagram

```
READ UNCOMMITTED สาธิต:

Time →
T1:  [BEGIN]--[UPDATE bal=9999]------[ROLLBACK]
T2:          [BEGIN]--[SELECT bal]--[COMMIT]
                           ↓
                      sees 9999 (DIRTY!)
                      
ปัญหา: T2 ใช้ค่า 9999 ที่ไม่เคยมีจริง
```

---

## 4. READ COMMITTED

ระดับนี้อนุญาตให้อ่านเฉพาะข้อมูลที่ commit แล้ว ป้องกัน Dirty Reads แต่ยังเกิด Non-Repeatable Reads ได้

```sql
-- PostgreSQL: ค่าเริ่มต้น
SHOW transaction_isolation;  -- read committed

BEGIN TRANSACTION ISOLATION LEVEL READ COMMITTED;
    -- อ่านครั้งแรก
    SELECT balance FROM accounts WHERE account_id = 1;  -- 1000
    
    -- ระหว่างนี้ transaction อื่น UPDATE และ COMMIT
    
    -- อ่านครั้งที่สอง
    SELECT balance FROM accounts WHERE account_id = 1;  -- อาจเห็น 1500 (Non-Repeatable Read!)
COMMIT;

-- MySQL: ตั้งค่า
SET SESSION TRANSACTION ISOLATION LEVEL READ COMMITTED;
START TRANSACTION;
    SELECT price FROM products WHERE product_id = 101;  -- 100
    
    -- Transaction อื่น UPDATE price เป็น 200 และ COMMIT
    
    SELECT price FROM products WHERE product_id = 101;  -- เห็น 200 (Non-Repeatable!)
COMMIT;

-- ตัวอย่างปัญหา: report ที่คำนวณ total สองครั้ง
BEGIN TRANSACTION ISOLATION LEVEL READ COMMITTED;
    -- First pass: count orders
    SELECT COUNT(*) FROM orders WHERE created_at::DATE = TODAY;  -- 100 orders
    
    -- Transaction อื่น insert 5 orders ใหม่
    
    -- Second pass: sum amounts (ของ "same" 100 orders)
    SELECT SUM(amount) FROM orders WHERE created_at::DATE = TODAY;  -- total ของ 105 orders!
    -- ข้อมูลไม่สอดคล้องกัน!
COMMIT;
```

### Transaction Timeline: READ COMMITTED

```
READ COMMITTED สาธิต:

Time →
T1:  [BEGIN]--[SELECT price=100]------[SELECT price=200]--[COMMIT]
T2:                     [BEGIN]--[UPDATE price=200]--[COMMIT]
                                                ↑
                                    T1 เห็น 200 ในการอ่านครั้งที่สอง
                                    (Non-Repeatable Read)
```

---

## 5. REPEATABLE READ

ระดับนี้รับประกันว่าข้อมูลที่อ่านไปแล้วจะไม่เปลี่ยนแปลงตลอด transaction ป้องกัน Dirty Reads และ Non-Repeatable Reads

```sql
-- PostgreSQL: REPEATABLE READ
BEGIN TRANSACTION ISOLATION LEVEL REPEATABLE READ;
    -- อ่านครั้งแรก - ได้ snapshot
    SELECT balance FROM accounts WHERE account_id = 1;  -- 1000
    
    -- Transaction อื่น UPDATE เป็น 1500 และ COMMIT
    
    -- อ่านครั้งที่สอง - ยังเห็น snapshot เดิม!
    SELECT balance FROM accounts WHERE account_id = 1;  -- ยังเห็น 1000
COMMIT;

-- แต่ถ้าพยายาม UPDATE ข้อมูลที่ถูก UPDATE โดย transaction อื่นแล้ว
BEGIN TRANSACTION ISOLATION LEVEL REPEATABLE READ;
    SELECT balance FROM accounts WHERE account_id = 1;  -- 1000 (snapshot)
    
    -- Transaction อื่น UPDATE เป็น 1500 และ COMMIT
    
    -- พยายาม UPDATE เอง
    UPDATE accounts SET balance = balance + 100 WHERE account_id = 1;
    -- PostgreSQL: จะ block จนกว่า T2 จะ COMMIT แล้วอ่านค่าใหม่
    -- หรือใน SERIALIZABLE: อาจเกิด serialization error
COMMIT;

-- MySQL REPEATABLE READ (ค่าเริ่มต้น)
-- MySQL ใช้ Next-Key Locking ป้องกัน Phantom Reads ใน REPEATABLE READ
-- ซึ่งต่างจาก PostgreSQL ที่ใช้ MVCC
SET SESSION TRANSACTION ISOLATION LEVEL REPEATABLE READ;
START TRANSACTION;
    SELECT COUNT(*) FROM orders WHERE amount > 1000;  -- 5
    
    -- Transaction อื่น INSERT order ใหม่และ COMMIT
    
    SELECT COUNT(*) FROM orders WHERE amount > 1000;  -- ยังเห็น 5 (snapshot!)
COMMIT;
```

### Transaction Timeline: REPEATABLE READ

```
REPEATABLE READ สาธิต:

Time →
T1:  [BEGIN, snapshot@T1]--[SELECT bal=1000]--[SELECT bal=1000]--[COMMIT]
T2:                [BEGIN]--[UPDATE bal=1500]--[COMMIT]
                                    ↑
                     T1 ยังเห็น 1000 (จาก snapshot)
                     เพราะ MVCC ใช้ snapshot ตั้งแต่ BEGIN
```

---

## 6. SERIALIZABLE

ระดับสูงสุด รับประกันว่า transactions ทำงานราวกับว่าทำทีละ transaction (แบบ serial) ป้องกัน Dirty Reads, Non-Repeatable Reads, และ Phantom Reads ทั้งหมด

```sql
-- PostgreSQL SERIALIZABLE (SSI - Serializable Snapshot Isolation)
BEGIN TRANSACTION ISOLATION LEVEL SERIALIZABLE;
    SELECT SUM(balance) FROM accounts WHERE branch_id = 1;  -- 100,000
    
    -- Transaction อื่น transfer เงินระหว่าง branches
    
    SELECT SUM(balance) FROM accounts WHERE branch_id = 2;  -- 200,000
    
    -- ถ้า Transaction อื่น commit ก่อนและ conflict กับ read pattern นี้:
    -- ERROR: could not serialize access due to read/write dependencies
    -- Transaction ต้อง retry!
COMMIT;

-- ตัวอย่างที่ SERIALIZABLE ป้องกัน Phantom Read:
BEGIN TRANSACTION ISOLATION LEVEL SERIALIZABLE;
    SELECT * FROM time_slots WHERE doctor_id = 1 AND slot_time = '09:00';
    -- ไม่มีใครจอง ดูว่าว่าง
    
    -- Transaction อื่น: patient 2 จอง slot 09:00 และ COMMIT
    
    -- ถ้า T1 ก็จะจอง slot 09:00 เช่นกัน:
    INSERT INTO appointments (doctor_id, patient_id, slot_time)
    VALUES (1, 1, '09:00');
    
    -- SERIALIZABLE จะ detect ว่ามี conflict และ raise error:
    -- ERROR: could not serialize access due to read/write dependencies
COMMIT;

-- การจัดการ Serialization Failure
DO $$
DECLARE
    max_retries CONSTANT INT := 3;
    attempts INT := 0;
    success BOOLEAN := FALSE;
BEGIN
    WHILE NOT success AND attempts < max_retries LOOP
        attempts := attempts + 1;
        
        BEGIN
            -- ตั้ง isolation level
            SET LOCAL transaction_isolation TO 'serializable';
            
            -- ทำ critical operations
            PERFORM SUM(balance) FROM accounts WHERE branch_id = 1;
            INSERT INTO transfers VALUES (...);
            
            success := TRUE;
            
        EXCEPTION
            WHEN serialization_failure THEN
                RAISE NOTICE 'Serialization failure, retry %', attempts;
                -- Exponential backoff
                PERFORM pg_sleep(0.1 * attempts);
            WHEN OTHERS THEN
                RAISE;  -- ไม่ retry
        END;
    END LOOP;
    
    IF NOT success THEN
        RAISE EXCEPTION 'Could not complete after % retries', max_retries;
    END IF;
END;
$$;
```

### Transaction Timeline: SERIALIZABLE

```
SERIALIZABLE สาธิต (Phantom Read Prevention):

Time →
T1:  [BEGIN SERIALIZABLE]--[SELECT slots for 09:00 = EMPTY]--[INSERT appt]--[ERROR!]
T2:        [BEGIN SERIALIZABLE]--[SELECT slots for 09:00 = EMPTY]--[INSERT appt]--[COMMIT]
                                                                          ↑
                                                              T2 COMMIT ก่อน T1
                                                              T1 detect conflict → ERROR
                                                              
ผล: T2 ได้ appointment, T1 ต้อง retry
ข้อมูลถูกต้อง: ไม่มี double booking
```

---

## 7. Snapshot Isolation (MVCC)

PostgreSQL ใช้ **MVCC (Multi-Version Concurrency Control)** ซึ่งเป็นรูปแบบของ Snapshot Isolation

```sql
-- MVCC ทำงานอย่างไร:
-- แทนที่จะ lock data สำหรับ reads
-- PostgreSQL สร้าง "snapshot" ของข้อมูล ณ เวลาที่ transaction เริ่ม
-- Readers ไม่ block Writers, Writers ไม่ block Readers

-- ดู system columns ที่ MVCC ใช้
SELECT xmin, xmax, ctid, * FROM accounts LIMIT 5;
-- xmin: transaction ID ที่สร้าง row นี้
-- xmax: transaction ID ที่ลบ/update row นี้ (0 = ยังมีอยู่)
-- ctid: physical location ของ row

-- เมื่อ UPDATE เกิดขึ้น:
BEGIN;
UPDATE accounts SET balance = 2000 WHERE account_id = 1;

-- ตรวจสอบ:
SELECT xmin, xmax, balance FROM accounts WHERE account_id = 1;
-- xmin = current_txn_id, xmax = 0  ← new version
-- นอกจากนี้ยังมี old version: xmin = old_txn_id, xmax = current_txn_id

COMMIT;

-- ดู transaction ID ปัจจุบัน
SELECT txid_current();

-- ดู transaction snapshot ปัจจุบัน
SELECT txid_current_snapshot();
-- ตัวอย่าง: 1000:1005:1003 
-- หมายถึง: transactions ที่ active คือ 1003, xmin=1000, xmax=1005

-- Row visibility:
-- Row จะ visible ถ้า xmin <= current_snapshot AND xmax = 0 หรือ xmax > current_snapshot
```

---

## 8. Default Isolation Levels ตามฐานข้อมูล

```sql
-- PostgreSQL: READ COMMITTED (ค่าเริ่มต้น)
SHOW transaction_isolation;
-- read committed

-- ตรวจสอบและตั้งค่า default
SELECT current_setting('default_transaction_isolation');
-- ตั้งค่าใน postgresql.conf:
-- default_transaction_isolation = 'read committed'

-- ตั้งค่าสำหรับ session:
SET SESSION CHARACTERISTICS AS TRANSACTION ISOLATION LEVEL REPEATABLE READ;

-- ตั้งค่าสำหรับ transaction เดียว:
BEGIN TRANSACTION ISOLATION LEVEL SERIALIZABLE;

-- MySQL: REPEATABLE READ (ค่าเริ่มต้น)
SHOW VARIABLES LIKE 'transaction_isolation';
-- REPEATABLE-READ

-- ตั้งค่า:
SET SESSION TRANSACTION ISOLATION LEVEL READ COMMITTED;
SET GLOBAL TRANSACTION ISOLATION LEVEL SERIALIZABLE;

-- SQL Server: READ COMMITTED (ค่าเริ่มต้น)
SELECT transaction_isolation_level FROM sys.dm_exec_sessions 
WHERE session_id = @@SPID;
-- 2 = READ COMMITTED

-- ตั้งค่า:
SET TRANSACTION ISOLATION LEVEL SNAPSHOT;

-- Oracle: READ COMMITTED (ค่าเริ่มต้น)
-- Oracle รองรับแค่ READ COMMITTED และ SERIALIZABLE (ไม่มี READ UNCOMMITTED)

-- SQLite: SERIALIZABLE (ค่าเริ่มต้น ผ่าน file locking)
PRAGMA read_uncommitted = true;  -- เปิด READ UNCOMMITTED ใน WAL mode
```

---

## 9. Setting Isolation Levels

```sql
-- PostgreSQL: วิธีต่างๆ ในการตั้งค่า

-- 1. Per-transaction
BEGIN;
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;
-- หรือ
BEGIN TRANSACTION ISOLATION LEVEL SERIALIZABLE;

-- 2. Per-session
SET SESSION CHARACTERISTICS AS TRANSACTION ISOLATION LEVEL REPEATABLE READ;

-- 3. Per-database (ตั้งใน postgresql.conf หรือ ALTER)
ALTER DATABASE mydb SET default_transaction_isolation TO 'serializable';

-- 4. Per-role
ALTER ROLE reporting_user SET default_transaction_isolation TO 'repeatable read';

-- ตรวจสอบ
SHOW transaction_isolation;  -- สำหรับ current transaction
SELECT current_setting('transaction_isolation');

-- ตั้งค่า และ reset
SET transaction_isolation = 'serializable';
RESET transaction_isolation;

-- MySQL: วิธีต่างๆ
-- 1. Per-session
SET SESSION TRANSACTION ISOLATION LEVEL READ COMMITTED;

-- 2. Per-transaction
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;
START TRANSACTION;
-- ...

-- 3. Global (ต้องมี SUPER privilege)
SET GLOBAL TRANSACTION ISOLATION LEVEL REPEATABLE READ;

-- ตรวจสอบ
SELECT @@SESSION.transaction_isolation;
SELECT @@GLOBAL.transaction_isolation;

-- SQL Server:
-- 1. Per-session
SET TRANSACTION ISOLATION LEVEL READ COMMITTED;

-- 2. Per-database
ALTER DATABASE dbname SET READ_COMMITTED_SNAPSHOT ON;
```

---

## 10. Performance vs Safety Tradeoff

```sql
-- Benchmark: ผลกระทบของ Isolation Level ต่อ performance

-- Setup
CREATE TABLE perf_test (
    id      SERIAL PRIMARY KEY,
    value   INT,
    updated TIMESTAMPTZ DEFAULT NOW()
);

INSERT INTO perf_test (value) 
SELECT generate_series(1, 100000);

-- Test READ COMMITTED (default)
\timing
BEGIN TRANSACTION ISOLATION LEVEL READ COMMITTED;
SELECT COUNT(*), SUM(value), AVG(value) FROM perf_test;
COMMIT;

-- Test REPEATABLE READ
BEGIN TRANSACTION ISOLATION LEVEL REPEATABLE READ;
SELECT COUNT(*), SUM(value), AVG(value) FROM perf_test;
COMMIT;

-- Test SERIALIZABLE
BEGIN TRANSACTION ISOLATION LEVEL SERIALIZABLE;
SELECT COUNT(*), SUM(value), AVG(value) FROM perf_test;
COMMIT;

-- สังเกต: SERIALIZABLE อาจช้ากว่าเมื่อมี concurrent transactions มาก
-- เพราะต้องตรวจสอบ serialization conflicts

-- Throughput comparison (concurrent writes)
-- READ COMMITTED: highest throughput
-- REPEATABLE READ: medium throughput
-- SERIALIZABLE: lowest throughput (conflict detection overhead)

-- แนะนำ:
-- READ COMMITTED: ใช้เป็น default สำหรับ OLTP workloads
-- REPEATABLE READ: ใช้สำหรับ long-running reports
-- SERIALIZABLE: ใช้เมื่อต้องการ correctness สูงสุด (ยอมรับ retries)

-- ตัวอย่าง: Web application ที่เหมาะสม
-- OLTP (e-commerce, banking): READ COMMITTED + careful locking
-- Analytics reports: REPEATABLE READ (snapshot-based)
-- Financial reconciliation: SERIALIZABLE
-- Social media (likes, views): READ COMMITTED (performance > accuracy)
```

---

## 11. การสาธิต Isolation Level ด้วย ASCII Diagrams

### สาธิต Non-Repeatable Read ใน READ COMMITTED

```sql
-- สร้าง demo script
-- Session A (ต้องรันใน terminal แยก):
-- \set AUTOCOMMIT off

-- Step 1: Session A - start transaction
-- BEGIN TRANSACTION ISOLATION LEVEL READ COMMITTED;
-- SELECT balance FROM accounts WHERE account_id = 1;
-- (รอ Session B ทำงาน)

-- Step 2: Session B - update and commit
-- BEGIN;
-- UPDATE accounts SET balance = 5000 WHERE account_id = 1;
-- COMMIT;

-- Step 3: Session A - read again
-- SELECT balance FROM accounts WHERE account_id = 1;  -- เห็นค่าใหม่!
-- COMMIT;

/*
Timeline:
t=0  Session A: BEGIN (READ COMMITTED)
t=1  Session A: SELECT balance → 1000
t=2  Session B: UPDATE balance = 5000; COMMIT
t=3  Session A: SELECT balance → 5000 (NON-REPEATABLE!)
t=4  Session A: COMMIT

ปัญหาที่เกิด:
Session A เห็นค่าที่ต่างกันในการอ่านสองครั้ง
แม้อยู่ใน transaction เดียวกัน
*/

-- สาธิตใน PostgreSQL (ใน single session)
DO $$
DECLARE
    v_initial DECIMAL;
    v_after   DECIMAL;
BEGIN
    -- จำลอง Session A อ่านครั้งแรก
    SELECT balance INTO v_initial FROM accounts WHERE account_id = 1;
    RAISE NOTICE 'Initial read: %', v_initial;
    
    -- จำลอง Session B update (ใน real world นี่จะเป็น process แยก)
    -- ที่นี่เราใช้ dblink หรือ pg_background ถ้าต้องการ simulate จริงๆ
    
    -- อ่านอีกครั้ง
    SELECT balance INTO v_after FROM accounts WHERE account_id = 1;
    RAISE NOTICE 'Second read: %', v_after;
    
    IF v_initial <> v_after THEN
        RAISE NOTICE 'Non-Repeatable Read detected!';
    ELSE
        RAISE NOTICE 'No Non-Repeatable Read (same value)';
    END IF;
END;
$$;
```

### สาธิต Phantom Read

```sql
-- Phantom Read demonstration
CREATE TABLE appointments (
    appt_id   SERIAL PRIMARY KEY,
    doctor_id INT NOT NULL,
    patient_id INT NOT NULL,
    slot_date DATE NOT NULL,
    slot_hour INT NOT NULL CHECK (slot_hour BETWEEN 8 AND 17),
    UNIQUE(doctor_id, slot_date, slot_hour)
);

INSERT INTO appointments VALUES
    (DEFAULT, 1, 101, '2024-01-15', 9),
    (DEFAULT, 1, 102, '2024-01-15', 10),
    (DEFAULT, 1, 103, '2024-01-15', 14);

/*
Phantom Read Timeline (READ COMMITTED):

t=0  Session A: BEGIN (READ COMMITTED)
t=1  Session A: SELECT COUNT(*) FROM appointments 
                WHERE doctor_id = 1 AND slot_date = '2024-01-15'
                → COUNT = 3

t=2  Session B: BEGIN
t=3  Session B: INSERT INTO appointments VALUES (DEFAULT, 1, 104, '2024-01-15', 11)
t=4  Session B: COMMIT

t=5  Session A: SELECT COUNT(*) FROM appointments 
                WHERE doctor_id = 1 AND slot_date = '2024-01-15'
                → COUNT = 4 (PHANTOM ROW!)

ปัญหา: Session A ตัดสินใจโดยอิงจาก COUNT = 3
       แต่พอดำเนินงานต่อ COUNT = 4
       อาจทำให้สั่งซื้ออุปกรณ์ไม่ครบ หรือกำหนดเวลาผิด
*/

-- ทดสอบด้วย Repeatable Read:
BEGIN TRANSACTION ISOLATION LEVEL REPEATABLE READ;
    SELECT slot_hour FROM appointments WHERE doctor_id = 1 AND slot_date = '2024-01-15';
    -- ได้ 9, 10, 14
    
    -- Session B insert slot 11 และ commit
    
    SELECT slot_hour FROM appointments WHERE doctor_id = 1 AND slot_date = '2024-01-15';
    -- PostgreSQL REPEATABLE READ: ยังเห็น 9, 10, 14 (ไม่เห็น 11)
    -- MySQL REPEATABLE READ: ยังเห็น 9, 10, 14 (ด้วย Next-Key Lock)
COMMIT;
```

---

## 12. ตัวอย่าง Setting Isolation Levels ในแต่ละ Database

```sql
-- PostgreSQL: Complete examples
-- สำหรับ report ที่ต้องการ consistent snapshot
BEGIN TRANSACTION ISOLATION LEVEL REPEATABLE READ READ ONLY;
    -- ทุก query ใช้ snapshot เดียวกัน
    SELECT SUM(amount) FROM sales WHERE sale_date = CURRENT_DATE;
    SELECT COUNT(*) FROM new_customers WHERE created_at::DATE = CURRENT_DATE;
    SELECT * FROM inventory WHERE quantity < reorder_point;
COMMIT;

-- สำหรับ critical financial operations
BEGIN TRANSACTION ISOLATION LEVEL SERIALIZABLE;
    -- Transfer ระหว่าง accounts
    DECLARE
        v_from_balance DECIMAL;
        v_transfer_amount DECIMAL := 1000;
    BEGIN
        SELECT balance INTO v_from_balance 
        FROM accounts WHERE account_id = 1 FOR UPDATE;
        
        IF v_from_balance < v_transfer_amount THEN
            RAISE EXCEPTION 'Insufficient funds';
        END IF;
        
        UPDATE accounts SET balance = balance - v_transfer_amount WHERE account_id = 1;
        UPDATE accounts SET balance = balance + v_transfer_amount WHERE account_id = 2;
    END;
COMMIT;

-- MySQL: Set for session
SET SESSION TRANSACTION ISOLATION LEVEL SERIALIZABLE;
-- ตอนนี้ทุก transaction ใน session นี้จะเป็น SERIALIZABLE

-- MySQL: Set for next transaction only
SET TRANSACTION ISOLATION LEVEL READ COMMITTED;
-- ใช้กับ transaction ถัดไปครั้งเดียว

-- SQL Server:
SET TRANSACTION ISOLATION LEVEL SNAPSHOT;
BEGIN TRANSACTION;
    SELECT * FROM LargeTable WHERE created_date > GETDATE() - 7;
    -- Snapshot isolation ป้องกัน blocking จาก writers
COMMIT;

-- ตรวจสอบ isolation level ที่ใช้อยู่จริง
-- PostgreSQL:
SELECT current_setting('transaction_isolation') AS isolation_level,
       pg_backend_pid() AS session_pid;

-- MySQL:
SELECT @@SESSION.transaction_isolation AS session_isolation,
       @@GLOBAL.transaction_isolation AS global_isolation;
```

---

## 13. Choosing the Right Isolation Level

```sql
-- Decision Tree สำหรับเลือก Isolation Level

-- Q: ข้อมูลต้องการ consistency ระดับไหน?

-- ระบบการเงิน / Banking:
-- → SERIALIZABLE หรือ REPEATABLE READ + explicit locking

-- OLTP (online orders, bookings):
-- → READ COMMITTED (ค่าเริ่มต้น)
-- ใช้ FOR UPDATE สำหรับ critical operations

-- Long-running reports:
-- → REPEATABLE READ
-- ป้องกัน inconsistent reads ในระหว่าง report generation

-- Analytics / BI:
-- → READ COMMITTED หรือ READ UNCOMMITTED
-- ยอมรับ approximate data เพื่อ performance

-- Log/Audit tables:
-- → READ COMMITTED
-- ข้อมูลใหม่ๆ ควรปรากฏทันที

-- ตัวอย่าง Business Logic:
CREATE OR REPLACE FUNCTION choose_isolation_level(
    p_operation_type TEXT
) RETURNS TEXT AS $$
BEGIN
    RETURN CASE p_operation_type
        WHEN 'financial_transfer'     THEN 'SERIALIZABLE'
        WHEN 'inventory_update'       THEN 'REPEATABLE READ'
        WHEN 'user_profile_update'    THEN 'READ COMMITTED'
        WHEN 'analytics_report'       THEN 'REPEATABLE READ'
        WHEN 'audit_log_read'         THEN 'READ COMMITTED'
        WHEN 'cache_refresh'          THEN 'READ UNCOMMITTED'
        ELSE 'READ COMMITTED'
    END;
END;
$$ LANGUAGE plpgsql;

-- ตัวอย่างการใช้:
SELECT choose_isolation_level('financial_transfer');  -- SERIALIZABLE
SELECT choose_isolation_level('analytics_report');    -- REPEATABLE READ
```

---

## 14. Monitoring Isolation Level Issues

```sql
-- ดู serialization failures
SELECT 
    datname,
    conflicts,
    deadlocks
FROM pg_stat_database
WHERE datname = current_database();

-- Monitor transactions กับ isolation levels
SELECT 
    pid,
    usename,
    application_name,
    now() - xact_start AS age,
    current_setting('transaction_isolation') AS isolation_level,
    state,
    left(query, 100) AS current_query
FROM pg_stat_activity
WHERE xact_start IS NOT NULL
ORDER BY age DESC;

-- ดู serialization conflicts
CREATE TABLE isolation_monitoring AS
SELECT 
    datname,
    xact_commit,
    xact_rollback,
    deadlocks,
    conflicts
FROM pg_stat_database
WHERE datname = current_database();

-- Alert เมื่อ rollback rate สูง (อาจมาจาก serialization failures)
SELECT 
    datname,
    xact_rollback,
    xact_commit,
    ROUND(xact_rollback::NUMERIC / NULLIF(xact_commit + xact_rollback, 0) * 100, 2) AS rollback_pct
FROM pg_stat_database
WHERE datname NOT IN ('postgres', 'template0', 'template1')
  AND xact_rollback::NUMERIC / NULLIF(xact_commit + xact_rollback, 0) > 0.05;  -- > 5% rollback
```

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: Setup สำหรับทดสอบ

```sql
-- คำตอบ
CREATE TABLE iso_test_accounts (
    account_id   SERIAL PRIMARY KEY,
    owner_name   VARCHAR(100) NOT NULL,
    balance      DECIMAL(15,2) NOT NULL DEFAULT 0,
    last_updated TIMESTAMPTZ DEFAULT NOW()
);

INSERT INTO iso_test_accounts (owner_name, balance) VALUES
    ('Alice', 10000),
    ('Bob', 5000),
    ('Charlie', 7500);

-- สร้าง transaction log
CREATE TABLE iso_test_log (
    log_id      SERIAL PRIMARY KEY,
    session_id  INT DEFAULT pg_backend_pid(),
    action      TEXT,
    old_balance DECIMAL(15,2),
    new_balance DECIMAL(15,2),
    isolation   TEXT DEFAULT current_setting('transaction_isolation'),
    logged_at   TIMESTAMPTZ DEFAULT NOW()
);
```

### แบบฝึกหัดที่ 2: สาธิต Read Committed Behavior

```sql
-- คำตอบ
-- ต้องรันใน 2 sessions
-- Session 1:
BEGIN TRANSACTION ISOLATION LEVEL READ COMMITTED;
SELECT balance FROM iso_test_accounts WHERE owner_name = 'Alice';
-- (รอ Session 2 ทำงาน แล้วรันบรรทัดถัดไป)
SELECT balance FROM iso_test_accounts WHERE owner_name = 'Alice';
-- เห็นค่าใหม่? → Non-Repeatable Read เกิดขึ้น
COMMIT;

-- Session 2:
BEGIN;
UPDATE iso_test_accounts SET balance = 99999 WHERE owner_name = 'Alice';
COMMIT;
```

### แบบฝึกหัดที่ 3: Repeatable Read Snapshot

```sql
-- คำตอบ
-- Session 1 (REPEATABLE READ):
BEGIN TRANSACTION ISOLATION LEVEL REPEATABLE READ;
SELECT balance FROM iso_test_accounts WHERE owner_name = 'Bob';  -- 5000
-- Session 2 update Bob's balance → 6000 และ commit
SELECT balance FROM iso_test_accounts WHERE owner_name = 'Bob';  -- ยังเห็น 5000!
COMMIT;

-- Verify: REPEATABLE READ ป้องกัน Non-Repeatable Read
DO $$
DECLARE
    v_first_read  DECIMAL;
    v_second_read DECIMAL;
BEGIN
    -- Set isolation level
    SET LOCAL transaction_isolation = 'repeatable read';
    
    SELECT balance INTO v_first_read 
    FROM iso_test_accounts WHERE owner_name = 'Bob';
    
    -- บันทึก first read
    RAISE NOTICE 'First read: %', v_first_read;
    
    -- รอสักครู่ (ใน real scenario, Session 2 จะ update ระหว่างนี้)
    PERFORM pg_sleep(0.01);
    
    SELECT balance INTO v_second_read 
    FROM iso_test_accounts WHERE owner_name = 'Bob';
    
    RAISE NOTICE 'Second read: %', v_second_read;
    RAISE NOTICE 'Same value: %', (v_first_read = v_second_read);
END;
$$;
```

### แบบฝึกหัดที่ 4: Phantom Read Protection

```sql
-- คำตอบ
CREATE TABLE iso_phantom_test (
    id       SERIAL PRIMARY KEY,
    category VARCHAR(50),
    amount   DECIMAL(10,2)
);

INSERT INTO iso_phantom_test (category, amount) VALUES
    ('A', 100), ('A', 200), ('B', 300);

-- Session 1 (REPEATABLE READ - ป้องกัน Phantom ใน PostgreSQL):
BEGIN TRANSACTION ISOLATION LEVEL REPEATABLE READ;
SELECT COUNT(*) FROM iso_phantom_test WHERE category = 'A';  -- 2

-- Session 2: INSERT ('A', 400) และ COMMIT

-- Session 1:
SELECT COUNT(*) FROM iso_phantom_test WHERE category = 'A';  -- ยังเห็น 2

-- Session 1 (READ COMMITTED - ไม่ป้องกัน Phantom):
-- SELECT COUNT(*) → จะเห็น 3 (phantom row)
COMMIT;
```

### แบบฝึกหัดที่ 5: Serializable Failure Handling

```sql
-- คำตอบ
CREATE OR REPLACE FUNCTION serializable_transfer(
    p_from_account INT,
    p_to_account   INT,
    p_amount       DECIMAL,
    p_max_retries  INT DEFAULT 3
) RETURNS TEXT AS $$
DECLARE
    v_attempts INT := 0;
    v_success  BOOLEAN := FALSE;
    v_result   TEXT;
BEGIN
    WHILE NOT v_success AND v_attempts < p_max_retries LOOP
        v_attempts := v_attempts + 1;
        
        BEGIN
            SET LOCAL transaction_isolation = 'serializable';
            
            -- Check balance
            IF (SELECT balance FROM iso_test_accounts WHERE account_id = p_from_account) < p_amount THEN
                RAISE EXCEPTION 'Insufficient funds';
            END IF;
            
            UPDATE iso_test_accounts SET balance = balance - p_amount 
            WHERE account_id = p_from_account;
            
            UPDATE iso_test_accounts SET balance = balance + p_amount 
            WHERE account_id = p_to_account;
            
            v_success := TRUE;
            v_result := format('Transfer completed on attempt %s', v_attempts);
            
        EXCEPTION
            WHEN serialization_failure THEN
                RAISE NOTICE 'Serialization failure on attempt %, retrying...', v_attempts;
                PERFORM pg_sleep(0.1 * v_attempts);
            WHEN OTHERS THEN
                RAISE;
        END;
    END LOOP;
    
    IF NOT v_success THEN
        RAISE EXCEPTION 'Transfer failed after % attempts', p_max_retries;
    END IF;
    
    RETURN v_result;
END;
$$ LANGUAGE plpgsql;

BEGIN;
SELECT serializable_transfer(1, 2, 500);
COMMIT;
```

### แบบฝึกหัดที่ 6: Compare Isolation Levels

```sql
-- คำตอบ
CREATE OR REPLACE FUNCTION compare_isolation_reads(
    p_target_account INT
) RETURNS TABLE(
    isolation_level TEXT,
    read1_value     DECIMAL,
    read2_value     DECIMAL,
    values_match    BOOLEAN
) AS $$
DECLARE
    v_val1 DECIMAL;
    v_val2 DECIMAL;
    v_levels TEXT[] := ARRAY['read committed', 'repeatable read'];
    v_level TEXT;
BEGIN
    FOREACH v_level IN ARRAY v_levels
    LOOP
        EXECUTE format('SET LOCAL transaction_isolation = %L', v_level);
        
        SELECT balance INTO v_val1 
        FROM iso_test_accounts WHERE account_id = p_target_account;
        
        -- Simulate time passing (in real test, update happens here)
        PERFORM pg_sleep(0.001);
        
        SELECT balance INTO v_val2 
        FROM iso_test_accounts WHERE account_id = p_target_account;
        
        isolation_level := v_level;
        read1_value     := v_val1;
        read2_value     := v_val2;
        values_match    := (v_val1 = v_val2);
        RETURN NEXT;
    END LOOP;
END;
$$ LANGUAGE plpgsql;

BEGIN;
SELECT * FROM compare_isolation_reads(1);
COMMIT;
```

### แบบฝึกหัดที่ 7: Lost Update Prevention

```sql
-- คำตอบ
-- สาธิต Lost Update และวิธีป้องกัน
CREATE TABLE iso_counter (
    counter_id INT PRIMARY KEY,
    value      INT NOT NULL DEFAULT 0
);

INSERT INTO iso_counter VALUES (1, 100);

-- Version ที่เกิด Lost Update (ไม่ดี):
-- Session 1:
-- BEGIN; SELECT value FROM iso_counter WHERE counter_id = 1; -- 100
-- (Session 2 runs and commits +50)
-- UPDATE iso_counter SET value = 100 + 10 WHERE counter_id = 1; -- Sets to 110!
-- COMMIT;
-- Lost Session 2's +50!

-- Version ที่ป้องกัน Lost Update (ดี):
CREATE OR REPLACE FUNCTION safe_increment(
    p_counter_id INT,
    p_increment  INT
) RETURNS INT AS $$
DECLARE
    v_new_value INT;
BEGIN
    -- Atomic update (ไม่ต้อง read-then-write)
    UPDATE iso_counter 
    SET value = value + p_increment
    WHERE counter_id = p_counter_id
    RETURNING value INTO v_new_value;
    
    RETURN v_new_value;
END;
$$ LANGUAGE plpgsql;

-- ทั้งสอง session จะได้ผลถูกต้อง
BEGIN;
SELECT safe_increment(1, 10);  -- 110
COMMIT;

-- อีก session:
BEGIN;
SELECT safe_increment(1, 50);  -- 160 (ไม่ใช่ 150!)
COMMIT;
```

### แบบฝึกหัดที่ 8: Isolation Level for Reports

```sql
-- คำตอบ
CREATE OR REPLACE FUNCTION generate_consistent_report()
RETURNS TABLE(
    metric      TEXT,
    value       NUMERIC,
    snapshot_id TEXT
) AS $$
DECLARE
    v_snapshot TEXT;
BEGIN
    SET LOCAL transaction_isolation = 'repeatable read';
    
    -- บันทึก snapshot ID
    v_snapshot := txid_current_snapshot()::TEXT;
    
    -- ทุก query ใช้ snapshot เดียวกัน
    metric := 'total_sales'; 
    value  := (SELECT COALESCE(SUM(amount), 0) FROM sales WHERE sale_date = CURRENT_DATE);
    snapshot_id := v_snapshot;
    RETURN NEXT;
    
    metric := 'total_orders';
    value  := (SELECT COUNT(*) FROM orders WHERE created_at::DATE = CURRENT_DATE);
    snapshot_id := v_snapshot;
    RETURN NEXT;
    
    metric := 'avg_order_value';
    value  := (SELECT ROUND(AVG(amount), 2) FROM sales WHERE sale_date = CURRENT_DATE);
    snapshot_id := v_snapshot;
    RETURN NEXT;
    
    metric := 'new_customers';
    value  := (SELECT COUNT(*) FROM customers WHERE created_at::DATE = CURRENT_DATE);
    snapshot_id := v_snapshot;
    RETURN NEXT;
END;
$$ LANGUAGE plpgsql;

BEGIN;
SELECT * FROM generate_consistent_report();
COMMIT;
```

### แบบฝึกหัดที่ 9: Monitoring Isolation Issues

```sql
-- คำตอบ
CREATE OR REPLACE VIEW isolation_health_check AS
WITH current_transactions AS (
    SELECT 
        pid,
        usename,
        state,
        now() - xact_start AS transaction_age,
        wait_event_type,
        wait_event
    FROM pg_stat_activity
    WHERE xact_start IS NOT NULL
),
db_stats AS (
    SELECT 
        xact_commit,
        xact_rollback,
        deadlocks,
        conflicts
    FROM pg_stat_database
    WHERE datname = current_database()
)
SELECT 
    (SELECT COUNT(*) FROM current_transactions) AS active_transactions,
    (SELECT COUNT(*) FROM current_transactions WHERE transaction_age > '5 minutes') AS long_transactions,
    (SELECT COUNT(*) FROM current_transactions WHERE wait_event IS NOT NULL) AS waiting_transactions,
    (SELECT deadlocks FROM db_stats) AS total_deadlocks,
    (SELECT conflicts FROM db_stats) AS total_conflicts,
    (SELECT ROUND(xact_rollback::NUMERIC / NULLIF(xact_commit + xact_rollback, 0) * 100, 2) FROM db_stats) AS rollback_pct;

SELECT * FROM isolation_health_check;
```

### แบบฝึกหัดที่ 10: Full Isolation Level Test Suite

```sql
-- คำตอบ
CREATE OR REPLACE FUNCTION test_isolation_levels()
RETURNS TABLE(
    test_name       TEXT,
    isolation_level TEXT,
    result          TEXT,
    explanation     TEXT
) AS $$
BEGIN
    -- Test 1: Dirty Read prevention
    test_name := 'Dirty Read Prevention';
    isolation_level := 'READ COMMITTED';
    result := 'PREVENTED';
    explanation := 'PostgreSQL READ COMMITTED (and above) prevents dirty reads';
    RETURN NEXT;
    
    -- Test 2: Non-Repeatable Read in READ COMMITTED
    test_name := 'Non-Repeatable Read';
    isolation_level := 'READ COMMITTED';
    result := 'POSSIBLE';
    explanation := 'READ COMMITTED allows non-repeatable reads';
    RETURN NEXT;
    
    -- Test 3: Non-Repeatable Read in REPEATABLE READ
    test_name := 'Non-Repeatable Read';
    isolation_level := 'REPEATABLE READ';
    result := 'PREVENTED';
    explanation := 'REPEATABLE READ uses snapshot to prevent non-repeatable reads';
    RETURN NEXT;
    
    -- Test 4: Phantom Read in REPEATABLE READ (PostgreSQL)
    test_name := 'Phantom Read (PostgreSQL)';
    isolation_level := 'REPEATABLE READ';
    result := 'PREVENTED';
    explanation := 'PostgreSQL MVCC prevents phantom reads even in REPEATABLE READ';
    RETURN NEXT;
    
    -- Test 5: All anomalies in SERIALIZABLE
    test_name := 'All Anomalies';
    isolation_level := 'SERIALIZABLE';
    result := 'PREVENTED';
    explanation := 'SERIALIZABLE prevents all concurrency anomalies using SSI';
    RETURN NEXT;
    
    -- Test 6: Performance check
    test_name := 'Performance Impact';
    isolation_level := 'SERIALIZABLE vs READ COMMITTED';
    result := 'SERIALIZABLE slower';
    explanation := 'SERIALIZABLE has overhead from conflict detection; retries may be needed';
    RETURN NEXT;
END;
$$ LANGUAGE plpgsql;

SELECT * FROM test_isolation_levels();
```

---

## สรุป

ใน Part นี้เราได้เรียนรู้:

1. **ปัญหา Concurrency** ทั้ง 4 ประเภท:
   - **Dirty Read**: อ่านข้อมูลที่ยังไม่ commit
   - **Non-Repeatable Read**: อ่านข้อมูลเดิมได้ค่าต่างกัน
   - **Phantom Read**: แถวใหม่ปรากฏหรือหายไปใน transaction
   - **Lost Update**: การแก้ไขหนึ่งถูกทับโดยอีกการแก้ไข

2. **Isolation Levels** ทั้ง 4 ระดับ:
   - **READ UNCOMMITTED**: เร็วที่สุด แต่อนุญาต dirty reads
   - **READ COMMITTED**: ค่าเริ่มต้น ป้องกัน dirty reads
   - **REPEATABLE READ**: ป้องกัน non-repeatable reads ด้วย snapshot
   - **SERIALIZABLE**: ปลอดภัยที่สุด ราวกับทำทีละ transaction

3. **MVCC (Snapshot Isolation)**: PostgreSQL ใช้ MVCC ทำให้ readers ไม่ block writers

4. **Database Defaults**: PostgreSQL=READ COMMITTED, MySQL=REPEATABLE READ

5. **Performance Trade-off**: ยิ่ง isolation สูง ยิ่งช้าลงและต้อง retry มากขึ้น

ใน Part 75 เราจะเรียนรู้เรื่อง Locking Mechanisms อย่างละเอียด
