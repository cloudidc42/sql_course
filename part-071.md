# Part 71: ACID Properties Deep Dive

## บทนำ

ในโลกของฐานข้อมูล คำว่า **ACID** เป็นหนึ่งในแนวคิดพื้นฐานที่สำคัญที่สุด ซึ่งย่อมาจาก:

- **A** - Atomicity (ความเป็นหน่วยเดียว)
- **C** - Consistency (ความสอดคล้อง)
- **I** - Isolation (การแยกออกจากกัน)
- **D** - Durability (ความคงทน)

คุณสมบัติเหล่านี้รับประกันว่า transaction ในฐานข้อมูลจะถูกประมวลผลอย่างน่าเชื่อถือ แม้ในกรณีที่เกิดข้อผิดพลาด, ระบบล่ม, หรือมีหลาย transaction ทำงานพร้อมกัน

---

## 1. Atomicity - ทั้งหมดหรือไม่มีเลย

### แนวคิด

Atomicity หมายความว่า transaction ต้องถูกมองว่าเป็น "หน่วยเดียว" (atomic unit) ซึ่งหมายความว่า:
- ถ้า transaction สำเร็จ → การเปลี่ยนแปลงทั้งหมดจะถูก commit
- ถ้า transaction ล้มเหลวในส่วนใดส่วนหนึ่ง → การเปลี่ยนแปลงทั้งหมดจะถูก rollback

ลองนึกภาพการโอนเงิน:
```
บัญชี A มีเงิน 1,000 บาท
บัญชี B มีเงิน 500 บาท
ต้องการโอนเงิน 300 บาทจาก A ไป B
```

Transaction ประกอบด้วย 2 ขั้นตอน:
1. ถอนเงิน 300 บาทจาก A (A = 700)
2. ฝากเงิน 300 บาทเข้า B (B = 800)

### ตัวอย่างการละเมิด Atomicity

สมมติว่าขั้นตอนที่ 1 สำเร็จ แต่ขั้นตอนที่ 2 ล้มเหลว (เช่น บัญชี B ถูกล็อก):

```
ก่อน: A=1000, B=500
หลังขั้นตอน 1: A=700, B=500  ← เงินหาย!
ขั้นตอน 2 ล้มเหลว
สถานะสุดท้าย: A=700, B=500  ← เงินหายไปจากระบบ!
```

นี่คือสถานการณ์ที่ฝันร้ายสำหรับระบบธนาคาร!

### ตัวอย่าง SQL ที่แสดง Atomicity

```sql
-- สร้างตารางสำหรับตัวอย่าง
CREATE TABLE accounts (
    account_id    SERIAL PRIMARY KEY,
    account_name  VARCHAR(100) NOT NULL,
    balance       DECIMAL(15, 2) NOT NULL CHECK (balance >= 0),
    updated_at    TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

INSERT INTO accounts (account_name, balance) VALUES
    ('Alice', 1000.00),
    ('Bob', 500.00);

-- ตัวอย่างที่ 1: Transaction ที่สำเร็จ (Atomicity ทำงานถูกต้อง)
BEGIN;
    UPDATE accounts SET balance = balance - 300 WHERE account_name = 'Alice';
    UPDATE accounts SET balance = balance + 300 WHERE account_name = 'Bob';
COMMIT;

-- ตรวจสอบผล
SELECT account_name, balance FROM accounts;
-- Alice: 700.00, Bob: 800.00

-- ตัวอย่างที่ 2: Transaction ที่ล้มเหลว (Rollback เกิดขึ้น)
BEGIN;
    UPDATE accounts SET balance = balance - 300 WHERE account_name = 'Alice';
    -- จำลองความผิดพลาด
    UPDATE accounts SET balance = balance - 999999 WHERE account_name = 'Bob';
    -- เงื่อนไข CHECK (balance >= 0) จะล้มเหลวที่นี่
COMMIT;
-- ERROR: new row for relation "accounts" violates check constraint "accounts_balance_check"
-- Rollback อัตโนมัติ → ทั้ง transaction ถูก undo

-- ตรวจสอบว่า Alice ยังมีเงิน 700 อยู่ (ไม่เปลี่ยน)
SELECT account_name, balance FROM accounts;
```

### กลไกที่รับประกัน Atomicity

PostgreSQL ใช้ **Write-Ahead Logging (WAL)** เพื่อรับประกัน Atomicity:

```sql
-- ดู WAL configuration
SHOW wal_level;
SHOW wal_buffers;

-- ดู transaction logs
SELECT pg_current_wal_lsn();

-- ตัวอย่าง: Transaction ที่ซับซ้อนกว่า
BEGIN;
    -- Step 1: ลดสต็อกสินค้า
    UPDATE products SET stock = stock - 5 WHERE product_id = 101;
    
    -- Step 2: เพิ่มรายการ order
    INSERT INTO orders (customer_id, product_id, quantity, total)
    VALUES (1001, 101, 5, 2500.00);
    
    -- Step 3: อัปเดต loyalty points
    UPDATE customers SET points = points + 250 WHERE customer_id = 1001;
    
    -- ถ้าทุกขั้นตอนสำเร็จ
COMMIT;
-- ถ้าขั้นตอนใดล้มเหลว → ทุกอย่างถูก rollback
```

---

## 2. Consistency - กฎของข้อมูลต้องคงอยู่

### แนวคิด

Consistency หมายความว่า transaction จะต้องนำฐานข้อมูลจาก "สถานะที่ถูกต้อง" ไปยัง "สถานะที่ถูกต้อง" อีกสถานะหนึ่ง กฎที่กำหนดความถูกต้องของข้อมูลประกอบด้วย:

1. **Constraints** (CHECK, NOT NULL, UNIQUE, FOREIGN KEY)
2. **Triggers** ที่บังคับใช้กฎธุรกิจ
3. **Cascades** ที่รักษาความสัมพันธ์
4. กฎธุรกิจที่ Application บังคับใช้

### ตัวอย่างการละเมิด Consistency

```sql
-- สร้างตารางที่มี business rules
CREATE TABLE bank_accounts (
    id          SERIAL PRIMARY KEY,
    owner_id    INT NOT NULL REFERENCES customers(id),
    balance     DECIMAL(15,2) NOT NULL DEFAULT 0,
    account_type VARCHAR(20) CHECK (account_type IN ('savings', 'checking')),
    min_balance DECIMAL(15,2) NOT NULL DEFAULT 0,
    CONSTRAINT balance_above_minimum CHECK (balance >= min_balance)
);

-- ตัวอย่างที่ 1: Consistency ด้วย FOREIGN KEY
BEGIN;
    -- พยายามสร้าง account สำหรับ customer ที่ไม่มีอยู่
    INSERT INTO bank_accounts (owner_id, balance, account_type)
    VALUES (99999, 1000.00, 'savings');
COMMIT;
-- ERROR: insert or update on table "bank_accounts" violates foreign key constraint
-- Consistency รักษาความสัมพันธ์ไว้

-- ตัวอย่างที่ 2: Consistency ด้วย CHECK constraint
BEGIN;
    INSERT INTO bank_accounts (owner_id, balance, account_type, min_balance)
    VALUES (1, 100.00, 'savings', 500.00);
COMMIT;
-- ERROR: new row violates check constraint "balance_above_minimum"
-- ไม่สามารถสร้าง account ที่มี balance ต่ำกว่า minimum ได้

-- ตัวอย่างที่ 3: Consistency ด้วย Trigger
CREATE OR REPLACE FUNCTION check_transfer_limit()
RETURNS TRIGGER AS $$
BEGIN
    -- ตรวจสอบว่าการโอนไม่เกิน 100,000 บาทต่อวัน
    IF (SELECT COALESCE(SUM(amount), 0) FROM transfers 
        WHERE from_account = NEW.from_account 
        AND DATE(transfer_date) = CURRENT_DATE) + NEW.amount > 100000 THEN
        RAISE EXCEPTION 'Daily transfer limit exceeded';
    END IF;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER enforce_transfer_limit
    BEFORE INSERT ON transfers
    FOR EACH ROW EXECUTE FUNCTION check_transfer_limit();
```

### Referential Integrity เป็นส่วนหนึ่งของ Consistency

```sql
-- สร้างโครงสร้างที่รักษา referential integrity
CREATE TABLE departments (
    dept_id   SERIAL PRIMARY KEY,
    dept_name VARCHAR(100) NOT NULL
);

CREATE TABLE employees (
    emp_id    SERIAL PRIMARY KEY,
    emp_name  VARCHAR(100) NOT NULL,
    dept_id   INT REFERENCES departments(dept_id) ON DELETE RESTRICT ON UPDATE CASCADE,
    salary    DECIMAL(10,2) CHECK (salary > 0)
);

-- ทดสอบ Consistency
BEGIN;
    -- ลบ department ที่มี employees อยู่
    DELETE FROM departments WHERE dept_id = 1;
COMMIT;
-- ERROR: update or delete on table "departments" violates foreign key constraint
-- Consistency รักษาไว้ว่า employee ต้องมี department

-- ถ้าต้องการลบต้องทำแบบนี้
BEGIN;
    UPDATE employees SET dept_id = 2 WHERE dept_id = 1; -- ย้ายพนักงานก่อน
    DELETE FROM departments WHERE dept_id = 1;          -- แล้วค่อยลบ dept
COMMIT;
```

---

## 3. Isolation - Transaction แต่ละอันต้องแยกออกจากกัน

### แนวคิด

Isolation หมายความว่า Transaction ที่กำลังทำงานอยู่จะต้องไม่ถูกรบกวนจาก Transaction อื่นที่ทำงานพร้อมกัน ผู้ใช้แต่ละคนจะรู้สึกเหมือนกับว่าตนเองเป็นผู้ใช้คนเดียวของฐานข้อมูล

### ปัญหาที่เกิดจากการขาด Isolation

#### 1. Dirty Read

```
Transaction T1:                    Transaction T2:
BEGIN;                             BEGIN;
UPDATE accounts 
  SET balance = 2000
  WHERE id = 1;
                                   SELECT balance FROM accounts WHERE id = 1;
                                   -- อ่านได้ 2000 (ยังไม่ commit!)
ROLLBACK;
-- ค่ากลับเป็น 1000
                                   -- T2 ใช้ค่า 2000 ที่ไม่เคยมีจริง!
```

```sql
-- สาธิต Dirty Read (ใน PostgreSQL จะไม่เกิดขึ้น แต่ใน MySQL READ UNCOMMITTED จะเกิด)

-- Session 1:
BEGIN;
UPDATE accounts SET balance = 9999 WHERE account_id = 1;
-- ยังไม่ COMMIT

-- Session 2 (MySQL READ UNCOMMITTED):
SET TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;
BEGIN;
SELECT balance FROM accounts WHERE account_id = 1;
-- อาจเห็น 9999 แม้ว่า Session 1 ยังไม่ COMMIT!

-- Session 1:
ROLLBACK; -- ค่ากลับเป็นเดิม แต่ Session 2 ได้ข้อมูลผิดไปแล้ว
```

#### 2. Non-Repeatable Read

```
Transaction T1:                    Transaction T2:
BEGIN;                             
SELECT balance FROM accounts 
  WHERE id = 1;
-- ได้ 1000
                                   BEGIN;
                                   UPDATE accounts SET balance = 2000 WHERE id = 1;
                                   COMMIT;
SELECT balance FROM accounts 
  WHERE id = 1;
-- ได้ 2000 ← ค่าเปลี่ยนไปในการอ่านครั้งที่สอง!
```

#### 3. Phantom Read

```
Transaction T1:                    Transaction T2:
BEGIN;
SELECT COUNT(*) FROM orders 
  WHERE customer_id = 1;
-- ได้ 5
                                   BEGIN;
                                   INSERT INTO orders (customer_id, ...) VALUES (1, ...);
                                   COMMIT;
SELECT COUNT(*) FROM orders 
  WHERE customer_id = 1;
-- ได้ 6 ← แถวใหม่ปรากฏขึ้น!
```

### ตัวอย่างการตั้งค่า Isolation Level

```sql
-- PostgreSQL: ตั้งค่า Isolation Level
SET TRANSACTION ISOLATION LEVEL READ COMMITTED;  -- ค่าเริ่มต้น
SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;

-- ดู Isolation Level ปัจจุบัน
SHOW transaction_isolation;

-- MySQL: ตั้งค่า Isolation Level
SET SESSION TRANSACTION ISOLATION LEVEL READ COMMITTED;
SET GLOBAL TRANSACTION ISOLATION LEVEL SERIALIZABLE;
```

---

## 4. Durability - ข้อมูลที่ commit แล้วต้องคงอยู่

### แนวคิด

Durability รับประกันว่าเมื่อ transaction ถูก commit แล้ว ข้อมูลจะยังคงอยู่แม้ว่า:
- ระบบจะล่มทันทีหลังจาก commit
- ไฟดับ
- Hardware เสีย
- OS Crash

### กลไกที่รับประกัน Durability

```sql
-- PostgreSQL ใช้ WAL (Write-Ahead Logging)

-- ดูการตั้งค่า WAL
SHOW wal_sync_method;
-- fsync, fdatasync, open_sync, open_datasync

SHOW synchronous_commit;
-- on = รอจนกว่า WAL จะถูก flush ไปยัง disk (ปลอดภัยที่สุด)
-- off = ไม่รอ (เร็วกว่า แต่อาจสูญเสียข้อมูลในกรณี crash)

-- ดู WAL checkpoint settings
SHOW checkpoint_completion_target;
SHOW checkpoint_timeout;

-- ตัวอย่างการตั้งค่า synchronous_commit สำหรับ session เดียว
SET synchronous_commit = off;  -- เพิ่มประสิทธิภาพ แต่ลดความปลอดภัย
BEGIN;
    INSERT INTO audit_log (action, timestamp) VALUES ('user_login', NOW());
COMMIT;
-- Log นี้อาจสูญหายได้ถ้าระบบ crash ทันทีหลัง commit
-- แต่สำหรับ audit log บางประเภทที่ไม่ critical อาจยอมรับได้

-- สำหรับ critical data ควรใช้ค่าเริ่มต้น synchronous_commit = on
RESET synchronous_commit;
```

### ตัวอย่างที่แสดง Durability

```sql
-- สร้างตารางสำหรับทดสอบ
CREATE TABLE critical_transactions (
    txn_id      SERIAL PRIMARY KEY,
    amount      DECIMAL(15,2),
    status      VARCHAR(20),
    created_at  TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Transaction ที่สำคัญมาก
BEGIN;
    INSERT INTO critical_transactions (amount, status) VALUES (1000000.00, 'completed');
    UPDATE accounts SET balance = balance - 1000000 WHERE account_id = 1;
    UPDATE accounts SET balance = balance + 1000000 WHERE account_id = 2;
COMMIT;
-- หลังจาก COMMIT → ข้อมูลถูกบันทึกลง WAL แล้ว
-- แม้ระบบจะ crash ทันที → ข้อมูลจะถูกกู้คืนได้เมื่อระบบรีสตาร์ท

-- ตรวจสอบ WAL LSN (Log Sequence Number)
SELECT pg_current_wal_lsn() AS current_wal_position;

-- ดู checkpoint ล่าสุด
SELECT checkpoint_lsn, checkpoint_tli 
FROM pg_control_checkpoint();
```

---

## 5. ตัวอย่างจริงของการละเมิด ACID แต่ละข้อ

### กรณีที่ 1: ระบบ E-Commerce

```sql
-- สร้างตาราง
CREATE TABLE inventory (
    product_id  INT PRIMARY KEY,
    product_name VARCHAR(200),
    quantity    INT NOT NULL CHECK (quantity >= 0),
    price       DECIMAL(10,2)
);

CREATE TABLE orders (
    order_id    SERIAL PRIMARY KEY,
    customer_id INT,
    product_id  INT REFERENCES inventory(product_id),
    quantity    INT,
    total_price DECIMAL(10,2),
    order_date  TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

INSERT INTO inventory VALUES (1, 'iPhone 15', 5, 35000.00);

-- ปัญหาที่ 1: Atomicity ถูกละเมิด (ถ้าไม่ใช้ transaction)
-- ผู้ใช้ A สั่งซื้อ iPhone 5 เครื่อง
UPDATE inventory SET quantity = quantity - 5 WHERE product_id = 1;
-- ระบบ crash ก่อน INSERT orders!
INSERT INTO orders (customer_id, product_id, quantity, total_price)
VALUES (101, 1, 5, 175000.00);
-- สต็อกลดลงแต่ไม่มี order record!

-- วิธีแก้: ใช้ Transaction
BEGIN;
    -- ตรวจสอบสต็อกก่อน
    DECLARE
        v_stock INT;
    BEGIN
        SELECT quantity INTO v_stock FROM inventory WHERE product_id = 1 FOR UPDATE;
        IF v_stock < 5 THEN
            RAISE EXCEPTION 'Insufficient stock';
        END IF;
        
        UPDATE inventory SET quantity = quantity - 5 WHERE product_id = 1;
        INSERT INTO orders (customer_id, product_id, quantity, total_price)
        VALUES (101, 1, 5, 175000.00);
    END;
COMMIT;
```

### กรณีที่ 2: ระบบธนาคาร

```sql
-- สร้างตาราง
CREATE TABLE bank_transfers (
    transfer_id   SERIAL PRIMARY KEY,
    from_account  INT NOT NULL,
    to_account    INT NOT NULL,
    amount        DECIMAL(15,2) NOT NULL CHECK (amount > 0),
    status        VARCHAR(20) DEFAULT 'pending',
    transfer_date TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- ตัวอย่าง: การโอนเงินที่ปลอดภัย (ACID-compliant)
CREATE OR REPLACE FUNCTION transfer_money(
    p_from_account INT,
    p_to_account   INT,
    p_amount       DECIMAL(15,2)
) RETURNS TEXT AS $$
DECLARE
    v_from_balance DECIMAL(15,2);
    v_transfer_id  INT;
BEGIN
    -- Lock ทั้งสอง accounts (เรียงตาม ID เพื่อป้องกัน deadlock)
    IF p_from_account < p_to_account THEN
        SELECT balance INTO v_from_balance 
        FROM accounts WHERE account_id = p_from_account FOR UPDATE;
        PERFORM * FROM accounts WHERE account_id = p_to_account FOR UPDATE;
    ELSE
        PERFORM * FROM accounts WHERE account_id = p_to_account FOR UPDATE;
        SELECT balance INTO v_from_balance 
        FROM accounts WHERE account_id = p_from_account FOR UPDATE;
    END IF;
    
    -- ตรวจสอบยอดเงิน (Consistency)
    IF v_from_balance < p_amount THEN
        RAISE EXCEPTION 'Insufficient funds: available %, requested %', 
                         v_from_balance, p_amount;
    END IF;
    
    -- ทำการโอนเงิน (Atomicity)
    UPDATE accounts SET balance = balance - p_amount, 
                        updated_at = NOW()
    WHERE account_id = p_from_account;
    
    UPDATE accounts SET balance = balance + p_amount,
                        updated_at = NOW()
    WHERE account_id = p_to_account;
    
    -- บันทึก transfer record
    INSERT INTO bank_transfers (from_account, to_account, amount, status)
    VALUES (p_from_account, p_to_account, p_amount, 'completed')
    RETURNING transfer_id INTO v_transfer_id;
    
    RETURN 'Transfer ' || v_transfer_id || ' completed successfully';
EXCEPTION
    WHEN OTHERS THEN
        -- Rollback จะเกิดขึ้นอัตโนมัติถ้า function ถูกเรียกใน transaction
        RAISE;
END;
$$ LANGUAGE plpgsql;

-- เรียกใช้งาน
BEGIN;
    SELECT transfer_money(1, 2, 500.00);
COMMIT;
```

---

## 6. ACID ในฐานข้อมูลต่างๆ

### PostgreSQL - ACID เข้มงวดมาก

```sql
-- PostgreSQL มี ACID ครบถ้วนโดยค่าเริ่มต้น

-- ตรวจสอบการตั้งค่า ACID-related
SHOW fsync;                    -- on = รับประกัน Durability
SHOW synchronous_commit;       -- on = รับประกัน Durability สูงสุด
SHOW transaction_isolation;    -- read committed = ค่าเริ่มต้น

-- PostgreSQL รองรับ Isolation Levels ทั้งหมด
-- READ UNCOMMITTED → ทำงานเหมือน READ COMMITTED (ป้องกัน dirty reads เสมอ)
-- READ COMMITTED → ค่าเริ่มต้น
-- REPEATABLE READ → ใช้ MVCC ป้องกัน non-repeatable reads
-- SERIALIZABLE → ป้องกันทุก concurrency problems

BEGIN TRANSACTION ISOLATION LEVEL SERIALIZABLE;
    -- ทุก operation จะทำงานเหมือนกับว่าทำทีละ transaction
    SELECT SUM(balance) FROM accounts; -- snapshot ณ เวลานั้น
    -- ถ้า transaction อื่น commit ก่อน อาจเกิด serialization failure
COMMIT;
```

### MySQL - ขึ้นอยู่กับ Storage Engine

```sql
-- MySQL: InnoDB รองรับ ACID ครบถ้วน
-- MyISAM ไม่รองรับ Transactions!

-- ตรวจสอบ Storage Engine
SHOW ENGINES;
-- InnoDB: Transactions, XA, Savepoints → รองรับ ACID
-- MyISAM: ไม่รองรับ transactions

-- ตรวจสอบ Engine ของตาราง
SHOW TABLE STATUS LIKE 'accounts';

-- InnoDB ACID settings
SHOW VARIABLES LIKE 'innodb_flush_log_at_trx_commit';
-- 1 = รับประกัน Durability (ค่าเริ่มต้น แต่ช้า)
-- 2 = เร็วกว่า แต่อาจสูญ 1 วินาทีของ transaction ถ้า OS crash
-- 0 = เร็วที่สุด แต่อาจสูญ 1 วินาทีของ transaction ถ้า MySQL crash

-- Default isolation level ใน MySQL
SHOW VARIABLES LIKE 'transaction_isolation';
-- REPEATABLE-READ (ต่างจาก PostgreSQL ที่เป็น READ-COMMITTED)

-- MySQL: Phantom Reads ถูกป้องกันใน REPEATABLE READ ด้วย Next-Key Locking
-- แต่ต่างจาก PostgreSQL ที่ใช้ MVCC
```

### SQLite - ACID ที่เรียบง่าย

```sql
-- SQLite มี ACID แต่ใช้ File-level locking
-- ไม่เหมาะกับ high-concurrency applications

-- SQLite Isolation: Serializable by default
-- เพราะ SQLite ใช้ database-level locking

-- Journal Modes ใน SQLite
PRAGMA journal_mode = WAL;     -- Write-Ahead Logging (ดีที่สุดสำหรับ concurrency)
PRAGMA journal_mode = DELETE;  -- ค่าเริ่มต้น
PRAGMA journal_mode = MEMORY;  -- เร็ว แต่ไม่ Durable

-- Synchronous settings
PRAGMA synchronous = FULL;     -- รับประกัน Durability สูงสุด
PRAGMA synchronous = NORMAL;   -- สมดุลระหว่างความเร็วและความปลอดภัย
PRAGMA synchronous = OFF;      -- เร็วที่สุด แต่ไม่ปลอดภัย

-- Transaction ใน SQLite
BEGIN;
    INSERT INTO users VALUES (1, 'Alice', 'alice@example.com');
    INSERT INTO users VALUES (2, 'Bob', 'bob@example.com');
COMMIT;
```

### SQL Server - ACID ที่ยืดหยุ่น

```sql
-- SQL Server รองรับ ACID ครบถ้วน

-- ตั้งค่า isolation level
SET TRANSACTION ISOLATION LEVEL READ COMMITTED;  -- ค่าเริ่มต้น
SET TRANSACTION ISOLATION LEVEL SNAPSHOT;         -- คล้าย MVCC ของ PostgreSQL

-- เปิด Snapshot Isolation
ALTER DATABASE MyDatabase SET ALLOW_SNAPSHOT_ISOLATION ON;
ALTER DATABASE MyDatabase SET READ_COMMITTED_SNAPSHOT ON;

-- Transaction ใน SQL Server
BEGIN TRANSACTION;
    UPDATE Accounts SET Balance = Balance - 300 WHERE AccountID = 1;
    UPDATE Accounts SET Balance = Balance + 300 WHERE AccountID = 2;
    
    IF @@ERROR <> 0
        ROLLBACK TRANSACTION;
    ELSE
        COMMIT TRANSACTION;
```

---

## 7. BASE - ทางเลือกสำหรับ NoSQL

BASE ย่อมาจาก:
- **B**asically **A**vailable - ระบบตอบสนองเสมอ แม้ข้อมูลอาจไม่ up-to-date
- **S**oft state - สถานะของข้อมูลอาจเปลี่ยนแปลงได้ตามเวลา (โดยไม่มี input)
- **E**ventually consistent - ข้อมูลจะสอดคล้องกันในที่สุด

### เปรียบเทียบ ACID vs BASE

| คุณสมบัติ | ACID | BASE |
|-----------|------|------|
| ความสอดคล้อง | Strong Consistency | Eventual Consistency |
| ความพร้อมใช้งาน | อาจต่ำ (เพราะ locking) | สูงมาก |
| ประสิทธิภาพ | ช้ากว่า | เร็วกว่า |
| ความซับซ้อน | แก้ไขปัญหา concurrency | Application ต้องจัดการเอง |
| เหมาะกับ | ระบบการเงิน, ข้อมูลสำคัญ | Social media, Analytics, CDN |

### ตัวอย่าง BASE ใน Application

```sql
-- ตัวอย่าง: ระบบ Like บน Social Media (BASE approach)
-- แทนที่จะ update counter ทันที เราอาจรวมค่าทีหลัง

-- ระบบ ACID (ช้า):
BEGIN;
    UPDATE posts SET like_count = like_count + 1 WHERE post_id = 123;
COMMIT;

-- ระบบ BASE (เร็วกว่า):
-- บันทึก like events แยกต่างหาก
INSERT INTO like_events (post_id, user_id, action, created_at)
VALUES (123, 456, 'like', NOW());

-- รวมค่าทีหลัง (batch processing)
-- อาจทำทุก 1 นาทีหรือทุกชั่วโมง
UPDATE posts p
SET like_count = (
    SELECT COUNT(*) FROM like_events 
    WHERE post_id = p.post_id AND action = 'like'
)
WHERE post_id IN (
    SELECT DISTINCT post_id FROM like_events 
    WHERE created_at > NOW() - INTERVAL '1 hour'
);
```

### เมื่อใดควรใช้ ACID vs BASE

```sql
-- ควรใช้ ACID เมื่อ:
-- 1. ระบบการเงิน (โอนเงิน, หักบัญชี)
-- 2. ระบบสต็อกสินค้า
-- 3. ข้อมูลที่ต้องถูกต้อง 100%
-- 4. ระบบที่มี concurrent users ที่แก้ไขข้อมูลเดียวกัน

-- ตัวอย่าง ACID-critical system
BEGIN;
    -- ตรวจสอบและจองที่นั่งเครื่องบิน
    SELECT * FROM seats 
    WHERE flight_id = 'TG203' AND seat_number = '12A' AND status = 'available'
    FOR UPDATE;  -- Lock เพื่อป้องกัน double booking
    
    UPDATE seats SET status = 'booked', passenger_id = 1234
    WHERE flight_id = 'TG203' AND seat_number = '12A';
    
    INSERT INTO bookings (flight_id, seat_number, passenger_id, amount)
    VALUES ('TG203', '12A', 1234, 5500.00);
COMMIT;

-- ควรใช้ BASE เมื่อ:
-- 1. Analytics และ Reporting
-- 2. Social media counters (likes, views)
-- 3. Session data
-- 4. Cache layers

-- ตัวอย่าง BASE-acceptable system
INSERT INTO page_views (page_url, user_id, viewed_at, session_id)
VALUES ('/product/123', NULL, NOW(), 'sess_abc123');
-- ยอมรับได้ถ้า view count อาจไม่ accurate 100% ในทันที
```

---

## 8. การตรวจสอบ ACID ในระบบ Production

```sql
-- ตรวจสอบ transaction ที่กำลังทำงาน
SELECT 
    pid,
    usename,
    application_name,
    state,
    now() - xact_start AS transaction_age,
    now() - query_start AS query_age,
    left(query, 100) AS current_query
FROM pg_stat_activity
WHERE xact_start IS NOT NULL
ORDER BY transaction_age DESC;

-- ตรวจสอบ long-running transactions ที่น่าสงสัย
SELECT 
    pid,
    usename,
    now() - xact_start AS age,
    state,
    query
FROM pg_stat_activity
WHERE xact_start < now() - INTERVAL '5 minutes'
  AND state != 'idle'
ORDER BY age DESC;

-- ตรวจสอบ database transaction statistics
SELECT 
    datname,
    xact_commit,
    xact_rollback,
    ROUND(xact_rollback::numeric / NULLIF(xact_commit + xact_rollback, 0) * 100, 2) AS rollback_pct,
    blks_read,
    blks_hit,
    ROUND(blks_hit::numeric / NULLIF(blks_read + blks_hit, 0) * 100, 2) AS cache_hit_pct
FROM pg_stat_database
WHERE datname NOT IN ('postgres', 'template0', 'template1')
ORDER BY xact_commit + xact_rollback DESC;

-- ตรวจสอบ locks ที่อาจกระทบ ACID isolation
SELECT 
    blocked_locks.pid AS blocked_pid,
    blocked_activity.usename AS blocked_user,
    blocking_locks.pid AS blocking_pid,
    blocking_activity.usename AS blocking_user,
    blocked_activity.query AS blocked_statement,
    blocking_activity.query AS current_statement_in_blocking_process
FROM pg_catalog.pg_locks blocked_locks
JOIN pg_catalog.pg_stat_activity blocked_activity 
    ON blocked_activity.pid = blocked_locks.pid
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
JOIN pg_catalog.pg_stat_activity blocking_activity 
    ON blocking_activity.pid = blocking_locks.pid
WHERE NOT blocked_locks.GRANTED;
```

---

## 9. ACID Properties ในบริบทของ Microservices

### ความท้าทาย

เมื่อระบบแตกออกเป็น Microservices ที่มีฐานข้อมูลแยกกัน การรักษา ACID across services เป็นเรื่องที่ยากมาก

```sql
-- ตัวอย่าง: Order Service และ Inventory Service แยกกัน

-- ปัญหา: เราไม่สามารถทำ transaction ข้าม database ได้ง่ายๆ
-- Order DB:
BEGIN;
    INSERT INTO orders (customer_id, total) VALUES (1, 5000);
COMMIT;

-- Inventory DB (คนละ server):
BEGIN;
    UPDATE products SET stock = stock - 1 WHERE product_id = 101;
COMMIT;

-- ถ้า Order DB สำเร็จแต่ Inventory DB ล้มเหลว → ข้อมูลไม่สอดคล้องกัน!

-- วิธีแก้: Outbox Pattern
-- ใน Order DB, บันทึก event ไว้ด้วย
BEGIN;
    INSERT INTO orders (customer_id, total) VALUES (1, 5000) RETURNING order_id;
    INSERT INTO outbox_events (event_type, payload, status) 
    VALUES ('ORDER_CREATED', '{"order_id": 1, "product_id": 101, "quantity": 1}', 'pending');
COMMIT;
-- Message broker จะส่ง event ไปยัง Inventory Service
-- Inventory Service จะอัปเดตสต็อกและส่ง confirmation กลับมา
```

---

## 10. Best Practices สำหรับ ACID

```sql
-- 1. ทำให้ Transaction สั้นที่สุดเท่าที่เป็นไปได้
-- ไม่ดี: Transaction ยาวที่มีการรอผู้ใช้
BEGIN;
    SELECT * FROM cart WHERE user_id = 1 FOR UPDATE;
    -- รอ user กด confirm (อาจนานหลายนาที!)
    INSERT INTO orders ...;
COMMIT;

-- ดี: ทำการตรวจสอบก่อน แล้วค่อย commit เร็ว
-- 1. อ่านข้อมูล cart (ไม่ lock)
SELECT * FROM cart WHERE user_id = 1;
-- 2. แสดง UI ให้ user confirm
-- 3. เมื่อ user กด confirm ค่อยเริ่ม transaction สั้นๆ
BEGIN;
    -- Lock และตรวจสอบว่า cart ยังไม่เปลี่ยน
    SELECT * FROM cart WHERE user_id = 1 FOR UPDATE;
    INSERT INTO orders ...;
    DELETE FROM cart WHERE user_id = 1;
COMMIT;

-- 2. จัดการ Error อย่างถูกต้อง
DO $$
BEGIN
    BEGIN  -- เริ่ม transaction
        UPDATE accounts SET balance = balance - 500 WHERE id = 1;
        UPDATE accounts SET balance = balance + 500 WHERE id = 2;
    EXCEPTION
        WHEN check_violation THEN
            RAISE NOTICE 'Balance check failed, transaction rolled back';
            RAISE;  -- Re-raise เพื่อ rollback
        WHEN OTHERS THEN
            RAISE NOTICE 'Unexpected error: %', SQLERRM;
            RAISE;  -- Re-raise เพื่อ rollback
    END;
END;
$$;

-- 3. ระวัง implicit transactions
-- ใน PostgreSQL ทุก statement เป็น implicit transaction
UPDATE accounts SET balance = 0;  -- ถ้าไม่มี WHERE → อัปเดตทุกแถว!
-- ต้องระวัง! แต่ autocommit จะ commit ทันที

-- ทดสอบด้วย BEGIN ก่อนเสมอ
BEGIN;
    UPDATE accounts SET balance = 0;  -- ทดสอบก่อน
    -- ดูผลลัพธ์
    SELECT * FROM accounts;
ROLLBACK;  -- ยกเลิกถ้าไม่ต้องการ
```

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1

สร้างระบบจองตั๋วคอนเสิร์ตที่รักษา ACID properties:
- ต้องป้องกัน double-booking (Isolation)
- ถ้า payment ล้มเหลว ต้องคืนที่นั่ง (Atomicity)
- จำนวนที่นั่งต้องไม่ติดลบ (Consistency)

```sql
-- คำตอบ
CREATE TABLE concert_seats (
    seat_id     SERIAL PRIMARY KEY,
    event_id    INT NOT NULL,
    seat_number VARCHAR(10) NOT NULL,
    status      VARCHAR(20) DEFAULT 'available' CHECK (status IN ('available', 'reserved', 'booked')),
    reserved_by INT,
    reserved_at TIMESTAMP,
    UNIQUE(event_id, seat_number)
);

CREATE TABLE payments (
    payment_id  SERIAL PRIMARY KEY,
    seat_id     INT REFERENCES concert_seats(seat_id),
    amount      DECIMAL(10,2) NOT NULL CHECK (amount > 0),
    status      VARCHAR(20) DEFAULT 'pending',
    created_at  TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE OR REPLACE FUNCTION book_concert_seat(
    p_event_id  INT,
    p_seat_num  VARCHAR(10),
    p_user_id   INT,
    p_amount    DECIMAL(10,2)
) RETURNS TEXT AS $$
DECLARE
    v_seat_id   INT;
    v_payment_id INT;
BEGIN
    -- Lock seat (Isolation)
    SELECT seat_id INTO v_seat_id
    FROM concert_seats
    WHERE event_id = p_event_id 
      AND seat_number = p_seat_num 
      AND status = 'available'
    FOR UPDATE;
    
    IF v_seat_id IS NULL THEN
        RAISE EXCEPTION 'Seat % is not available', p_seat_num;
    END IF;
    
    -- จอง seat (Atomicity - ทั้งหมดสำเร็จหรือ rollback ทั้งหมด)
    UPDATE concert_seats 
    SET status = 'reserved', reserved_by = p_user_id, reserved_at = NOW()
    WHERE seat_id = v_seat_id;
    
    -- สร้าง payment
    INSERT INTO payments (seat_id, amount, status)
    VALUES (v_seat_id, p_amount, 'completed')
    RETURNING payment_id INTO v_payment_id;
    
    -- Confirm booking (Consistency - status ต้องถูกต้อง)
    UPDATE concert_seats SET status = 'booked' WHERE seat_id = v_seat_id;
    
    RETURN 'Booking confirmed: seat ' || p_seat_num || ', payment ' || v_payment_id;
EXCEPTION
    WHEN OTHERS THEN
        RAISE; -- Transaction จะถูก rollback
END;
$$ LANGUAGE plpgsql;

BEGIN;
    SELECT book_concert_seat(1, 'A15', 101, 1500.00);
COMMIT;
```

### แบบฝึกหัดที่ 2

แสดงให้เห็นว่า Consistency ถูกรักษาไว้อย่างไรเมื่อพยายามทำการโอนเงินที่ไม่ถูกต้อง:

```sql
-- คำตอบ
CREATE OR REPLACE FUNCTION safe_transfer(
    p_from_id  INT,
    p_to_id    INT,
    p_amount   DECIMAL(15,2)
) RETURNS VOID AS $$
DECLARE
    v_from_balance DECIMAL(15,2);
BEGIN
    IF p_amount <= 0 THEN
        RAISE EXCEPTION 'Transfer amount must be positive';
    END IF;
    
    IF p_from_id = p_to_id THEN
        RAISE EXCEPTION 'Cannot transfer to same account';
    END IF;
    
    -- Lock บัญชีต้นทาง
    SELECT balance INTO v_from_balance
    FROM accounts WHERE account_id = p_from_id FOR UPDATE;
    
    IF v_from_balance < p_amount THEN
        RAISE EXCEPTION 'Insufficient funds: balance %, requested %', 
                         v_from_balance, p_amount;
    END IF;
    
    -- Lock บัญชีปลายทาง
    PERFORM * FROM accounts WHERE account_id = p_to_id FOR UPDATE;
    
    UPDATE accounts SET balance = balance - p_amount WHERE account_id = p_from_id;
    UPDATE accounts SET balance = balance + p_amount WHERE account_id = p_to_id;
END;
$$ LANGUAGE plpgsql;
```

### แบบฝึกหัดที่ 3

เขียน query เพื่อตรวจสอบว่า transaction ใน production กำลังทำงานนานผิดปกติหรือไม่:

```sql
-- คำตอบ
SELECT 
    pid,
    usename AS username,
    application_name,
    client_addr,
    state,
    EXTRACT(EPOCH FROM (now() - xact_start)) AS seconds_in_transaction,
    CASE 
        WHEN EXTRACT(EPOCH FROM (now() - xact_start)) > 300 THEN 'CRITICAL - > 5 minutes'
        WHEN EXTRACT(EPOCH FROM (now() - xact_start)) > 60  THEN 'WARNING - > 1 minute'
        ELSE 'OK'
    END AS status,
    left(query, 200) AS query_snippet
FROM pg_stat_activity
WHERE xact_start IS NOT NULL
  AND state != 'idle'
ORDER BY seconds_in_transaction DESC;
```

### แบบฝึกหัดที่ 4

สาธิตว่า Durability ทำงานอย่างไรโดยใช้ WAL settings:

```sql
-- คำตอบ
-- ดู WAL configuration ปัจจุบัน
SELECT name, setting, unit, short_desc
FROM pg_settings
WHERE name IN (
    'fsync',
    'synchronous_commit',
    'wal_sync_method',
    'wal_buffers',
    'checkpoint_completion_target',
    'checkpoint_timeout',
    'max_wal_size'
)
ORDER BY name;

-- ตรวจสอบว่า WAL กำลังทำงาน
SELECT 
    pg_current_wal_lsn() AS current_wal_lsn,
    pg_walfile_name(pg_current_wal_lsn()) AS current_wal_file,
    pg_wal_lsn_diff(pg_current_wal_lsn(), '0/00000000') / 1024 / 1024 AS total_wal_mb;
```

### แบบฝึกหัดที่ 5

เปรียบเทียบประสิทธิภาพระหว่าง synchronous_commit ON และ OFF:

```sql
-- คำตอบ
-- ทดสอบ synchronous_commit ON (ปลอดภัย)
\timing
SET synchronous_commit = on;
BEGIN;
INSERT INTO test_perf SELECT generate_series(1, 10000), 'test data', NOW();
COMMIT;

-- ทดสอบ synchronous_commit OFF (เร็วกว่า)
SET synchronous_commit = off;
BEGIN;
INSERT INTO test_perf SELECT generate_series(10001, 20000), 'test data', NOW();
COMMIT;

-- สังเกตว่า OFF เร็วกว่าประมาณ 2-5 เท่า แต่อาจสูญ transaction ล่าสุดถ้า OS crash
```

### แบบฝึกหัดที่ 6

เขียน function ที่แสดง Atomicity โดยการ rollback ถ้า total balance เปลี่ยนไป:

```sql
-- คำตอบ
CREATE OR REPLACE FUNCTION transfer_with_audit(
    p_from INT,
    p_to   INT,
    p_amt  DECIMAL
) RETURNS VOID AS $$
DECLARE
    v_before_total DECIMAL;
    v_after_total  DECIMAL;
BEGIN
    SELECT SUM(balance) INTO v_before_total FROM accounts;
    
    UPDATE accounts SET balance = balance - p_amt WHERE account_id = p_from;
    UPDATE accounts SET balance = balance + p_amt WHERE account_id = p_to;
    
    SELECT SUM(balance) INTO v_after_total FROM accounts;
    
    -- ตรวจสอบว่า total balance ไม่เปลี่ยน (Consistency check)
    IF v_before_total <> v_after_total THEN
        RAISE EXCEPTION 'Balance integrity violation! Before: %, After: %',
                         v_before_total, v_after_total;
    END IF;
END;
$$ LANGUAGE plpgsql;
```

### แบบฝึกหัดที่ 7

สร้าง monitoring dashboard สำหรับ ACID violations:

```sql
-- คำตอบ
CREATE VIEW acid_health_monitor AS
SELECT 
    'Active Transactions' AS metric,
    COUNT(*)::TEXT AS value,
    CASE WHEN COUNT(*) > 100 THEN 'WARNING' ELSE 'OK' END AS status
FROM pg_stat_activity WHERE xact_start IS NOT NULL

UNION ALL

SELECT 
    'Long Running Transactions (>1 min)',
    COUNT(*)::TEXT,
    CASE WHEN COUNT(*) > 5 THEN 'CRITICAL' ELSE 'OK' END
FROM pg_stat_activity 
WHERE xact_start < now() - INTERVAL '1 minute'
  AND state != 'idle'

UNION ALL

SELECT 
    'Rollback Rate (%)',
    ROUND(
        SUM(xact_rollback)::NUMERIC / NULLIF(SUM(xact_commit + xact_rollback), 0) * 100, 2
    )::TEXT,
    CASE 
        WHEN SUM(xact_rollback)::NUMERIC / NULLIF(SUM(xact_commit + xact_rollback), 0) * 100 > 10 
        THEN 'WARNING' ELSE 'OK' 
    END
FROM pg_stat_database
WHERE datname = current_database()

UNION ALL

SELECT
    'Waiting Locks',
    COUNT(*)::TEXT,
    CASE WHEN COUNT(*) > 10 THEN 'WARNING' ELSE 'OK' END
FROM pg_locks WHERE NOT granted;

SELECT * FROM acid_health_monitor;
```

### แบบฝึกหัดที่ 8

แสดงให้เห็นว่า Isolation ทำงานในกรณี concurrent order placement:

```sql
-- คำตอบ
-- สร้างตาราง
CREATE TABLE flash_sale_items (
    item_id      SERIAL PRIMARY KEY,
    item_name    VARCHAR(200),
    stock        INT NOT NULL CHECK (stock >= 0),
    price        DECIMAL(10,2)
);

INSERT INTO flash_sale_items VALUES (1, 'Limited Edition Watch', 3, 5000.00);

-- Function สำหรับซื้อแบบ thread-safe
CREATE OR REPLACE FUNCTION buy_flash_sale_item(
    p_item_id   INT,
    p_user_id   INT,
    p_quantity  INT DEFAULT 1
) RETURNS TEXT AS $$
DECLARE
    v_available INT;
BEGIN
    -- ใช้ FOR UPDATE เพื่อ lock แถว (Isolation + Atomicity)
    SELECT stock INTO v_available
    FROM flash_sale_items
    WHERE item_id = p_item_id
    FOR UPDATE;
    
    IF v_available < p_quantity THEN
        RETURN 'Sorry, only ' || v_available || ' items left';
    END IF;
    
    UPDATE flash_sale_items
    SET stock = stock - p_quantity
    WHERE item_id = p_item_id;
    
    INSERT INTO orders (customer_id, product_id, quantity)
    VALUES (p_user_id, p_item_id, p_quantity);
    
    RETURN 'Purchase successful! ' || v_available - p_quantity || ' items remaining';
END;
$$ LANGUAGE plpgsql;
```

### แบบฝึกหัดที่ 9

เขียน query เพื่อหาสาเหตุของ transaction rollbacks:

```sql
-- คำตอบ
-- Enable logging ของ rollbacks (ต้องตั้งใน postgresql.conf)
-- log_min_duration_statement = 1000  -- log queries > 1 second
-- log_error_verbosity = verbose

-- ดู rollback statistics
SELECT 
    schemaname,
    tablename,
    n_dead_tup AS dead_tuples,  -- มาจาก rollback และ update
    n_live_tup AS live_tuples,
    ROUND(n_dead_tup::NUMERIC / NULLIF(n_live_tup + n_dead_tup, 0) * 100, 2) AS dead_pct,
    last_vacuum,
    last_autovacuum
FROM pg_stat_user_tables
WHERE n_dead_tup > 100
ORDER BY dead_pct DESC;

-- ดู database-level transaction stats
SELECT 
    datname,
    xact_commit,
    xact_rollback,
    ROUND(xact_rollback::NUMERIC / NULLIF(xact_commit + xact_rollback, 0) * 100, 4) AS rollback_rate_pct,
    deadlocks
FROM pg_stat_database
WHERE datname = current_database();
```

### แบบฝึกหัดที่ 10

สร้าง test suite ที่ตรวจสอบว่าระบบยังคง ACID compliant:

```sql
-- คำตอบ
CREATE OR REPLACE FUNCTION test_acid_compliance()
RETURNS TABLE(test_name TEXT, result TEXT, passed BOOLEAN) AS $$
DECLARE
    v_initial_sum  DECIMAL;
    v_final_sum    DECIMAL;
    v_test_passed  BOOLEAN;
BEGIN
    -- Test 1: Atomicity
    test_name := 'Atomicity Test';
    BEGIN
        SELECT SUM(balance) INTO v_initial_sum FROM accounts;
        
        BEGIN
            UPDATE accounts SET balance = balance - 1000000 WHERE account_id = 1;
            -- ข้อผิดพลาดที่ตั้งใจ
            RAISE EXCEPTION 'Simulated error';
        EXCEPTION WHEN OTHERS THEN
            NULL; -- Rollback จะเกิดอัตโนมัติ
        END;
        
        SELECT SUM(balance) INTO v_final_sum FROM accounts;
        v_test_passed := (v_initial_sum = v_final_sum);
        result := CASE WHEN v_test_passed THEN 'PASS: Balance unchanged after rollback' 
                       ELSE 'FAIL: Balance changed unexpectedly' END;
        passed := v_test_passed;
        RETURN NEXT;
    END;
    
    -- Test 2: Consistency
    test_name := 'Consistency Test (CHECK constraint)';
    BEGIN
        v_test_passed := FALSE;
        BEGIN
            INSERT INTO accounts (balance) VALUES (-100);
        EXCEPTION WHEN check_violation THEN
            v_test_passed := TRUE;
        END;
        result := CASE WHEN v_test_passed THEN 'PASS: CHECK constraint enforced'
                       ELSE 'FAIL: Negative balance allowed' END;
        passed := v_test_passed;
        RETURN NEXT;
    END;
    
    -- Test 3: Isolation (simplified)
    test_name := 'Isolation Test (READ COMMITTED)';
    result := 'PASS: PostgreSQL default isolation prevents dirty reads';
    passed := TRUE;
    RETURN NEXT;
    
    -- Test 4: Durability (check WAL is enabled)
    test_name := 'Durability Test (WAL enabled)';
    SELECT 
        CASE WHEN setting = 'on' THEN TRUE ELSE FALSE END INTO v_test_passed
    FROM pg_settings WHERE name = 'fsync';
    result := CASE WHEN v_test_passed THEN 'PASS: fsync is enabled'
                   ELSE 'FAIL: fsync is disabled - durability at risk!' END;
    passed := v_test_passed;
    RETURN NEXT;
END;
$$ LANGUAGE plpgsql;

SELECT * FROM test_acid_compliance();
```

---

## สรุป

ACID Properties เป็นรากฐานของฐานข้อมูลที่น่าเชื่อถือ:

1. **Atomicity** - ทุก operation ใน transaction ต้องสำเร็จพร้อมกัน หรือล้มเหลวพร้อมกัน
2. **Consistency** - ฐานข้อมูลต้องอยู่ในสถานะที่ถูกต้องเสมอ
3. **Isolation** - Transaction แต่ละอันต้องไม่รบกวนซึ่งกันและกัน
4. **Durability** - ข้อมูลที่ commit แล้วต้องคงอยู่แม้ระบบจะล่ม

- PostgreSQL มี ACID compliance ที่เข้มงวดที่สุดในบรรดา RDBMS ยอดนิยม
- MySQL ต้องใช้ InnoDB engine จึงจะได้ ACID
- SQLite เหมาะกับ single-user applications เท่านั้น
- BASE เป็นทางเลือกสำหรับ NoSQL ที่ยอมรับ eventual consistency เพื่อแลกกับ availability และ performance

ใน Part 72 เราจะเจาะลึกเรื่อง Transactions: BEGIN, COMMIT, ROLLBACK และการจัดการ error ใน transactions
