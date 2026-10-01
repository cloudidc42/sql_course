# Part 15: Deleting Data - DELETE

## บทนำ

`DELETE` เป็นคำสั่ง DML ที่ใช้ลบข้อมูลออกจากตาราง เช่นเดียวกับ UPDATE การลบข้อมูลผิดพลาดอาจเป็นหายนะได้ บทนี้จะสอน:
- วิธีลบข้อมูลอย่างปลอดภัย
- ความแตกต่างระหว่าง DELETE และ TRUNCATE
- Soft delete pattern
- Cascading deletes

---

## 1. DELETE Syntax พื้นฐาน

```sql
DELETE FROM table_name
WHERE condition;
```

**⚠️ คำเตือน**: ถ้าลืม WHERE clause จะลบข้อมูลทั้งตาราง!

### ตัวอย่าง 1: DELETE พื้นฐาน

```sql
-- ลบ employee คนเดียว
DELETE FROM employees
WHERE employee_id = 10;

-- ลบหลาย rows ด้วย IN
DELETE FROM employees
WHERE employee_id IN (8, 9, 10);

-- ลบด้วยเงื่อนไข
DELETE FROM products
WHERE stock_quantity = 0 AND is_available = FALSE;

-- ลบ orders เก่า
DELETE FROM orders
WHERE order_date < '2022-01-01'
  AND status = 'cancelled';

-- ดูจำนวนที่ถูกลบ
SELECT ROW_COUNT() AS rows_deleted;
```

### ตัวอย่าง 2: DELETE พร้อม ORDER BY และ LIMIT (MySQL)

```sql
-- ลบ records เก่าที่สุด 1000 rows (batch delete)
DELETE FROM audit_log
ORDER BY created_at ASC
LIMIT 1000;

-- ลบ notifications ที่อ่านแล้ว เก่ากว่า 30 วัน
DELETE FROM notifications
WHERE is_read = TRUE
  AND created_at < DATE_SUB(NOW(), INTERVAL 30 DAY)
ORDER BY created_at ASC
LIMIT 500;
```

---

## 2. DELETE กับ Subqueries

```sql
-- ตัวอย่าง 3: DELETE ด้วย subquery ใน WHERE
-- ลบ customers ที่ไม่เคยสั่งซื้อ
DELETE FROM customers
WHERE customer_id NOT IN (
    SELECT DISTINCT customer_id FROM orders
);

-- ⚠️ MySQL ไม่อนุญาต delete จากตารางเดียวกับ subquery โดยตรง
-- ต้องใช้ derived table:
DELETE FROM customers
WHERE customer_id NOT IN (
    SELECT customer_id FROM (
        SELECT DISTINCT customer_id FROM orders
    ) AS active_customers
);

-- ตัวอย่าง 4: DELETE ด้วย EXISTS
DELETE FROM sessions
WHERE NOT EXISTS (
    SELECT 1 FROM users u
    WHERE u.user_id = sessions.user_id
      AND u.is_active = TRUE
);

-- ตัวอย่าง 5: DELETE rows ซ้ำ (duplicate removal)
-- เก็บ ID ต่ำสุดไว้ ลบที่เหลือ
DELETE FROM email_subscriptions
WHERE id NOT IN (
    SELECT min_id FROM (
        SELECT MIN(id) AS min_id
        FROM email_subscriptions
        GROUP BY email, category
    ) AS keep_rows
);
```

---

## 3. DELETE กับ JOIN

```sql
-- ตัวอย่าง 6: MySQL DELETE...JOIN
-- ลบ order items ของ cancelled orders
DELETE oi
FROM order_items oi
INNER JOIN orders o ON oi.order_id = o.order_id
WHERE o.status = 'cancelled'
  AND o.order_date < DATE_SUB(NOW(), INTERVAL 1 YEAR);

-- ตัวอย่าง 7: DELETE หลายตารางพร้อมกัน (MySQL)
DELETE o, oi
FROM orders o
INNER JOIN order_items oi ON o.order_id = oi.order_id
WHERE o.status = 'cancelled'
  AND o.customer_id = 5;

-- ตัวอย่าง 8: PostgreSQL DELETE...USING
/*
DELETE FROM order_items oi
USING orders o
WHERE oi.order_id = o.order_id
  AND o.status = 'cancelled';
*/

-- ตัวอย่าง 9: ลบ users ที่ไม่ active มากกว่า 1 ปี
DELETE u
FROM users u
LEFT JOIN user_sessions s ON u.user_id = s.user_id 
    AND s.last_active > DATE_SUB(NOW(), INTERVAL 1 YEAR)
WHERE s.session_id IS NULL  -- ไม่มี recent session
  AND u.created_at < DATE_SUB(NOW(), INTERVAL 1 YEAR);
```

---

## 4. DELETE WITH RETURNING (PostgreSQL)

```sql
-- ตัวอย่าง 10: PostgreSQL RETURNING
/*
-- ลบและดูข้อมูลที่ถูกลบ
DELETE FROM sessions
WHERE expires_at < NOW()
RETURNING session_id, user_id, created_at;

-- ใช้ใน CTE เพื่อ archive ก่อนลบ
WITH deleted_orders AS (
    DELETE FROM orders
    WHERE status = 'cancelled'
      AND created_at < NOW() - INTERVAL '2 years'
    RETURNING *
)
INSERT INTO orders_archive
SELECT * FROM deleted_orders;
*/
```

---

## 5. TRUNCATE vs DELETE

```sql
-- ตัวอย่าง 11: TRUNCATE TABLE
-- ลบข้อมูลทั้งหมดอย่างรวดเร็ว
TRUNCATE TABLE audit_log;

-- TRUNCATE รีเซ็ต AUTO_INCREMENT ด้วย
TRUNCATE TABLE temp_data;
INSERT INTO temp_data (name) VALUES ('test');
SELECT LAST_INSERT_ID();  -- 1 (เริ่มใหม่)

-- ตัวอย่าง 12: เปรียบเทียบ DELETE vs TRUNCATE
/*
DELETE FROM large_table;       -- ช้า, ทำทีละ row, สามารถ ROLLBACK ได้
TRUNCATE TABLE large_table;    -- เร็วมาก, ลบทั้งตารางครั้งเดียว, ไม่สามารถ ROLLBACK (บางกรณี)

ความแตกต่างหลัก:
┌─────────────────┬─────────────────────┬──────────────────────┐
│  คุณสมบัติ     │      DELETE         │      TRUNCATE        │
├─────────────────┼─────────────────────┼──────────────────────┤
│ WHERE clause    │ รองรับ             │ ไม่รองรับ           │
│ Triggers        │ รันได่             │ ไม่รัน              │
│ ROLLBACK        │ ได้ (InnoDB)       │ ไม่ได้ (บางกรณี)   │
│ Auto-increment  │ ไม่รีเซ็ต         │ รีเซ็ต              │
│ ความเร็ว       │ ช้า               │ เร็วมาก             │
│ Foreign Keys    │ ตรวจสอบ           │ ตรวจสอบ (หรือไม่)  │
│ Log             │ บันทึกทุก row     │ บันทึก minimal      │
└─────────────────┴─────────────────────┴──────────────────────┘
*/

-- ตัวอย่าง 13: TRUNCATE พร้อม FOREIGN KEYS
-- ต้อง disable FK ก่อน TRUNCATE ตารางที่มี FK references
SET FOREIGN_KEY_CHECKS = 0;
TRUNCATE TABLE order_items;
TRUNCATE TABLE orders;
SET FOREIGN_KEY_CHECKS = 1;

-- หรือ TRUNCATE ... CASCADE (PostgreSQL)
-- TRUNCATE orders, order_items CASCADE;
```

---

## 6. Soft Delete Pattern

```sql
-- ตัวอย่าง 14: Soft Delete - ไม่ลบจริง แต่ mark ว่าถูกลบ

-- เพิ่ม column สำหรับ soft delete
ALTER TABLE customers
ADD COLUMN deleted_at TIMESTAMP NULL DEFAULT NULL,
ADD COLUMN deleted_by INT UNSIGNED NULL;

ALTER TABLE products
ADD COLUMN deleted_at TIMESTAMP NULL DEFAULT NULL;

-- Soft delete customer
UPDATE customers
SET 
    deleted_at = NOW(),
    deleted_by = @current_user_id,
    email = CONCAT('deleted_', customer_id, '_', email)  -- ป้องกัน unique conflict
WHERE customer_id = 5;

-- Soft delete product
UPDATE products
SET 
    deleted_at = NOW(),
    is_available = FALSE
WHERE product_id = 8;

-- ตัวอย่าง 15: Query ที่รวม soft delete
-- ดูเฉพาะข้อมูลที่ไม่ถูกลบ
SELECT * FROM customers
WHERE deleted_at IS NULL;

-- ดูข้อมูลที่ถูกลบด้วย
SELECT * FROM customers;

-- VIEW สำหรับ active records
CREATE VIEW active_customers AS
SELECT * FROM customers
WHERE deleted_at IS NULL;

CREATE VIEW active_products AS
SELECT * FROM products
WHERE deleted_at IS NULL;

-- ใช้ VIEW แทน table
SELECT * FROM active_customers WHERE city = 'กรุงเทพ';

-- ตัวอย่าง 16: Restore soft-deleted record
UPDATE customers
SET 
    deleted_at = NULL,
    deleted_by = NULL,
    email = REGEXP_REPLACE(email, '^deleted_[0-9]+_', '')
WHERE customer_id = 5;

-- ตัวอย่าง 17: Hard delete หลังจาก soft delete นาน
-- ลบจริงหลังจาก soft delete มากกว่า 90 วัน
DELETE FROM customers
WHERE deleted_at IS NOT NULL
  AND deleted_at < DATE_SUB(NOW(), INTERVAL 90 DAY);
```

---

## 7. Cascading Deletes

```sql
-- ตัวอย่าง 18: ON DELETE CASCADE
-- ลบ parent -> ลบ children อัตโนมัติ
CREATE TABLE parent_orders (
    order_id    INT PRIMARY KEY AUTO_INCREMENT,
    customer_id INT NOT NULL,
    total       DECIMAL(10,2)
);

CREATE TABLE parent_order_items (
    item_id     INT PRIMARY KEY AUTO_INCREMENT,
    order_id    INT NOT NULL,
    product_id  INT NOT NULL,
    quantity    INT,
    FOREIGN KEY (order_id) 
        REFERENCES parent_orders(order_id)
        ON DELETE CASCADE  -- ลบ order -> ลบ items อัตโนมัติ
);

-- ลบ order 1 -> items ทั้งหมดของ order 1 ถูกลบด้วย
DELETE FROM parent_orders WHERE order_id = 1;

-- ตัวอย่าง 19: ON DELETE SET NULL
CREATE TABLE employees_cascade (
    employee_id INT PRIMARY KEY,
    name        VARCHAR(100),
    manager_id  INT,
    FOREIGN KEY (manager_id) 
        REFERENCES employees_cascade(employee_id)
        ON DELETE SET NULL  -- ลบ manager -> subordinates manager_id = NULL
);

-- ตัวอย่าง 20: ตรวจสอบ cascade dependencies ก่อนลบ
-- ดูว่า customer มี orders กี่อัน ก่อนลบ
SELECT 
    c.customer_id,
    c.first_name,
    COUNT(o.order_id) AS order_count,
    SUM(o.total_amount) AS total_spent
FROM customers c
LEFT JOIN orders o ON c.customer_id = o.customer_id
WHERE c.customer_id = 5
GROUP BY c.customer_id, c.first_name;

-- ดูว่ามี FK references อะไรบ้าง (MySQL)
SELECT 
    TABLE_NAME,
    COLUMN_NAME,
    CONSTRAINT_NAME,
    REFERENCED_TABLE_NAME,
    REFERENCED_COLUMN_NAME
FROM information_schema.KEY_COLUMN_USAGE
WHERE REFERENCED_TABLE_SCHEMA = DATABASE()
  AND REFERENCED_TABLE_NAME = 'customers';
```

---

## 8. Safe Deletion Practices

```sql
-- ตัวอย่าง 21: Always SELECT before DELETE
-- ก่อนลบ ดูก่อนว่ามีอะไรบ้าง
SELECT COUNT(*), MIN(created_at), MAX(created_at)
FROM audit_log
WHERE created_at < '2022-01-01';
-- ดูตัวเลขว่าสมเหตุสมผลไหม

-- ค่อย DELETE
DELETE FROM audit_log
WHERE created_at < '2022-01-01';

-- ตัวอย่าง 22: Transaction + Savepoint
START TRANSACTION;

-- ลบ order items ก่อน
DELETE FROM order_items WHERE order_id = 1001;
SAVEPOINT after_items;

-- ลบ order
DELETE FROM orders WHERE order_id = 1001;
SAVEPOINT after_order;

-- ตรวจสอบ
SELECT 'Items deleted:', ROW_COUNT();

-- ถ้าผิดพลาด rollback to savepoint
-- ROLLBACK TO SAVEPOINT after_items;

-- ถ้า OK
COMMIT;

-- ตัวอย่าง 23: Archive ก่อนลบ
-- สร้างตาราง archive
CREATE TABLE IF NOT EXISTS orders_archive LIKE orders;
CREATE TABLE IF NOT EXISTS order_items_archive LIKE order_items;

-- Archive ก่อน
INSERT INTO order_items_archive
SELECT * FROM order_items
WHERE order_id IN (
    SELECT order_id FROM orders
    WHERE order_date < '2022-01-01'
);

INSERT INTO orders_archive
SELECT * FROM orders
WHERE order_date < '2022-01-01';

-- ลบจากตารางหลัก
DELETE FROM order_items
WHERE order_id IN (
    SELECT order_id FROM (
        SELECT order_id FROM orders WHERE order_date < '2022-01-01'
    ) AS old_orders
);

DELETE FROM orders
WHERE order_date < '2022-01-01';

-- ตัวอย่าง 24: Batch Delete สำหรับข้อมูลจำนวนมาก
DELIMITER //
CREATE PROCEDURE batch_delete_old_logs(IN p_days INT)
BEGIN
    DECLARE v_deleted INT DEFAULT 1;
    DECLARE v_total INT DEFAULT 0;
    DECLARE v_cutoff DATE;
    
    SET v_cutoff = DATE_SUB(CURRENT_DATE, INTERVAL p_days DAY);
    
    WHILE v_deleted > 0 DO
        DELETE FROM application_logs
        WHERE log_date < v_cutoff
        LIMIT 1000;
        
        SET v_deleted = ROW_COUNT();
        SET v_total = v_total + v_deleted;
        
        -- หยุดพักเล็กน้อย
        DO SLEEP(0.1);
    END WHILE;
    
    SELECT v_total AS total_deleted, 
           v_cutoff AS deleted_before;
END //
DELIMITER ;

-- ลบ logs เก่ากว่า 365 วัน
CALL batch_delete_old_logs(365);
```

---

## 9. Delete Patterns สำหรับ Real-World Scenarios

### 9.1 User Account Management

```sql
-- ตัวอย่าง 25: ลบ account พร้อม data (GDPR compliance)
DELIMITER //
CREATE PROCEDURE delete_user_account(IN p_user_id INT)
BEGIN
    START TRANSACTION;
    
    -- ลบ sensitive data ก่อน
    DELETE FROM user_payment_methods WHERE user_id = p_user_id;
    DELETE FROM user_sessions WHERE user_id = p_user_id;
    DELETE FROM user_addresses WHERE user_id = p_user_id;
    
    -- Anonymize orders (ไม่ลบ เพราะ financial records ต้องเก็บ)
    UPDATE orders SET customer_id = NULL
    WHERE customer_id = p_user_id;
    
    -- Soft delete user
    UPDATE users
    SET 
        email = CONCAT('deleted_', p_user_id, '@removed.invalid'),
        username = CONCAT('deleted_', p_user_id),
        first_name = 'Deleted',
        last_name = 'User',
        phone = NULL,
        deleted_at = NOW()
    WHERE user_id = p_user_id;
    
    COMMIT;
    
    SELECT 'Account deleted successfully' AS result;
END //
DELIMITER ;

-- ตัวอย่าง 26: Cleanup expired tokens
DELETE FROM password_reset_tokens
WHERE expires_at < NOW();

DELETE FROM email_verification_tokens
WHERE created_at < DATE_SUB(NOW(), INTERVAL 24 HOUR)
  AND used_at IS NULL;
```

### 9.2 Session Management

```sql
-- ตัวอย่าง 27: Cleanup old sessions
DELETE FROM user_sessions
WHERE expires_at < NOW();

-- ลบ sessions ของ inactive users
DELETE s
FROM user_sessions s
JOIN users u ON s.user_id = u.user_id
WHERE u.is_active = FALSE;

-- ตัวอย่าง 28: Logout (ลบ session เฉพาะ)
DELETE FROM user_sessions
WHERE 
    session_token = 'abc123...'
    AND user_id = 1;

-- Logout ทุก devices
DELETE FROM user_sessions
WHERE user_id = 1;
```

### 9.3 Content Management

```sql
-- ตัวอย่าง 29: ลบ orphaned records
-- ลบ images ที่ไม่มี product อ้างอิงแล้ว
DELETE FROM product_images
WHERE product_id NOT IN (
    SELECT product_id FROM products
);

-- ลบ comments ของ deleted posts
DELETE c
FROM comments c
LEFT JOIN posts p ON c.post_id = p.post_id
WHERE p.post_id IS NULL;

-- ตัวอย่าง 30: ลบ spam/abuse content
DELETE FROM comments
WHERE 
    spam_score > 0.9
    AND is_approved = FALSE
    AND created_at < DATE_SUB(NOW(), INTERVAL 7 DAY);
```

---

## 10. Performance Considerations

```sql
-- ตัวอย่าง 31: DELETE Performance
-- ช้า: ไม่มี index บน WHERE condition
DELETE FROM logs WHERE status = 'processed';  -- ถ้า status ไม่มี index

-- เร็ว: มี index
CREATE INDEX idx_logs_status ON logs(status);
DELETE FROM logs WHERE status = 'processed';

-- ตัวอย่าง 32: Partitioned table delete
-- ถ้าตาราง partition ตามวันที่ การลบทั้ง partition เร็วกว่า DELETE rows
-- MySQL:
ALTER TABLE sales_log DROP PARTITION p2021;  -- เร็วมาก!

-- เปรียบเทียบกับ:
DELETE FROM sales_log WHERE YEAR(sale_date) = 2021;  -- ช้ากว่ามาก

-- ตัวอย่าง 33: DELETE พร้อม explain
EXPLAIN DELETE FROM audit_log
WHERE created_at < '2022-01-01';
-- ดูว่าใช้ index ไหม
```

---

## แบบฝึกหัด 10 ข้อ

### ข้อ 1
ลบ products ที่ไม่มีการสั่งซื้อเลยและ stock_quantity = 0

**เฉลย:**
```sql
-- ดูก่อน
SELECT p.product_id, p.product_name, p.stock_quantity
FROM products p
LEFT JOIN order_items oi ON p.product_id = oi.product_id
WHERE oi.product_id IS NULL
  AND p.stock_quantity = 0;

-- ลบ
DELETE FROM products
WHERE product_id NOT IN (SELECT DISTINCT product_id FROM order_items)
  AND stock_quantity = 0;
```

### ข้อ 2
Soft delete customers ที่ไม่มีการสั่งซื้อมากกว่า 2 ปี

**เฉลย:**
```sql
UPDATE customers
SET deleted_at = NOW()
WHERE customer_id NOT IN (
    SELECT DISTINCT customer_id
    FROM orders
    WHERE order_date > DATE_SUB(NOW(), INTERVAL 2 YEAR)
)
AND deleted_at IS NULL;
```

### ข้อ 3
ลบ duplicate rows ใน email_subscriptions เก็บแค่ row ล่าสุด

**เฉลย:**
```sql
DELETE FROM email_subscriptions
WHERE id NOT IN (
    SELECT max_id FROM (
        SELECT MAX(id) AS max_id
        FROM email_subscriptions
        GROUP BY email
    ) AS latest
);
```

### ข้อ 4
Archive orders ที่ completed มากกว่า 3 ปี แล้วลบออกจากตารางหลัก

**เฉลย:**
```sql
-- Archive order_items ก่อน
INSERT INTO order_items_archive
SELECT oi.* FROM order_items oi
JOIN orders o ON oi.order_id = o.order_id
WHERE o.status = 'completed'
  AND o.order_date < DATE_SUB(NOW(), INTERVAL 3 YEAR);

-- Archive orders
INSERT INTO orders_archive
SELECT * FROM orders
WHERE status = 'completed'
  AND order_date < DATE_SUB(NOW(), INTERVAL 3 YEAR);

-- ลบ items
DELETE oi
FROM order_items oi
JOIN orders o ON oi.order_id = o.order_id
WHERE o.status = 'completed'
  AND o.order_date < DATE_SUB(NOW(), INTERVAL 3 YEAR);

-- ลบ orders
DELETE FROM orders
WHERE status = 'completed'
  AND order_date < DATE_SUB(NOW(), INTERVAL 3 YEAR);

SELECT ROW_COUNT() AS orders_deleted;
```

### ข้อ 5
สร้าง GDPR data deletion procedure สำหรับ customers

**เฉลย:**
```sql
DELIMITER //
CREATE PROCEDURE gdpr_delete_customer(IN p_customer_id INT)
BEGIN
    DECLARE v_exists INT;
    
    SELECT COUNT(*) INTO v_exists
    FROM customers WHERE customer_id = p_customer_id;
    
    IF v_exists = 0 THEN
        SIGNAL SQLSTATE '45000' SET MESSAGE_TEXT = 'Customer not found';
    END IF;
    
    START TRANSACTION;
    
    -- ลบ sensitive data
    DELETE FROM customer_payment_info WHERE customer_id = p_customer_id;
    DELETE FROM customer_sessions WHERE customer_id = p_customer_id;
    
    -- Anonymize orders
    UPDATE orders 
    SET customer_id = NULL, notes = '[GDPR DELETED]'
    WHERE customer_id = p_customer_id;
    
    -- Anonymize customer record
    UPDATE customers SET
        first_name = 'Anonymous',
        last_name = 'User',
        email = CONCAT('gdpr_deleted_', p_customer_id, '@anonymous.invalid'),
        phone = NULL,
        address = NULL,
        birth_date = NULL,
        deleted_at = NOW()
    WHERE customer_id = p_customer_id;
    
    COMMIT;
    
    SELECT 'GDPR deletion completed' AS status, p_customer_id AS customer_id;
END //
DELIMITER ;

CALL gdpr_delete_customer(5);
```

### ข้อ 6
ลบ orphaned order_items (ที่ order ไม่มีอยู่แล้ว)

**เฉลย:**
```sql
-- ตรวจสอบ
SELECT COUNT(*) AS orphaned_items
FROM order_items oi
LEFT JOIN orders o ON oi.order_id = o.order_id
WHERE o.order_id IS NULL;

-- ลบ
DELETE oi
FROM order_items oi
LEFT JOIN orders o ON oi.order_id = o.order_id
WHERE o.order_id IS NULL;
```

### ข้อ 7
เปรียบเทียบ DELETE vs TRUNCATE ในทางปฏิบัติพร้อมเวลาดำเนินการ

**เฉลย:**
```sql
-- สร้าง test data
CREATE TABLE test_delete (id INT PRIMARY KEY AUTO_INCREMENT, data VARCHAR(100));
INSERT INTO test_delete (data)
SELECT REPEAT('x', 100) FROM information_schema.columns LIMIT 10000;

-- วัดเวลา DELETE
SET @start = NOW(6);
DELETE FROM test_delete;
SELECT TIMESTAMPDIFF(MICROSECOND, @start, NOW(6)) / 1000 AS delete_ms;

-- Insert data กลับ
INSERT INTO test_delete (data)
SELECT REPEAT('x', 100) FROM information_schema.columns LIMIT 10000;

-- วัดเวลา TRUNCATE
SET @start = NOW(6);
TRUNCATE TABLE test_delete;
SELECT TIMESTAMPDIFF(MICROSECOND, @start, NOW(6)) / 1000 AS truncate_ms;

-- TRUNCATE เร็วกว่ามาก (อาจ 10-100x)
-- ตรวจสอบ AUTO_INCREMENT reset
INSERT INTO test_delete (data) VALUES ('after truncate');
SELECT LAST_INSERT_ID();  -- 1 (reset แล้ว)

DROP TABLE test_delete;
```

### ข้อ 8
ลบ sessions หมดอายุโดยใช้ batch delete

**เฉลย:**
```sql
DELIMITER //
CREATE PROCEDURE cleanup_expired_sessions()
BEGIN
    DECLARE v_deleted INT DEFAULT 1;
    DECLARE v_total INT DEFAULT 0;
    
    WHILE v_deleted > 0 DO
        DELETE FROM user_sessions
        WHERE expires_at < NOW()
        LIMIT 500;
        
        SET v_deleted = ROW_COUNT();
        SET v_total = v_total + v_deleted;
        
        DO SLEEP(0.05);
    END WHILE;
    
    INSERT INTO cleanup_log (action, rows_affected, executed_at)
    VALUES ('cleanup_sessions', v_total, NOW());
    
    SELECT v_total AS sessions_deleted;
END //
DELIMITER ;

CALL cleanup_expired_sessions();
```

### ข้อ 9
สร้าง Soft Delete pattern ที่สมบูรณ์พร้อม VIEW และ TRIGGER

**เฉลย:**
```sql
-- เพิ่ม soft delete columns
ALTER TABLE products
ADD COLUMN IF NOT EXISTS deleted_at TIMESTAMP NULL,
ADD COLUMN IF NOT EXISTS deleted_by INT UNSIGNED NULL;

-- View สำหรับ active products
CREATE OR REPLACE VIEW v_active_products AS
SELECT * FROM products
WHERE deleted_at IS NULL;

-- Trigger ป้องกัน hard delete ผ่าน table โดยตรง
DELIMITER //
CREATE TRIGGER prevent_hard_delete_products
BEFORE DELETE ON products
FOR EACH ROW
BEGIN
    SIGNAL SQLSTATE '45000'
    SET MESSAGE_TEXT = 'Direct DELETE not allowed. Use soft delete procedure instead.';
END //
DELIMITER ;

-- Soft delete procedure
DELIMITER //
CREATE PROCEDURE soft_delete_product(IN p_product_id INT, IN p_user_id INT)
BEGIN
    UPDATE products
    SET 
        deleted_at = NOW(),
        deleted_by = p_user_id,
        is_available = FALSE
    WHERE product_id = p_product_id
      AND deleted_at IS NULL;
    
    IF ROW_COUNT() = 0 THEN
        SIGNAL SQLSTATE '45000' SET MESSAGE_TEXT = 'Product not found or already deleted';
    END IF;
END //
DELIMITER ;

CALL soft_delete_product(8, 1);
```

### ข้อ 10
สร้าง scheduled cleanup job สำหรับลบข้อมูลเก่าอัตโนมัติ

**เฉลย:**
```sql
-- ตาราง log สำหรับ cleanup
CREATE TABLE IF NOT EXISTS cleanup_log (
    log_id      INT AUTO_INCREMENT PRIMARY KEY,
    action      VARCHAR(100),
    rows_affected INT,
    executed_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Stored Procedure cleanup ทั้งหมด
DELIMITER //
CREATE PROCEDURE daily_cleanup()
BEGIN
    DECLARE v_deleted INT;
    
    -- 1. ลบ expired sessions
    DELETE FROM user_sessions WHERE expires_at < NOW();
    SET v_deleted = ROW_COUNT();
    INSERT INTO cleanup_log (action, rows_affected) VALUES ('expired_sessions', v_deleted);
    
    -- 2. ลบ expired reset tokens  
    DELETE FROM password_reset_tokens WHERE expires_at < NOW();
    SET v_deleted = ROW_COUNT();
    INSERT INTO cleanup_log (action, rows_affected) VALUES ('expired_tokens', v_deleted);
    
    -- 3. ลบ audit logs เก่ากว่า 1 ปี (batch)
    DELETE FROM audit_log
    WHERE created_at < DATE_SUB(NOW(), INTERVAL 1 YEAR)
    LIMIT 5000;
    SET v_deleted = ROW_COUNT();
    INSERT INTO cleanup_log (action, rows_affected) VALUES ('old_audit_logs', v_deleted);
    
    -- 4. Hard delete soft-deleted records เก่ากว่า 90 วัน
    DELETE FROM customers
    WHERE deleted_at IS NOT NULL
      AND deleted_at < DATE_SUB(NOW(), INTERVAL 90 DAY);
    SET v_deleted = ROW_COUNT();
    INSERT INTO cleanup_log (action, rows_affected) VALUES ('hard_delete_customers', v_deleted);
    
    SELECT 'Daily cleanup completed' AS status;
END //
DELIMITER ;

-- Schedule ด้วย MySQL Event
CREATE EVENT daily_cleanup_event
ON SCHEDULE EVERY 1 DAY
STARTS CURRENT_DATE + INTERVAL 2 HOUR  -- รันตี 2 ทุกวัน
DO
    CALL daily_cleanup();
```

---

## สรุป

ในบทนี้เราเรียนรู้:
1. **DELETE พื้นฐาน**: syntax, WHERE clause, หลาย rows
2. **DELETE กับ Subqueries**: IN, NOT IN, EXISTS, correlated
3. **DELETE กับ JOIN**: multi-table delete (MySQL/PostgreSQL)
4. **TRUNCATE vs DELETE**: ความแตกต่างและเมื่อใดใช้อะไร
5. **Soft Delete Pattern**: ไม่ลบจริง แต่ mark deleted_at
6. **Cascading Deletes**: ON DELETE CASCADE/SET NULL
7. **Safe practices**: SELECT ก่อน DELETE, Transaction, Archive
8. **Batch Delete**: ลบทีละ batch ลด lock pressure
9. **Real-world patterns**: GDPR, session cleanup, orphaned records

> **กฎทองของ DELETE**: 
> 1. ทำ SELECT COUNT ก่อน DELETE เสมอ
> 2. ใช้ Transaction สำหรับการลบที่สำคัญ  
> 3. Archive ก่อนลบสำหรับข้อมูล business-critical
> 4. พิจารณา Soft Delete สำหรับข้อมูลสำคัญ
> 5. ใช้ Batch Delete สำหรับข้อมูลจำนวนมาก
