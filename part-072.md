# Part 72: Transactions - BEGIN, COMMIT, ROLLBACK

## บทนำ

Transaction คือกลุ่มของ SQL statements ที่ถูกมองว่าเป็นหน่วยเดียว ซึ่งต้องสำเร็จทั้งหมดหรือล้มเหลวทั้งหมด ในบทนี้เราจะเรียนรู้การใช้งาน Transaction ใน Database ต่างๆ อย่างละเอียด

---

## 1. Implicit vs Explicit Transactions

### Implicit Transaction (Autocommit)

โดยค่าเริ่มต้น SQL databases ทำงานในโหมด **autocommit** ซึ่งหมายความว่าแต่ละ statement จะถูก commit อัตโนมัติทันทีที่ execute

```sql
-- Autocommit mode (ค่าเริ่มต้น)
-- แต่ละ statement เป็น transaction ของตัวเอง

UPDATE accounts SET balance = 1000 WHERE id = 1;  -- commit ทันที
INSERT INTO logs VALUES ('balance updated');        -- commit ทันที
-- ถ้า INSERT ล้มเหลว UPDATE ก็ยังคงอยู่!

-- ตรวจสอบสถานะ autocommit
-- PostgreSQL: ไม่มี autocommit setting แบบ explicit
-- แต่ทุก statement ที่ไม่อยู่ใน BEGIN...COMMIT จะ autocommit

-- MySQL: ตรวจสอบ autocommit
SHOW VARIABLES LIKE 'autocommit';
-- Value: ON = autocommit เปิดอยู่

-- ปิด autocommit ใน MySQL
SET autocommit = 0;
-- ตอนนี้ต้องใช้ COMMIT หรือ ROLLBACK เอง

-- เปิด autocommit กลับ
SET autocommit = 1;
```

### Explicit Transaction

```sql
-- Explicit Transaction ด้วย BEGIN...COMMIT
BEGIN;
    UPDATE accounts SET balance = balance - 500 WHERE id = 1;
    UPDATE accounts SET balance = balance + 500 WHERE id = 2;
COMMIT;

-- ถ้าต้องการยกเลิก
BEGIN;
    UPDATE accounts SET balance = balance - 500 WHERE id = 1;
    UPDATE accounts SET balance = balance + 500 WHERE id = 2;
ROLLBACK;  -- ยกเลิกทั้งหมด
```

---

## 2. BEGIN / START TRANSACTION

### PostgreSQL

```sql
-- วิธีที่ 1: BEGIN
BEGIN;
    -- SQL statements
COMMIT;

-- วิธีที่ 2: BEGIN TRANSACTION
BEGIN TRANSACTION;
    -- SQL statements
COMMIT;

-- วิธีที่ 3: START TRANSACTION (ANSI SQL standard)
START TRANSACTION;
    -- SQL statements
COMMIT;

-- กำหนด Isolation Level ตอน BEGIN
BEGIN TRANSACTION ISOLATION LEVEL SERIALIZABLE;
    -- SQL statements
COMMIT;

-- กำหนด READ ONLY / READ WRITE
BEGIN TRANSACTION READ ONLY;
    SELECT * FROM accounts;  -- อ่านได้อย่างเดียว
COMMIT;

BEGIN TRANSACTION READ WRITE;
    UPDATE accounts SET balance = 1000 WHERE id = 1;
COMMIT;

-- ดูสถานะของ transaction ปัจจุบัน
SELECT current_setting('transaction_isolation') AS isolation_level;
```

### MySQL

```sql
-- MySQL: BEGIN หรือ START TRANSACTION
BEGIN;
    UPDATE users SET last_login = NOW() WHERE user_id = 1;
COMMIT;

START TRANSACTION;
    INSERT INTO orders (user_id, total) VALUES (1, 5000);
    INSERT INTO order_items (order_id, product_id, quantity) VALUES (LAST_INSERT_ID(), 101, 2);
COMMIT;

-- MySQL: WITH CONSISTENT SNAPSHOT
START TRANSACTION WITH CONSISTENT SNAPSHOT;
-- เริ่ม transaction พร้อม snapshot ของข้อมูล ณ เวลานั้น
-- ใช้สำหรับ long-running reports

-- MySQL: READ WRITE / READ ONLY
START TRANSACTION READ ONLY;
    SELECT SUM(total) FROM orders WHERE DATE(created_at) = CURDATE();
COMMIT;

-- ดูสถานะ autocommit
SELECT @@autocommit;
```

### SQLite

```sql
-- SQLite: BEGIN TRANSACTION
BEGIN TRANSACTION;
    INSERT INTO products (name, price) VALUES ('Widget', 9.99);
    UPDATE inventory SET quantity = quantity - 1 WHERE product_id = 1;
COMMIT;

-- SQLite: BEGIN DEFERRED (ค่าเริ่มต้น - lock เมื่อ access ครั้งแรก)
BEGIN DEFERRED;
    SELECT * FROM orders;
COMMIT;

-- SQLite: BEGIN IMMEDIATE (shared read lock ทันที)
BEGIN IMMEDIATE;
    SELECT * FROM orders;
    UPDATE orders SET status = 'processing' WHERE order_id = 1;
COMMIT;

-- SQLite: BEGIN EXCLUSIVE (exclusive lock ทันที - ป้องกัน readers ด้วย)
BEGIN EXCLUSIVE;
    -- ไม่มีใช้ database นี้ได้เลยในช่วงนี้
    PRAGMA wal_checkpoint(FULL);
COMMIT;
```

### SQL Server

```sql
-- SQL Server: BEGIN TRANSACTION
BEGIN TRANSACTION;
    UPDATE Products SET StockQuantity = StockQuantity - 1 WHERE ProductID = 101;
    INSERT INTO SalesOrders (CustomerID, ProductID, Quantity) VALUES (1001, 101, 1);
COMMIT TRANSACTION;

-- ย่อเป็น COMMIT หรือ COMMIT TRAN
BEGIN TRAN;
    DELETE FROM TempData WHERE ExpiryDate < GETDATE();
COMMIT TRAN;

-- ดูจำนวน open transactions
SELECT @@TRANCOUNT;

-- Named Transaction ใน SQL Server
BEGIN TRANSACTION CreateOrder;
    INSERT INTO Orders VALUES (...);
COMMIT TRANSACTION CreateOrder;

-- ตั้งชื่อ Transaction สำหรับ ROLLBACK บางส่วน
BEGIN TRANSACTION;
    SAVE TRANSACTION CheckPoint1;
    -- บางอย่าง
    ROLLBACK TRANSACTION CheckPoint1;
    -- ยังอยู่ใน transaction
COMMIT TRANSACTION;
```

---

## 3. COMMIT - ยืนยัน Transaction

```sql
-- PostgreSQL: COMMIT หรือ END
BEGIN;
    UPDATE products SET stock = stock - 10 WHERE product_id = 1;
    INSERT INTO shipments (product_id, quantity) VALUES (1, 10);
COMMIT;
-- หรือ
END;

-- ตรวจสอบว่า commit สำเร็จ
BEGIN;
    INSERT INTO test_table VALUES (1, 'test');
COMMIT;
SELECT * FROM test_table;  -- ควรเห็น row ใหม่

-- Two-Phase Commit (2PC) - สำหรับ distributed transactions
PREPARE TRANSACTION 'my_transaction';
-- ... ตรวจสอบสถานะในระบบอื่น ...
COMMIT PREPARED 'my_transaction';
-- หรือ
ROLLBACK PREPARED 'my_transaction';

-- ดู prepared transactions
SELECT * FROM pg_prepared_xacts;
```

---

## 4. ROLLBACK - ยกเลิก Transaction

```sql
-- ROLLBACK พื้นฐาน
BEGIN;
    UPDATE accounts SET balance = 0 WHERE id = 1;  -- ลบเงินทั้งหมด!
ROLLBACK;  -- ยกเลิก ไม่มีอะไรเปลี่ยน

-- ROLLBACK เมื่อเกิด Error
BEGIN;
    UPDATE accounts SET balance = balance - 500 WHERE id = 1;
    -- จำลอง error
    DO $$ BEGIN RAISE EXCEPTION 'Something went wrong'; END; $$;
    UPDATE accounts SET balance = balance + 500 WHERE id = 2;  -- ไม่ถึงบรรทัดนี้
ROLLBACK;

-- PostgreSQL: ROLLBACK TO SAVEPOINT
BEGIN;
    UPDATE accounts SET balance = 1000 WHERE id = 1;
    SAVEPOINT sp1;
    UPDATE accounts SET balance = 2000 WHERE id = 1;
    ROLLBACK TO SAVEPOINT sp1;  -- กลับไปที่ 1000
COMMIT;  -- บันทึกค่า 1000

-- ROLLBACK PREPARED (2PC)
BEGIN;
    INSERT INTO critical_log VALUES ('important event');
PREPARE TRANSACTION 'critical_txn';
-- ถ้าตัดสินใจยกเลิก:
ROLLBACK PREPARED 'critical_txn';
```

---

## 5. Autocommit Mode ในรายละเอียด

```sql
-- PostgreSQL: ทุก statement ที่ไม่อยู่ใน transaction block จะ autocommit
-- แต่ psql client มี \set AUTOCOMMIT off ได้

-- ใน psql:
-- \set AUTOCOMMIT off
-- ตอนนี้ต้องใช้ COMMIT หรือ ROLLBACK เอง

-- ทดสอบ autocommit behavior
-- Session 1:
INSERT INTO test_vals VALUES (1);  -- autocommit ทันที

-- Session 2 (immediately after):
SELECT * FROM test_vals;  -- เห็น row 1 ทันที (already committed)

-- MySQL autocommit
SET autocommit = 0;
INSERT INTO test_vals VALUES (1);  -- ยังไม่ commit
INSERT INTO test_vals VALUES (2);  -- ยังไม่ commit
COMMIT;  -- commit ทั้งสอง
-- หรือ
ROLLBACK;  -- ยกเลิกทั้งสอง

-- ข้อควรระวัง: DDL statements ใน MySQL จะ auto-commit implicit transaction
SET autocommit = 0;
INSERT INTO test_vals VALUES (1);  -- อยู่ใน transaction
CREATE TABLE new_table (id INT);   -- DDL triggers implicit COMMIT!
-- ตอนนี้ INSERT ถูก commit ไปแล้ว!
ROLLBACK;  -- ไม่มีผล เพราะ implicit COMMIT เกิดไปแล้ว
```

---

## 6. Transaction Blocks ที่ซับซ้อน

```sql
-- Transaction พร้อม Error Handling ใน PostgreSQL (PL/pgSQL)
DO $$
DECLARE
    v_order_id INT;
    v_customer_id INT := 1001;
    v_product_id INT := 501;
    v_quantity INT := 3;
    v_price DECIMAL := 299.99;
BEGIN
    -- Step 1: ตรวจสอบว่า customer มีอยู่จริง
    IF NOT EXISTS (SELECT 1 FROM customers WHERE customer_id = v_customer_id) THEN
        RAISE EXCEPTION 'Customer % not found', v_customer_id;
    END IF;
    
    -- Step 2: ตรวจสอบสต็อก
    IF (SELECT quantity FROM products WHERE product_id = v_product_id) < v_quantity THEN
        RAISE EXCEPTION 'Insufficient stock for product %', v_product_id;
    END IF;
    
    -- Step 3: สร้าง order
    INSERT INTO orders (customer_id, order_date, status)
    VALUES (v_customer_id, CURRENT_TIMESTAMP, 'pending')
    RETURNING order_id INTO v_order_id;
    
    -- Step 4: เพิ่ม order items
    INSERT INTO order_items (order_id, product_id, quantity, unit_price)
    VALUES (v_order_id, v_product_id, v_quantity, v_price);
    
    -- Step 5: อัปเดตสต็อก
    UPDATE products 
    SET quantity = quantity - v_quantity,
        updated_at = CURRENT_TIMESTAMP
    WHERE product_id = v_product_id;
    
    -- Step 6: คำนวณและอัปเดต order total
    UPDATE orders 
    SET total_amount = v_quantity * v_price,
        status = 'confirmed'
    WHERE order_id = v_order_id;
    
    -- Step 7: สร้าง invoice
    INSERT INTO invoices (order_id, amount, due_date, status)
    VALUES (v_order_id, v_quantity * v_price, CURRENT_DATE + 30, 'unpaid');
    
    RAISE NOTICE 'Order % created successfully', v_order_id;
    
EXCEPTION
    WHEN OTHERS THEN
        RAISE NOTICE 'Error occurred: %, rolling back', SQLERRM;
        RAISE;  -- Re-raise เพื่อให้ transaction rollback
END;
$$;
```

---

## 7. Error Handling ใน Transactions

```sql
-- Pattern 1: Simple BEGIN...EXCEPTION...END ใน PL/pgSQL
CREATE OR REPLACE FUNCTION process_payment(
    p_order_id INT,
    p_amount DECIMAL,
    p_card_number VARCHAR
) RETURNS JSONB AS $$
DECLARE
    v_result JSONB;
BEGIN
    -- ตรวจสอบ order
    IF NOT EXISTS (SELECT 1 FROM orders WHERE order_id = p_order_id AND status = 'pending') THEN
        RETURN jsonb_build_object('success', false, 'error', 'Order not found or not pending');
    END IF;
    
    -- อัปเดตสถานะ
    UPDATE orders SET status = 'processing' WHERE order_id = p_order_id;
    
    -- บันทึก payment
    INSERT INTO payments (order_id, amount, card_last4, payment_date)
    VALUES (p_order_id, p_amount, RIGHT(p_card_number, 4), NOW());
    
    -- อัปเดต order เป็น completed
    UPDATE orders SET status = 'paid' WHERE order_id = p_order_id;
    
    RETURN jsonb_build_object('success', true, 'order_id', p_order_id);
    
EXCEPTION
    WHEN unique_violation THEN
        RETURN jsonb_build_object('success', false, 'error', 'Duplicate payment detected');
    WHEN foreign_key_violation THEN
        RETURN jsonb_build_object('success', false, 'error', 'Invalid reference data');
    WHEN check_violation THEN
        RETURN jsonb_build_object('success', false, 'error', 'Data validation failed');
    WHEN OTHERS THEN
        RETURN jsonb_build_object('success', false, 'error', SQLERRM);
END;
$$ LANGUAGE plpgsql;

-- Pattern 2: Nested Error Handling
CREATE OR REPLACE FUNCTION complex_operation() RETURNS VOID AS $$
BEGIN
    BEGIN  -- Outer transaction
        INSERT INTO main_table VALUES (1, 'main data');
        
        BEGIN  -- Inner block with its own exception handler
            INSERT INTO secondary_table VALUES (1, 'secondary data');
        EXCEPTION
            WHEN OTHERS THEN
                -- ล้มเหลว แต่ outer transaction ยังดำเนินต่อได้
                INSERT INTO error_log VALUES (NOW(), 'secondary_table insert failed', SQLERRM);
        END;
        
        UPDATE main_table SET status = 'completed' WHERE id = 1;
        
    EXCEPTION
        WHEN OTHERS THEN
            -- Outer ล้มเหลว → ทุกอย่างถูก rollback
            RAISE;
    END;
END;
$$ LANGUAGE plpgsql;

-- Pattern 3: Transaction ใน Application Code (Python psycopg2 style)
-- จำลองใน SQL:
DO $$
DECLARE
    max_retries CONSTANT INT := 3;
    retry_count INT := 0;
    success BOOLEAN := FALSE;
BEGIN
    WHILE retry_count < max_retries AND NOT success LOOP
        BEGIN
            -- Attempt transaction
            UPDATE accounts SET balance = balance - 100 WHERE id = 1;
            UPDATE accounts SET balance = balance + 100 WHERE id = 2;
            success := TRUE;
            
        EXCEPTION
            WHEN serialization_failure OR deadlock_detected THEN
                retry_count := retry_count + 1;
                RAISE NOTICE 'Retry % due to %', retry_count, SQLERRM;
                PERFORM pg_sleep(0.1 * retry_count);  -- Exponential backoff
            WHEN OTHERS THEN
                RAISE;  -- ไม่ retry สำหรับ errors อื่น
        END;
    END LOOP;
    
    IF NOT success THEN
        RAISE EXCEPTION 'Transaction failed after % retries', max_retries;
    END IF;
END;
$$;
```

---

## 8. Long-Running Transactions และอันตรายของมัน

```sql
-- อันตราย 1: Bloating ใน PostgreSQL (dead tuples สะสม)
-- Long transaction ป้องกัน VACUUM จากการเก็บ dead tuples

-- ดู long-running transactions
SELECT 
    pid,
    now() - pg_stat_activity.xact_start AS duration,
    query,
    state
FROM pg_stat_activity
WHERE (now() - pg_stat_activity.xact_start) > INTERVAL '5 minutes'
ORDER BY duration DESC;

-- อันตราย 2: Lock contention
-- Long transaction ที่ hold lock จะ block transaction อื่น
BEGIN;
    SELECT * FROM orders WHERE status = 'pending' FOR UPDATE;
    -- ทำอะไรบางอย่างที่ใช้เวลานาน...
    -- ระหว่างนี้ transaction อื่นที่ต้องการ update orders จะถูก block!

-- วิธีป้องกัน: ตั้ง lock timeout
SET lock_timeout = '10s';  -- รอ lock ไม่เกิน 10 วินาที

-- อันตราย 3: Transaction ID Wraparound (PostgreSQL)
-- PostgreSQL ใช้ 32-bit transaction IDs → ครบรอบทุกๆ ~2 billion transactions
-- ถ้า long transaction ค้าง → อาจป้องกัน wraparound protection

-- ดู oldest transaction
SELECT min(xact_start) AS oldest_transaction,
       now() - min(xact_start) AS age
FROM pg_stat_activity
WHERE xact_start IS NOT NULL;

-- ดู transaction ID ที่ใกล้ wraparound
SELECT 
    datname,
    age(datfrozenxid) AS transaction_age,
    2147483648 - age(datfrozenxid) AS remaining_before_wraparound
FROM pg_database
ORDER BY age(datfrozenxid) DESC;

-- วิธีแก้: ตั้ง idle_in_transaction_session_timeout
-- postgresql.conf:
-- idle_in_transaction_session_timeout = '5min'
SET idle_in_transaction_session_timeout = '300000';  -- 5 minutes in ms

-- อันตราย 4: Connection Exhaustion
-- Long transactions ใช้ connections นาน → connection pool หมด
-- ดู connections ทั้งหมด
SELECT 
    state,
    COUNT(*) AS connection_count,
    MAX(now() - xact_start) AS max_transaction_age
FROM pg_stat_activity
GROUP BY state;

-- Kill long-running transactions ถ้าจำเป็น
SELECT pg_terminate_backend(pid)
FROM pg_stat_activity
WHERE (now() - pg_stat_activity.xact_start) > INTERVAL '30 minutes'
  AND state = 'idle in transaction';
```

---

## 9. Transactions ในแต่ละ Database อย่างละเอียด

### PostgreSQL Transactions

```sql
-- PostgreSQL Transaction Features ทั้งหมด

-- 1. Transaction Isolation Levels
BEGIN TRANSACTION ISOLATION LEVEL READ COMMITTED;
BEGIN TRANSACTION ISOLATION LEVEL REPEATABLE READ;
BEGIN TRANSACTION ISOLATION LEVEL SERIALIZABLE;

-- 2. Deferrable transactions (ใช้กับ SERIALIZABLE เท่านั้น)
BEGIN TRANSACTION ISOLATION LEVEL SERIALIZABLE DEFERRABLE;
-- DEFERRABLE transaction จะรอจนกว่าจะมั่นใจว่าไม่มี conflicts
-- ช้ากว่า แต่ไม่เกิด serialization failures

-- 3. Transaction ใน PL/pgSQL function
CREATE OR REPLACE PROCEDURE process_batch(p_batch_size INT DEFAULT 1000)
LANGUAGE plpgsql AS $$
DECLARE
    v_processed INT := 0;
    v_total INT;
BEGIN
    SELECT COUNT(*) INTO v_total FROM pending_records;
    
    WHILE v_processed < v_total LOOP
        -- Process batch
        WITH batch AS (
            SELECT id FROM pending_records
            WHERE processed = FALSE
            LIMIT p_batch_size
            FOR UPDATE SKIP LOCKED
        )
        UPDATE pending_records p
        SET processed = TRUE, processed_at = NOW()
        FROM batch b
        WHERE p.id = b.id;
        
        GET DIAGNOSTICS v_processed = ROW_COUNT;
        
        COMMIT;  -- Commit ทีละ batch (PostgreSQL 11+)
        
        RAISE NOTICE 'Processed % records', v_processed;
    END LOOP;
END;
$$;

-- 4. Advisory Locks ใน transaction context
BEGIN;
    SELECT pg_advisory_xact_lock(12345);  -- Lock จนกว่า transaction จะจบ
    -- ทำ critical section
COMMIT;

-- 5. Two-Phase Commit
PREPARE TRANSACTION 'order_processing_txn';
-- ระหว่างนี้ transaction ถูก "suspend"
-- ใน distributed coordinator:
COMMIT PREPARED 'order_processing_txn';
-- หรือ:
ROLLBACK PREPARED 'order_processing_txn';

-- ดู prepared transactions ที่ค้างอยู่
SELECT * FROM pg_prepared_xacts;
-- IMPORTANT: Prepared transactions ที่ค้างนานมาก = อันตราย!
-- ควร ROLLBACK PREPARED ถ้าไม่แน่ใจ
```

### MySQL Transactions

```sql
-- MySQL Transaction Features

-- 1. Basic Transaction
START TRANSACTION;
    INSERT INTO orders (customer_id, total) VALUES (1, 5000);
    SET @last_order_id = LAST_INSERT_ID();
    INSERT INTO order_items (order_id, product_id, quantity, price)
    VALUES (@last_order_id, 101, 2, 2500);
COMMIT;

-- 2. Chained Transactions
-- COMMIT AND CHAIN - commit และเริ่ม transaction ใหม่ทันที
START TRANSACTION;
    UPDATE stock SET quantity = quantity - 1 WHERE product_id = 101;
COMMIT AND CHAIN;
    INSERT INTO stock_log VALUES (101, -1, NOW());
COMMIT;

-- 3. Transaction ที่มี Savepoints
START TRANSACTION;
    INSERT INTO headers (order_id, total) VALUES (1, 1000);
    SAVEPOINT sp_after_header;
    
    INSERT INTO items (order_id, item, price) VALUES (1, 'Widget', 500);
    SAVEPOINT sp_after_item1;
    
    INSERT INTO items (order_id, item, price) VALUES (1, 'Gadget', 500);
    -- ถ้า Gadget ไม่มีสต็อก:
    ROLLBACK TO SAVEPOINT sp_after_item1;
    
    -- Order ยังคงมี Widget
COMMIT;

-- 4. XA Transactions (Distributed)
XA START 'order_xa';
    INSERT INTO orders VALUES (...);
XA END 'order_xa';
XA PREPARE 'order_xa';
-- Coordinator ตัดสินใจ:
XA COMMIT 'order_xa';
-- หรือ:
XA ROLLBACK 'order_xa';

-- 5. ดู active transactions
SELECT 
    trx_id,
    trx_state,
    trx_started,
    trx_mysql_thread_id,
    trx_query,
    trx_rows_locked,
    trx_rows_modified
FROM information_schema.innodb_trx
ORDER BY trx_started;

-- 6. ปัญหา DDL และ autocommit ใน MySQL
START TRANSACTION;
    INSERT INTO users VALUES (1, 'Alice');  -- อยู่ใน transaction
    CREATE TABLE temp_table (id INT);        -- DDL! ทำให้ implicit COMMIT
    -- ตอนนี้ INSERT ถูก commit ไปแล้ว!
    INSERT INTO users VALUES (2, 'Bob');    -- อยู่ใน transaction ใหม่
ROLLBACK;
-- Bob ถูก rollback แต่ Alice ไม่ถูก!
```

### SQLite Transactions

```sql
-- SQLite Transaction Features

-- 1. Transaction Modes
BEGIN;                    -- DEFERRED (ค่าเริ่มต้น)
BEGIN DEFERRED;           -- เหมือนกัน
BEGIN IMMEDIATE;          -- RESERVED lock ทันที
BEGIN EXCLUSIVE;          -- EXCLUSIVE lock ทันที

-- DEFERRED: Database ไม่ acquire lock จนกว่าจะมีการ access จริง
-- IMMEDIATE: Acquire reserved lock ทันที (ป้องกัน writer อื่น)  
-- EXCLUSIVE: Acquire exclusive lock ทันที (ป้องกันทุกคน)

-- 2. WAL Mode สำหรับ Concurrency ที่ดีกว่า
PRAGMA journal_mode = WAL;
-- WAL Mode อนุญาตให้ readers และ 1 writer ทำงานพร้อมกัน

-- 3. Nested Transactions (ใช้ Savepoints)
BEGIN;
    INSERT INTO table_a VALUES (1);
    SAVEPOINT sp1;
    INSERT INTO table_b VALUES (1);
    SAVEPOINT sp2;
    INSERT INTO table_c VALUES (1);
    ROLLBACK TO sp2;  -- ยกเลิกแค่ table_c
    RELEASE sp2;
    -- table_a และ table_b ยังอยู่
COMMIT;

-- 4. Transaction Statistics
SELECT 
    changes() AS rows_affected_in_last_statement;

SELECT 
    total_changes() AS total_rows_changed_in_connection;
```

### SQL Server Transactions

```sql
-- SQL Server Transaction Features

-- 1. Explicit Transactions
BEGIN TRANSACTION;
    INSERT INTO Customers (Name, Email) VALUES ('Alice', 'alice@example.com');
    INSERT INTO Accounts (CustomerID, Balance) VALUES (SCOPE_IDENTITY(), 1000);
COMMIT TRANSACTION;

-- 2. Named Transactions
BEGIN TRANSACTION OrderCreation;
    INSERT INTO Orders (CustomerID, OrderDate) VALUES (1, GETDATE());
    
    IF @@ERROR <> 0
    BEGIN
        ROLLBACK TRANSACTION OrderCreation;
        RETURN;
    END
    
    INSERT INTO OrderDetails (OrderID, ProductID, Quantity) 
    VALUES (SCOPE_IDENTITY(), 101, 2);
    
    IF @@ERROR <> 0
    BEGIN
        ROLLBACK TRANSACTION OrderCreation;
        RETURN;
    END
    
COMMIT TRANSACTION OrderCreation;

-- 3. TRY...CATCH ใน Transactions
BEGIN TRY
    BEGIN TRANSACTION;
    
    UPDATE Accounts SET Balance = Balance - 500 WHERE AccountID = 1;
    UPDATE Accounts SET Balance = Balance + 500 WHERE AccountID = 2;
    
    COMMIT TRANSACTION;
    PRINT 'Transaction committed successfully';
    
END TRY
BEGIN CATCH
    IF @@TRANCOUNT > 0
        ROLLBACK TRANSACTION;
    
    PRINT 'Error: ' + ERROR_MESSAGE();
    THROW;  -- Re-throw ข้อผิดพลาด
END CATCH;

-- 4. Nested Transactions ใน SQL Server (@@TRANCOUNT)
BEGIN TRANSACTION;    -- @@TRANCOUNT = 1
    BEGIN TRANSACTION;  -- @@TRANCOUNT = 2
        INSERT INTO Table1 VALUES (1);
    COMMIT TRANSACTION;  -- @@TRANCOUNT = 1 (ยังไม่ commit จริง!)
COMMIT TRANSACTION;      -- @@TRANCOUNT = 0 (commit จริง!)

-- ข้อควรระวัง: ROLLBACK ใน nested transaction ทำให้ทุกอย่างถูก rollback
BEGIN TRANSACTION;    -- @@TRANCOUNT = 1
    BEGIN TRANSACTION;  -- @@TRANCOUNT = 2
        INSERT INTO Table1 VALUES (1);
    ROLLBACK TRANSACTION;  -- @@TRANCOUNT = 0! ทุกอย่างถูก rollback!
-- ไม่สามารถ COMMIT ได้แล้ว!

-- 5. Distributed Transactions (MSDTC)
BEGIN DISTRIBUTED TRANSACTION;
    -- Transaction กับ linked server
    INSERT INTO LinkedServer.RemoteDB.dbo.Orders VALUES (...);
    INSERT INTO LocalDB.dbo.LocalOrders VALUES (...);
COMMIT TRANSACTION;

-- 6. Snapshot Isolation
ALTER DATABASE MyDB SET ALLOW_SNAPSHOT_ISOLATION ON;
SET TRANSACTION ISOLATION LEVEL SNAPSHOT;
BEGIN TRANSACTION;
    SELECT * FROM Accounts;  -- Snapshot ณ เวลา BEGIN
COMMIT;
```

---

## 10. Transaction Best Practices

```sql
-- Practice 1: ทำ Transactions ให้สั้นที่สุด
-- ไม่ดี:
BEGIN;
    SELECT * FROM products WHERE category = 'electronics';
    -- รอ user เลือก product (อาจนาน 5 นาที!)
    INSERT INTO cart VALUES (...);
COMMIT;

-- ดี:
-- Step 1: อ่านข้อมูล (ไม่ต้องอยู่ใน transaction)
SELECT * FROM products WHERE category = 'electronics';
-- Step 2: User เลือก product
-- Step 3: Transaction สั้นๆ
BEGIN;
    INSERT INTO cart VALUES (...);
COMMIT;

-- Practice 2: Lock ตามลำดับที่สอดคล้องกันเสมอ
-- ไม่ดี (อาจเกิด deadlock):
-- Transaction 1: Lock account 1 แล้ว lock account 2
-- Transaction 2: Lock account 2 แล้ว lock account 1

-- ดี: Lock ตาม account_id เสมอ
CREATE OR REPLACE FUNCTION transfer(from_id INT, to_id INT, amount DECIMAL) 
RETURNS VOID AS $$
BEGIN
    IF from_id < to_id THEN
        PERFORM * FROM accounts WHERE account_id = from_id FOR UPDATE;
        PERFORM * FROM accounts WHERE account_id = to_id FOR UPDATE;
    ELSE
        PERFORM * FROM accounts WHERE account_id = to_id FOR UPDATE;
        PERFORM * FROM accounts WHERE account_id = from_id FOR UPDATE;
    END IF;
    
    UPDATE accounts SET balance = balance - amount WHERE account_id = from_id;
    UPDATE accounts SET balance = balance + amount WHERE account_id = to_id;
END;
$$ LANGUAGE plpgsql;

-- Practice 3: ใช้ SKIP LOCKED สำหรับ queue processing
BEGIN;
    SELECT * FROM job_queue
    WHERE status = 'pending'
    ORDER BY created_at
    LIMIT 10
    FOR UPDATE SKIP LOCKED;  -- ข้าม records ที่ถูก lock อยู่แล้ว
    
    -- Process jobs...
    
    UPDATE job_queue SET status = 'completed'
    WHERE job_id = ANY(ARRAY[1, 2, 3, ...]);
COMMIT;

-- Practice 4: ใช้ Retry Logic
CREATE OR REPLACE FUNCTION transfer_with_retry(
    from_id INT, 
    to_id INT, 
    amount DECIMAL,
    max_attempts INT DEFAULT 3
) RETURNS BOOLEAN AS $$
DECLARE
    attempt INT := 0;
BEGIN
    LOOP
        BEGIN
            UPDATE accounts SET balance = balance - amount WHERE account_id = from_id;
            UPDATE accounts SET balance = balance + amount WHERE account_id = to_id;
            RETURN TRUE;
        EXCEPTION
            WHEN serialization_failure OR deadlock_detected THEN
                attempt := attempt + 1;
                IF attempt >= max_attempts THEN
                    RAISE EXCEPTION 'Transfer failed after % attempts', max_attempts;
                END IF;
                PERFORM pg_sleep(0.05 * attempt);  -- Backoff
        END;
    END LOOP;
END;
$$ LANGUAGE plpgsql;

-- Practice 5: Monitoring ใน Production
CREATE OR REPLACE VIEW transaction_health AS
SELECT 
    state,
    COUNT(*) AS count,
    MAX(EXTRACT(EPOCH FROM (NOW() - xact_start))) AS max_age_seconds,
    AVG(EXTRACT(EPOCH FROM (NOW() - xact_start))) AS avg_age_seconds
FROM pg_stat_activity
WHERE xact_start IS NOT NULL
GROUP BY state;
```

---

## 11. ตัวอย่าง Real-World: E-Commerce Order System

```sql
-- Complete order processing system with full transaction handling

CREATE TABLE customers (
    customer_id   SERIAL PRIMARY KEY,
    name          VARCHAR(200) NOT NULL,
    email         VARCHAR(200) UNIQUE NOT NULL,
    credit_limit  DECIMAL(15,2) DEFAULT 10000,
    current_debt  DECIMAL(15,2) DEFAULT 0
);

CREATE TABLE products (
    product_id    SERIAL PRIMARY KEY,
    name          VARCHAR(200) NOT NULL,
    price         DECIMAL(10,2) NOT NULL,
    stock         INT NOT NULL DEFAULT 0,
    reserved      INT NOT NULL DEFAULT 0,
    CHECK (stock >= 0),
    CHECK (reserved >= 0),
    CHECK (stock >= reserved)
);

CREATE TABLE orders (
    order_id      SERIAL PRIMARY KEY,
    customer_id   INT REFERENCES customers(customer_id),
    status        VARCHAR(20) DEFAULT 'draft' 
                  CHECK (status IN ('draft', 'confirmed', 'paid', 'shipped', 'cancelled')),
    total_amount  DECIMAL(15,2) DEFAULT 0,
    created_at    TIMESTAMP DEFAULT NOW(),
    updated_at    TIMESTAMP DEFAULT NOW()
);

CREATE TABLE order_items (
    item_id       SERIAL PRIMARY KEY,
    order_id      INT REFERENCES orders(order_id) ON DELETE CASCADE,
    product_id    INT REFERENCES products(product_id),
    quantity      INT NOT NULL CHECK (quantity > 0),
    unit_price    DECIMAL(10,2) NOT NULL,
    subtotal      DECIMAL(15,2) GENERATED ALWAYS AS (quantity * unit_price) STORED
);

-- Main function สำหรับสร้าง order
CREATE OR REPLACE FUNCTION create_order(
    p_customer_id INT,
    p_items       JSONB  -- [{"product_id": 1, "quantity": 2}, ...]
) RETURNS JSONB AS $$
DECLARE
    v_order_id     INT;
    v_total        DECIMAL(15,2) := 0;
    v_item         JSONB;
    v_product_id   INT;
    v_quantity     INT;
    v_price        DECIMAL(10,2);
    v_available    INT;
    v_credit_limit DECIMAL(15,2);
    v_current_debt DECIMAL(15,2);
BEGIN
    -- Validation
    IF p_items IS NULL OR jsonb_array_length(p_items) = 0 THEN
        RAISE EXCEPTION 'Order must have at least one item';
    END IF;
    
    -- ตรวจสอบ customer
    SELECT credit_limit, current_debt 
    INTO v_credit_limit, v_current_debt
    FROM customers 
    WHERE customer_id = p_customer_id
    FOR UPDATE;  -- Lock customer record
    
    IF NOT FOUND THEN
        RAISE EXCEPTION 'Customer % not found', p_customer_id;
    END IF;
    
    -- สร้าง order header
    INSERT INTO orders (customer_id, status)
    VALUES (p_customer_id, 'draft')
    RETURNING order_id INTO v_order_id;
    
    -- Process each item
    FOR v_item IN SELECT * FROM jsonb_array_elements(p_items)
    LOOP
        v_product_id := (v_item->>'product_id')::INT;
        v_quantity   := (v_item->>'quantity')::INT;
        
        -- Lock product และตรวจสอบสต็อก
        SELECT price, stock - reserved INTO v_price, v_available
        FROM products
        WHERE product_id = v_product_id
        FOR UPDATE;
        
        IF NOT FOUND THEN
            RAISE EXCEPTION 'Product % not found', v_product_id;
        END IF;
        
        IF v_available < v_quantity THEN
            RAISE EXCEPTION 'Insufficient stock for product %. Available: %, Requested: %',
                             v_product_id, v_available, v_quantity;
        END IF;
        
        -- Reserve สต็อก
        UPDATE products 
        SET reserved = reserved + v_quantity
        WHERE product_id = v_product_id;
        
        -- เพิ่ม order item
        INSERT INTO order_items (order_id, product_id, quantity, unit_price)
        VALUES (v_order_id, v_product_id, v_quantity, v_price);
        
        v_total := v_total + (v_quantity * v_price);
    END LOOP;
    
    -- ตรวจสอบ credit limit
    IF v_current_debt + v_total > v_credit_limit THEN
        RAISE EXCEPTION 'Credit limit exceeded. Limit: %, Current debt: %, Order total: %',
                         v_credit_limit, v_current_debt, v_total;
    END IF;
    
    -- Confirm order
    UPDATE orders 
    SET status = 'confirmed', 
        total_amount = v_total,
        updated_at = NOW()
    WHERE order_id = v_order_id;
    
    -- อัปเดต customer debt
    UPDATE customers
    SET current_debt = current_debt + v_total
    WHERE customer_id = p_customer_id;
    
    RETURN jsonb_build_object(
        'success', true,
        'order_id', v_order_id,
        'total_amount', v_total,
        'status', 'confirmed'
    );
    
EXCEPTION
    WHEN OTHERS THEN
        -- Transaction จะถูก rollback โดยอัตโนมัติ
        RETURN jsonb_build_object(
            'success', false,
            'error', SQLERRM
        );
END;
$$ LANGUAGE plpgsql;

-- ทดสอบ
BEGIN;
SELECT create_order(
    1,
    '[{"product_id": 1, "quantity": 2}, {"product_id": 2, "quantity": 1}]'::JSONB
);
COMMIT;
```

---

## 12. Transaction Monitoring และ Debugging

```sql
-- ดูสถานะ transactions ทั้งหมด
SELECT 
    pid,
    usename,
    application_name,
    client_addr,
    backend_start,
    xact_start,
    query_start,
    state,
    wait_event_type,
    wait_event,
    now() - xact_start AS transaction_duration,
    left(query, 100) AS current_query
FROM pg_stat_activity
WHERE state != 'idle'
ORDER BY transaction_duration DESC NULLS LAST;

-- ดู transaction ที่กำลัง waiting
SELECT 
    pid,
    usename,
    now() - xact_start AS waiting_duration,
    wait_event_type,
    wait_event,
    query
FROM pg_stat_activity
WHERE wait_event IS NOT NULL
  AND xact_start IS NOT NULL
ORDER BY waiting_duration DESC;

-- Statistics ของ transactions
SELECT 
    datname AS database,
    xact_commit AS committed_transactions,
    xact_rollback AS rolled_back_transactions,
    ROUND(
        xact_rollback::NUMERIC / NULLIF(xact_commit + xact_rollback, 0) * 100, 2
    ) AS rollback_percentage,
    deadlocks
FROM pg_stat_database
WHERE datname NOT IN ('postgres', 'template0', 'template1')
ORDER BY xact_commit + xact_rollback DESC;

-- Log slow transactions (ต้องตั้งใน postgresql.conf)
-- log_min_duration_statement = 5000  -- log statements > 5 seconds

-- ดู lock waits
SELECT 
    l1.pid AS waiting_pid,
    a1.usename AS waiting_user,
    l1.relation::regclass AS waiting_on_table,
    l1.locktype AS lock_type,
    l2.pid AS holding_pid,
    a2.usename AS holding_user,
    left(a2.query, 100) AS holding_query
FROM pg_locks l1
JOIN pg_locks l2 ON l1.relation = l2.relation 
                 AND l2.granted = TRUE
                 AND l1.granted = FALSE
JOIN pg_stat_activity a1 ON l1.pid = a1.pid
JOIN pg_stat_activity a2 ON l2.pid = a2.pid;
```

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: Basic Transaction

สร้าง transaction สำหรับการโอนคะแนน loyalty points ระหว่างสมาชิก:

```sql
-- คำตอบ
CREATE TABLE loyalty_points (
    member_id   INT PRIMARY KEY,
    points      INT NOT NULL DEFAULT 0 CHECK (points >= 0),
    updated_at  TIMESTAMP DEFAULT NOW()
);

INSERT INTO loyalty_points VALUES (1, 1000, NOW()), (2, 500, NOW());

CREATE OR REPLACE FUNCTION transfer_points(
    from_member INT,
    to_member   INT,
    points      INT
) RETURNS TEXT AS $$
BEGIN
    IF points <= 0 THEN
        RAISE EXCEPTION 'Transfer points must be positive';
    END IF;
    
    UPDATE loyalty_points 
    SET points = points - $3, updated_at = NOW()
    WHERE member_id = from_member;
    
    IF NOT FOUND THEN
        RAISE EXCEPTION 'From member % not found', from_member;
    END IF;
    
    UPDATE loyalty_points 
    SET points = points + $3, updated_at = NOW()
    WHERE member_id = to_member;
    
    IF NOT FOUND THEN
        RAISE EXCEPTION 'To member % not found', to_member;
    END IF;
    
    RETURN format('Transferred %s points from member %s to member %s', 
                   $3, from_member, to_member);
END;
$$ LANGUAGE plpgsql;

BEGIN;
SELECT transfer_points(1, 2, 200);
COMMIT;
```

### แบบฝึกหัดที่ 2: Transaction with Error Handling

สร้าง transaction ที่จัดการ error ได้อย่างถูกต้อง:

```sql
-- คำตอบ
CREATE OR REPLACE FUNCTION safe_insert_order(
    p_customer_id INT,
    p_product_ids INT[]
) RETURNS JSONB AS $$
DECLARE
    v_order_id INT;
    v_product_id INT;
    v_errors TEXT[] := ARRAY[]::TEXT[];
BEGIN
    INSERT INTO orders (customer_id, status) 
    VALUES (p_customer_id, 'pending')
    RETURNING order_id INTO v_order_id;
    
    FOREACH v_product_id IN ARRAY p_product_ids
    LOOP
        BEGIN
            INSERT INTO order_items (order_id, product_id, quantity, unit_price)
            SELECT v_order_id, v_product_id, 1, price
            FROM products WHERE product_id = v_product_id;
            
            IF NOT FOUND THEN
                v_errors := array_append(v_errors, 
                    format('Product %s not found', v_product_id));
            END IF;
        EXCEPTION WHEN OTHERS THEN
            v_errors := array_append(v_errors, SQLERRM);
        END;
    END LOOP;
    
    IF array_length(v_errors, 1) > 0 THEN
        RAISE EXCEPTION 'Errors: %', array_to_string(v_errors, '; ');
    END IF;
    
    RETURN jsonb_build_object('order_id', v_order_id, 'status', 'success');
EXCEPTION
    WHEN OTHERS THEN
        RETURN jsonb_build_object('error', SQLERRM);
END;
$$ LANGUAGE plpgsql;
```

### แบบฝึกหัดที่ 3: Transaction Isolation Demo

สาธิต autocommit ด้วย session สองอัน:

```sql
-- Session 1:
-- คำตอบ
CREATE TABLE isolation_demo (
    id    INT PRIMARY KEY,
    value TEXT,
    created_at TIMESTAMP DEFAULT NOW()
);

-- Session 1: เริ่ม transaction
BEGIN;
INSERT INTO isolation_demo VALUES (1, 'from session 1');
-- ยังไม่ COMMIT

-- Session 2: ลองอ่าน (ในโหมด READ COMMITTED จะไม่เห็น)
SELECT * FROM isolation_demo WHERE id = 1;  -- ไม่เห็น!

-- Session 1: COMMIT
COMMIT;

-- Session 2: ลองอ่านอีกครั้ง (ตอนนี้เห็นแล้ว)
SELECT * FROM isolation_demo WHERE id = 1;  -- เห็นแล้ว!
```

### แบบฝึกหัดที่ 4: Long Transaction Detection

เขียน script เพื่อตรวจจับและ kill long transactions:

```sql
-- คำตอบ
CREATE OR REPLACE FUNCTION kill_long_transactions(
    max_duration INTERVAL DEFAULT '30 minutes',
    dry_run BOOLEAN DEFAULT TRUE
) RETURNS TABLE(
    pid INT,
    username TEXT,
    duration INTERVAL,
    action TEXT
) AS $$
BEGIN
    RETURN QUERY
    SELECT 
        a.pid,
        a.usename::TEXT,
        now() - a.xact_start AS duration,
        CASE 
            WHEN dry_run THEN 'Would terminate'
            ELSE CASE 
                WHEN pg_terminate_backend(a.pid) THEN 'Terminated'
                ELSE 'Failed to terminate'
            END
        END AS action
    FROM pg_stat_activity a
    WHERE a.xact_start IS NOT NULL
      AND now() - a.xact_start > max_duration
      AND a.pid != pg_backend_pid()  -- ไม่ kill ตัวเอง
    ORDER BY duration DESC;
END;
$$ LANGUAGE plpgsql;

-- ดูว่าจะ kill อะไรบ้าง (dry run):
SELECT * FROM kill_long_transactions('5 minutes', TRUE);

-- Kill จริง:
-- SELECT * FROM kill_long_transactions('5 minutes', FALSE);
```

### แบบฝึกหัดที่ 5: Transaction Retry Pattern

```sql
-- คำตอบ
CREATE OR REPLACE FUNCTION retry_transaction(
    p_max_retries INT DEFAULT 3,
    p_backoff_ms INT DEFAULT 100
) RETURNS TEXT AS $$
DECLARE
    v_attempt INT := 0;
    v_success BOOLEAN := FALSE;
    v_result TEXT;
BEGIN
    WHILE v_attempt < p_max_retries AND NOT v_success LOOP
        v_attempt := v_attempt + 1;
        
        BEGIN
            -- Simulated work that might fail
            UPDATE accounts 
            SET balance = balance - 100,
                updated_at = NOW()
            WHERE account_id = 1;
            
            IF NOT FOUND THEN
                RAISE EXCEPTION 'Account not found';
            END IF;
            
            v_success := TRUE;
            v_result := format('Success on attempt %s', v_attempt);
            
        EXCEPTION
            WHEN serialization_failure OR deadlock_detected THEN
                IF v_attempt < p_max_retries THEN
                    PERFORM pg_sleep(p_backoff_ms * v_attempt / 1000.0);
                    RAISE NOTICE 'Retry % after %, error: %', v_attempt, SQLERRM;
                ELSE
                    v_result := format('Failed after %s attempts: %s', v_attempt, SQLERRM);
                END IF;
            WHEN OTHERS THEN
                RAISE;  -- ไม่ retry สำหรับ errors อื่น
        END;
    END LOOP;
    
    RETURN v_result;
END;
$$ LANGUAGE plpgsql;
```

### แบบฝึกหัดที่ 6: Batch Processing with Transactions

```sql
-- คำตอบ
CREATE OR REPLACE PROCEDURE process_pending_orders(
    p_batch_size INT DEFAULT 100
)
LANGUAGE plpgsql AS $$
DECLARE
    v_processed INT := 0;
    v_batch_count INT := 0;
    r RECORD;
BEGIN
    FOR r IN 
        SELECT order_id FROM orders 
        WHERE status = 'pending' 
        ORDER BY created_at
        LIMIT p_batch_size
        FOR UPDATE SKIP LOCKED
    LOOP
        UPDATE orders SET status = 'processing', updated_at = NOW()
        WHERE order_id = r.order_id;
        
        -- Process order logic here...
        
        UPDATE orders SET status = 'completed', updated_at = NOW()
        WHERE order_id = r.order_id;
        
        v_processed := v_processed + 1;
    END LOOP;
    
    COMMIT;
    
    RAISE NOTICE 'Processed % orders', v_processed;
END;
$$;
```

### แบบฝึกหัดที่ 7: Two-Phase Commit Example

```sql
-- คำตอบ
-- Step 1: เริ่ม transaction และทำงาน
BEGIN;
    INSERT INTO orders (customer_id, total) VALUES (1, 5000);
    UPDATE inventory SET quantity = quantity - 1 WHERE product_id = 101;

-- Step 2: Prepare (suspend transaction)
PREPARE TRANSACTION 'order_txn_001';

-- ระหว่างนี้: ตรวจสอบกับระบบอื่น (payment gateway, etc.)

-- Step 3a: ถ้าทุกระบบพร้อม → Commit
COMMIT PREPARED 'order_txn_001';

-- หรือ Step 3b: ถ้ามีปัญหา → Rollback  
-- ROLLBACK PREPARED 'order_txn_001';

-- ตรวจสอบ prepared transactions ที่ยังค้างอยู่
SELECT gid, prepared, owner, database
FROM pg_prepared_xacts
ORDER BY prepared;
```

### แบบฝึกหัดที่ 8: MySQL Transactions

```sql
-- คำตอบ (MySQL syntax)
-- สร้าง stored procedure สำหรับ payment processing
-- DELIMITER $$
-- CREATE PROCEDURE process_payment(
--     IN p_order_id INT,
--     IN p_amount DECIMAL(10,2),
--     OUT p_result VARCHAR(200)
-- )
-- BEGIN
--     DECLARE EXIT HANDLER FOR SQLEXCEPTION
--     BEGIN
--         ROLLBACK;
--         GET DIAGNOSTICS CONDITION 1
--             p_result = MESSAGE_TEXT;
--     END;
--     
--     START TRANSACTION;
--     
--     UPDATE orders SET status = 'paid' WHERE order_id = p_order_id;
--     INSERT INTO payments (order_id, amount) VALUES (p_order_id, p_amount);
--     UPDATE customers SET total_purchases = total_purchases + p_amount 
--         WHERE customer_id = (SELECT customer_id FROM orders WHERE order_id = p_order_id);
--     
--     COMMIT;
--     SET p_result = 'Payment processed successfully';
-- END$$
-- DELIMITER ;

-- PostgreSQL equivalent:
CREATE OR REPLACE FUNCTION process_payment_pg(
    p_order_id INT,
    p_amount DECIMAL(10,2)
) RETURNS TEXT AS $$
BEGIN
    UPDATE orders SET status = 'paid' WHERE order_id = p_order_id;
    INSERT INTO payments (order_id, amount) VALUES (p_order_id, p_amount);
    UPDATE customers SET total_purchases = total_purchases + p_amount 
    WHERE customer_id = (SELECT customer_id FROM orders WHERE order_id = p_order_id);
    
    RETURN 'Payment processed successfully';
EXCEPTION
    WHEN OTHERS THEN
        RAISE;
END;
$$ LANGUAGE plpgsql;
```

### แบบฝึกหัดที่ 9: Transaction Statistics Dashboard

```sql
-- คำตอบ
CREATE OR REPLACE VIEW transaction_stats AS
WITH stats AS (
    SELECT 
        datname,
        xact_commit,
        xact_rollback,
        blks_read,
        blks_hit,
        tup_returned,
        tup_fetched,
        tup_inserted,
        tup_updated,
        tup_deleted,
        deadlocks,
        stats_reset
    FROM pg_stat_database
    WHERE datname = current_database()
)
SELECT
    datname AS database_name,
    xact_commit AS total_commits,
    xact_rollback AS total_rollbacks,
    ROUND(xact_rollback::NUMERIC / NULLIF(xact_commit + xact_rollback, 0) * 100, 4) 
        AS rollback_pct,
    ROUND(blks_hit::NUMERIC / NULLIF(blks_hit + blks_read, 0) * 100, 2) 
        AS cache_hit_pct,
    tup_inserted AS rows_inserted,
    tup_updated AS rows_updated,
    tup_deleted AS rows_deleted,
    deadlocks,
    stats_reset
FROM stats;

SELECT * FROM transaction_stats;
```

### แบบฝึกหัดที่ 10: Complete Transaction System Test

```sql
-- คำตอบ
CREATE OR REPLACE FUNCTION run_transaction_tests()
RETURNS TABLE(test TEXT, status TEXT, message TEXT) AS $$
BEGIN
    -- Test 1: Basic commit
    test := 'Basic COMMIT';
    BEGIN
        CREATE TEMP TABLE txn_test (id INT, val TEXT) ON COMMIT DROP;
        INSERT INTO txn_test VALUES (1, 'test');
        status := 'PASS';
        message := 'Transaction committed successfully';
    EXCEPTION WHEN OTHERS THEN
        status := 'FAIL';
        message := SQLERRM;
    END;
    RETURN NEXT;
    
    -- Test 2: Rollback on error
    test := 'ROLLBACK on error';
    BEGIN
        BEGIN
            INSERT INTO txn_test VALUES (1, 'duplicate');
            INSERT INTO txn_test VALUES (1, 'duplicate again');
        EXCEPTION WHEN unique_violation THEN
            status := 'PASS';
            message := 'Rollback triggered correctly on unique violation';
        END;
    EXCEPTION WHEN OTHERS THEN
        status := 'FAIL';
        message := SQLERRM;
    END;
    RETURN NEXT;
    
    -- Test 3: Isolation
    test := 'READ COMMITTED isolation';
    BEGIN
        status := 'PASS';
        message := 'PostgreSQL default READ COMMITTED prevents dirty reads';
        RETURN NEXT;
    END;
    
    -- Test 4: Autocommit behavior
    test := 'Autocommit mode';
    BEGIN
        CREATE TEMP TABLE auto_test (id SERIAL, val TEXT);
        INSERT INTO auto_test (val) VALUES ('autocommitted');
        -- ถ้าไม่มี BEGIN → commit ทันที
        status := 'PASS';
        message := 'Single statements autocommit correctly';
    EXCEPTION WHEN OTHERS THEN
        status := 'FAIL';
        message := SQLERRM;
    END;
    RETURN NEXT;
END;
$$ LANGUAGE plpgsql;

SELECT * FROM run_transaction_tests();
```

---

## สรุป

ใน Part นี้เราได้เรียนรู้:

1. **Implicit vs Explicit Transactions** - ความแตกต่างระหว่าง autocommit และ explicit transaction blocks
2. **BEGIN/START TRANSACTION** - วิธีเริ่ม transaction ในแต่ละ database
3. **COMMIT** - การยืนยัน transaction รวมถึง Two-Phase Commit
4. **ROLLBACK** - การยกเลิก transaction และ ROLLBACK TO SAVEPOINT
5. **Error Handling** - รูปแบบต่างๆ ในการจัดการ errors ใน transactions
6. **Long-Running Transactions** - อันตรายและวิธีป้องกัน
7. **Database-specific features** - PostgreSQL, MySQL, SQLite, SQL Server

ใน Part 73 เราจะเรียนรู้เรื่อง Savepoints อย่างละเอียด ซึ่งช่วยให้เราสามารถทำ partial rollbacks ภายใน transaction ได้
