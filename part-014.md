# Part 14: Updating Data - UPDATE

## บทนำ

`UPDATE` เป็นคำสั่ง DML ที่ใช้แก้ไขข้อมูลที่มีอยู่แล้วในตาราง การทำ UPDATE อย่างถูกต้องและปลอดภัยเป็นสิ่งสำคัญมาก เพราะ UPDATE ที่ผิดพลาดอาจทำลายข้อมูลได้โดยไม่สามารถกู้คืน

---

## 1. UPDATE Syntax พื้นฐาน

```sql
UPDATE table_name
SET column1 = value1,
    column2 = value2,
    ...
WHERE condition;
```

**⚠️ คำเตือน**: ถ้าลืม WHERE clause จะ update ทุก row!

### ตัวอย่าง 1: UPDATE พื้นฐาน

```sql
-- อัปเดตข้อมูล employee
UPDATE employees
SET salary = 60000
WHERE employee_id = 1;

-- อัปเดตหลาย columns พร้อมกัน
UPDATE employees
SET 
    salary = 65000,
    job_title = 'Senior Engineer',
    department_id = 1
WHERE employee_id = 1;

-- ตรวจสอบผล
SELECT employee_id, first_name, salary, job_title 
FROM employees 
WHERE employee_id = 1;
```

### ตัวอย่าง 2: UPDATE ด้วยเงื่อนไขต่างๆ

```sql
-- อัปเดตโดยใช้ช่วงค่า
UPDATE employees
SET salary = salary * 1.10  -- ขึ้นเงินเดือน 10%
WHERE hire_date < '2020-01-01';

-- อัปเดตด้วย NULL
UPDATE employees
SET phone = NULL
WHERE employee_id = 5;

-- อัปเดต boolean
UPDATE products
SET is_available = FALSE
WHERE stock_quantity = 0;

-- อัปเดตด้วย expression
UPDATE products
SET price = price * 0.90  -- ลดราคา 10%
WHERE category = 'Electronics';

-- อัปเดต timestamp
UPDATE orders
SET 
    status = 'shipped',
    shipped_at = NOW()
WHERE order_id = 1001
  AND status = 'processing';
```

---

## 2. UPDATE หลาย Rows

```sql
-- ตัวอย่าง 3: UPDATE ทุก rows ที่ตรงเงื่อนไข
-- ปรับ salary ของทุกคนในแผนก Engineering
UPDATE employees
SET salary = salary * 1.15
WHERE department_id = (
    SELECT department_id 
    FROM departments 
    WHERE department_name = 'Engineering'
);

-- ตัวอย่าง 4: UPDATE พร้อม CASE expression
UPDATE employees
SET salary = CASE 
    WHEN salary < 30000 THEN salary * 1.20  -- เพิ่ม 20%
    WHEN salary < 50000 THEN salary * 1.10  -- เพิ่ม 10%
    WHEN salary < 70000 THEN salary * 1.05  -- เพิ่ม 5%
    ELSE salary * 1.03                       -- เพิ่ม 3%
END;

-- ตัวอย่าง 5: UPDATE ด้วย CASE แบบ conditional columns
UPDATE products
SET 
    is_available = CASE 
        WHEN stock_quantity > 0 THEN TRUE 
        ELSE FALSE 
    END,
    updated_at = NOW();
```

---

## 3. UPDATE กับ Subqueries

```sql
-- ตัวอย่าง 6: UPDATE ด้วย subquery ใน WHERE
UPDATE orders
SET status = 'vip_priority'
WHERE customer_id IN (
    SELECT customer_id 
    FROM customers 
    WHERE city = 'กรุงเทพ'
);

-- ตัวอย่าง 7: UPDATE ด้วย correlated subquery
-- อัปเดต total_amount ในตาราง orders จาก order_items
UPDATE orders o
SET total_amount = (
    SELECT SUM(quantity * unit_price - discount)
    FROM order_items oi
    WHERE oi.order_id = o.order_id
)
WHERE status != 'cancelled';

-- ตัวอย่าง 8: UPDATE ด้วย scalar subquery
UPDATE employees
SET salary = (
    SELECT AVG(salary) * 1.1  -- อัปเดตเป็น 110% ของค่าเฉลี่ย
    FROM employees             -- ⚠️ MySQL: ไม่สามารถ query ตัวเองโดยตรง
) 
WHERE employee_id = 99;

-- MySQL ต้องใช้ derived table:
UPDATE employees
SET salary = (
    SELECT avg_sal * 1.1
    FROM (SELECT AVG(salary) AS avg_sal FROM employees) AS temp
)
WHERE employee_id = 99;

-- ตัวอย่าง 9: UPDATE โดยใช้ subquery ใน SET
UPDATE employees e
SET department_id = (
    SELECT department_id 
    FROM departments 
    WHERE department_name = 'Sales'
)
WHERE e.job_title LIKE '%Sales%';
```

---

## 4. UPDATE with JOIN (Multi-table UPDATE)

```sql
-- ตัวอย่าง 10: MySQL UPDATE...JOIN
-- อัปเดต employees โดยดึงข้อมูลจาก departments
UPDATE employees e
INNER JOIN departments d ON e.department_id = d.department_id
SET e.salary = e.salary * 1.10
WHERE d.department_name = 'Engineering';

-- ตัวอย่าง 11: UPDATE หลายตารางพร้อมกัน (MySQL)
UPDATE orders o
INNER JOIN customers c ON o.customer_id = c.customer_id
SET 
    o.status = 'vip_order',
    c.last_activity = NOW()  -- อัปเดต customers ด้วย
WHERE o.total_amount > 10000;

-- ตัวอย่าง 12: UPDATE ด้วย LEFT JOIN
-- อัปเดต orders ที่ยังไม่มี items (empty orders)
UPDATE orders o
LEFT JOIN order_items oi ON o.order_id = oi.order_id
SET o.status = 'empty'
WHERE oi.item_id IS NULL;

-- ตัวอย่าง 13: PostgreSQL UPDATE...FROM
/*
UPDATE employees e
SET salary = e.salary * 1.10
FROM departments d
WHERE e.department_id = d.department_id
  AND d.department_name = 'Engineering';

-- อัปเดตหลายตาราง PostgreSQL
UPDATE orders o
SET status = 'vip_priority'
FROM customers c
WHERE o.customer_id = c.customer_id
  AND c.city = 'Bangkok';
*/

-- ตัวอย่าง 14: SQL Server UPDATE...FROM
/*
UPDATE e
SET e.salary = e.salary * 1.10
FROM employees e
INNER JOIN departments d ON e.department_id = d.department_id
WHERE d.department_name = 'Engineering';
*/
```

---

## 5. UPDATE พร้อม RETURNING (PostgreSQL)

```sql
-- ตัวอย่าง 15: PostgreSQL RETURNING
/*
-- อัปเดตและดูค่าใหม่
UPDATE employees
SET salary = salary * 1.10
WHERE department_id = 1
RETURNING employee_id, first_name, salary AS new_salary;

-- ใช้ใน CTE
WITH updated_orders AS (
    UPDATE orders
    SET status = 'processing'
    WHERE status = 'confirmed'
      AND order_date < NOW() - INTERVAL '1 hour'
    RETURNING order_id, customer_id
)
SELECT uo.order_id, c.email
FROM updated_orders uo
JOIN customers c ON uo.customer_id = c.customer_id;
*/
```

---

## 6. Safe UPDATE Practices

```sql
-- ตัวอย่าง 16: ดู affected rows ก่อน UPDATE
-- Step 1: SELECT ก่อนเพื่อดูว่า WHERE condition ตรงกี่ rows
SELECT COUNT(*), employee_id, first_name, salary
FROM employees
WHERE department_id = 1 AND salary < 40000;

-- Step 2: ถ้า OK แล้วค่อย UPDATE
UPDATE employees
SET salary = 40000
WHERE department_id = 1 AND salary < 40000;

-- Step 3: ดูผลลัพธ์
SELECT ROW_COUNT() AS rows_updated;

-- ตัวอย่าง 17: ใช้ Transaction + Rollback pattern
START TRANSACTION;

-- ทำ UPDATE
UPDATE employees
SET salary = salary * 2  -- เพิ่ม 2 เท่า
WHERE department_id = 1;

-- ตรวจสอบผล
SELECT employee_id, first_name, salary 
FROM employees 
WHERE department_id = 1;

-- ถ้า OK ทำ: COMMIT;
-- ถ้าผิด ทำ: ROLLBACK;
ROLLBACK;  -- ยกเลิกการเปลี่ยนแปลง

-- ตัวอย่าง 18: SET SQL_SAFE_UPDATES (MySQL)
-- ป้องกัน UPDATE โดยไม่มี WHERE ที่ใช้ indexed column
SET SQL_SAFE_UPDATES = 1;

-- query นี้จะ error ถ้า department_name ไม่มี index
UPDATE employees SET salary = 0 WHERE department_id = 1;

-- ปิด safe mode สำหรับ bulk updates
SET SQL_SAFE_UPDATES = 0;
UPDATE employees SET is_active = FALSE;  -- อัปเดตทุก row
SET SQL_SAFE_UPDATES = 1;

-- ตัวอย่าง 19: LIMIT ใน UPDATE (MySQL)
-- อัปเดตแค่ 10 rows ครั้งละ (เพื่อไม่ให้ lock table นาน)
UPDATE orders
SET status = 'archived'
WHERE created_at < '2022-01-01'
  AND status = 'completed'
LIMIT 1000;
-- รัน loop นี้จนกว่า ROW_COUNT() = 0
```

---

## 7. UPDATE Patterns ใน Real-World

### 7.1 E-Commerce Order Management

```sql
-- ตัวอย่าง 20: Order status transitions
-- pending -> confirmed
UPDATE orders
SET 
    status = 'confirmed',
    updated_at = NOW()
WHERE 
    order_id = 1003
    AND status = 'pending';  -- ป้องกัน race condition

-- ตรวจว่า update สำเร็จ (ROW_COUNT = 0 = ใครอื่น update ก่อน)
SELECT ROW_COUNT() AS updated;

-- confirmed -> processing
UPDATE orders
SET 
    status = 'processing',
    updated_at = NOW()
WHERE 
    order_id = 1003
    AND status = 'confirmed';

-- processing -> shipped
UPDATE orders
SET 
    status = 'shipped',
    shipped_at = NOW(),
    updated_at = NOW()
WHERE 
    order_id = 1003
    AND status = 'processing';

-- shipped -> delivered
UPDATE orders
SET 
    status = 'delivered',
    delivered_at = NOW(),
    updated_at = NOW()
WHERE 
    order_id = 1003
    AND status = 'shipped';

-- ตัวอย่าง 21: Update stock หลังจ่ายเงิน
UPDATE products p
INNER JOIN order_items oi ON p.product_id = oi.product_id
SET p.stock_quantity = p.stock_quantity - oi.quantity
WHERE oi.order_id = 1001
  AND p.stock_quantity >= oi.quantity;

-- ตรวจสอบว่า stock ไม่ติดลบ
SELECT product_id, product_name, stock_quantity
FROM products
WHERE product_id IN (
    SELECT product_id FROM order_items WHERE order_id = 1001
);
```

### 7.2 HR Salary Management

```sql
-- ตัวอย่าง 22: Annual salary review
-- ปรับเงินเดือนตาม performance rating
UPDATE employees e
JOIN performance_reviews pr ON e.employee_id = pr.employee_id
    AND pr.review_year = 2023
SET e.salary = CASE 
    WHEN pr.rating = 5 THEN e.salary * 1.20
    WHEN pr.rating = 4 THEN e.salary * 1.10
    WHEN pr.rating = 3 THEN e.salary * 1.05
    WHEN pr.rating = 2 THEN e.salary * 1.02
    ELSE e.salary  -- rating 1: ไม่ขึ้น
END,
e.updated_at = NOW()
WHERE e.is_active = TRUE;

-- ตัวอย่าง 23: Batch salary update สำหรับ departments
UPDATE employees
SET salary = salary * 1.08
WHERE department_id IN (
    SELECT department_id FROM departments
    WHERE budget > 3000000
);

-- ตัวอย่าง 24: เปลี่ยนแผนก manager
UPDATE departments
SET manager_id = (
    SELECT employee_id
    FROM employees
    WHERE department_id = 1
    ORDER BY salary DESC
    LIMIT 1
)
WHERE department_id = 1;
```

### 7.3 Inventory Management

```sql
-- ตัวอย่าง 25: Restock ใส่สินค้าเพิ่ม
UPDATE products
SET 
    stock_quantity = stock_quantity + 100,
    updated_at = NOW()
WHERE product_id = 1;

-- ตัวอย่าง 26: Mark products out of stock
UPDATE products
SET is_available = FALSE
WHERE stock_quantity <= 0;

-- ตัวอย่าง 27: Bulk price update จาก supplier
CREATE TEMPORARY TABLE price_list (
    sku     VARCHAR(50),
    new_price DECIMAL(10,2)
);

INSERT INTO price_list VALUES
    ('SKU001', 43900.00),
    ('SKU002', 36900.00),
    ('SKU003', 50900.00);

UPDATE products p
JOIN price_list pl ON p.sku = pl.sku
SET p.price = pl.new_price,
    p.updated_at = NOW();

DROP TEMPORARY TABLE price_list;
```

### 7.4 User Management

```sql
-- ตัวอย่าง 28: Update last login
UPDATE customers
SET last_login = NOW()
WHERE customer_id = 1;

-- ตัวอย่าง 29: Soft delete (ไม่ลบจริง)
UPDATE customers
SET 
    is_active = FALSE,
    deleted_at = NOW(),
    email = CONCAT('deleted_', customer_id, '_', email)  -- ป้องกัน email ชน
WHERE customer_id = 5;

-- ตัวอย่าง 30: Reset password (update hash เท่านั้น)
UPDATE users
SET 
    password_hash = '$2y$10$newhashedpassword...',
    password_reset_token = NULL,
    password_changed_at = NOW()
WHERE 
    password_reset_token = 'valid-token-here'
    AND token_expires_at > NOW();

-- ตัวอย่าง 31: Bulk email unsubscribe
UPDATE email_subscriptions
SET 
    is_subscribed = FALSE,
    unsubscribed_at = NOW()
WHERE email IN (
    SELECT email FROM unsubscribe_requests
    WHERE processed_at IS NULL
);

-- Mark as processed
UPDATE unsubscribe_requests
SET processed_at = NOW()
WHERE processed_at IS NULL;
```

---

## 8. UPDATE Performance

```sql
-- ตัวอย่าง 32: UPDATE performance - ใช้ index
-- ช้า: ไม่ใช้ index
UPDATE employees SET salary = salary * 1.1
WHERE YEAR(hire_date) = 2020;  -- function ทำให้ไม่ใช้ index บน hire_date

-- เร็ว: ใช้ range condition (index เข้าถึงได้)
UPDATE employees SET salary = salary * 1.1
WHERE hire_date BETWEEN '2020-01-01' AND '2020-12-31';

-- ตัวอย่าง 33: UPDATE ใหญ่แบบ batch
-- แทนที่จะ update ทีเดียว (lock table นาน)
-- ให้ update ทีละ batch
DELIMITER //
CREATE PROCEDURE batch_update_prices(IN p_batch_size INT)
BEGIN
    DECLARE done INT DEFAULT FALSE;
    
    WHILE NOT done DO
        UPDATE products
        SET price = price * 0.95
        WHERE updated_at < '2023-01-01'
          AND is_discounted = FALSE
        LIMIT p_batch_size;
        
        IF ROW_COUNT() < p_batch_size THEN
            SET done = TRUE;
        END IF;
        
        -- Sleep เล็กน้อยเพื่อลด lock pressure
        DO SLEEP(0.1);
    END WHILE;
END //
DELIMITER ;

-- ตัวอย่าง 34: อย่าทำ UPDATE ใหญ่ใน production hours
-- แนะนำใช้ event scheduler หรือ cron job ทำในช่วงกลางคืน
CREATE EVENT nightly_price_update
ON SCHEDULE EVERY 1 DAY
STARTS '2024-01-01 02:00:00'
DO
    UPDATE products 
    SET is_available = (stock_quantity > 0)
    WHERE is_available != (stock_quantity > 0);
```

---

## 9. UPDATE และ Locking

```sql
-- ตัวอย่าง 35: Optimistic Locking
-- เพิ่ม version column ในตาราง
ALTER TABLE orders ADD COLUMN version INT DEFAULT 0;

-- อ่านข้อมูลพร้อม version
SELECT order_id, status, total_amount, version
FROM orders
WHERE order_id = 1001;
-- ได้: version = 5

-- UPDATE โดยตรวจสอบ version (ป้องกัน lost update)
UPDATE orders
SET 
    status = 'shipped',
    version = version + 1
WHERE 
    order_id = 1001
    AND version = 5;  -- ต้องตรงกับที่อ่านมา

-- ถ้า version เปลี่ยน (ใครอื่น update ก่อน) -> ROW_COUNT() = 0
SELECT ROW_COUNT() AS success;  -- 0 = conflict, 1 = success

-- ตัวอย่าง 36: SELECT FOR UPDATE (Pessimistic Locking)
START TRANSACTION;

-- Lock row ไว้ก่อน
SELECT * FROM orders 
WHERE order_id = 1001
FOR UPDATE;  -- ล็อก row นี้ไว้

-- ทำ update
UPDATE orders 
SET status = 'processing'
WHERE order_id = 1001;

COMMIT;  -- ปล่อย lock
```

---

## 10. Preventing Accidental Mass Updates

```sql
-- ตัวอย่าง 37: Pattern ป้องกัน accidental full-table update
-- ❌ อันตราย: ลืม WHERE
-- UPDATE employees SET salary = 50000;  -- update ทุก employee!

-- ✅ ปลอดภัยกว่า: ใส่ WHERE เสมอ
UPDATE employees 
SET salary = 50000
WHERE employee_id = 1;  -- ระบุ ID ชัดเจน

-- ✅ ป้องกันด้วย LIMIT
UPDATE employees 
SET salary = 50000
WHERE department_id = 1
LIMIT 100;  -- update สูงสุด 100 rows

-- ตัวอย่าง 38: Double-check pattern
-- ก่อน UPDATE สำคัญ: ทำ SELECT COUNT ก่อน
SELECT COUNT(*) AS will_be_updated
FROM employees
WHERE salary < 30000;
-- ดูตัวเลขก่อนว่าสมเหตุสมผลไหม

-- ถ้าสมเหตุสมผล ค่อย UPDATE
UPDATE employees
SET salary = 30000
WHERE salary < 30000;

-- ตัวอย่าง 39: ใช้ Stored Procedure พร้อม parameter validation
DELIMITER //
CREATE PROCEDURE safe_bulk_salary_update(
    IN p_department_id  INT,
    IN p_percentage     DECIMAL(5,2),
    IN p_dry_run        BOOLEAN
)
BEGIN
    DECLARE v_affected INT;
    
    -- Validate percentage (ป้องกันขึ้นเงินเดือน 100x)
    IF p_percentage < -50 OR p_percentage > 50 THEN
        SIGNAL SQLSTATE '45000' 
        SET MESSAGE_TEXT = 'Invalid percentage: must be between -50 and 50';
    END IF;
    
    -- ดูจะ affect กี่ rows
    SELECT COUNT(*) INTO v_affected
    FROM employees
    WHERE department_id = p_department_id AND is_active = TRUE;
    
    SELECT v_affected AS rows_to_update, 
           CONCAT(p_percentage, '%') AS adjustment;
    
    IF NOT p_dry_run THEN
        UPDATE employees
        SET salary = salary * (1 + p_percentage / 100)
        WHERE department_id = p_department_id 
          AND is_active = TRUE;
        
        SELECT ROW_COUNT() AS rows_updated;
    ELSE
        SELECT 'DRY RUN - no changes made' AS message;
    END IF;
END //
DELIMITER ;

-- Dry run ก่อน
CALL safe_bulk_salary_update(1, 10.0, TRUE);

-- ถ้า OK แล้วค่อย run จริง
CALL safe_bulk_salary_update(1, 10.0, FALSE);
```

---

## แบบฝึกหัด 10 ข้อ

### ข้อ 1
อัปเดตราคา products ในหมวด Electronics ลด 5%

**เฉลย:**
```sql
-- ดูก่อนว่ามีกี่ products
SELECT COUNT(*), AVG(price), MIN(price), MAX(price)
FROM products WHERE category = 'Electronics';

-- UPDATE
UPDATE products
SET 
    price = ROUND(price * 0.95, 2),
    updated_at = NOW()
WHERE category = 'Electronics';

SELECT ROW_COUNT() AS updated;
```

### ข้อ 2
อัปเดต status ของ orders ที่ค้างเกิน 7 วัน จาก 'pending' เป็น 'cancelled'

**เฉลย:**
```sql
UPDATE orders
SET 
    status = 'cancelled',
    notes = CONCAT(COALESCE(notes,''), ' | Auto-cancelled: no payment after 7 days'),
    updated_at = NOW()
WHERE 
    status = 'pending'
    AND order_date < DATE_SUB(NOW(), INTERVAL 7 DAY);

SELECT ROW_COUNT() AS orders_cancelled;
```

### ข้อ 3
อัปเดต employees ให้ขึ้นเงินเดือนตามอายุงาน (1-3 ปี: 5%, 3-5 ปี: 8%, 5+ ปี: 12%)

**เฉลย:**
```sql
UPDATE employees
SET salary = CASE
    WHEN TIMESTAMPDIFF(YEAR, hire_date, CURRENT_DATE) >= 5 THEN salary * 1.12
    WHEN TIMESTAMPDIFF(YEAR, hire_date, CURRENT_DATE) >= 3 THEN salary * 1.08
    WHEN TIMESTAMPDIFF(YEAR, hire_date, CURRENT_DATE) >= 1 THEN salary * 1.05
    ELSE salary
END,
updated_at = NOW()
WHERE is_active = TRUE;
```

### ข้อ 4
อัปเดต order total_amount ให้ตรงกับผลรวมจาก order_items จริง

**เฉลย:**
```sql
UPDATE orders o
SET o.total_amount = (
    SELECT SUM(oi.quantity * oi.unit_price - oi.discount)
    FROM order_items oi
    WHERE oi.order_id = o.order_id
)
WHERE o.status != 'cancelled'
  AND EXISTS (
    SELECT 1 FROM order_items WHERE order_id = o.order_id
  );
```

### ข้อ 5
อัปเดต products ให้ is_available = FALSE ถ้า stock = 0

**เฉลย:**
```sql
UPDATE products
SET is_available = (stock_quantity > 0);

-- ดูผล
SELECT 
    SUM(CASE WHEN is_available THEN 1 ELSE 0 END) AS available,
    SUM(CASE WHEN NOT is_available THEN 1 ELSE 0 END) AS unavailable
FROM products;
```

### ข้อ 6
UPDATE ด้วย Transaction: โยกย้ายพนักงานจากแผนกหนึ่งไปอีกแผนก

**เฉลย:**
```sql
START TRANSACTION;

-- บันทึก current state
INSERT INTO employee_history (employee_id, field_changed, old_value, new_value, changed_at)
SELECT employee_id, 'department_id', department_id, 2, NOW()
FROM employees WHERE employee_id IN (3, 5);

-- ย้ายแผนก
UPDATE employees
SET 
    department_id = 2,  -- ย้ายไปแผนก Marketing
    updated_at = NOW()
WHERE employee_id IN (3, 5);

-- ตรวจสอบ
SELECT employee_id, first_name, department_id FROM employees WHERE employee_id IN (3, 5);

-- ถ้า OK
COMMIT;
-- ถ้าผิด: ROLLBACK;
```

### ข้อ 7
UPDATE ด้วย JOIN: อัปเดตราคา products โดยดึงราคาจาก supplier catalog

**เฉลย:**
```sql
CREATE TEMPORARY TABLE supplier_prices (
    product_sku     VARCHAR(50),
    supplier_price  DECIMAL(10,2)
);

INSERT INTO supplier_prices VALUES
    ('PHONE-001', 35000),
    ('PHONE-002', 28000),
    ('LAPTOP-001', 42000);

UPDATE products p
JOIN supplier_prices sp ON p.sku = sp.product_sku
SET 
    p.cost = sp.supplier_price,
    p.updated_at = NOW()
WHERE p.cost != sp.supplier_price OR p.cost IS NULL;

SELECT ROW_COUNT() AS prices_updated;
DROP TEMPORARY TABLE supplier_prices;
```

### ข้อ 8
Implement Optimistic Locking สำหรับการอัปเดต order status

**เฉลย:**
```sql
-- เพิ่ม version ถ้ายังไม่มี
ALTER TABLE orders ADD COLUMN IF NOT EXISTS version INT DEFAULT 0;

-- อ่านข้อมูลพร้อม version
SELECT order_id, status, version FROM orders WHERE order_id = 1002;

-- สมมติได้ version = 0
SET @current_version = 0;

-- UPDATE พร้อมตรวจ version
UPDATE orders
SET 
    status = 'confirmed',
    version = version + 1,
    updated_at = NOW()
WHERE 
    order_id = 1002
    AND version = @current_version;

-- ตรวจสอบว่าสำเร็จ
SELECT 
    ROW_COUNT() AS success,
    CASE ROW_COUNT() 
        WHEN 1 THEN 'Updated successfully'
        ELSE 'Conflict: record was modified by another user'
    END AS message;
```

### ข้อ 9
อัปเดต customer tier ตามยอดซื้อสะสม

**เฉลย:**
```sql
-- เพิ่ม tier column ถ้ายังไม่มี
ALTER TABLE customers 
ADD COLUMN IF NOT EXISTS tier VARCHAR(20) DEFAULT 'Bronze';

-- คำนวณและอัปเดต tier
UPDATE customers c
SET c.tier = (
    SELECT CASE 
        WHEN SUM(o.total_amount) >= 100000 THEN 'Platinum'
        WHEN SUM(o.total_amount) >= 50000  THEN 'Gold'
        WHEN SUM(o.total_amount) >= 10000  THEN 'Silver'
        ELSE 'Bronze'
    END
    FROM orders o
    WHERE o.customer_id = c.customer_id
      AND o.status = 'completed'
);

SELECT customer_id, first_name, tier FROM customers ORDER BY tier;
```

### ข้อ 10
Batch UPDATE: อัปเดต products ทีละ 100 rows ด้วย stored procedure

**เฉลย:**
```sql
DELIMITER //
CREATE PROCEDURE batch_archive_old_products()
BEGIN
    DECLARE v_batch_size INT DEFAULT 100;
    DECLARE v_updated INT DEFAULT 1;
    DECLARE v_total INT DEFAULT 0;
    
    WHILE v_updated > 0 DO
        UPDATE products
        SET is_available = FALSE
        WHERE last_sold_date < DATE_SUB(CURRENT_DATE, INTERVAL 1 YEAR)
          AND is_available = TRUE
        LIMIT v_batch_size;
        
        SET v_updated = ROW_COUNT();
        SET v_total = v_total + v_updated;
        
        -- หยุดพักเพื่อลด load
        IF v_updated > 0 THEN
            DO SLEEP(0.05);
        END IF;
    END WHILE;
    
    SELECT v_total AS total_archived;
END //
DELIMITER ;

CALL batch_archive_old_products();
```

---

## สรุป

ในบทนี้เราเรียนรู้:
1. **UPDATE พื้นฐาน**: syntax, หลาย columns, expressions
2. **UPDATE กับ Subqueries**: correlated subquery, scalar subquery
3. **UPDATE กับ JOIN**: multi-table update (MySQL/PostgreSQL/SQL Server)
4. **Safe practices**: SELECT ก่อน UPDATE, Transaction, SQL_SAFE_UPDATES
5. **LIMIT**: ป้องกัน mass update, batch processing
6. **Optimistic Locking**: version column ป้องกัน race condition
7. **Real-world patterns**: order management, HR, inventory

> **กฎทอง**: ก่อน UPDATE สำคัญทุกครั้ง ให้ทำ SELECT ด้วย WHERE เดียวกันก่อน เพื่อตรวจสอบว่าจะ affect กี่ rows แล้วค่อยเปลี่ยนเป็น UPDATE
