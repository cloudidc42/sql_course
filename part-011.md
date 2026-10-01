# Part 11: SQL Data Types - คู่มือฉบับสมบูรณ์

## บทนำ

ชนิดข้อมูล (Data Types) คือพื้นฐานสำคัญที่สุดของการออกแบบฐานข้อมูล การเลือกชนิดข้อมูลที่เหมาะสมส่งผลต่อ:
- **ประสิทธิภาพ** (Performance) - ขนาดข้อมูลที่เล็กกว่า = เร็วกว่า
- **ความถูกต้อง** (Integrity) - ป้องกันข้อมูลผิดประเภท
- **พื้นที่จัดเก็บ** (Storage) - ประหยัดพื้นที่ดิสก์

## ตารางฐานข้อมูลตัวอย่าง

```sql
-- ตารางหลักที่ใช้ตลอดคอร์ส
CREATE TABLE departments (
    department_id   INT PRIMARY KEY,
    department_name VARCHAR(100) NOT NULL,
    location        VARCHAR(100),
    budget          DECIMAL(15,2),
    created_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE employees (
    employee_id     INT PRIMARY KEY,
    first_name      VARCHAR(50) NOT NULL,
    last_name       VARCHAR(50) NOT NULL,
    email           VARCHAR(100) UNIQUE,
    phone           VARCHAR(20),
    hire_date       DATE NOT NULL,
    job_title       VARCHAR(100),
    salary          DECIMAL(10,2),
    department_id   INT,
    is_active       BOOLEAN DEFAULT TRUE,
    FOREIGN KEY (department_id) REFERENCES departments(department_id)
);

CREATE TABLE products (
    product_id      INT PRIMARY KEY,
    product_name    VARCHAR(200) NOT NULL,
    category        VARCHAR(100),
    price           DECIMAL(10,2) NOT NULL,
    cost            DECIMAL(10,2),
    stock_quantity  INT DEFAULT 0,
    weight_kg       DECIMAL(8,3),
    description     TEXT,
    is_available    BOOLEAN DEFAULT TRUE,
    created_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE customers (
    customer_id     INT PRIMARY KEY,
    first_name      VARCHAR(50) NOT NULL,
    last_name       VARCHAR(50) NOT NULL,
    email           VARCHAR(100) UNIQUE NOT NULL,
    phone           VARCHAR(20),
    address         TEXT,
    city            VARCHAR(100),
    country         VARCHAR(100) DEFAULT 'Thailand',
    birth_date      DATE,
    created_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE orders (
    order_id        INT PRIMARY KEY,
    customer_id     INT NOT NULL,
    order_date      TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    status          VARCHAR(20) DEFAULT 'pending',
    total_amount    DECIMAL(12,2),
    shipping_fee    DECIMAL(8,2) DEFAULT 0,
    notes           TEXT,
    FOREIGN KEY (customer_id) REFERENCES customers(customer_id)
);

CREATE TABLE order_items (
    item_id         INT PRIMARY KEY,
    order_id        INT NOT NULL,
    product_id      INT NOT NULL,
    quantity        INT NOT NULL,
    unit_price      DECIMAL(10,2) NOT NULL,
    discount        DECIMAL(5,2) DEFAULT 0,
    FOREIGN KEY (order_id) REFERENCES orders(order_id),
    FOREIGN KEY (product_id) REFERENCES products(product_id)
);
```

---

## 1. ชนิดข้อมูลตัวเลข (Numeric Data Types)

### 1.1 Integer Types - จำนวนเต็ม

```sql
-- TINYINT: -128 ถึง 127 (signed) หรือ 0 ถึง 255 (unsigned)
-- ใช้งาน: อายุ, คะแนน, สถานะ
CREATE TABLE example_tinyint (
    age         TINYINT,           -- อายุ 0-127
    score       TINYINT UNSIGNED,  -- คะแนน 0-255 (MySQL)
    status_code TINYINT            -- -128 ถึง 127
);

-- SMALLINT: -32,768 ถึง 32,767
-- ใช้งาน: จำนวนสินค้าคงคลังขนาดเล็ก, ปี
CREATE TABLE example_smallint (
    year_value      SMALLINT,   -- เก็บปี เช่น 2024
    quantity        SMALLINT,   -- จำนวนน้อย
    port_number     SMALLINT    -- หมายเลข port 0-65535
);

-- INT / INTEGER: -2,147,483,648 ถึง 2,147,483,647
-- ใช้งาน: ID, จำนวนทั่วไป (ใช้บ่อยที่สุด)
CREATE TABLE example_int (
    product_id      INT PRIMARY KEY,
    stock           INT DEFAULT 0,
    view_count      INT DEFAULT 0,
    population      INT
);

-- BIGINT: -9,223,372,036,854,775,808 ถึง 9,223,372,036,854,775,807
-- ใช้งาน: ID ขนาดใหญ่, จำนวนเงิน (ถ้าเก็บเป็นสตางค์), timestamps
CREATE TABLE example_bigint (
    transaction_id  BIGINT PRIMARY KEY,
    amount_satang   BIGINT,         -- เก็บเงินเป็นสตางค์
    unix_timestamp  BIGINT,         -- Unix timestamp milliseconds
    total_clicks    BIGINT DEFAULT 0
);
```

### ตัวอย่างการใช้งานจริง - ระบบ HR

```sql
-- ตัวอย่าง 1: สร้างตาราง employees พร้อม integer types ที่เหมาะสม
CREATE TABLE hr_employees (
    employee_id     INT PRIMARY KEY AUTO_INCREMENT,    -- INT พอสำหรับ 2 พันล้านคน
    department_id   SMALLINT,                          -- แผนกไม่เกิน 32,767
    years_exp       TINYINT,                           -- ประสบการณ์ 0-127 ปี
    salary_thb      INT,                               -- เงินเดือนเป็นบาท
    total_sales     BIGINT DEFAULT 0                   -- ยอดขายรวมตลอดอาชีพ
);

-- ตัวอย่าง 2: Query เพื่อดูขนาด integer
SELECT 
    employee_id,
    CHAR_LENGTH(CAST(employee_id AS CHAR)) AS id_digits,
    salary_thb,
    FORMAT(salary_thb, 0) AS salary_formatted
FROM hr_employees;
```

### 1.2 Decimal Types - ทศนิยมแบบแม่นยำ

```sql
-- DECIMAL(precision, scale) / NUMERIC(precision, scale)
-- precision = จำนวนหลักทั้งหมด
-- scale = จำนวนหลักหลังจุดทศนิยม

-- ตัวอย่าง 3: DECIMAL สำหรับเงิน (ห้ามใช้ FLOAT!)
CREATE TABLE financial_records (
    record_id       INT PRIMARY KEY,
    -- DECIMAL(10,2) = สูงสุด 99,999,999.99
    amount          DECIMAL(10,2) NOT NULL,
    -- DECIMAL(15,4) = สูงสุด 99,999,999,999.9999
    exchange_rate   DECIMAL(15,4),
    -- DECIMAL(5,2) = สูงสุด 999.99 (เปอร์เซ็นต์)
    tax_rate        DECIMAL(5,2),
    -- DECIMAL(19,4) สำหรับเงินจำนวนมาก
    total_revenue   DECIMAL(19,4)
);

-- ตัวอย่าง 4: ใส่ข้อมูลตัวเลขทศนิยม
INSERT INTO financial_records VALUES 
(1, 1500.50, 35.2500, 7.00, 1500000.0000),
(2, 999.99, 35.1234, 7.00, 999990.0000),
(3, 0.01, 35.0000, 0.00, 10.0000);

-- ตัวอย่าง 5: การคำนวณกับ DECIMAL
SELECT 
    amount,
    tax_rate,
    amount * (tax_rate / 100) AS tax_amount,
    amount + (amount * tax_rate / 100) AS total_with_tax
FROM financial_records;
```

### 1.3 Floating Point Types - ทศนิยมแบบประมาณ

```sql
-- FLOAT / REAL: ความแม่นยำประมาณ 7 หลัก
-- DOUBLE / DOUBLE PRECISION: ความแม่นยำประมาณ 15 หลัก

-- ตัวอย่าง 6: เมื่อใช้ FLOAT (ข้อมูลวิทยาศาสตร์ที่ไม่ต้องแม่นยำ 100%)
CREATE TABLE sensor_readings (
    reading_id      INT PRIMARY KEY,
    temperature     FLOAT,          -- อุณหภูมิ (ทศนิยม ~7 หลัก)
    humidity        FLOAT,          -- ความชื้น
    pressure        DOUBLE,         -- ความดัน (ต้องการความแม่นยำสูงกว่า)
    latitude        DOUBLE,         -- พิกัด GPS (ต้องแม่นยำ)
    longitude       DOUBLE          -- พิกัด GPS (ต้องแม่นยำ)
);

-- ตัวอย่าง 7: ปัญหาของ FLOAT กับเงิน (อย่าทำแบบนี้!)
-- ผิด: ใช้ FLOAT เก็บเงิน
CREATE TABLE bad_example_prices (
    price FLOAT  -- อันตราย! อาจได้ 19.999999 แทน 20.00
);

INSERT INTO bad_example_prices VALUES (19.99);
SELECT price, price * 3 FROM bad_example_prices;
-- อาจได้ผล: 59.97000122070312 (ไม่แม่นยำ!)

-- ถูก: ใช้ DECIMAL เก็บเงิน
CREATE TABLE good_example_prices (
    price DECIMAL(10,2)  -- ดี! แม่นยำ 100%
);
```

---

## 2. ชนิดข้อมูลข้อความ (String Data Types)

### 2.1 CHAR - ความยาวคงที่

```sql
-- CHAR(n): ความยาวคงที่ n ตัวอักษร (เติม space ถ้าสั้นกว่า)
-- เหมาะกับ: รหัสที่มีความยาวคงที่ เช่น รหัสประเทศ, รหัสไปรษณีย์

-- ตัวอย่าง 8: CHAR สำหรับรหัสคงที่
CREATE TABLE location_codes (
    country_code    CHAR(2),        -- TH, US, JP
    postal_code     CHAR(5),        -- 10100
    currency_code   CHAR(3),        -- THB, USD
    language_code   CHAR(5),        -- th-TH, en-US
    isbn_code       CHAR(13)        -- ISBN หนังสือ
);

INSERT INTO location_codes VALUES 
('TH', '10100', 'THB', 'th-TH', '9780306406157'),
('US', '90210', 'USD', 'en-US', '9780131872752');

-- ตัวอย่าง 9: CHAR vs VARCHAR ขนาดจริง
-- CHAR(10) เก็บ 'Hi' = 'Hi        ' (10 bytes เสมอ)
-- VARCHAR(10) เก็บ 'Hi' = 'Hi' (2 bytes + 1 byte overhead)
SELECT 
    country_code,
    LENGTH(country_code) AS char_length,        -- ความยาวข้อมูล
    CHAR_LENGTH(country_code) AS string_length  -- ความยาวจริง
FROM location_codes;
```

### 2.2 VARCHAR - ความยาวแปรผัน

```sql
-- VARCHAR(n): ความยาวสูงสุด n ตัวอักษร
-- เหมาะกับ: ชื่อ, อีเมล, URL, ข้อความทั่วไป

-- ตัวอย่าง 10: VARCHAR ในระบบจริง
CREATE TABLE user_profiles (
    user_id         INT PRIMARY KEY,
    username        VARCHAR(30) NOT NULL,   -- ชื่อผู้ใช้
    email           VARCHAR(254) NOT NULL,  -- อีเมล (RFC 5321 max = 254)
    full_name       VARCHAR(200),           -- ชื่อเต็ม
    website_url     VARCHAR(2048),          -- URL (บาง browser รองรับ 2048)
    bio             VARCHAR(500),           -- ประวัติย่อ
    avatar_path     VARCHAR(500)            -- path รูปภาพ
);

-- ตัวอย่าง 11: เลือกความยาว VARCHAR ที่เหมาะสม
-- ชื่อคน: 50-100 ตัวอักษร
-- อีเมล: 254 ตัวอักษร (มาตรฐาน RFC)
-- URL: 2048 ตัวอักษร
-- รหัสผ่าน hash: 60-128 ตัวอักษร
-- UUID: 36 ตัวอักษร

CREATE TABLE auth_users (
    user_id         INT PRIMARY KEY,
    email           VARCHAR(254) UNIQUE NOT NULL,
    password_hash   VARCHAR(128) NOT NULL,  -- bcrypt hash
    reset_token     VARCHAR(64),            -- token รีเซ็ตรหัสผ่าน
    api_key         VARCHAR(64) UNIQUE      -- API key
);
```

### 2.3 TEXT Types - ข้อความยาว

```sql
-- TEXT: ข้อความยาว (ขนาดแตกต่างกันตาม DB)
-- PostgreSQL: TEXT ไม่จำกัดความยาว
-- MySQL: TINYTEXT(255), TEXT(65,535), MEDIUMTEXT(16MB), LONGTEXT(4GB)
-- SQLite: TEXT ไม่จำกัด

-- ตัวอย่าง 12: TEXT สำหรับเนื้อหายาว
CREATE TABLE articles (
    article_id      INT PRIMARY KEY,
    title           VARCHAR(300) NOT NULL,
    summary         VARCHAR(1000),          -- สรุปย่อ (VARCHAR เพราะรู้ขนาด)
    content         TEXT,                   -- เนื้อหาทั้งหมด (TEXT)
    meta_keywords   VARCHAR(500),
    meta_description VARCHAR(200),
    created_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- ตัวอย่าง 13: MySQL TEXT sizes
CREATE TABLE mysql_text_examples (
    tiny_text       TINYTEXT,       -- สูงสุด 255 bytes
    normal_text     TEXT,           -- สูงสุด 65,535 bytes (~64KB)
    medium_text     MEDIUMTEXT,     -- สูงสุด 16,777,215 bytes (~16MB)
    long_text       LONGTEXT        -- สูงสุด 4,294,967,295 bytes (~4GB)
);

-- ตัวอย่าง 14: ข้อจำกัดของ TEXT
-- TEXT ไม่สามารถ:
-- 1. ใช้เป็น PRIMARY KEY
-- 2. มี DEFAULT value (บางฐานข้อมูล)
-- 3. ใช้ใน index โดยตรง (ต้องระบุ prefix length)

-- MySQL: สร้าง index บน TEXT ต้องระบุความยาว
CREATE TABLE content_search (
    id      INT PRIMARY KEY,
    content TEXT,
    INDEX idx_content (content(100))  -- index 100 ตัวอักษรแรก
);
```

---

## 3. ชนิดข้อมูลวันที่และเวลา (Date/Time Data Types)

### 3.1 DATE

```sql
-- DATE: เก็บเฉพาะวันที่ 'YYYY-MM-DD'
-- ช่วง: 0001-01-01 ถึง 9999-12-31

-- ตัวอย่าง 15: DATE สำหรับวันเกิด, วันที่จ้างงาน
CREATE TABLE hr_info (
    employee_id     INT PRIMARY KEY,
    birth_date      DATE,           -- 1990-05-15
    hire_date       DATE NOT NULL,  -- 2020-01-01
    contract_end    DATE,           -- วันหมดสัญญา (NULL = ไม่มีกำหนด)
    last_review     DATE            -- วันประเมินล่าสุด
);

INSERT INTO hr_info VALUES
(1, '1990-05-15', '2020-01-01', NULL, '2023-12-01'),
(2, '1985-11-30', '2018-06-15', '2025-06-14', '2023-11-01');

-- ตัวอย่าง 16: การคำนวณกับ DATE
SELECT 
    employee_id,
    hire_date,
    CURRENT_DATE AS today,
    -- จำนวนปีที่ทำงาน
    TIMESTAMPDIFF(YEAR, hire_date, CURRENT_DATE) AS years_worked,  -- MySQL
    -- หรือใน PostgreSQL:
    -- EXTRACT(YEAR FROM AGE(CURRENT_DATE, hire_date)) AS years_worked
    -- หรือใน SQLite:
    -- (strftime('%Y', 'now') - strftime('%Y', hire_date)) AS years_worked
    birth_date,
    -- อายุ
    TIMESTAMPDIFF(YEAR, birth_date, CURRENT_DATE) AS age  -- MySQL
FROM hr_info;

-- ตัวอย่าง 17: DATE arithmetic
SELECT 
    hire_date,
    DATE_ADD(hire_date, INTERVAL 90 DAY) AS probation_end,    -- MySQL
    DATE_ADD(hire_date, INTERVAL 1 YEAR) AS first_anniversary, -- MySQL
    DATE_SUB(CURRENT_DATE, INTERVAL 30 DAY) AS thirty_days_ago -- MySQL
FROM hr_info;
```

### 3.2 TIME

```sql
-- TIME: เก็บเวลา 'HH:MM:SS'

-- ตัวอย่าง 18: TIME สำหรับตารางเวลาร้าน
CREATE TABLE store_hours (
    store_id        INT,
    day_of_week     TINYINT,        -- 0=อาทิตย์, 1=จันทร์, ...
    open_time       TIME NOT NULL,  -- 09:00:00
    close_time      TIME NOT NULL,  -- 22:00:00
    is_open         BOOLEAN DEFAULT TRUE
);

INSERT INTO store_hours VALUES
(1, 1, '09:00:00', '22:00:00', TRUE),
(1, 0, '10:00:00', '21:00:00', TRUE),
(1, 6, '10:00:00', '23:00:00', TRUE);

-- ตัวอย่าง 19: ตรวจสอบเวลาเปิด-ปิด
SELECT 
    store_id,
    day_of_week,
    open_time,
    close_time,
    TIMEDIFF(close_time, open_time) AS hours_open,
    CURRENT_TIME AS current_time,
    CASE 
        WHEN CURRENT_TIME BETWEEN open_time AND close_time THEN 'เปิด'
        ELSE 'ปิด'
    END AS status
FROM store_hours
WHERE day_of_week = DAYOFWEEK(CURRENT_DATE) - 1;
```

### 3.3 TIMESTAMP และ DATETIME

```sql
-- TIMESTAMP: วันที่และเวลา พร้อม timezone
-- MySQL: '1970-01-01 00:00:01' ถึง '2038-01-19 03:14:07' (Y2K38!)
-- PostgreSQL: timestamp with time zone ช่วงกว้างกว่า
-- DATETIME (MySQL): '1000-01-01' ถึง '9999-12-31' ไม่มี timezone

-- ตัวอย่าง 20: TIMESTAMP กับ audit trail
CREATE TABLE audit_log (
    log_id          BIGINT PRIMARY KEY AUTO_INCREMENT,
    table_name      VARCHAR(100),
    record_id       INT,
    action          VARCHAR(10),        -- INSERT, UPDATE, DELETE
    old_values      TEXT,
    new_values      TEXT,
    changed_by      INT,
    changed_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    -- MySQL: อัปเดตอัตโนมัติเมื่อ row เปลี่ยน
    updated_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);

-- ตัวอย่าง 21: การทำงานกับ TIMESTAMP
INSERT INTO audit_log (table_name, record_id, action, changed_by)
VALUES ('employees', 1, 'UPDATE', 5);

SELECT 
    log_id,
    changed_at,
    -- แปลง timezone
    CONVERT_TZ(changed_at, '+00:00', '+07:00') AS thai_time,  -- MySQL
    -- PostgreSQL: changed_at AT TIME ZONE 'Asia/Bangkok'
    DATE_FORMAT(changed_at, '%d/%m/%Y %H:%i:%s') AS formatted_time,  -- MySQL
    UNIX_TIMESTAMP(changed_at) AS unix_ts,  -- MySQL
    -- PostgreSQL: EXTRACT(EPOCH FROM changed_at)
    action
FROM audit_log;

-- ตัวอย่าง 22: INTERVAL สำหรับ PostgreSQL
-- PostgreSQL มี INTERVAL type พิเศษ
/*
CREATE TABLE events (
    event_id        SERIAL PRIMARY KEY,
    event_name      VARCHAR(200),
    start_time      TIMESTAMP WITH TIME ZONE,
    duration        INTERVAL,           -- เช่น '2 hours 30 minutes'
    recurring_every INTERVAL            -- เช่น '1 week'
);

INSERT INTO events VALUES
(1, 'Weekly Meeting', '2024-01-08 09:00:00+07', '1 hour', '1 week'),
(2, 'Annual Review', '2024-01-15 10:00:00+07', '2 hours 30 minutes', '1 year');

SELECT 
    event_name,
    start_time,
    start_time + duration AS end_time,
    start_time + recurring_every AS next_occurrence
FROM events;
*/
```

---

## 4. Boolean

```sql
-- BOOLEAN / BOOL: TRUE หรือ FALSE
-- MySQL: TINYINT(1) จริงๆ (0 = false, 1 = true)
-- PostgreSQL: TRUE native boolean
-- SQLite: INTEGER (0 หรือ 1)

-- ตัวอย่าง 23: Boolean ในระบบจริง
CREATE TABLE feature_flags (
    flag_id         INT PRIMARY KEY,
    flag_name       VARCHAR(100) UNIQUE,
    is_enabled      BOOLEAN DEFAULT FALSE,
    description     TEXT,
    updated_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

INSERT INTO feature_flags VALUES
(1, 'dark_mode', TRUE, 'เปิด dark mode', NOW()),
(2, 'beta_features', FALSE, 'ฟีเจอร์ทดสอบ', NOW()),
(3, 'email_notifications', TRUE, 'แจ้งเตือนทางอีเมล', NOW());

-- ตัวอย่าง 24: Query boolean
SELECT 
    flag_name,
    is_enabled,
    CASE is_enabled WHEN TRUE THEN 'เปิดใช้งาน' ELSE 'ปิดใช้งาน' END AS status
FROM feature_flags
WHERE is_enabled = TRUE;

-- ตัวอย่าง 25: Boolean arithmetic (MySQL)
SELECT 
    SUM(is_enabled) AS total_enabled,
    COUNT(*) - SUM(is_enabled) AS total_disabled,
    ROUND(SUM(is_enabled) / COUNT(*) * 100, 1) AS percent_enabled
FROM feature_flags;
```

---

## 5. Binary Data Types

```sql
-- BINARY(n): binary string ความยาวคงที่
-- VARBINARY(n): binary string ความยาวแปรผัน
-- BLOB: Binary Large Object

-- ตัวอย่าง 26: Binary สำหรับเก็บ hash
CREATE TABLE user_sessions (
    session_id      VARBINARY(32) PRIMARY KEY,  -- random bytes
    user_id         INT NOT NULL,
    session_token   BINARY(32),                 -- fixed-length token hash
    created_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    expires_at      TIMESTAMP
);

-- ตัวอย่าง 27: BLOB สำหรับ file storage (ไม่แนะนำสำหรับไฟล์ใหญ่)
CREATE TABLE document_storage (
    doc_id          INT PRIMARY KEY,
    filename        VARCHAR(255),
    file_type       VARCHAR(50),
    file_size       INT,
    -- MySQL BLOB types
    thumbnail       TINYBLOB,       -- สูงสุด 255 bytes
    preview         BLOB,           -- สูงสุด 65KB
    content         MEDIUMBLOB,     -- สูงสุด 16MB
    large_file      LONGBLOB        -- สูงสุด 4GB
    -- PostgreSQL: ใช้ BYTEA แทน
);

-- ตัวอย่าง 28: PostgreSQL BYTEA
/*
CREATE TABLE pg_files (
    file_id     SERIAL PRIMARY KEY,
    filename    VARCHAR(255),
    content     BYTEA   -- binary data
);

-- Insert binary data
INSERT INTO pg_files (filename, content)
VALUES ('test.txt', E'\\x48656c6c6f20576f726c64');  -- "Hello World" in hex

SELECT filename, encode(content, 'hex') AS hex_content FROM pg_files;
*/
```

---

## 6. ชนิดข้อมูลพิเศษ

### 6.1 JSON

```sql
-- JSON: ข้อมูล JSON format
-- MySQL 5.7+, PostgreSQL 9.2+ (JSON), 9.4+ (JSONB), SQLite ไม่รองรับ native

-- ตัวอย่าง 29: JSON ใน MySQL
CREATE TABLE product_attributes (
    product_id      INT PRIMARY KEY,
    product_name    VARCHAR(200),
    attributes      JSON,           -- {"color": "red", "size": "L", "weight": 1.5}
    tags            JSON,           -- ["electronics", "mobile", "5G"]
    metadata        JSON
);

INSERT INTO product_attributes VALUES
(1, 'iPhone 15', 
    '{"color": "black", "storage": "128GB", "display": "6.1 inch"}',
    '["smartphone", "apple", "ios"]',
    '{"sku": "IP15-BLK-128", "warranty_months": 12}'
);

-- ตัวอย่าง 30: Query JSON ใน MySQL
SELECT 
    product_name,
    attributes->>'$.color' AS color,               -- MySQL 5.7+
    JSON_EXTRACT(attributes, '$.storage') AS storage,
    JSON_UNQUOTE(JSON_EXTRACT(attributes, '$.display')) AS display,
    tags->>'$[0]' AS first_tag,
    JSON_LENGTH(tags) AS tag_count
FROM product_attributes;

-- ตัวอย่าง 31: JSON ใน PostgreSQL
/*
CREATE TABLE pg_products (
    id      SERIAL PRIMARY KEY,
    name    VARCHAR(200),
    specs   JSONB  -- JSONB ดีกว่า JSON เพราะ index ได้
);

INSERT INTO pg_products (name, specs) VALUES
('Laptop', '{"ram": 16, "cpu": "i7", "os": "Windows 11"}');

-- Query JSONB
SELECT 
    name,
    specs->>'ram' AS ram,
    specs->'cpu' AS cpu_json,        -- ได้ JSON value
    specs->>'cpu' AS cpu_text,       -- ได้ text value
    specs @> '{"ram": 16}' AS has_16gb_ram  -- containment check
FROM pg_products
WHERE specs->>'os' = 'Windows 11';

-- JSONB Index
CREATE INDEX idx_specs ON pg_products USING GIN(specs);
*/
```

### 6.2 UUID

```sql
-- UUID: Universally Unique Identifier
-- รูปแบบ: xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
-- ตัวอย่าง: 550e8400-e29b-41d4-a716-446655440000

-- ตัวอย่าง 32: UUID เป็น Primary Key
-- MySQL (ใช้ VARCHAR หรือ BINARY)
CREATE TABLE orders_uuid (
    order_id        CHAR(36) PRIMARY KEY DEFAULT (UUID()),  -- MySQL 8.0+
    customer_id     INT NOT NULL,
    total           DECIMAL(10,2),
    created_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- ตัวอย่าง 33: PostgreSQL UUID native type
/*
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";

CREATE TABLE pg_orders (
    order_id        UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    customer_id     INT NOT NULL,
    total           DECIMAL(10,2)
);

INSERT INTO pg_orders (customer_id, total) VALUES (1, 999.99);
SELECT * FROM pg_orders;
*/

-- ตัวอย่าง 34: UUID vs INT performance
-- INT: 4 bytes, เร็วกว่า, ต่อเนื่อง (ทำนายได้)
-- UUID: 16 bytes, ช้ากว่า แต่ unique across databases
-- ใช้ UUID เมื่อ: ต้องการ merge ข้อมูลจากหลาย DB, public-facing ID
```

### 6.3 ARRAY (PostgreSQL)

```sql
-- ตัวอย่าง 35: PostgreSQL ARRAY
/*
CREATE TABLE pg_user_preferences (
    user_id         INT PRIMARY KEY,
    favorite_colors TEXT[],         -- array of text
    scores          INT[],          -- array of integers
    work_days       SMALLINT[]      -- 1=จันทร์, ..., 7=อาทิตย์
);

INSERT INTO pg_user_preferences VALUES
(1, ARRAY['red', 'blue', 'green'], ARRAY[95, 87, 92], ARRAY[1,2,3,4,5]),
(2, '{"black", "white"}', '{100, 95}', '{1,2,3,4,5,6}');

-- Query Arrays
SELECT 
    user_id,
    favorite_colors,
    favorite_colors[1] AS first_color,    -- 1-indexed
    array_length(favorite_colors, 1) AS color_count,
    'blue' = ANY(favorite_colors) AS likes_blue
FROM pg_user_preferences;
*/
```

---

## 7. ตารางเปรียบเทียบระหว่าง Database Systems

| ชนิดข้อมูล | PostgreSQL | MySQL | SQLite | SQL Server |
|-----------|------------|-------|--------|------------|
| จำนวนเต็มเล็ก | SMALLINT | TINYINT/SMALLINT | INTEGER | TINYINT/SMALLINT |
| จำนวนเต็มปกติ | INTEGER | INT | INTEGER | INT |
| จำนวนเต็มใหญ่ | BIGINT | BIGINT | INTEGER | BIGINT |
| ทศนิยมแม่นยำ | DECIMAL/NUMERIC | DECIMAL/NUMERIC | REAL | DECIMAL/NUMERIC |
| ทศนิยมประมาณ | REAL/DOUBLE PRECISION | FLOAT/DOUBLE | REAL | FLOAT/REAL |
| ข้อความคงที่ | CHAR | CHAR | TEXT | CHAR |
| ข้อความแปรผัน | VARCHAR | VARCHAR | TEXT | VARCHAR/NVARCHAR |
| ข้อความยาว | TEXT | TEXT/MEDIUMTEXT/LONGTEXT | TEXT | TEXT/VARCHAR(MAX) |
| วันที่ | DATE | DATE | TEXT/INTEGER | DATE |
| วันเวลา | TIMESTAMP | DATETIME/TIMESTAMP | TEXT/INTEGER | DATETIME/DATETIME2 |
| Boolean | BOOLEAN | TINYINT(1) | INTEGER | BIT |
| Binary | BYTEA | BLOB/VARBINARY | BLOB | VARBINARY/IMAGE |
| JSON | JSON/JSONB | JSON | TEXT | NVARCHAR(MAX) |
| UUID | UUID | CHAR(36)/BINARY(16) | TEXT | UNIQUEIDENTIFIER |
| Auto ID | SERIAL/GENERATED | AUTO_INCREMENT | ROWID | IDENTITY |

---

## 8. Type Conversion และ Casting

```sql
-- ตัวอย่าง 36: CAST() - มาตรฐาน SQL
SELECT 
    CAST('123' AS INT) AS str_to_int,
    CAST(123.456 AS DECIMAL(5,1)) AS decimal_round,
    CAST(123 AS CHAR(10)) AS int_to_str,
    CAST('2024-01-15' AS DATE) AS str_to_date;

-- ตัวอย่าง 37: CONVERT() - MySQL
SELECT 
    CONVERT('123', UNSIGNED INT) AS str_to_uint,
    CONVERT(NOW(), DATE) AS datetime_to_date,
    CONVERT(123.456, DECIMAL(6,2)) AS convert_decimal;

-- ตัวอย่าง 38: PostgreSQL :: shorthand casting
/*
SELECT 
    '123'::INTEGER AS str_to_int,
    '2024-01-15'::DATE AS str_to_date,
    123::TEXT AS int_to_str,
    '3.14'::NUMERIC(5,2) AS str_to_decimal;
*/

-- ตัวอย่าง 39: Implicit vs Explicit casting
-- Implicit (อัตโนมัติ)
SELECT '5' + 3;          -- MySQL: 8 (แปลง '5' เป็น 5 อัตโนมัติ)
SELECT '5abc' + 3;       -- MySQL: 8 (แปลง '5abc' เป็น 5 แล้วบวก 3!)

-- Explicit (ระบุชัดเจน - ดีกว่า)
SELECT CAST('5' AS INT) + 3;   -- 8, ชัดเจน

-- ตัวอย่าง 40: การแปลงที่อาจผิดพลาด
-- ถ้าแปลงไม่ได้จะ error
-- SELECT CAST('abc' AS INT);  -- ERROR!

-- ป้องกัน error ด้วย NULLIF
SELECT CAST(NULLIF(TRIM(some_column), '') AS DECIMAL(10,2))
FROM some_table;

-- ตัวอย่าง 41: Format functions
SELECT 
    -- วันที่เป็น string
    DATE_FORMAT(NOW(), '%d/%m/%Y') AS thai_date,         -- MySQL
    -- PostgreSQL: TO_CHAR(NOW(), 'DD/MM/YYYY')
    -- SQLite: strftime('%d/%m/%Y', 'now')
    
    -- เลขเป็น string พร้อม format
    FORMAT(1234567.89, 2) AS formatted_number,            -- MySQL: 1,234,567.89
    -- PostgreSQL: TO_CHAR(1234567.89, 'FM9,999,999.99')
    
    -- string เป็นวันที่
    STR_TO_DATE('15/01/2024', '%d/%m/%Y') AS parsed_date; -- MySQL
    -- PostgreSQL: TO_DATE('15/01/2024', 'DD/MM/YYYY')
```

---

## 9. ขนาดข้อมูลและผลต่อ Performance

```sql
-- ตัวอย่าง 42: ขนาดข้อมูลแต่ละ type (โดยประมาณ)
/*
TINYINT:    1 byte
SMALLINT:   2 bytes
INT:        4 bytes
BIGINT:     8 bytes
FLOAT:      4 bytes
DOUBLE:     8 bytes
DECIMAL(p,s): ceil((p/2)+1) bytes (MySQL)
CHAR(n):    n bytes (MySQL/PostgreSQL)
VARCHAR(n): actual_length + 1-2 bytes
DATE:       3 bytes (MySQL)
TIME:       3 bytes (MySQL)
DATETIME:   8 bytes (MySQL)
TIMESTAMP:  4 bytes (MySQL) - Y2038 problem!
*/

-- ตัวอย่าง 43: ดูขนาดของ columns จริง
-- MySQL
SELECT 
    table_name,
    column_name,
    data_type,
    character_maximum_length,
    numeric_precision,
    numeric_scale,
    column_type
FROM information_schema.columns
WHERE table_schema = 'mydb'
  AND table_name = 'employees'
ORDER BY ordinal_position;

-- ตัวอย่าง 44: Performance เปรียบเทียบ
-- ตาราง 10 ล้าน rows
-- ค้นหา user ด้วย email (VARCHAR 254)
-- vs ค้นหา user ด้วย user_id (INT)
-- INT index เร็วกว่า VARCHAR ประมาณ 3-5 เท่า

-- ดังนั้น: ใช้ INT เป็น FK แทน VARCHAR/EMAIL ที่เป็น FK
CREATE TABLE good_design_orders (
    order_id        INT PRIMARY KEY,
    user_id         INT,            -- FK ไปยัง INT user_id (เร็ว)
    created_at      TIMESTAMP
);

-- ไม่ดี:
CREATE TABLE bad_design_orders (
    order_id        INT PRIMARY KEY,
    user_email      VARCHAR(254),   -- FK ไปยัง email (ช้ากว่า)
    created_at      TIMESTAMP
);
```

---

## 10. Common Mistakes และวิธีแก้ไข

```sql
-- ตัวอย่าง 45: ผิด - ใช้ VARCHAR เก็บตัวเลข
CREATE TABLE mistakes_example (
    phone           VARCHAR(20),     -- OK สำหรับโทรศัพท์
    zip_code        VARCHAR(10),     -- OK สำหรับรหัสไปรษณีย์
    price           VARCHAR(20),     -- ผิด! ควรใช้ DECIMAL
    quantity        VARCHAR(10)      -- ผิด! ควรใช้ INT
);

-- ถูก:
CREATE TABLE correct_example (
    phone           VARCHAR(20),     -- OK: โทรศัพท์มี +, -, space
    zip_code        CHAR(5),         -- OK: รหัสไปรษณีย์ความยาวคงที่
    price           DECIMAL(10,2),   -- ถูก: ตัวเลขทศนิยมที่แม่นยำ
    quantity        INT              -- ถูก: จำนวนเต็ม
);

-- ตัวอย่าง 46: ผิด - ใช้ FLOAT เก็บเงิน
-- ปัญหา floating point:
SELECT 0.1 + 0.2;                    -- MySQL: 0.30000000000000004!
SELECT CAST(0.1 AS DECIMAL(5,2)) + CAST(0.2 AS DECIMAL(5,2));  -- 0.30 (ถูก)

-- ตัวอย่าง 47: ผิด - TIMESTAMP Y2038 problem
-- TIMESTAMP ใน MySQL เก็บได้ถึงแค่ 2038-01-19!
-- ใช้ DATETIME แทนถ้าต้องการวันที่หลัง 2038
CREATE TABLE future_events (
    event_date      DATETIME,        -- ถูก: รองรับถึง 9999
    -- event_date   TIMESTAMP        -- ผิด: Y2038 problem
);

-- ตัวอย่าง 48: ผิด - ใช้ INT เก็บ ID ที่อาจเกิน 2 พันล้าน
-- social media platform ควรใช้ BIGINT
CREATE TABLE social_posts (
    post_id         BIGINT PRIMARY KEY AUTO_INCREMENT,  -- ถูก
    like_count      BIGINT DEFAULT 0,                   -- ถูก: อาจมีมากกว่า 2 พันล้าน
    -- post_id      INT  -- ผิด: อาจ overflow
);

-- ตัวอย่าง 49: ผิด - ใช้ TEXT สำหรับ column ที่ต้องใช้ index
-- TEXT ไม่สามารถ index ทั้งหมดได้ (ต้องระบุ prefix)
CREATE TABLE searchable_content (
    id              INT PRIMARY KEY,
    title           VARCHAR(500),    -- ถูก: index ได้ทั้งหมด
    -- title        TEXT,            -- ผิด: ถ้าต้องการ index
    FULLTEXT INDEX idx_title (title) -- Full-text search สำหรับ VARCHAR/TEXT
);

-- ตัวอย่าง 50: ผิด - ใช้ NULL กับ NOT NULL ไม่ถูกต้อง
-- NULL หมายถึง "ไม่มีข้อมูล" ไม่ใช่ 0 หรือ ''
CREATE TABLE careful_nulls (
    name            VARCHAR(100) NOT NULL,   -- ชื่อต้องมีเสมอ
    middle_name     VARCHAR(100),            -- ชื่อกลางอาจไม่มี (NULL OK)
    age             INT,                     -- อายุอาจไม่รู้ (NULL OK)
    balance         DECIMAL(10,2) DEFAULT 0, -- ยอดเงินเริ่มที่ 0 ไม่ใช่ NULL
    deleted_at      TIMESTAMP NULL           -- NULL = ยังไม่ถูกลบ
);
```

---

## 11. ตัวอย่างการออกแบบตารางสมบูรณ์

```sql
-- ตัวอย่าง 51: ระบบ E-commerce สมบูรณ์
CREATE TABLE ecommerce_products (
    -- Primary Key
    product_id          INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    
    -- ข้อมูลพื้นฐาน
    sku                 CHAR(20) UNIQUE NOT NULL,        -- SKU คงที่ 20 ตัว
    product_name        VARCHAR(300) NOT NULL,
    slug                VARCHAR(300) UNIQUE,             -- URL-friendly name
    
    -- ราคาและต้นทุน
    list_price          DECIMAL(10,2) NOT NULL,          -- ราคาตั้ง
    sale_price          DECIMAL(10,2),                   -- ราคาลด (NULL = ไม่ลด)
    cost_price          DECIMAL(10,2),                   -- ต้นทุน
    
    -- สต็อก
    stock_quantity      INT UNSIGNED DEFAULT 0,          -- ไม่ติดลบ
    min_stock_alert     SMALLINT UNSIGNED DEFAULT 10,    -- แจ้งเตือนเมื่อต่ำกว่านี้
    
    -- ขนาด/น้ำหนัก
    weight_grams        INT UNSIGNED,                    -- น้ำหนักหน่วย กรัม
    length_mm           SMALLINT UNSIGNED,               -- ความยาว mm
    width_mm            SMALLINT UNSIGNED,               -- ความกว้าง mm
    height_mm           SMALLINT UNSIGNED,               -- ความสูง mm
    
    -- เนื้อหา
    short_description   VARCHAR(1000),
    full_description    TEXT,
    specifications      TEXT,                            -- JSON string
    
    -- สถานะ
    is_active           BOOLEAN DEFAULT TRUE,
    is_featured         BOOLEAN DEFAULT FALSE,
    is_digital          BOOLEAN DEFAULT FALSE,
    
    -- SEO
    meta_title          VARCHAR(200),
    meta_description    VARCHAR(500),
    meta_keywords       VARCHAR(300),
    
    -- Timestamps
    created_at          TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at          TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    published_at        TIMESTAMP NULL,                  -- NULL = ยังไม่ publish
    
    -- Category
    category_id         SMALLINT UNSIGNED,
    brand_id            SMALLINT UNSIGNED
);

-- ตัวอย่าง 52: ระบบ HR สมบูรณ์
CREATE TABLE hr_full_employees (
    employee_id         INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    employee_code       CHAR(10) UNIQUE NOT NULL,       -- รหัสพนักงาน
    
    -- ชื่อ
    first_name          VARCHAR(100) NOT NULL,
    last_name           VARCHAR(100) NOT NULL,
    first_name_en       VARCHAR(100),
    last_name_en        VARCHAR(100),
    nick_name           VARCHAR(50),
    
    -- ติดต่อ
    personal_email      VARCHAR(254),
    work_email          VARCHAR(254) UNIQUE,
    personal_phone      VARCHAR(20),
    work_phone          VARCHAR(20),
    extension           CHAR(6),                        -- เบอร์ต่อ
    
    -- ข้อมูลส่วนตัว
    gender              CHAR(1),                        -- M, F, O
    birth_date          DATE,
    national_id         CHAR(13) UNIQUE,                -- บัตรประชาชน 13 หลัก
    passport_number     VARCHAR(20),
    
    -- ที่อยู่
    address_line1       VARCHAR(300),
    address_line2       VARCHAR(300),
    city                VARCHAR(100),
    province            VARCHAR(100),
    postal_code         CHAR(5),
    country             CHAR(2) DEFAULT 'TH',
    
    -- การทำงาน
    department_id       SMALLINT UNSIGNED,
    position_id         SMALLINT UNSIGNED,
    manager_id          INT UNSIGNED,                   -- self-referencing FK
    hire_date           DATE NOT NULL,
    contract_type       VARCHAR(20),                    -- FULL_TIME, PART_TIME, CONTRACT
    
    -- เงินเดือน
    base_salary         DECIMAL(10,2),
    currency            CHAR(3) DEFAULT 'THB',
    
    -- สถานะ
    employment_status   VARCHAR(20) DEFAULT 'ACTIVE',   -- ACTIVE, INACTIVE, TERMINATED
    is_active           BOOLEAN DEFAULT TRUE,
    termination_date    DATE NULL,
    termination_reason  VARCHAR(500),
    
    -- Timestamps
    created_at          TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at          TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);
```

---

## แบบฝึกหัด 10 ข้อ

### ข้อ 1
ออกแบบตาราง `bank_accounts` สำหรับระบบธนาคาร ระบุชนิดข้อมูลที่เหมาะสมสำหรับ: account_number, holder_name, balance, account_type, opened_date, is_active

**เฉลย:**
```sql
CREATE TABLE bank_accounts (
    account_id      BIGINT PRIMARY KEY AUTO_INCREMENT,
    account_number  CHAR(12) UNIQUE NOT NULL,      -- 12 หลักคงที่
    holder_name     VARCHAR(200) NOT NULL,
    balance         DECIMAL(15,2) DEFAULT 0.00,    -- เงินต้องแม่นยำ
    account_type    VARCHAR(20) NOT NULL,           -- SAVING, CURRENT, FIXED
    opened_date     DATE NOT NULL,
    is_active       BOOLEAN DEFAULT TRUE,
    created_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

### ข้อ 2
อธิบายความแตกต่างระหว่าง DECIMAL(10,2) กับ FLOAT และเมื่อใดควรใช้อะไร

**เฉลย:**
```sql
-- DECIMAL(10,2): แม่นยำ 100%, เก็บเป็น string internally
-- ใช้กับ: เงิน, ราคา, อัตราดอกเบี้ย
SELECT CAST(0.1 AS DECIMAL(5,2)) + CAST(0.2 AS DECIMAL(5,2));  -- 0.30 (ถูกต้อง)

-- FLOAT: ประมาณ, เร็วกว่า, อาจผิดพลาดเล็กน้อย
-- ใช้กับ: วิทยาศาสตร์, พิกัด GPS, สถิติที่ไม่ต้องแม่นยำ 100%
SELECT 0.1 + 0.2;  -- 0.30000000000000004 (ไม่แม่นยำ)
```

### ข้อ 3
เขียน query เพื่อแสดง employees ที่ทำงานมากกว่า 5 ปีแล้ว พร้อมแสดงจำนวนปีที่ทำงาน

**เฉลย:**
```sql
SELECT 
    employee_id,
    first_name,
    last_name,
    hire_date,
    TIMESTAMPDIFF(YEAR, hire_date, CURRENT_DATE) AS years_worked
FROM employees
WHERE TIMESTAMPDIFF(YEAR, hire_date, CURRENT_DATE) > 5
ORDER BY years_worked DESC;
```

### ข้อ 4
แปลงข้อมูลในตาราง products: price จาก VARCHAR เป็น DECIMAL

**เฉลย:**
```sql
-- ก่อนอื่น ตรวจสอบข้อมูล
SELECT price FROM products WHERE price NOT REGEXP '^[0-9]+(\.[0-9]+)?$';

-- แปลงโดยใช้ CAST
SELECT 
    product_id,
    product_name,
    price AS original,
    CAST(price AS DECIMAL(10,2)) AS converted
FROM products;

-- อัปเดตจริง (ทำหลัง backup)
ALTER TABLE products MODIFY COLUMN price DECIMAL(10,2);
```

### ข้อ 5
สร้างตาราง `sensor_data` สำหรับ IoT โดยมีข้อมูล: device_id, temperature, humidity, timestamp, location coordinates

**เฉลย:**
```sql
CREATE TABLE sensor_data (
    reading_id      BIGINT PRIMARY KEY AUTO_INCREMENT,  -- อาจมีข้อมูลมาก
    device_id       VARCHAR(36) NOT NULL,               -- UUID ของอุปกรณ์
    temperature     DECIMAL(5,2),                       -- -99.99 ถึง 999.99 °C
    humidity        DECIMAL(5,2),                       -- 0.00 ถึง 100.00 %
    pressure        DECIMAL(7,2),                       -- hPa
    latitude        DECIMAL(10,7),                      -- ±90.0000000
    longitude       DECIMAL(11,7),                      -- ±180.0000000
    recorded_at     TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_device (device_id),
    INDEX idx_time (recorded_at)
);
```

### ข้อ 6
เขียน query แสดง products ที่ stock หมด (0) และราคาต่ำกว่า 500 บาท พร้อม format ราคา

**เฉลย:**
```sql
SELECT 
    product_id,
    product_name,
    stock_quantity,
    FORMAT(price, 2) AS price_formatted,
    CONCAT('฿', FORMAT(price, 2)) AS price_with_currency
FROM products
WHERE stock_quantity = 0 
  AND price < 500.00
ORDER BY price ASC;
```

### ข้อ 7
อธิบายและแสดงตัวอย่าง Y2038 Problem ของ MySQL TIMESTAMP

**เฉลย:**
```sql
-- TIMESTAMP MySQL เก็บได้ถึง 2038-01-19 03:14:07 UTC
-- หลังจากนั้น overflow เป็น 0 (1970-01-01)

-- ทดสอบ
CREATE TABLE timestamp_test (
    id          INT PRIMARY KEY,
    ts          TIMESTAMP,
    dt          DATETIME
);

INSERT INTO timestamp_test VALUES 
(1, '2038-01-19 03:14:06', '2038-01-19 03:14:06'),  -- ก่อน Y2038 (OK)
(2, '2040-12-31 00:00:00', '2040-12-31 00:00:00');   -- หลัง Y2038 (TIMESTAMP อาจมีปัญหา)

-- ป้องกัน: ใช้ DATETIME แทน TIMESTAMP สำหรับวันที่หลัง 2038
```

### ข้อ 8
สร้างตาราง `user_addresses` ที่รองรับที่อยู่หลายประเทศ

**เฉลย:**
```sql
CREATE TABLE user_addresses (
    address_id      INT PRIMARY KEY AUTO_INCREMENT,
    user_id         INT NOT NULL,
    address_type    VARCHAR(20) NOT NULL,           -- HOME, WORK, SHIPPING
    address_line1   VARCHAR(300) NOT NULL,
    address_line2   VARCHAR(300),
    city            VARCHAR(100) NOT NULL,
    state_province  VARCHAR(100),                   -- NULL ได้ (บางประเทศไม่มี)
    postal_code     VARCHAR(20),                    -- VARCHAR เพราะรูปแบบต่างกัน
    country_code    CHAR(2) NOT NULL DEFAULT 'TH',  -- ISO 3166-1 alpha-2
    is_default      BOOLEAN DEFAULT FALSE,
    created_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

### ข้อ 9
เขียน query แปลง unix timestamp เป็นวันที่ไทย

**เฉลย:**
```sql
-- สมมติมี unix timestamp column
SELECT 
    1704067200 AS unix_ts,
    -- MySQL
    FROM_UNIXTIME(1704067200) AS mysql_datetime,
    DATE_FORMAT(FROM_UNIXTIME(1704067200), '%d/%m/%Y %H:%i:%s') AS thai_format,
    CONVERT_TZ(FROM_UNIXTIME(1704067200), '+00:00', '+07:00') AS thai_timezone;
    
-- PostgreSQL
-- SELECT to_timestamp(1704067200) AS pg_timestamp;
```

### ข้อ 10
ออกแบบตาราง `product_variants` สำหรับสินค้าที่มีหลายตัวเลือก (สี, ขนาด) พร้อมชนิดข้อมูลที่เหมาะสม

**เฉลย:**
```sql
CREATE TABLE product_variants (
    variant_id      INT UNSIGNED PRIMARY KEY AUTO_INCREMENT,
    product_id      INT UNSIGNED NOT NULL,
    sku             VARCHAR(50) UNIQUE NOT NULL,
    
    -- ตัวเลือก
    color           VARCHAR(50),
    size            VARCHAR(20),        -- S, M, L, XL หรือ 38, 40, 42
    material        VARCHAR(100),
    
    -- ราคา/ต้นทุน ที่อาจต่างจาก product หลัก
    price_adjustment DECIMAL(8,2) DEFAULT 0.00,  -- บวก/ลบจาก base price
    
    -- สต็อก
    stock_quantity  INT UNSIGNED DEFAULT 0,
    
    -- รูปภาพ
    image_url       VARCHAR(2048),
    
    -- ขนาด
    weight_grams    INT UNSIGNED,
    
    -- สถานะ
    is_active       BOOLEAN DEFAULT TRUE,
    
    created_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    
    FOREIGN KEY (product_id) REFERENCES products(product_id) ON DELETE CASCADE
);
```

---

## สรุป

ในบทนี้เราเรียนรู้:
1. **Numeric types**: INT สำหรับ ID/จำนวน, DECIMAL สำหรับเงิน, FLOAT สำหรับวิทยาศาสตร์
2. **String types**: CHAR สำหรับความยาวคงที่, VARCHAR สำหรับทั่วไป, TEXT สำหรับข้อความยาว
3. **Date/Time types**: DATE, TIME, TIMESTAMP/DATETIME และปัญหา Y2038
4. **Boolean**: TRUE/FALSE (TINYINT(1) ใน MySQL)
5. **Binary**: BLOB/BYTEA สำหรับไฟล์ binary
6. **Special**: JSON, UUID, ARRAY (PostgreSQL)
7. **Casting**: CAST(), CONVERT() และ :: ใน PostgreSQL
8. **Best practices**: เลือก type ให้เหมาะสม ไม่ใช้ FLOAT กับเงิน ระวัง Y2038

> **กฎทอง**: เลือก data type ที่เล็กที่สุดที่รองรับข้อมูลได้ถูกต้อง = ประสิทธิภาพดีที่สุด
