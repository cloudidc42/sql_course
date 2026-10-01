# Part 13: Inserting Data - INSERT

## บทนำ

`INSERT` เป็นคำสั่ง DML (Data Manipulation Language) สำหรับเพิ่มข้อมูลลงในตาราง การเข้าใจ INSERT อย่างลึกซึ้งช่วยให้:
- เพิ่มข้อมูลได้อย่างถูกต้องและรวดเร็ว
- จัดการกับ duplicate data ได้อย่างชาญฉลาด
- Import ข้อมูลจำนวนมากได้อย่างมีประสิทธิภาพ

---

## 1. INSERT พื้นฐาน

### 1.1 INSERT INTO ... VALUES

```sql
-- ตัวอย่าง 1: INSERT แบบระบุ columns (แนะนำ)
INSERT INTO departments (department_id, department_name, location, budget)
VALUES (1, 'Engineering', 'Bangkok', 5000000.00);

-- ตัวอย่าง 2: INSERT แบบไม่ระบุ columns (ต้องใส่ครบทุก column ตามลำดับ)
-- ไม่แนะนำ เพราะถ้าตารางเปลี่ยน column จะ error
INSERT INTO departments
VALUES (2, 'Marketing', 'Bangkok', 3000000.00, NOW());

-- ตัวอย่าง 3: INSERT บาง columns (columns ที่ไม่ระบุจะได้ DEFAULT หรือ NULL)
INSERT INTO departments (department_name, location)
VALUES ('HR', 'Chiang Mai');

-- ดูผล
SELECT * FROM departments;
```

### 1.2 INSERT Multiple Rows

```sql
-- ตัวอย่าง 4: INSERT หลาย rows ครั้งเดียว (ประหยัด network round-trip)
INSERT INTO departments (department_id, department_name, location, budget)
VALUES 
    (1, 'Engineering',  'Bangkok',      5000000.00),
    (2, 'Marketing',    'Bangkok',      3000000.00),
    (3, 'HR',           'Bangkok',      2000000.00),
    (4, 'Finance',      'Bangkok',      4000000.00),
    (5, 'Operations',   'Chiang Mai',   2500000.00),
    (6, 'Sales',        'Phuket',       3500000.00),
    (7, 'IT Support',   'Bangkok',      1500000.00),
    (8, 'Legal',        'Bangkok',      2000000.00);

-- ตัวอย่าง 5: INSERT หลาย rows สำหรับ employees
INSERT INTO employees 
    (employee_id, first_name, last_name, email, hire_date, salary, department_id)
VALUES 
    (1,  'สมชาย',   'ใจดี',      'somchai@company.com',   '2020-01-15', 55000, 1),
    (2,  'สมหญิง',  'รักดี',     'somying@company.com',   '2019-06-01', 65000, 1),
    (3,  'วิชัย',   'พงษ์ดี',   'wichai@company.com',    '2021-03-10', 45000, 2),
    (4,  'นงนุช',   'สุขใส',    'nongnuch@company.com',  '2018-11-20', 70000, 3),
    (5,  'ประยุทธ', 'ชัยดี',    'prayuth@company.com',   '2022-01-01', 50000, 2),
    (6,  'รัชดา',   'มีสุข',    'ratchada@company.com',  '2017-09-15', 80000, 4),
    (7,  'กาญจนา', 'ดีงาม',     'kanjana@company.com',   '2023-02-28', 42000, 5),
    (8,  'ธนภัทร', 'รุ่งเรือง', 'tanaphat@company.com',  '2016-07-01', 90000, 1),
    (9,  'มาลี',    'สุวรรณ',   'mali@company.com',      '2020-08-15', 48000, 3),
    (10, 'ชัชวาล', 'ดำรงค์',    'chatchawal@company.com','2019-04-20', 72000, 4);
```

### 1.3 INSERT สำหรับ customers และ products

```sql
-- ตัวอย่าง 6: INSERT customers
INSERT INTO customers 
    (customer_id, first_name, last_name, email, phone, city, birth_date)
VALUES 
    (1, 'อนุชา',   'สมบูรณ์',  'anucha@email.com',  '081-234-5678', 'กรุงเทพ',    '1990-05-15'),
    (2, 'บุษบา',   'รุ่งเรือง', 'bussaba@email.com', '082-345-6789', 'เชียงใหม่',  '1985-11-30'),
    (3, 'คงเดช',   'ทองดี',    'kongdech@email.com','083-456-7890', 'ภูเก็ต',     '1992-08-22'),
    (4, 'ดวงใจ',   'แสงทอง',   'duangjai@email.com','084-567-8901', 'กรุงเทพ',    '1988-03-10'),
    (5, 'เอกชัย',  'ศิริมงคล', 'ekachai@email.com', '085-678-9012', 'ขอนแก่น',    '1995-07-05'),
    (6, 'ฟ้าใส',   'ชัยพฤกษ์', 'fasai@email.com',   '086-789-0123', 'กรุงเทพ',    '1991-01-18'),
    (7, 'ไกรสร',   'เพชรดำ',   'kraisorn@email.com','087-890-1234', 'สงขลา',     '1987-09-25'),
    (8, 'หฤทัย',   'สุขสันต์', 'haruthai@email.com','088-901-2345', 'กรุงเทพ',    '1993-12-08');

-- ตัวอย่าง 7: INSERT products
INSERT INTO products 
    (product_id, product_name, category, price, cost, stock_quantity)
VALUES 
    (1,  'iPhone 15 Pro',          'Electronics', 42900.00, 35000.00, 50),
    (2,  'Samsung Galaxy S24',     'Electronics', 35900.00, 28000.00, 30),
    (3,  'MacBook Air M2',         'Electronics', 49900.00, 42000.00, 20),
    (4,  'Nike Air Max',           'Shoes',       4500.00,  2200.00,  100),
    (5,  'Adidas Ultraboost',      'Shoes',       5200.00,  2500.00,  80),
    (6,  'เสื้อยืด Uniqlo',        'Clothing',    590.00,   250.00,   200),
    (7,  'กางเกงยีนส์ Levi''s',   'Clothing',    2200.00,  900.00,   150),
    (8,  'หนังสือ SQL Mastery',    'Books',       350.00,   150.00,   500),
    (9,  'โต๊ะทำงาน IKEA',        'Furniture',   8500.00,  4500.00,  25),
    (10, 'เก้าอี้สำนักงาน',       'Furniture',   12000.00, 6000.00,  15),
    (11, 'กล้อง Canon EOS R50',   'Electronics', 29900.00, 24000.00, 10),
    (12, 'AirPods Pro',            'Electronics', 8900.00,  6500.00,  40);
```

---

## 2. INSERT INTO ... SELECT

```sql
-- ตัวอย่าง 8: Copy ข้อมูลจากตารางหนึ่งไปยังอีกตาราง
-- สมมติมีตาราง employees_old จากระบบเก่า
INSERT INTO employees (first_name, last_name, email, hire_date, salary)
SELECT 
    name_first,
    name_last,
    email_addr,
    start_date,
    monthly_pay
FROM employees_old
WHERE is_active = 1;

-- ตัวอย่าง 9: INSERT พร้อม transformation
INSERT INTO employees (first_name, last_name, email, hire_date, salary, department_id)
SELECT 
    TRIM(UPPER(first_name)),        -- ทำเป็นตัวพิมพ์ใหญ่และตัดช่องว่าง
    TRIM(UPPER(last_name)),
    LOWER(email),                   -- อีเมลเป็นตัวพิมพ์เล็ก
    hire_date,
    COALESCE(salary, 25000),        -- ถ้า salary NULL ใช้ 25000
    COALESCE(dept_id, 3)            -- ถ้าไม่มีแผนก ใส่แผนก 3 (HR)
FROM staging_employees;

-- ตัวอย่าง 10: Archive ข้อมูลเก่า
-- สร้างตาราง archive ก่อน
CREATE TABLE orders_archive AS SELECT * FROM orders WHERE 1=0;

-- ย้ายข้อมูลเก่ากว่า 2 ปี
INSERT INTO orders_archive
SELECT * FROM orders
WHERE order_date < DATE_SUB(CURRENT_DATE, INTERVAL 2 YEAR);

-- ลบจากตารางหลัก (หลังตรวจสอบแล้ว)
DELETE FROM orders
WHERE order_date < DATE_SUB(CURRENT_DATE, INTERVAL 2 YEAR);

-- ตัวอย่าง 11: INSERT INTO ... SELECT พร้อม aggregate
CREATE TABLE IF NOT EXISTS sales_daily_summary (
    summary_date    DATE PRIMARY KEY,
    total_orders    INT,
    total_revenue   DECIMAL(15,2),
    avg_order_value DECIMAL(10,2)
);

INSERT INTO sales_daily_summary (summary_date, total_orders, total_revenue, avg_order_value)
SELECT 
    DATE(order_date) AS summary_date,
    COUNT(*) AS total_orders,
    SUM(total_amount) AS total_revenue,
    AVG(total_amount) AS avg_order_value
FROM orders
WHERE status = 'completed'
  AND DATE(order_date) = CURRENT_DATE - INTERVAL 1 DAY  -- เมื่อวาน
GROUP BY DATE(order_date);
```

---

## 3. INSERT สำหรับ orders ระบบ

```sql
-- ตัวอย่าง 12: สร้าง orders พร้อม order_items
-- Step 1: ใส่ order หลัก
INSERT INTO orders (order_id, customer_id, order_date, status, total_amount)
VALUES 
    (1001, 1, '2024-01-15 10:30:00', 'completed', 47450.00),
    (1002, 2, '2024-01-16 14:00:00', 'completed', 5090.00),
    (1003, 3, '2024-01-17 09:15:00', 'pending',   42900.00),
    (1004, 1, '2024-01-18 16:45:00', 'completed', 350.00),
    (1005, 4, '2024-01-19 11:00:00', 'processing',8900.00),
    (1006, 5, '2024-01-20 13:30:00', 'shipped',   35900.00),
    (1007, 6, '2024-01-21 10:00:00', 'completed', 12700.00),
    (1008, 7, '2024-01-22 15:20:00', 'cancelled', 4500.00);

-- Step 2: ใส่ order items
INSERT INTO order_items (item_id, order_id, product_id, quantity, unit_price, discount)
VALUES 
    (1, 1001, 1,  1, 42900.00, 0),      -- iPhone 15 Pro
    (2, 1001, 6,  5, 590.00,   50.00),  -- เสื้อยืด 5 ตัว ลด 50
    (3, 1002, 4,  1, 4500.00,  0),      -- Nike Air Max
    (4, 1002, 8,  1, 350.00,   0),      -- หนังสือ SQL
    (5, 1002, 6,  1, 590.00,   0),      -- เสื้อยืด
    (6, 1003, 1,  1, 42900.00, 0),      -- iPhone 15 Pro
    (7, 1004, 8,  1, 350.00,   0),      -- หนังสือ
    (8, 1005, 12, 1, 8900.00,  0),      -- AirPods Pro
    (9, 1006, 2,  1, 35900.00, 0),      -- Samsung
    (10,1007, 9,  1, 8500.00,  0),      -- โต๊ะ IKEA
    (11,1007, 7,  1, 2200.00,  0),      -- กางเกง Levi's
    (12,1007, 6,  2, 590.00,   0),      -- เสื้อยืด 2 ตัว
    (13,1007, 8,  2, 350.00,   0),      -- หนังสือ 2 เล่ม
    (14,1008, 4,  1, 4500.00,  0);      -- Nike (cancelled)
```

---

## 4. INSERT OR IGNORE / INSERT IGNORE

```sql
-- ตัวอย่าง 13: MySQL - INSERT IGNORE (ข้ามถ้า duplicate key)
INSERT IGNORE INTO customers (customer_id, email, first_name, last_name)
VALUES 
    (1, 'anucha@email.com', 'อนุชา', 'สมบูรณ์'),  -- มีอยู่แล้ว -> ข้าม
    (9, 'new@email.com',    'ใหม่',   'นามสกุล');   -- ใหม่ -> ใส่

-- ตรวจสอบจำนวน rows ที่ถูกแทรก
SELECT ROW_COUNT() AS rows_affected;

-- ตัวอย่าง 14: SQLite - INSERT OR IGNORE
-- INSERT OR IGNORE INTO customers ...
-- SQLite รองรับ: INSERT OR FAIL, INSERT OR IGNORE, INSERT OR REPLACE, 
--               INSERT OR ROLLBACK, INSERT OR ABORT

-- ตัวอย่าง 15: PostgreSQL - INSERT ... ON CONFLICT DO NOTHING
INSERT INTO customers (customer_id, email, first_name, last_name)
VALUES (1, 'anucha@email.com', 'อนุชา', 'สมบูรณ์')
ON CONFLICT (customer_id) DO NOTHING;

-- ON CONFLICT กับ unique column อื่น
INSERT INTO customers (email, first_name, last_name)
VALUES ('anucha@email.com', 'อนุชา New', 'สมบูรณ์')
ON CONFLICT (email) DO NOTHING;
```

---

## 5. UPSERT - INSERT หรือ UPDATE ถ้ามีอยู่แล้ว

### 5.1 MySQL - ON DUPLICATE KEY UPDATE

```sql
-- ตัวอย่าง 16: ON DUPLICATE KEY UPDATE พื้นฐาน
INSERT INTO products (product_id, product_name, price, stock_quantity)
VALUES (1, 'iPhone 15 Pro', 42900.00, 50)
ON DUPLICATE KEY UPDATE
    product_name   = VALUES(product_name),
    price          = VALUES(price),
    stock_quantity = stock_quantity + VALUES(stock_quantity);
-- ถ้า product_id=1 มีอยู่แล้ว: update ชื่อ, ราคา, เพิ่มสต็อก
-- ถ้าไม่มี: insert ใหม่

-- ตัวอย่าง 17: Upsert สำหรับ inventory
INSERT INTO inventory (product_id, warehouse_id, quantity)
VALUES (1, 1, 100)
ON DUPLICATE KEY UPDATE
    quantity = quantity + VALUES(quantity),
    last_updated = NOW();

-- ตัวอย่าง 18: Upsert สำหรับ daily summary
INSERT INTO sales_daily_summary 
    (summary_date, total_orders, total_revenue)
SELECT 
    CURRENT_DATE,
    COUNT(*),
    SUM(total_amount)
FROM orders
WHERE DATE(order_date) = CURRENT_DATE
  AND status = 'completed'
ON DUPLICATE KEY UPDATE
    total_orders   = VALUES(total_orders),
    total_revenue  = VALUES(total_revenue);

-- ตัวอย่าง 19: Upsert พร้อมนับว่าเป็น insert หรือ update
INSERT INTO page_views (page_url, view_count, last_viewed)
VALUES ('/home', 1, NOW())
ON DUPLICATE KEY UPDATE
    view_count = view_count + 1,
    last_viewed = NOW();

SELECT 
    page_url,
    view_count,
    last_viewed,
    ROW_COUNT() AS affected  -- 1 = INSERT, 2 = UPDATE
FROM page_views
WHERE page_url = '/home';
```

### 5.2 PostgreSQL - ON CONFLICT DO UPDATE

```sql
-- ตัวอย่าง 20: PostgreSQL UPSERT
/*
INSERT INTO products (product_id, product_name, price, stock_quantity)
VALUES (1, 'iPhone 15 Pro', 42900.00, 50)
ON CONFLICT (product_id) DO UPDATE SET
    product_name   = EXCLUDED.product_name,
    price          = EXCLUDED.price,
    stock_quantity = products.stock_quantity + EXCLUDED.stock_quantity,
    updated_at     = NOW();
-- EXCLUDED. = ค่าที่พยายาม INSERT แต่ชนกัน

-- Upsert สำหรับ settings
INSERT INTO settings (key, value, updated_at)
VALUES ('theme', 'dark', NOW())
ON CONFLICT (key) DO UPDATE SET
    value = EXCLUDED.value,
    updated_at = EXCLUDED.updated_at
WHERE settings.value != EXCLUDED.value;  -- อัปเดตเฉพาะตอนค่าเปลี่ยน

-- ON CONFLICT DO NOTHING
INSERT INTO audit_log (event_id, event_data)
VALUES ('evt-001', '{"action": "login"}')
ON CONFLICT (event_id) DO NOTHING;
*/
```

### 5.3 SQLite - INSERT OR REPLACE

```sql
-- ตัวอย่าง 21: SQLite INSERT OR REPLACE
-- ลบ row เดิมทิ้งแล้ว insert ใหม่ (attention: ถ้ามี FK อาจมีปัญหา)
INSERT OR REPLACE INTO settings (key, value)
VALUES ('theme', 'dark');

-- SQLite INSERT OR IGNORE
INSERT OR IGNORE INTO customers (customer_id, email)
VALUES (1, 'existing@email.com');
```

---

## 6. INSERT WITH RETURNING (PostgreSQL)

```sql
-- ตัวอย่าง 22: PostgreSQL RETURNING
/*
-- ดึง auto-generated ID กลับมา
INSERT INTO customers (first_name, last_name, email)
VALUES ('John', 'Doe', 'john@example.com')
RETURNING customer_id, created_at;

-- RETURNING หลาย rows
INSERT INTO products (product_name, price)
VALUES 
    ('Product A', 100.00),
    ('Product B', 200.00),
    ('Product C', 300.00)
RETURNING product_id, product_name;

-- ใช้ RETURNING ใน CTE
WITH new_order AS (
    INSERT INTO orders (customer_id, total_amount)
    VALUES (1, 5000.00)
    RETURNING order_id
)
INSERT INTO order_items (order_id, product_id, quantity, unit_price)
SELECT 
    new_order.order_id,
    1,
    1,
    42900.00
FROM new_order;
*/

-- MySQL เทียบเท่า (ใช้ LAST_INSERT_ID())
INSERT INTO orders (customer_id, total_amount)
VALUES (1, 5000.00);

SET @new_order_id = LAST_INSERT_ID();

INSERT INTO order_items (order_id, product_id, quantity, unit_price)
VALUES (@new_order_id, 1, 1, 42900.00);

SELECT @new_order_id AS new_order_id;
```

---

## 7. Bulk Insert Patterns

```sql
-- ตัวอย่าง 23: Batch insert สำหรับข้อมูลจำนวนมาก
-- แทนที่จะ insert ทีละ row (ช้ามาก)
-- ผิด: 10,000 individual inserts
-- INSERT INTO data VALUES (1, 'a');
-- INSERT INTO data VALUES (2, 'b');
-- ... x 10,000

-- ถูก: batch insert (เร็วกว่ามาก)
INSERT INTO data VALUES 
    (1, 'a'), (2, 'b'), (3, 'c'), ..., (1000, 'zzz');
-- หรือใส่ 500-1000 rows ต่อ statement

-- ตัวอย่าง 24: Disable indexes ระหว่าง bulk insert (MySQL)
-- สำหรับ MyISAM:
ALTER TABLE large_table DISABLE KEYS;
INSERT INTO large_table SELECT * FROM source_table;
ALTER TABLE large_table ENABLE KEYS;

-- สำหรับ InnoDB:
SET foreign_key_checks = 0;
SET unique_checks = 0;
SET sql_log_bin = 0;  -- ถ้าไม่ต้องการ replication

-- Bulk insert
INSERT INTO large_table (col1, col2, col3)
SELECT col1, col2, col3 FROM source_table;

SET foreign_key_checks = 1;
SET unique_checks = 1;

-- ตัวอย่าง 25: Transaction สำหรับ Bulk Insert
START TRANSACTION;

INSERT INTO products (product_name, price, stock_quantity)
SELECT 
    CONCAT('Product ', n),
    ROUND(RAND() * 1000 + 10, 2),
    FLOOR(RAND() * 100)
FROM (
    SELECT a.N + b.N * 10 + c.N * 100 + 1 AS n
    FROM (SELECT 0 AS N UNION SELECT 1 UNION SELECT 2 UNION SELECT 3 UNION SELECT 4 
          UNION SELECT 5 UNION SELECT 6 UNION SELECT 7 UNION SELECT 8 UNION SELECT 9) a,
         (SELECT 0 AS N UNION SELECT 1 UNION SELECT 2 UNION SELECT 3 UNION SELECT 4 
          UNION SELECT 5 UNION SELECT 6 UNION SELECT 7 UNION SELECT 8 UNION SELECT 9) b,
         (SELECT 0 AS N UNION SELECT 1 UNION SELECT 2 UNION SELECT 3 UNION SELECT 4 
          UNION SELECT 5 UNION SELECT 6 UNION SELECT 7 UNION SELECT 8 UNION SELECT 9) c
) AS numbers
WHERE n <= 100;

COMMIT;
```

---

## 8. INSERT Patterns สำหรับ Real-World Scenarios

### 8.1 Order Processing

```sql
-- ตัวอย่าง 26: สร้าง order ใหม่พร้อม transaction
DELIMITER //
CREATE PROCEDURE create_order(
    IN p_customer_id    INT,
    IN p_product_ids    VARCHAR(500),  -- comma-separated product IDs
    IN p_quantities     VARCHAR(500)   -- comma-separated quantities
)
BEGIN
    DECLARE v_order_id  INT;
    DECLARE v_total     DECIMAL(12,2) DEFAULT 0;
    
    START TRANSACTION;
    
    -- สร้าง order
    INSERT INTO orders (customer_id, status, total_amount)
    VALUES (p_customer_id, 'pending', 0);
    
    SET v_order_id = LAST_INSERT_ID();
    
    -- ใส่ items (simplified)
    INSERT INTO order_items (order_id, product_id, quantity, unit_price)
    SELECT 
        v_order_id,
        p.product_id,
        1,          -- simplified
        p.price
    FROM products p
    WHERE p.product_id IN (1, 2, 3);  -- simplified
    
    -- อัปเดต total
    UPDATE orders SET total_amount = (
        SELECT SUM(quantity * unit_price) FROM order_items WHERE order_id = v_order_id
    ) WHERE order_id = v_order_id;
    
    COMMIT;
    
    SELECT v_order_id AS new_order_id;
END //
DELIMITER ;
```

### 8.2 Audit Logging

```sql
-- ตัวอย่าง 27: Insert ลง audit log อัตโนมัติ
CREATE TABLE IF NOT EXISTS audit_trail (
    audit_id        BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    table_name      VARCHAR(100)    NOT NULL,
    record_id       INT UNSIGNED,
    action          VARCHAR(10)     NOT NULL,   -- INSERT, UPDATE, DELETE
    old_values      JSON,
    new_values      JSON,
    user_id         INT UNSIGNED,
    ip_address      VARCHAR(45),
    created_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Trigger INSERT audit
DELIMITER //
CREATE TRIGGER trg_employee_insert_audit
AFTER INSERT ON employees
FOR EACH ROW
BEGIN
    INSERT INTO audit_trail (table_name, record_id, action, new_values, user_id)
    VALUES (
        'employees',
        NEW.employee_id,
        'INSERT',
        JSON_OBJECT(
            'first_name', NEW.first_name,
            'last_name', NEW.last_name,
            'email', NEW.email,
            'salary', NEW.salary,
            'department_id', NEW.department_id
        ),
        @current_user_id
    );
END //
DELIMITER ;

-- ทดสอบ
SET @current_user_id = 1;
INSERT INTO employees (first_name, last_name, email, hire_date, salary, department_id)
VALUES ('ทดสอบ', 'ระบบ', 'test@company.com', CURRENT_DATE, 30000, 1);

SELECT * FROM audit_trail ORDER BY created_at DESC LIMIT 5;
```

### 8.3 Session/Token Management

```sql
-- ตัวอย่าง 28: สร้าง session token
CREATE TABLE IF NOT EXISTS user_sessions (
    session_id      CHAR(64)        PRIMARY KEY,    -- random hex token
    user_id         INT UNSIGNED    NOT NULL,
    ip_address      VARCHAR(45),
    user_agent      VARCHAR(500),
    created_at      TIMESTAMP       DEFAULT CURRENT_TIMESTAMP,
    expires_at      TIMESTAMP       NOT NULL,
    last_active     TIMESTAMP       DEFAULT CURRENT_TIMESTAMP,
    is_active       BOOLEAN         DEFAULT TRUE
);

-- สร้าง session ใหม่
INSERT INTO user_sessions (session_id, user_id, ip_address, expires_at)
VALUES (
    SHA2(CONCAT(UUID(), RAND()), 256),  -- random session token
    1,
    '127.0.0.1',
    DATE_ADD(NOW(), INTERVAL 30 DAY)
);

-- Cleanup sessions หมดอายุ
DELETE FROM user_sessions 
WHERE expires_at < NOW() OR is_active = FALSE;
```

---

## 9. INSERT Best Practices

```sql
-- ตัวอย่าง 29: ตรวจสอบก่อน INSERT
-- ป้องกัน duplicate ด้วย NOT EXISTS
INSERT INTO email_subscriptions (email, category)
SELECT 'user@example.com', 'newsletter'
WHERE NOT EXISTS (
    SELECT 1 FROM email_subscriptions 
    WHERE email = 'user@example.com' 
      AND category = 'newsletter'
);

-- ตัวอย่าง 30: INSERT พร้อม validation
-- MySQL: ใช้ CHECK constraint (MySQL 8.0.16+)
CREATE TABLE validated_orders (
    order_id        INT PRIMARY KEY AUTO_INCREMENT,
    customer_id     INT NOT NULL,
    amount          DECIMAL(10,2) NOT NULL,
    status          VARCHAR(20) DEFAULT 'pending',
    
    CONSTRAINT chk_amount   CHECK (amount > 0),
    CONSTRAINT chk_status   CHECK (status IN ('pending','confirmed','cancelled'))
);

-- ถ้า insert ด้วย amount ติดลบ -> error
INSERT INTO validated_orders (customer_id, amount, status)
VALUES (1, -100.00, 'pending');  -- ERROR: Check constraint violated

-- ตัวอย่าง 31: INSERT performance tips
-- 1. Wrap multiple inserts ใน transaction
START TRANSACTION;
INSERT INTO logs VALUES (1, 'event1');
INSERT INTO logs VALUES (2, 'event2');
-- ... many inserts
COMMIT;

-- 2. ใช้ multi-row insert แทนหลาย single inserts
-- ช้า:
INSERT INTO t VALUES (1);
INSERT INTO t VALUES (2);
INSERT INTO t VALUES (3);

-- เร็ว:
INSERT INTO t VALUES (1), (2), (3);

-- 3. สำหรับ bulk insert ใหญ่ๆ: disable autocommit
SET autocommit = 0;
-- ... หลาย inserts
COMMIT;
SET autocommit = 1;
```

---

## 10. Database-Specific INSERT Features

### MySQL Special Features

```sql
-- ตัวอย่าง 32: MySQL INSERT ... SET syntax (alternative)
INSERT INTO employees SET
    first_name = 'มาใหม่',
    last_name = 'พนักงาน',
    email = 'newstaff@company.com',
    hire_date = CURRENT_DATE,
    salary = 35000,
    department_id = 1;

-- ตัวอย่าง 33: MySQL HIGH_PRIORITY / LOW_PRIORITY
-- LOW_PRIORITY: รอจนกว่า read operations จะเสร็จ (ใช้กับ MyISAM)
INSERT LOW_PRIORITY INTO logs (message) VALUES ('background log');

-- HIGH_PRIORITY: ทำก่อน read operations (ใช้กับ MyISAM)
INSERT HIGH_PRIORITY INTO critical_logs (message) VALUES ('critical event');

-- ตัวอย่าง 34: MySQL DELAYED (deprecated ใน 5.6)
-- INSERT DELAYED ใช้ได้กับ MyISAM เท่านั้น และถูกยกเลิกแล้ว
-- ทางเลือก: ใช้ Message Queue เช่น Redis, RabbitMQ แทน
```

### PostgreSQL Special Features

```sql
-- ตัวอย่าง 35: PostgreSQL COPY (เร็วกว่า INSERT มาก)
/*
-- Copy จาก CSV file
COPY customers (first_name, last_name, email)
FROM '/path/to/customers.csv'
WITH (FORMAT CSV, HEADER TRUE);

-- Copy ไปยัง CSV file
COPY (SELECT * FROM customers WHERE city = 'Bangkok')
TO '/path/to/bangkok_customers.csv'
WITH (FORMAT CSV, HEADER TRUE);
*/
```

### SQL Server Special Features

```sql
-- ตัวอย่าง 36: SQL Server OUTPUT clause
/*
DECLARE @InsertedRows TABLE (
    employee_id INT,
    email VARCHAR(254),
    inserted_at DATETIME
);

INSERT INTO employees (first_name, last_name, email, hire_date, salary)
OUTPUT INSERTED.employee_id, INSERTED.email, GETDATE()
INTO @InsertedRows
VALUES ('John', 'Doe', 'john@company.com', GETDATE(), 50000);

SELECT * FROM @InsertedRows;
*/
```

---

## 11. Handling NULL in INSERT

```sql
-- ตัวอย่าง 37: การจัดการ NULL
-- Explicit NULL
INSERT INTO employees (first_name, last_name, email, phone, hire_date, salary, department_id)
VALUES ('นาม', 'สกุล', 'nam@company.com', NULL, CURRENT_DATE, 40000, NULL);
-- phone = NULL, department_id = NULL

-- Implicit NULL (ไม่ระบุ column)
INSERT INTO employees (first_name, last_name, email, hire_date, salary)
VALUES ('นาม2', 'สกุล2', 'nam2@company.com', CURRENT_DATE, 40000);
-- phone, department_id จะเป็น NULL (ถ้าไม่มี DEFAULT)

-- DEFAULT keyword
INSERT INTO employees (first_name, last_name, email, hire_date, salary, is_active)
VALUES ('นาม3', 'สกุล3', 'nam3@company.com', CURRENT_DATE, 40000, DEFAULT);
-- is_active จะได้ค่า DEFAULT (TRUE)

-- ตัวอย่าง 38: COALESCE ใน INSERT
INSERT INTO employees (first_name, last_name, email, hire_date, salary, department_id)
SELECT 
    first_name,
    last_name,
    email,
    COALESCE(hire_date, CURRENT_DATE),    -- ถ้า NULL ใช้วันนี้
    COALESCE(salary, 25000),              -- ถ้า NULL ใช้ 25000
    COALESCE(department_id, 3)            -- ถ้า NULL ใช้ HR (3)
FROM new_employee_import;
```

---

## แบบฝึกหัด 10 ข้อ

### ข้อ 1
Insert ข้อมูลพนักงานใหม่ 3 คน โดยใช้ multi-row INSERT

**เฉลย:**
```sql
INSERT INTO employees (first_name, last_name, email, hire_date, salary, department_id)
VALUES 
    ('สุภาพ', 'ใจงาม',   'supap@company.com',  CURRENT_DATE, 45000, 2),
    ('มณี',   'แก้วดี',  'manee@company.com',  CURRENT_DATE, 38000, 3),
    ('สิทธิ', 'พลังดี',  'sitti@company.com',  CURRENT_DATE, 52000, 1);
```

### ข้อ 2
Copy ข้อมูล customers ที่อยู่กรุงเทพไปยัง table `vip_customers` (สร้าง table และ insert)

**เฉลย:**
```sql
CREATE TABLE vip_customers AS
SELECT * FROM customers WHERE 1=0;  -- สร้างโครงสร้าง

INSERT INTO vip_customers
SELECT * FROM customers
WHERE city = 'กรุงเทพ';

SELECT COUNT(*) FROM vip_customers;
```

### ข้อ 3
สร้าง order ใหม่สำหรับ customer_id=1 สั่งสินค้า product_id=1 จำนวน 2 ชิ้น

**เฉลย:**
```sql
START TRANSACTION;

INSERT INTO orders (customer_id, order_date, status, total_amount)
VALUES (1, NOW(), 'pending', 0);

SET @oid = LAST_INSERT_ID();

INSERT INTO order_items (order_id, product_id, quantity, unit_price)
SELECT @oid, 1, 2, price FROM products WHERE product_id = 1;

UPDATE orders SET total_amount = (
    SELECT SUM(quantity * unit_price) FROM order_items WHERE order_id = @oid
) WHERE order_id = @oid;

COMMIT;

SELECT o.*, oi.product_id, oi.quantity, oi.unit_price
FROM orders o
JOIN order_items oi ON o.order_id = oi.order_id
WHERE o.order_id = @oid;
```

### ข้อ 4
ใช้ INSERT IGNORE เพื่อ import customers โดยข้าม duplicates (email ซ้ำ)

**เฉลย:**
```sql
-- สร้าง staging table พร้อมข้อมูลทดสอบ
CREATE TEMPORARY TABLE staging_customers (
    email       VARCHAR(254),
    first_name  VARCHAR(100),
    last_name   VARCHAR(100),
    city        VARCHAR(100)
);

INSERT INTO staging_customers VALUES
    ('anucha@email.com', 'อนุชา', 'สมบูรณ์', 'กรุงเทพ'),   -- duplicate
    ('new1@email.com', 'ใหม่1', 'สกุล', 'กรุงเทพ'),         -- new
    ('new2@email.com', 'ใหม่2', 'สกุล', 'เชียงใหม่');        -- new

-- Insert โดยข้าม duplicates
INSERT IGNORE INTO customers (email, first_name, last_name, city)
SELECT email, first_name, last_name, city FROM staging_customers;

SELECT ROW_COUNT() AS rows_inserted;
```

### ข้อ 5
Implement upsert สำหรับ product price update (MySQL)

**เฉลย:**
```sql
-- สร้าง table สำหรับ price updates
CREATE TEMPORARY TABLE price_updates (
    product_id  INT,
    new_price   DECIMAL(10,2)
);

INSERT INTO price_updates VALUES
    (1, 41900.00),  -- iPhone ลดราคา
    (2, 36900.00),  -- Samsung เพิ่มราคา
    (99, 999.00);   -- สินค้าใหม่ (ไม่มีใน products)

-- Upsert price
INSERT INTO products (product_id, product_name, price, stock_quantity)
SELECT pu.product_id, CONCAT('Product ', pu.product_id), pu.new_price, 0
FROM price_updates pu
ON DUPLICATE KEY UPDATE
    price = VALUES(price),
    updated_at = NOW();

SELECT product_id, product_name, price FROM products WHERE product_id IN (1, 2, 99);
```

### ข้อ 6
INSERT ข้อมูลจาก subquery: เพิ่ม employees ที่มีเงินเดือนสูงกว่าค่าเฉลี่ยลงใน high_earners table

**เฉลย:**
```sql
CREATE TABLE IF NOT EXISTS high_earners (
    employee_id     INT PRIMARY KEY,
    full_name       VARCHAR(200),
    salary          DECIMAL(10,2),
    dept_name       VARCHAR(100),
    categorized_at  TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

INSERT INTO high_earners (employee_id, full_name, salary, dept_name)
SELECT 
    e.employee_id,
    CONCAT(e.first_name, ' ', e.last_name),
    e.salary,
    d.department_name
FROM employees e
JOIN departments d ON e.department_id = d.department_id
WHERE e.salary > (SELECT AVG(salary) FROM employees);
```

### ข้อ 7
Insert พร้อมตรวจสอบ stock ก่อน (ป้องกัน negative stock)

**เฉลย:**
```sql
DELIMITER //
CREATE PROCEDURE safe_order_insert(
    IN p_customer_id    INT,
    IN p_product_id     INT,
    IN p_quantity       INT
)
BEGIN
    DECLARE v_stock     INT;
    DECLARE v_price     DECIMAL(10,2);
    DECLARE v_order_id  INT;
    
    -- ตรวจสอบ stock
    SELECT stock_quantity, price 
    INTO v_stock, v_price
    FROM products 
    WHERE product_id = p_product_id;
    
    IF v_stock IS NULL THEN
        SIGNAL SQLSTATE '45000' SET MESSAGE_TEXT = 'Product not found';
    ELSEIF v_stock < p_quantity THEN
        SIGNAL SQLSTATE '45000' SET MESSAGE_TEXT = 'Insufficient stock';
    END IF;
    
    -- ถ้า stock พอ ทำ order
    START TRANSACTION;
    
    INSERT INTO orders (customer_id, status, total_amount)
    VALUES (p_customer_id, 'confirmed', v_price * p_quantity);
    
    SET v_order_id = LAST_INSERT_ID();
    
    INSERT INTO order_items (order_id, product_id, quantity, unit_price)
    VALUES (v_order_id, p_product_id, p_quantity, v_price);
    
    -- ลด stock
    UPDATE products 
    SET stock_quantity = stock_quantity - p_quantity
    WHERE product_id = p_product_id;
    
    COMMIT;
    
    SELECT v_order_id AS order_id, 'Order created successfully' AS message;
END //
DELIMITER ;
```

### ข้อ 8
INSERT สร้าง report ของวันนี้ใน table daily_stats

**เฉลย:**
```sql
CREATE TABLE IF NOT EXISTS daily_stats (
    stat_date           DATE PRIMARY KEY,
    new_customers       INT DEFAULT 0,
    new_orders          INT DEFAULT 0,
    revenue             DECIMAL(15,2) DEFAULT 0,
    avg_order_value     DECIMAL(10,2) DEFAULT 0,
    top_product_id      INT
);

INSERT INTO daily_stats (stat_date, new_customers, new_orders, revenue, avg_order_value, top_product_id)
SELECT 
    CURRENT_DATE,
    (SELECT COUNT(*) FROM customers WHERE DATE(created_at) = CURRENT_DATE),
    COUNT(o.order_id),
    COALESCE(SUM(o.total_amount), 0),
    COALESCE(AVG(o.total_amount), 0),
    (SELECT oi2.product_id 
     FROM order_items oi2 
     JOIN orders o2 ON oi2.order_id = o2.order_id
     WHERE DATE(o2.order_date) = CURRENT_DATE
     GROUP BY oi2.product_id 
     ORDER BY SUM(oi2.quantity) DESC 
     LIMIT 1)
FROM orders o
WHERE DATE(o.order_date) = CURRENT_DATE
  AND o.status != 'cancelled'
ON DUPLICATE KEY UPDATE
    new_customers = VALUES(new_customers),
    new_orders = VALUES(new_orders),
    revenue = VALUES(revenue),
    avg_order_value = VALUES(avg_order_value),
    top_product_id = VALUES(top_product_id);
```

### ข้อ 9
INSERT ข้อมูลสำหรับ many-to-many relationship (product_tags)

**เฉลย:**
```sql
CREATE TABLE IF NOT EXISTS product_tags_table (
    product_id  INT UNSIGNED NOT NULL,
    tag_name    VARCHAR(50) NOT NULL,
    PRIMARY KEY (product_id, tag_name)
);

-- Insert tags สำหรับ products หลายตัวพร้อมกัน
INSERT INTO product_tags_table (product_id, tag_name)
VALUES 
    (1, 'smartphone'), (1, 'apple'), (1, 'ios'), (1, 'premium'),
    (2, 'smartphone'), (2, 'samsung'), (2, 'android'), (2, 'premium'),
    (3, 'laptop'), (3, 'apple'), (3, 'macos'),
    (4, 'shoes'), (4, 'sport'), (4, 'nike'),
    (8, 'book'), (8, 'education'), (8, 'technology');
```

### ข้อ 10
INSERT ข้อมูลทดสอบ (seed data) สำหรับ development environment

**เฉลย:**
```sql
-- seed.sql - ข้อมูลทดสอบ
-- ตรวจสอบก่อนว่าอยู่ใน development
SET @env = 'development';  -- เปลี่ยนตาม environment

-- ล้างข้อมูลเก่า (development เท่านั้น)
-- SET FOREIGN_KEY_CHECKS = 0;
-- TRUNCATE TABLE order_items;
-- TRUNCATE TABLE orders;
-- TRUNCATE TABLE products;
-- TRUNCATE TABLE customers;
-- SET FOREIGN_KEY_CHECKS = 1;

-- Insert seed data
INSERT INTO customers (customer_id, first_name, last_name, email, phone, city)
VALUES 
    (1001, 'Test', 'User1', 'test1@test.com', '000-000-0001', 'Bangkok'),
    (1002, 'Test', 'User2', 'test2@test.com', '000-000-0002', 'Chiang Mai'),
    (1003, 'Test', 'User3', 'test3@test.com', '000-000-0003', 'Phuket')
ON DUPLICATE KEY UPDATE
    first_name = VALUES(first_name);

INSERT INTO products (product_id, product_name, price, stock_quantity)
VALUES 
    (9001, 'Test Product A', 100.00, 999),
    (9002, 'Test Product B', 200.00, 999),
    (9003, 'Test Product C', 999.99, 999)
ON DUPLICATE KEY UPDATE
    product_name = VALUES(product_name);

SELECT 'Seed data inserted successfully' AS result;
```

---

## สรุป

ในบทนี้เราเรียนรู้:
1. **INSERT พื้นฐาน**: single row, multi-row, ระบุ columns
2. **INSERT INTO ... SELECT**: copy/transform ข้อมูล
3. **INSERT IGNORE**: ข้ามถ้า duplicate (MySQL)
4. **UPSERT patterns**: ON DUPLICATE KEY UPDATE (MySQL), ON CONFLICT (PostgreSQL), OR REPLACE (SQLite)
5. **RETURNING**: ดึงข้อมูลที่ insert กลับมา (PostgreSQL)
6. **Bulk Insert**: performance tips, transaction wrapping
7. **NULL handling**: explicit, implicit, DEFAULT, COALESCE
8. **Real-world patterns**: order processing, audit log, session management

> **กฎสำคัญ**: ใช้ multi-row INSERT แทนหลาย single-row INSERT, ห่อ batch insert ด้วย transaction, และใช้ ON DUPLICATE KEY UPDATE สำหรับ upsert patterns
