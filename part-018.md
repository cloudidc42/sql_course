# Part 18: AUTO INCREMENT และ Sequences

## บทนำ

การสร้าง ID อัตโนมัติเป็นความต้องการพื้นฐานของทุกระบบฐานข้อมูล แต่ละ database system มีวิธีการที่ต่างกัน บทนี้ครอบคลุม:
- AUTO_INCREMENT (MySQL)
- SERIAL/BIGSERIAL (PostgreSQL)
- IDENTITY (SQL Server)
- AUTOINCREMENT (SQLite)
- CREATE SEQUENCE
- UUID เป็น Primary Key

---

## 1. AUTO_INCREMENT - MySQL

### 1.1 พื้นฐาน AUTO_INCREMENT

```sql
-- ตัวอย่าง 1: AUTO_INCREMENT พื้นฐาน
CREATE TABLE auto_basic (
    id      INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    name    VARCHAR(100) NOT NULL
);

INSERT INTO auto_basic (name) VALUES ('First');   -- id = 1
INSERT INTO auto_basic (name) VALUES ('Second');  -- id = 2
INSERT INTO auto_basic (name) VALUES ('Third');   -- id = 3

SELECT * FROM auto_basic;

-- LAST_INSERT_ID() - ดู ID ล่าสุด
INSERT INTO auto_basic (name) VALUES ('Fourth');
SELECT LAST_INSERT_ID() AS last_id;  -- 4

-- ตัวอย่าง 2: AUTO_INCREMENT กับ types ต่างๆ
CREATE TABLE auto_types (
    -- TINYINT: สูงสุด 127 (signed) หรือ 255 (unsigned)
    tiny_id     TINYINT UNSIGNED AUTO_INCREMENT,
    
    -- SMALLINT: สูงสุด 32,767 (signed) หรือ 65,535 (unsigned)  
    small_id    SMALLINT UNSIGNED AUTO_INCREMENT,
    
    -- INT: สูงสุด 2,147,483,647 (signed) หรือ 4,294,967,295 (unsigned)
    normal_id   INT UNSIGNED AUTO_INCREMENT,
    
    -- BIGINT: สูงสุด 9,223,372,036,854,775,807 (signed)
    big_id      BIGINT UNSIGNED AUTO_INCREMENT
    -- (เลือกแค่ 1 auto_increment ต่อตาราง)
);

-- ตัวอย่าง 3: ดู AUTO_INCREMENT value ปัจจุบัน
SHOW TABLE STATUS LIKE 'employees';
-- Column: Auto_increment แสดงค่า next AUTO_INCREMENT

-- หรือ
SELECT AUTO_INCREMENT
FROM information_schema.TABLES
WHERE TABLE_SCHEMA = DATABASE()
  AND TABLE_NAME = 'employees';
```

### 1.2 กำหนดค่าเริ่มต้น AUTO_INCREMENT

```sql
-- ตัวอย่าง 4: ตั้งค่าเริ่มต้น AUTO_INCREMENT
CREATE TABLE employees_numbered (
    employee_id INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    name        VARCHAR(100)
) AUTO_INCREMENT = 1000;  -- เริ่มที่ 1000

INSERT INTO employees_numbered (name) VALUES ('พนักงาน A');  -- id = 1000
INSERT INTO employees_numbered (name) VALUES ('พนักงาน B');  -- id = 1001

-- ตัวอย่าง 5: เปลี่ยน AUTO_INCREMENT ด้วย ALTER TABLE
ALTER TABLE employees AUTO_INCREMENT = 5000;
-- INSERT ถัดไปจะเป็น 5000

-- รีเซ็ต AUTO_INCREMENT
-- ✅ วิธีที่ปลอดภัย:
ALTER TABLE employees AUTO_INCREMENT = 1;
-- MySQL จะใช้ค่า MAX(id) + 1 ถ้า MAX(id) >= 1

-- ลบข้อมูลทั้งหมดและรีเซ็ต
TRUNCATE TABLE temp_data;  -- รีเซ็ต AUTO_INCREMENT เป็น 1

-- ตัวอย่าง 6: INSERT ด้วย explicit ID
INSERT INTO auto_basic (id, name) VALUES (100, 'Jump to 100');
INSERT INTO auto_basic (name) VALUES ('After jump');  -- id = 101
```

### 1.3 AUTO_INCREMENT Gaps

```sql
-- ตัวอย่าง 7: ช่องว่างใน AUTO_INCREMENT (Gaps)
-- เกิดจาก: DELETE, ROLLBACK, failed INSERT

INSERT INTO auto_basic (name) VALUES ('Row 5');  -- id = 5
INSERT INTO auto_basic (name) VALUES ('Row 6');  -- id = 6 (แต่จะ rollback)

START TRANSACTION;
INSERT INTO auto_basic (name) VALUES ('Row 7');  -- id = 7 แต่ถูก rollback
ROLLBACK;

INSERT INTO auto_basic (name) VALUES ('Row 8');  -- id = 8! (7 ถูกข้าม)

-- ✅ Gaps เป็นเรื่องปกติ และไม่ควรพยายามแก้ไข
-- ❌ อย่าเขียน business logic ที่ขึ้นกับว่า ID ต้องต่อเนื่อง

-- ตัวอย่าง 8: ดู gaps ใน IDs
SELECT 
    a.id + 1 AS gap_start,
    MIN(b.id) - 1 AS gap_end
FROM auto_basic a
LEFT JOIN auto_basic b ON a.id + 1 = b.id
WHERE b.id IS NULL
  AND a.id < (SELECT MAX(id) FROM auto_basic)
GROUP BY a.id;
```

### 1.4 AUTO_INCREMENT ใน Multi-Row Insert

```sql
-- ตัวอย่าง 9: Multi-row INSERT กับ AUTO_INCREMENT
INSERT INTO auto_basic (name) VALUES ('A'), ('B'), ('C');
SELECT LAST_INSERT_ID();  -- ID ของ row แรกที่ insert!
-- ถ้า insert ได้ id 10, 11, 12 -> LAST_INSERT_ID() = 10

-- ดู ID ทั้งหมดที่ insert
INSERT INTO auto_basic (name) VALUES ('X'), ('Y'), ('Z');
SELECT LAST_INSERT_ID() AS first_id, 
       LAST_INSERT_ID() + ROW_COUNT() - 1 AS last_id;
```

---

## 2. SERIAL/BIGSERIAL - PostgreSQL

```sql
-- ตัวอย่าง 10: PostgreSQL SERIAL
/*
-- SERIAL = SEQUENCE + DEFAULT nextval + NOT NULL
CREATE TABLE pg_employees (
    employee_id SERIAL PRIMARY KEY,  -- equivalent to INT
    name        VARCHAR(100) NOT NULL
);

-- BIGSERIAL สำหรับ ID ขนาดใหญ่
CREATE TABLE pg_big_table (
    id      BIGSERIAL PRIMARY KEY,
    data    TEXT
);

-- SMALLSERIAL สำหรับตารางเล็ก
CREATE TABLE pg_small_table (
    id      SMALLSERIAL PRIMARY KEY,
    code    VARCHAR(10)
);

-- SERIAL ถูก expand เป็น:
-- CREATE SEQUENCE pg_employees_employee_id_seq;
-- employee_id INTEGER NOT NULL DEFAULT nextval('pg_employees_employee_id_seq')
-- ALTER SEQUENCE pg_employees_employee_id_seq OWNED BY pg_employees.employee_id;

INSERT INTO pg_employees (name) VALUES ('John Doe');
SELECT LASTVAL();      -- last sequence value used
SELECT currval('pg_employees_employee_id_seq');  -- current value of specific sequence

-- GENERATED AS IDENTITY (SQL Standard, PostgreSQL 10+)
CREATE TABLE pg_identity (
    id      INT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    name    VARCHAR(100)
);

-- GENERATED BY DEFAULT AS IDENTITY (อนุญาต manual insert)
CREATE TABLE pg_identity2 (
    id      INT GENERATED BY DEFAULT AS IDENTITY PRIMARY KEY,
    name    VARCHAR(100)
);

INSERT INTO pg_identity (name) VALUES ('Test');
SELECT * FROM pg_identity;

-- กำหนดค่าเริ่มต้น
CREATE TABLE pg_custom_start (
    id      INT GENERATED ALWAYS AS IDENTITY 
            (START WITH 1000 INCREMENT BY 1) PRIMARY KEY,
    name    VARCHAR(100)
);
*/
```

---

## 3. IDENTITY - SQL Server

```sql
-- ตัวอย่าง 11: SQL Server IDENTITY
/*
-- IDENTITY(seed, increment)
CREATE TABLE ss_employees (
    employee_id INT IDENTITY(1,1) PRIMARY KEY,  -- เริ่มที่ 1 เพิ่มทีละ 1
    name        NVARCHAR(100) NOT NULL
);

-- IDENTITY เริ่มที่ 1000 เพิ่มทีละ 5
CREATE TABLE ss_custom (
    id      INT IDENTITY(1000, 5) PRIMARY KEY,
    data    NVARCHAR(200)
);
-- Insert ครั้งแรก = 1000, ครั้งต่อไป = 1005, 1010, ...

-- ดู last identity
INSERT INTO ss_employees (name) VALUES ('John');
SELECT SCOPE_IDENTITY() AS last_id;   -- ปลอดภัยที่สุด (same scope)
SELECT @@IDENTITY AS last_id;          -- อาจได้ identity จาก trigger
SELECT IDENT_CURRENT('ss_employees') AS current_id;  -- current value ของ table

-- Insert explicit ID (ต้อง SET IDENTITY_INSERT ON)
SET IDENTITY_INSERT ss_employees ON;
INSERT INTO ss_employees (employee_id, name) VALUES (999, 'Special');
SET IDENTITY_INSERT ss_employees OFF;
*/
```

---

## 4. AUTOINCREMENT - SQLite

```sql
-- ตัวอย่าง 12: SQLite ROWID และ AUTOINCREMENT
/*
-- SQLite: INTEGER PRIMARY KEY = alias for ROWID (เร็วกว่า)
CREATE TABLE sqlite_basic (
    id      INTEGER PRIMARY KEY,  -- alias for ROWID, reuses gaps
    name    TEXT
);

-- AUTOINCREMENT: ป้องกัน reuse ของ IDs ที่เคยใช้แล้ว (ช้ากว่าเล็กน้อย)
CREATE TABLE sqlite_autoincrement (
    id      INTEGER PRIMARY KEY AUTOINCREMENT,  -- ไม่ reuse IDs
    name    TEXT
);

-- ความแตกต่าง:
-- INTEGER PRIMARY KEY: อาจ reuse ID ที่เคยถูกลบ
-- AUTOINCREMENT: ไม่ reuse, เก็บ max id ใน sqlite_sequence table

-- ดู current sequence
SELECT * FROM sqlite_sequence WHERE name = 'sqlite_autoincrement';

-- last inserted rowid
SELECT last_insert_rowid();
*/
```

---

## 5. CREATE SEQUENCE

### 5.1 PostgreSQL SEQUENCE

```sql
-- ตัวอย่าง 13: สร้าง Sequence เอง
/*
-- Basic sequence
CREATE SEQUENCE order_number_seq
    START WITH 10000
    INCREMENT BY 1
    MINVALUE 10000
    MAXVALUE 99999
    CYCLE;  -- วนกลับหลังถึง MAXVALUE

-- ใช้ sequence
SELECT nextval('order_number_seq');  -- 10000
SELECT nextval('order_number_seq');  -- 10001
SELECT currval('order_number_seq');  -- 10001 (ค่าปัจจุบัน)
SELECT lastval();                     -- 10001 (ค่าล่าสุดของ session นี้)

-- ตั้งค่าใหม่
SELECT setval('order_number_seq', 5000);  -- ตั้งเป็น 5000, nextval = 5001
SELECT setval('order_number_seq', 5000, false);  -- nextval = 5000

-- DROP SEQUENCE
DROP SEQUENCE order_number_seq;

-- Sequence สำหรับ order numbers
CREATE SEQUENCE invoice_seq
    START WITH 202400001
    INCREMENT BY 1
    NO CYCLE;

CREATE TABLE invoices (
    invoice_id  INT PRIMARY KEY,
    invoice_no  VARCHAR(20) DEFAULT ('INV-' || to_char(nextval('invoice_seq'), 'FM000000')),
    customer_id INT,
    amount      DECIMAL(12,2)
);
*/
```

### 5.2 MySQL SEQUENCE (ไม่มี native, ใช้ workaround)

```sql
-- ตัวอย่าง 14: MySQL Sequence Emulation
-- วิธีที่ 1: ใช้ AUTO_INCREMENT table
CREATE TABLE sequences (
    seq_name    VARCHAR(50) PRIMARY KEY,
    seq_value   BIGINT UNSIGNED NOT NULL DEFAULT 0
);

INSERT INTO sequences (seq_name) VALUES 
    ('order_number'),
    ('invoice_number'),
    ('employee_code');

-- Function สร้าง next value
DELIMITER //
CREATE FUNCTION nextval(p_seq_name VARCHAR(50)) 
RETURNS BIGINT
DETERMINISTIC
MODIFIES SQL DATA
BEGIN
    DECLARE v_value BIGINT;
    
    UPDATE sequences 
    SET seq_value = seq_value + 1
    WHERE seq_name = p_seq_name;
    
    SELECT seq_value INTO v_value
    FROM sequences
    WHERE seq_name = p_seq_name;
    
    RETURN v_value;
END //
DELIMITER ;

-- ใช้งาน
SELECT nextval('order_number');  -- 1
SELECT nextval('order_number');  -- 2

-- ตัวอย่าง 15: ใช้ sequence ใน INSERT
INSERT INTO orders (order_id, customer_id, total_amount)
VALUES (nextval('order_number'), 1, 500.00);

-- วิธีที่ 2: LAST_INSERT_ID() trick
CREATE TABLE sequence_table (id INT NOT NULL);
INSERT INTO sequence_table VALUES (0);

-- ดึง next value แบบ atomic
UPDATE sequence_table SET id = LAST_INSERT_ID(id + 1);
SELECT LAST_INSERT_ID() AS next_id;
```

---

## 6. Resetting Sequences

```sql
-- ตัวอย่าง 16: Reset AUTO_INCREMENT ใน MySQL

-- วิธีที่ 1: ALTER TABLE
ALTER TABLE employees AUTO_INCREMENT = 1;

-- วิธีที่ 2: TRUNCATE (ลบข้อมูลทั้งหมด)
TRUNCATE TABLE temp_test;

-- วิธีที่ 3: ตั้งให้เป็น max + 1 (ปลอดภัยสุด)
SET @max_id = (SELECT MAX(employee_id) + 1 FROM employees);
SET @sql = CONCAT('ALTER TABLE employees AUTO_INCREMENT = ', @max_id);
PREPARE stmt FROM @sql;
EXECUTE stmt;
DEALLOCATE PREPARE stmt;

-- ตัวอย่าง 17: Reset PostgreSQL Sequence
/*
-- รีเซ็ตเป็น 1
ALTER SEQUENCE employees_employee_id_seq RESTART WITH 1;

-- รีเซ็ตให้ต่อจาก max ID
SELECT setval('employees_employee_id_seq', MAX(employee_id)) FROM employees;

-- หรือ
SELECT setval(pg_get_serial_sequence('employees', 'employee_id'), 
              COALESCE((SELECT MAX(employee_id) FROM employees), 0) + 1, false);
*/
```

---

## 7. UUID เป็น Primary Key

```sql
-- ตัวอย่าง 18: UUID ใน MySQL 8.0+
CREATE TABLE orders_uuid (
    order_id    CHAR(36) PRIMARY KEY DEFAULT (UUID()),
    customer_id INT NOT NULL,
    total       DECIMAL(10,2),
    created_at  TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

INSERT INTO orders_uuid (customer_id, total) VALUES (1, 999.99);
SELECT * FROM orders_uuid;

-- ตัวอย่าง 19: UUID ด้วย BINARY(16) (ประหยัดพื้นที่)
CREATE TABLE orders_uuid_binary (
    order_id    BINARY(16) PRIMARY KEY DEFAULT (UUID_TO_BIN(UUID())),
    customer_id INT NOT NULL,
    total       DECIMAL(10,2)
);

INSERT INTO orders_uuid_binary (customer_id, total) VALUES (1, 500.00);

-- แสดง UUID แบบ readable
SELECT BIN_TO_UUID(order_id) AS order_id, customer_id, total 
FROM orders_uuid_binary;

-- ตัวอย่าง 20: UUID v4 ใน PostgreSQL
/*
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
-- หรือ
CREATE EXTENSION IF NOT EXISTS "pgcrypto";

CREATE TABLE pg_orders (
    order_id    UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    -- หรือ: DEFAULT gen_random_uuid() (pgcrypto)
    customer_id INT,
    total       DECIMAL(10,2)
);

INSERT INTO pg_orders (customer_id, total) VALUES (1, 999.99);
SELECT * FROM pg_orders;

-- UUID v7 (ordered by time, better index performance)
-- ใน PostgreSQL 17+: gen_random_uuid() type v4
-- ใน application: ใช้ library ที่รองรับ UUID v7
*/

-- ตัวอย่าง 21: UUID vs AUTO_INCREMENT เปรียบเทียบ
/*
AUTO_INCREMENT / SERIAL:
✅ เล็ก (4-8 bytes)
✅ เรียงตามเวลา (predictable)
✅ เร็วสำหรับ index (sequential)
❌ expose business info (order count)
❌ merge ข้อมูลจากหลาย DB ยาก

UUID:
✅ Globally unique
✅ ไม่ expose business info
✅ merge ข้อมูลจากหลาย DB ง่าย
✅ generate ใน application ได้
❌ ใหญ่กว่า (16 bytes)
❌ random = fragmented index (ช้ากว่า)
❌ อ่านยากสำหรับ human
*/
```

---

## 8. GENERATED ALWAYS AS IDENTITY (SQL Standard)

```sql
-- ตัวอย่าง 22: SQL Standard Identity (PostgreSQL 10+, SQL Server)
/*
-- GENERATED ALWAYS AS IDENTITY: ไม่อนุญาต manual insert
CREATE TABLE pg_products (
    product_id  INT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    name        VARCHAR(200) NOT NULL
);

INSERT INTO pg_products (name) VALUES ('Product A');  -- OK
INSERT INTO pg_products (product_id, name) VALUES (1, 'Manual');  -- ERROR!

-- GENERATED BY DEFAULT AS IDENTITY: อนุญาต manual insert
CREATE TABLE pg_products2 (
    product_id  INT GENERATED BY DEFAULT AS IDENTITY 
                (START WITH 1000 INCREMENT BY 1) PRIMARY KEY,
    name        VARCHAR(200) NOT NULL
);

INSERT INTO pg_products2 (name) VALUES ('Product B');           -- id = 1000
INSERT INTO pg_products2 (product_id, name) VALUES (5, 'Manual');  -- id = 5, OK

-- Reset identity
ALTER TABLE pg_products2 
ALTER COLUMN product_id RESTART WITH 2000;
*/
```

---

## 9. Practical Patterns

### 9.1 Human-Readable IDs

```sql
-- ตัวอย่าง 23: สร้าง Order Number แบบ readable
CREATE TABLE order_numbers (
    order_id        INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    order_number    VARCHAR(20) NOT NULL UNIQUE,
    customer_id     INT UNSIGNED,
    total           DECIMAL(10,2),
    created_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- สร้าง order number: ORD-YYYYMM-00001
DELIMITER //
CREATE TRIGGER generate_order_number
BEFORE INSERT ON order_numbers
FOR EACH ROW
BEGIN
    DECLARE v_seq INT;
    DECLARE v_prefix VARCHAR(10);
    
    SET v_prefix = CONCAT('ORD-', DATE_FORMAT(NOW(), '%Y%m'));
    
    -- ดูเลขลำดับล่าสุดของเดือนนี้
    SELECT COALESCE(MAX(CAST(SUBSTRING(order_number, 9) AS UNSIGNED)), 0) + 1
    INTO v_seq
    FROM order_numbers
    WHERE order_number LIKE CONCAT(v_prefix, '%');
    
    SET NEW.order_number = CONCAT(v_prefix, '-', LPAD(v_seq, 5, '0'));
END //
DELIMITER ;

INSERT INTO order_numbers (customer_id, total) VALUES (1, 500.00);
SELECT order_number FROM order_numbers;  -- ORD-202401-00001

-- ตัวอย่าง 24: Employee Code Generation
DELIMITER //
CREATE TRIGGER generate_employee_code
BEFORE INSERT ON employees
FOR EACH ROW
BEGIN
    DECLARE v_seq INT;
    DECLARE v_dept_code VARCHAR(5);
    
    -- ดูรหัสแผนก
    SELECT LEFT(department_name, 3) INTO v_dept_code
    FROM departments WHERE department_id = NEW.department_id;
    
    SET v_dept_code = UPPER(COALESCE(v_dept_code, 'GEN'));
    
    -- ดูเลขล่าสุดของแผนก
    SELECT COALESCE(MAX(CAST(SUBSTRING(employee_code, 4) AS UNSIGNED)), 0) + 1
    INTO v_seq
    FROM employees
    WHERE employee_code LIKE CONCAT(v_dept_code, '%');
    
    SET NEW.employee_code = CONCAT(v_dept_code, LPAD(v_seq, 4, '0'));
END //
DELIMITER ;
```

### 9.2 ULID - Universally Unique Lexicographically Sortable Identifier

```sql
-- ตัวอย่าง 25: ULID concept
-- ULID = timestamp (10 chars) + random (16 chars) = 26 chars
-- ข้อดี: sortable by time, unique, URL-safe
-- ตัวอย่าง: 01ARZ3NDEKTSV4RRFFQ69G5FAV

-- Store as VARCHAR(26)
CREATE TABLE ulid_example (
    id      VARCHAR(26) PRIMARY KEY,  -- ULID
    data    TEXT
);

-- ใน MySQL ต้องใช้ library ภายนอก หรือ UDF
-- ใน application (PHP, Node.js, Python) ใช้ library ที่รองรับ ULID
```

---

## 10. Auto-increment ใน Distributed Systems

```sql
-- ตัวอย่าง 26: Snowflake ID (Twitter-style)
-- 64-bit integer: 1 bit sign + 41 bit timestamp + 10 bit machine + 12 bit sequence
-- ทำให้ generate ID ได้หลาย servers พร้อมกันโดยไม่ชน
-- ตัวอย่าง implementation อยู่ใน application layer

-- ตัวอย่าง 27: Composite key สำหรับ sharding
CREATE TABLE sharded_orders (
    shard_id    TINYINT UNSIGNED NOT NULL,  -- 0-255
    order_id    INT UNSIGNED AUTO_INCREMENT,
    customer_id INT UNSIGNED NOT NULL,
    total       DECIMAL(10,2),
    
    PRIMARY KEY (shard_id, order_id)
) AUTO_INCREMENT = 1;

-- ข้อมูลถูก shard โดย shard_id
-- order_id เป็น auto increment ภายใน shard

-- ตัวอย่าง 28: Global sequence table สำหรับ multi-table IDs
CREATE TABLE id_generator (
    stub CHAR(1) NOT NULL DEFAULT 'a',
    id   BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
    PRIMARY KEY (stub, id),
    UNIQUE KEY stub (stub)
) ENGINE=MyISAM;  -- MyISAM สำหรับ faster auto_increment

-- Generate unique ID
REPLACE INTO id_generator (stub) VALUES ('a');
SELECT LAST_INSERT_ID() AS new_id;
```

---

## แบบฝึกหัด 10 ข้อ

### ข้อ 1
สร้างตาราง products ด้วย AUTO_INCREMENT เริ่มที่ 1001

**เฉลย:**
```sql
CREATE TABLE products_auto (
    product_id  INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    name        VARCHAR(200) NOT NULL,
    price       DECIMAL(10,2)
) AUTO_INCREMENT = 1001;

INSERT INTO products_auto (name, price) VALUES ('Test Product', 99.99);
SELECT product_id FROM products_auto;  -- 1001
```

### ข้อ 2
เขียน function เพื่อ generate invoice number แบบ INV-YYYY-NNNNNN

**เฉลย:**
```sql
CREATE TABLE invoice_counter (
    year        SMALLINT PRIMARY KEY,
    last_seq    INT UNSIGNED DEFAULT 0
);

DELIMITER //
CREATE FUNCTION generate_invoice_no() RETURNS VARCHAR(20)
DETERMINISTIC
MODIFIES SQL DATA
BEGIN
    DECLARE v_year SMALLINT DEFAULT YEAR(NOW());
    DECLARE v_seq INT;
    
    INSERT INTO invoice_counter (year, last_seq) VALUES (v_year, 1)
    ON DUPLICATE KEY UPDATE last_seq = last_seq + 1;
    
    SELECT last_seq INTO v_seq FROM invoice_counter WHERE year = v_year;
    
    RETURN CONCAT('INV-', v_year, '-', LPAD(v_seq, 6, '0'));
END //
DELIMITER ;

SELECT generate_invoice_no();  -- INV-2024-000001
SELECT generate_invoice_no();  -- INV-2024-000002
```

### ข้อ 3
แสดงปัญหา AUTO_INCREMENT gaps และ best practices

**เฉลย:**
```sql
CREATE TABLE gap_demo (
    id   INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(50)
);

-- สร้าง gaps
INSERT INTO gap_demo (name) VALUES ('Row 1'), ('Row 2'), ('Row 3');
DELETE FROM gap_demo WHERE id = 2;  -- สร้าง gap

START TRANSACTION;
INSERT INTO gap_demo (name) VALUES ('Row 4');  -- id = 4
ROLLBACK;  -- สร้าง gap ที่ 4

INSERT INTO gap_demo (name) VALUES ('Row 5');  -- id = 5

SELECT * FROM gap_demo;  -- 1, 3, 5 (gaps ที่ 2, 4)

-- Best practices:
-- 1. ไม่ expose ID ให้ user เห็น (ใช้ UUID แทน)
-- 2. ไม่ assume IDs เป็น sequential
-- 3. ไม่ calculate "number of records" จาก max ID
-- 4. Gaps เป็นเรื่องปกติ ไม่ต้องแก้
```

### ข้อ 4
เขียน query ดู AUTO_INCREMENT status ของทุกตารางในฐานข้อมูล

**เฉลย:**
```sql
SELECT 
    TABLE_NAME,
    AUTO_INCREMENT AS next_id,
    TABLE_ROWS AS estimated_rows,
    ROUND(DATA_LENGTH / 1024 / 1024, 2) AS data_mb
FROM information_schema.TABLES
WHERE TABLE_SCHEMA = DATABASE()
  AND AUTO_INCREMENT IS NOT NULL
ORDER BY TABLE_NAME;
```

### ข้อ 5
สร้าง PostgreSQL sequence สำหรับ custom order numbering

**เฉลย:**
```sql
-- PostgreSQL syntax
/*
CREATE SEQUENCE monthly_order_seq
    START WITH 1
    INCREMENT BY 1
    MINVALUE 1
    MAXVALUE 99999
    CYCLE;  -- รีเซ็ตทุกเดือน (ต้องรีเซ็ต manually)

CREATE TABLE pg_orders (
    order_id    SERIAL PRIMARY KEY,
    order_no    VARCHAR(15) DEFAULT (
        'ORD-' || to_char(now(), 'YYYYMM') || '-' || 
        lpad(nextval('monthly_order_seq')::text, 5, '0')
    ),
    customer_id INT,
    total       DECIMAL(10,2)
);
*/

-- MySQL equivalent:
CREATE TABLE mysql_seq_orders (
    order_id    INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    order_no    VARCHAR(15),
    customer_id INT UNSIGNED,
    total       DECIMAL(10,2)
);

DELIMITER //
CREATE TRIGGER gen_order_no
BEFORE INSERT ON mysql_seq_orders
FOR EACH ROW
BEGIN
    DECLARE v_cnt INT;
    SELECT COUNT(*) + 1 INTO v_cnt
    FROM mysql_seq_orders
    WHERE LEFT(order_no, 11) = CONCAT('ORD-', DATE_FORMAT(NOW(), '%Y%m'), '-');
    
    SET NEW.order_no = CONCAT('ORD-', DATE_FORMAT(NOW(), '%Y%m'), '-', LPAD(v_cnt, 5, '0'));
END //
DELIMITER ;
```

### ข้อ 6
Implement UUID v4 เป็น Primary Key แบบ Binary สำหรับ performance

**เฉลย:**
```sql
CREATE TABLE products_uuid (
    product_id  BINARY(16) PRIMARY KEY DEFAULT (UUID_TO_BIN(UUID(), 1)),
    -- UUID_TO_BIN(uuid, 1) = swap time bits สำหรับ ordered UUID
    sku         VARCHAR(50) UNIQUE NOT NULL,
    name        VARCHAR(200) NOT NULL,
    price       DECIMAL(10,2),
    created_at  TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

INSERT INTO products_uuid (sku, name, price)
VALUES ('SKU001', 'Product A', 99.99);

-- แสดง UUID แบบ readable
SELECT 
    BIN_TO_UUID(product_id, 1) AS uuid,
    sku,
    name,
    price
FROM products_uuid;
```

### ข้อ 7
สร้าง sequence emulation ใน MySQL ที่ thread-safe

**เฉลย:**
```sql
CREATE TABLE sequence_store (
    seq_name    VARCHAR(100) PRIMARY KEY,
    seq_value   BIGINT UNSIGNED NOT NULL DEFAULT 1,
    step        INT UNSIGNED NOT NULL DEFAULT 1,
    created_at  TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

INSERT INTO sequence_store (seq_name, seq_value, step) VALUES
    ('order_id', 1, 1),
    ('invoice_id', 100000, 1),
    ('employee_id', 1, 1);

-- Thread-safe next value
DELIMITER //
CREATE PROCEDURE get_next_seq(IN p_name VARCHAR(100), OUT p_value BIGINT)
BEGIN
    UPDATE sequence_store 
    SET seq_value = seq_value + step
    WHERE seq_name = p_name;
    
    SELECT seq_value INTO p_value
    FROM sequence_store
    WHERE seq_name = p_name;
END //
DELIMITER ;

CALL get_next_seq('order_id', @next_id);
SELECT @next_id;  -- thread-safe auto-increment
```

### ข้อ 8
Reset AUTO_INCREMENT หลังจาก bulk delete

**เฉลย:**
```sql
-- ลบข้อมูลบางส่วน
DELETE FROM products WHERE category = 'discontinued';

-- ดู current state
SELECT AUTO_INCREMENT, TABLE_ROWS
FROM information_schema.TABLES
WHERE TABLE_SCHEMA = DATABASE() AND TABLE_NAME = 'products';

-- Reset AUTO_INCREMENT เป็น max_id + 1
SET @new_auto_inc = (SELECT MAX(product_id) + 1 FROM products);

-- ใช้ dynamic SQL
SET @sql = CONCAT('ALTER TABLE products AUTO_INCREMENT = ', @new_auto_inc);
PREPARE stmt FROM @sql;
EXECUTE stmt;
DEALLOCATE PREPARE stmt;

-- ยืนยัน
SELECT AUTO_INCREMENT FROM information_schema.TABLES
WHERE TABLE_SCHEMA = DATABASE() AND TABLE_NAME = 'products';
```

### ข้อ 9
เปรียบเทียบ performance ระหว่าง INT AUTO_INCREMENT และ UUID PRIMARY KEY

**เฉลย:**
```sql
-- สร้างตารางทดสอบ
CREATE TABLE perf_int (
    id      INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    data    VARCHAR(100),
    ts      TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_ts (ts)
);

CREATE TABLE perf_uuid (
    id      CHAR(36) PRIMARY KEY DEFAULT (UUID()),
    data    VARCHAR(100),
    ts      TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_ts (ts)
);

-- Insert 10,000 rows (ทดสอบด้วย Loop ใน Stored Procedure)
DELIMITER //
CREATE PROCEDURE insert_test_data()
BEGIN
    DECLARE i INT DEFAULT 1;
    WHILE i <= 10000 DO
        INSERT INTO perf_int (data) VALUES (CONCAT('Data-', i));
        INSERT INTO perf_uuid (data) VALUES (CONCAT('Data-', i));
        SET i = i + 1;
    END WHILE;
END //
DELIMITER ;

-- INT: insert เร็วกว่า (sequential insert = no B-tree rebalancing)
-- UUID: insert ช้ากว่า (random = B-tree fragmentation)
-- SELECT by PK: INT เร็วกว่า (4 bytes vs 36 bytes)
-- UUID: ดีกว่าสำหรับ distributed systems

-- เปรียบเทียบขนาด index
SELECT 
    table_name,
    ROUND(index_length / 1024 / 1024, 2) AS index_mb
FROM information_schema.TABLES
WHERE TABLE_SCHEMA = DATABASE()
  AND TABLE_NAME IN ('perf_int', 'perf_uuid');
```

### ข้อ 10
สร้าง Multi-tenant sequence ที่แต่ละ tenant มี auto-increment ของตัวเอง

**เฉลย:**
```sql
CREATE TABLE tenant_sequences (
    tenant_id   INT UNSIGNED NOT NULL,
    seq_name    VARCHAR(50) NOT NULL,
    seq_value   BIGINT UNSIGNED NOT NULL DEFAULT 0,
    PRIMARY KEY (tenant_id, seq_name)
);

DELIMITER //
CREATE FUNCTION tenant_nextval(p_tenant_id INT, p_seq_name VARCHAR(50))
RETURNS BIGINT
DETERMINISTIC
MODIFIES SQL DATA
BEGIN
    DECLARE v_value BIGINT;
    
    INSERT INTO tenant_sequences (tenant_id, seq_name, seq_value)
    VALUES (p_tenant_id, p_seq_name, 1)
    ON DUPLICATE KEY UPDATE seq_value = seq_value + 1;
    
    SELECT seq_value INTO v_value
    FROM tenant_sequences
    WHERE tenant_id = p_tenant_id AND seq_name = p_seq_name;
    
    RETURN v_value;
END //
DELIMITER ;

-- Tenant A: order ของตัวเอง
SELECT tenant_nextval(1, 'order');  -- 1
SELECT tenant_nextval(1, 'order');  -- 2

-- Tenant B: order เริ่มที่ 1 ด้วย (แยกกัน)
SELECT tenant_nextval(2, 'order');  -- 1
SELECT tenant_nextval(2, 'order');  -- 2

-- ดูสถานะ
SELECT * FROM tenant_sequences ORDER BY tenant_id, seq_name;
```

---

## สรุป

ในบทนี้เราเรียนรู้:
1. **MySQL AUTO_INCREMENT**: syntax, LAST_INSERT_ID(), กำหนดค่าเริ่มต้น
2. **PostgreSQL SERIAL/BIGSERIAL**: equivalent ของ AUTO_INCREMENT
3. **GENERATED AS IDENTITY**: SQL Standard ที่ปลอดภัยกว่า SERIAL
4. **SQL Server IDENTITY**: SCOPE_IDENTITY() vs @@IDENTITY
5. **SQLite AUTOINCREMENT**: ป้องกัน ID reuse
6. **CREATE SEQUENCE**: control สูงกว่า, PostgreSQL native
7. **UUID**: globally unique, ใช้กับ distributed systems
8. **Gaps**: เป็นเรื่องปกติ อย่า assume sequential
9. **Human-readable IDs**: order numbers, employee codes

> **กฎทอง**: เลือก INT AUTO_INCREMENT สำหรับทั่วไป, BIGINT สำหรับข้อมูลขนาดใหญ่, UUID สำหรับ distributed systems หรือเมื่อต้องการ merge ข้อมูลจากหลาย databases
