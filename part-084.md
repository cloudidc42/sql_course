# Part 084: Stored Procedures - Introduction (Stored Procedures - บทนำ)

## บทนำ

Stored Procedures (Stored Procedures) คือชุดคำสั่ง SQL ที่ถูกรวบรวมไว้ด้วยกัน บันทึกไว้ในฐานข้อมูล และสามารถเรียกใช้ได้ด้วยชื่อ เปรียบเสมือน "โปรแกรมย่อย" หรือ "function" ที่อยู่ในฐานข้อมูล ช่วยให้สามารถ reuse code ได้และลดความซับซ้อนในการเขียน query

---

## 1. What is a Stored Procedure (Stored Procedure คืออะไร)

```sql
-- Stored Procedure คือ:
-- 1. ชุดคำสั่ง SQL ที่ถูกเก็บในฐานข้อมูล
-- 2. สามารถ parameterized ได้ (รับ input และ return output)
-- 3. Compiled และ optimized ไว้แล้ว (บางระบบ)
-- 4. เรียกใช้ได้ด้วย CALL หรือ EXECUTE

-- ตัวอย่างที่ 1: Stored Procedure อย่างง่าย (PostgreSQL)
CREATE OR REPLACE PROCEDURE sp_hello_world()
LANGUAGE plpgsql
AS $$
BEGIN
    RAISE NOTICE 'Hello, World!';
END;
$$;

-- เรียกใช้
CALL sp_hello_world();

-- ตัวอย่างที่ 2: Stored Procedure อย่างง่าย (MySQL)
DELIMITER //
CREATE PROCEDURE sp_hello_world()
BEGIN
    SELECT 'Hello, World!' AS message;
END //
DELIMITER ;

-- เรียกใช้ MySQL
CALL sp_hello_world();

-- ตัวอย่างที่ 3: Stored Procedure อย่างง่าย (SQL Server)
CREATE PROCEDURE sp_hello_world
AS
BEGIN
    SELECT 'Hello, World!' AS message;
END;

-- เรียกใช้ SQL Server
EXEC sp_hello_world;
-- หรือ
EXECUTE sp_hello_world;
```

---

## 2. Benefits of Stored Procedures (ประโยชน์)

```sql
-- ประโยชน์หลักๆ:

-- 1. CODE REUSE - ใช้ซ้ำได้
-- แทนที่จะเขียน query ซับซ้อนซ้ำๆ
-- เขียนครั้งเดียวแล้วเรียกใช้บ่อยๆ

-- ตัวอย่างที่ 4: แทนที่ Query ซับซ้อนด้วย Procedure
-- โดยปกติต้องเขียน query นี้ทุกครั้ง:
SELECT 
    c.customer_id, c.first_name, c.last_name,
    COUNT(o.order_id) AS total_orders,
    SUM(o.total_amount) AS lifetime_value
FROM customers c
LEFT JOIN orders o ON c.customer_id = o.customer_id
WHERE c.customer_id = 1001 AND o.status = 'completed'
GROUP BY c.customer_id, c.first_name, c.last_name;

-- แทนที่ด้วย Procedure:
CREATE OR REPLACE PROCEDURE sp_get_customer_summary(p_customer_id INT)
LANGUAGE plpgsql
AS $$
BEGIN
    SELECT 
        c.customer_id, c.first_name, c.last_name,
        COUNT(o.order_id) AS total_orders,
        SUM(o.total_amount) AS lifetime_value
    FROM customers c
    LEFT JOIN orders o ON c.customer_id = o.customer_id
    WHERE c.customer_id = p_customer_id AND o.status = 'completed'
    GROUP BY c.customer_id, c.first_name, c.last_name;
END;
$$;

-- เรียกใช้ง่ายมาก
CALL sp_get_customer_summary(1001);

-- 2. SECURITY - ความปลอดภัย
-- ให้สิทธิ์ execute procedure แทนที่จะให้สิทธิ์ table โดยตรง
GRANT EXECUTE ON PROCEDURE sp_get_customer_summary TO app_user;
REVOKE SELECT ON customers FROM app_user;

-- 3. NETWORK EFFICIENCY - ประหยัด bandwidth
-- ส่งแค่ชื่อ procedure และ parameters แทนที่จะส่ง SQL query ทั้งหมด

-- 4. BUSINESS LOGIC CENTRALIZATION - รวม logic ไว้ที่เดียว
-- ตัวอย่างที่ 5: Business Logic ที่ซับซ้อน
CREATE OR REPLACE PROCEDURE sp_process_order(
    p_customer_id INT,
    p_product_ids INT[],
    p_quantities INT[]
)
LANGUAGE plpgsql
AS $$
DECLARE
    v_order_id INT;
    v_total DECIMAL := 0;
    v_price DECIMAL;
    v_stock INT;
    i INT;
BEGIN
    -- สร้าง Order
    INSERT INTO orders (customer_id, order_date, status)
    VALUES (p_customer_id, NOW(), 'pending')
    RETURNING order_id INTO v_order_id;
    
    -- เพิ่ม order details สำหรับแต่ละสินค้า
    FOR i IN 1..array_length(p_product_ids, 1)
    LOOP
        -- ตรวจสอบ stock
        SELECT unit_price, units_in_stock INTO v_price, v_stock
        FROM products WHERE product_id = p_product_ids[i];
        
        IF v_stock < p_quantities[i] THEN
            RAISE EXCEPTION 'Insufficient stock for product %', p_product_ids[i];
        END IF;
        
        -- เพิ่ม order detail
        INSERT INTO order_details (order_id, product_id, quantity, unit_price)
        VALUES (v_order_id, p_product_ids[i], p_quantities[i], v_price);
        
        -- อัพเดท stock
        UPDATE products 
        SET units_in_stock = units_in_stock - p_quantities[i]
        WHERE product_id = p_product_ids[i];
        
        v_total := v_total + (v_price * p_quantities[i]);
    END LOOP;
    
    -- อัพเดท order total
    UPDATE orders SET total_amount = v_total WHERE order_id = v_order_id;
    
    RAISE NOTICE 'Order % created successfully. Total: %', v_order_id, v_total;
END;
$$;
```

---

## 3. CREATE PROCEDURE Syntax

### 3.1 PostgreSQL PL/pgSQL

```sql
-- ตัวอย่างที่ 6: PostgreSQL - Syntax พื้นฐาน
CREATE [OR REPLACE] PROCEDURE procedure_name(
    [IN|OUT|INOUT] parameter_name data_type [DEFAULT default_value],
    ...
)
LANGUAGE plpgsql
AS $$
[DECLARE
    variable declarations]
BEGIN
    -- SQL statements
    [EXCEPTION
        WHEN exception_type THEN
            -- error handling
    ]
END;
$$;

-- ตัวอย่างที่ 7: PostgreSQL Procedure ที่มีตัวแปรและ logic
CREATE OR REPLACE PROCEDURE sp_update_product_price(
    IN p_product_id INT,
    IN p_new_price DECIMAL(10,2),
    OUT p_old_price DECIMAL(10,2),
    OUT p_success BOOLEAN
)
LANGUAGE plpgsql
AS $$
DECLARE
    v_product_exists BOOLEAN;
BEGIN
    p_success := FALSE;
    
    -- ตรวจสอบว่า product มีอยู่
    SELECT EXISTS(SELECT 1 FROM products WHERE product_id = p_product_id)
    INTO v_product_exists;
    
    IF NOT v_product_exists THEN
        RAISE EXCEPTION 'Product % not found', p_product_id;
    END IF;
    
    -- ดึง old price
    SELECT unit_price INTO p_old_price
    FROM products WHERE product_id = p_product_id;
    
    -- Update price
    UPDATE products
    SET unit_price = p_new_price, updated_at = NOW()
    WHERE product_id = p_product_id;
    
    p_success := TRUE;
    
    RAISE NOTICE 'Product % price changed from % to %', p_product_id, p_old_price, p_new_price;
END;
$$;

-- เรียกใช้พร้อมรับ OUT parameters
DO $$
DECLARE
    v_old_price DECIMAL;
    v_success BOOLEAN;
BEGIN
    CALL sp_update_product_price(1, 49.99, v_old_price, v_success);
    RAISE NOTICE 'Old price: %, Success: %', v_old_price, v_success;
END;
$$;
```

### 3.2 MySQL Procedures

```sql
-- ตัวอย่างที่ 8: MySQL - CREATE PROCEDURE
DELIMITER //

CREATE PROCEDURE sp_get_order_details(
    IN p_order_id INT
)
BEGIN
    SELECT 
        o.order_id,
        o.order_date,
        c.first_name,
        c.last_name,
        od.product_id,
        p.product_name,
        od.quantity,
        od.unit_price,
        od.quantity * od.unit_price AS line_total
    FROM orders o
    JOIN customers c ON o.customer_id = c.customer_id
    JOIN order_details od ON o.order_id = od.order_id
    JOIN products p ON od.product_id = p.product_id
    WHERE o.order_id = p_order_id;
END //

DELIMITER ;

-- เรียกใช้
CALL sp_get_order_details(1001);

-- ตัวอย่างที่ 9: MySQL - Procedure พร้อม OUT parameter
DELIMITER //

CREATE PROCEDURE sp_get_customer_order_count(
    IN p_customer_id INT,
    OUT p_order_count INT,
    OUT p_total_spent DECIMAL(15,2)
)
BEGIN
    SELECT 
        COUNT(*),
        SUM(total_amount)
    INTO p_order_count, p_total_spent
    FROM orders
    WHERE customer_id = p_customer_id
      AND status = 'completed';
    
    -- Handle NULL case
    SET p_order_count = COALESCE(p_order_count, 0);
    SET p_total_spent = COALESCE(p_total_spent, 0.00);
END //

DELIMITER ;

-- เรียกใช้
CALL sp_get_customer_order_count(1001, @count, @total);
SELECT @count AS order_count, @total AS total_spent;
```

### 3.3 SQL Server T-SQL

```sql
-- ตัวอย่างที่ 10: SQL Server - CREATE PROCEDURE
CREATE PROCEDURE sp_search_products
    @SearchTerm NVARCHAR(100),
    @MaxPrice DECIMAL(10,2) = NULL,  -- Optional parameter with default
    @CategoryId INT = NULL            -- Optional parameter with default
AS
BEGIN
    SET NOCOUNT ON;
    
    SELECT 
        p.product_id,
        p.product_name,
        c.category_name,
        p.unit_price,
        p.units_in_stock
    FROM products p
    JOIN categories c ON p.category_id = c.category_id
    WHERE p.product_name LIKE '%' + @SearchTerm + '%'
      AND (@MaxPrice IS NULL OR p.unit_price <= @MaxPrice)
      AND (@CategoryId IS NULL OR p.category_id = @CategoryId)
      AND p.discontinued = 0
    ORDER BY p.product_name;
END;

-- เรียกใช้
EXEC sp_search_products @SearchTerm = 'widget';
EXEC sp_search_products @SearchTerm = 'widget', @MaxPrice = 50.00;
EXEC sp_search_products 'widget', 50.00, 1;  -- Positional parameters

-- ตัวอย่างที่ 11: SQL Server - Procedure พร้อม OUTPUT parameters
CREATE PROCEDURE sp_create_customer
    @FirstName NVARCHAR(50),
    @LastName NVARCHAR(50),
    @Email NVARCHAR(100),
    @NewCustomerId INT OUTPUT,
    @ErrorMessage NVARCHAR(500) OUTPUT
AS
BEGIN
    SET NOCOUNT ON;
    
    SET @ErrorMessage = NULL;
    
    -- ตรวจสอบ email ซ้ำ
    IF EXISTS (SELECT 1 FROM customers WHERE email = @Email)
    BEGIN
        SET @ErrorMessage = 'Email already exists: ' + @Email;
        RETURN;
    END;
    
    INSERT INTO customers (first_name, last_name, email, created_at)
    VALUES (@FirstName, @LastName, @Email, GETDATE());
    
    SET @NewCustomerId = SCOPE_IDENTITY();
END;

-- เรียกใช้
DECLARE @NewId INT, @ErrMsg NVARCHAR(500);
EXEC sp_create_customer 
    @FirstName = 'สมชาย',
    @LastName = 'ใจดี',
    @Email = 'somchai@test.com',
    @NewCustomerId = @NewId OUTPUT,
    @ErrorMessage = @ErrMsg OUTPUT;

SELECT @NewId AS new_id, @ErrMsg AS error_message;
```

---

## 4. Parameters (IN, OUT, INOUT)

```sql
-- ตัวอย่างที่ 12: IN Parameters - รับค่าเข้ามา
CREATE OR REPLACE PROCEDURE sp_deactivate_customer(
    IN p_customer_id INT,     -- รับ customer_id
    IN p_reason VARCHAR(200)  -- รับเหตุผล
)
LANGUAGE plpgsql
AS $$
BEGIN
    UPDATE customers
    SET 
        is_active = FALSE,
        deactivated_at = NOW(),
        deactivation_reason = p_reason
    WHERE customer_id = p_customer_id;
    
    IF NOT FOUND THEN
        RAISE EXCEPTION 'Customer % not found', p_customer_id;
    END IF;
    
    RAISE NOTICE 'Customer % deactivated. Reason: %', p_customer_id, p_reason;
END;
$$;

CALL sp_deactivate_customer(100, 'Customer requested account closure');

-- ตัวอย่างที่ 13: OUT Parameters - ส่งค่ากลับ
CREATE OR REPLACE PROCEDURE sp_calculate_discount(
    IN p_customer_id INT,
    IN p_order_amount DECIMAL(10,2),
    OUT p_discount_pct DECIMAL(5,2),
    OUT p_discount_amount DECIMAL(10,2),
    OUT p_final_amount DECIMAL(10,2)
)
LANGUAGE plpgsql
AS $$
DECLARE
    v_total_orders INT;
    v_lifetime_value DECIMAL(15,2);
BEGIN
    -- ดูประวัติการสั่งซื้อ
    SELECT COUNT(*), SUM(total_amount)
    INTO v_total_orders, v_lifetime_value
    FROM orders
    WHERE customer_id = p_customer_id AND status = 'completed';
    
    -- คำนวณ discount
    IF v_lifetime_value > 50000 THEN
        p_discount_pct := 15.00;
    ELSIF v_lifetime_value > 20000 THEN
        p_discount_pct := 10.00;
    ELSIF v_lifetime_value > 10000 THEN
        p_discount_pct := 5.00;
    ELSIF v_total_orders > 10 THEN
        p_discount_pct := 3.00;
    ELSE
        p_discount_pct := 0.00;
    END IF;
    
    p_discount_amount := p_order_amount * (p_discount_pct / 100);
    p_final_amount := p_order_amount - p_discount_amount;
END;
$$;

-- เรียกใช้
DO $$
DECLARE
    v_pct DECIMAL;
    v_disc DECIMAL;
    v_final DECIMAL;
BEGIN
    CALL sp_calculate_discount(1001, 5000.00, v_pct, v_disc, v_final);
    RAISE NOTICE 'Discount: %%, Amount: %, Final: %', v_pct, v_disc, v_final;
END;
$$;

-- ตัวอย่างที่ 14: INOUT Parameters - รับและส่งคืนค่า
CREATE OR REPLACE PROCEDURE sp_adjust_stock(
    IN p_product_id INT,
    INOUT p_quantity INT,  -- รับ requested qty, ส่งคืน actual qty
    OUT p_status VARCHAR(50)
)
LANGUAGE plpgsql
AS $$
DECLARE
    v_available INT;
BEGIN
    SELECT units_in_stock INTO v_available
    FROM products WHERE product_id = p_product_id;
    
    IF v_available IS NULL THEN
        p_status := 'PRODUCT_NOT_FOUND';
        p_quantity := 0;
    ELSIF v_available = 0 THEN
        p_status := 'OUT_OF_STOCK';
        p_quantity := 0;
    ELSIF v_available < p_quantity THEN
        -- ส่งคืน available quantity แทน requested
        p_quantity := v_available;
        p_status := 'PARTIAL_FILL';
    ELSE
        p_status := 'FILLED';
    END IF;
    
    -- Deduct stock
    IF p_quantity > 0 THEN
        UPDATE products
        SET units_in_stock = units_in_stock - p_quantity
        WHERE product_id = p_product_id;
    END IF;
END;
$$;

-- เรียกใช้
DO $$
DECLARE
    v_qty INT := 100;  -- ต้องการ 100 ชิ้น
    v_status VARCHAR;
BEGIN
    CALL sp_adjust_stock(5, v_qty, v_status);
    RAISE NOTICE 'Got % items, Status: %', v_qty, v_status;
END;
$$;
```

---

## 5. DROP PROCEDURE and Listing

```sql
-- ตัวอย่างที่ 15: DROP PROCEDURE
-- PostgreSQL
DROP PROCEDURE IF EXISTS sp_hello_world();
DROP PROCEDURE IF EXISTS sp_update_product_price(INT, DECIMAL, DECIMAL, BOOLEAN);

-- MySQL
DROP PROCEDURE IF EXISTS sp_hello_world;
DROP PROCEDURE IF EXISTS sp_get_order_details;

-- SQL Server
DROP PROCEDURE IF EXISTS sp_hello_world;
DROP PROCEDURE IF EXISTS sp_create_customer;

-- ตัวอย่างที่ 16: ดูรายการ Stored Procedures
-- PostgreSQL
SELECT 
    routine_name AS procedure_name,
    routine_type,
    specific_schema,
    data_type AS return_type
FROM information_schema.routines
WHERE routine_type = 'PROCEDURE'
  AND routine_schema = 'public'
ORDER BY routine_name;

-- ดูรายละเอียด procedure รวม parameter types
SELECT 
    p.proname AS procedure_name,
    pg_get_function_arguments(p.oid) AS arguments,
    l.lanname AS language
FROM pg_proc p
JOIN pg_language l ON p.prolang = l.oid
WHERE p.prokind = 'p'  -- 'p' = procedure
  AND p.pronamespace = 'public'::regnamespace
ORDER BY p.proname;

-- MySQL
SELECT 
    ROUTINE_NAME,
    ROUTINE_TYPE,
    CREATED,
    LAST_ALTERED,
    DEFINER
FROM information_schema.ROUTINES
WHERE ROUTINE_SCHEMA = DATABASE()
  AND ROUTINE_TYPE = 'PROCEDURE'
ORDER BY ROUTINE_NAME;

-- ดู procedure definition (MySQL)
SHOW CREATE PROCEDURE sp_get_order_details;

-- SQL Server
SELECT 
    name AS procedure_name,
    create_date,
    modify_date,
    OBJECT_DEFINITION(object_id) AS definition
FROM sys.procedures
ORDER BY name;
```

---

## 6. Real-World Examples

```sql
-- ตัวอย่างที่ 17: Procedure สำหรับ User Registration
CREATE OR REPLACE PROCEDURE sp_register_user(
    IN p_username VARCHAR(50),
    IN p_email VARCHAR(100),
    IN p_password_hash VARCHAR(255),
    IN p_first_name VARCHAR(50),
    IN p_last_name VARCHAR(50),
    OUT p_user_id INT,
    OUT p_error_code INT,
    OUT p_error_message TEXT
)
LANGUAGE plpgsql
AS $$
BEGIN
    p_error_code := 0;
    p_error_message := NULL;
    
    -- Validate email format
    IF p_email !~ '^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}$' THEN
        p_error_code := 400;
        p_error_message := 'Invalid email format';
        RETURN;
    END IF;
    
    -- Check duplicate username
    IF EXISTS (SELECT 1 FROM users WHERE username = p_username) THEN
        p_error_code := 409;
        p_error_message := 'Username already taken';
        RETURN;
    END IF;
    
    -- Check duplicate email
    IF EXISTS (SELECT 1 FROM users WHERE email = p_email) THEN
        p_error_code := 409;
        p_error_message := 'Email already registered';
        RETURN;
    END IF;
    
    -- Insert new user
    INSERT INTO users (username, email, password_hash, first_name, last_name, created_at)
    VALUES (p_username, p_email, p_password_hash, p_first_name, p_last_name, NOW())
    RETURNING user_id INTO p_user_id;
    
    -- Create default settings
    INSERT INTO user_settings (user_id, notification_email, theme)
    VALUES (p_user_id, TRUE, 'light');
    
    p_error_code := 201;
    p_error_message := 'User created successfully';
    
    RAISE NOTICE 'New user registered: % (ID: %)', p_username, p_user_id;
END;
$$;

-- ตัวอย่างที่ 18: Procedure สำหรับ Transfer เงิน
CREATE OR REPLACE PROCEDURE sp_transfer_funds(
    IN p_from_account INT,
    IN p_to_account INT,
    IN p_amount DECIMAL(15,2),
    IN p_description VARCHAR(200),
    OUT p_transaction_id INT,
    OUT p_status VARCHAR(20)
)
LANGUAGE plpgsql
AS $$
DECLARE
    v_from_balance DECIMAL(15,2);
    v_to_exists BOOLEAN;
BEGIN
    p_status := 'FAILED';
    
    -- ล็อค accounts เพื่อป้องกัน race condition
    SELECT balance INTO v_from_balance
    FROM bank_accounts
    WHERE account_id = p_from_account
    FOR UPDATE;
    
    IF NOT FOUND THEN
        RAISE EXCEPTION 'Source account % not found', p_from_account;
    END IF;
    
    -- ตรวจสอบยอดเงิน
    IF v_from_balance < p_amount THEN
        RAISE EXCEPTION 'Insufficient funds. Balance: %, Required: %', v_from_balance, p_amount;
    END IF;
    
    -- ตรวจสอบ destination account
    SELECT EXISTS(SELECT 1 FROM bank_accounts WHERE account_id = p_to_account)
    INTO v_to_exists;
    
    IF NOT v_to_exists THEN
        RAISE EXCEPTION 'Destination account % not found', p_to_account;
    END IF;
    
    -- ทำการโอนเงิน
    UPDATE bank_accounts SET balance = balance - p_amount WHERE account_id = p_from_account;
    UPDATE bank_accounts SET balance = balance + p_amount WHERE account_id = p_to_account;
    
    -- บันทึก transaction
    INSERT INTO transactions (from_account, to_account, amount, description, transaction_date, status)
    VALUES (p_from_account, p_to_account, p_amount, p_description, NOW(), 'completed')
    RETURNING transaction_id INTO p_transaction_id;
    
    p_status := 'SUCCESS';
END;
$$;

-- ตัวอย่างที่ 19: Procedure สำหรับ Report Generation
CREATE OR REPLACE PROCEDURE sp_generate_sales_report(
    IN p_start_date DATE,
    IN p_end_date DATE,
    IN p_group_by VARCHAR(20) DEFAULT 'day'  -- 'day', 'week', 'month'
)
LANGUAGE plpgsql
AS $$
BEGIN
    IF p_group_by = 'day' THEN
        SELECT 
            DATE(order_date) AS period,
            COUNT(*) AS total_orders,
            SUM(total_amount) AS revenue
        FROM orders
        WHERE order_date BETWEEN p_start_date AND p_end_date
          AND status = 'completed'
        GROUP BY DATE(order_date)
        ORDER BY period;
        
    ELSIF p_group_by = 'week' THEN
        SELECT 
            DATE_TRUNC('week', order_date) AS period,
            COUNT(*) AS total_orders,
            SUM(total_amount) AS revenue
        FROM orders
        WHERE order_date BETWEEN p_start_date AND p_end_date
          AND status = 'completed'
        GROUP BY DATE_TRUNC('week', order_date)
        ORDER BY period;
        
    ELSIF p_group_by = 'month' THEN
        SELECT 
            DATE_TRUNC('month', order_date) AS period,
            COUNT(*) AS total_orders,
            SUM(total_amount) AS revenue
        FROM orders
        WHERE order_date BETWEEN p_start_date AND p_end_date
          AND status = 'completed'
        GROUP BY DATE_TRUNC('month', order_date)
        ORDER BY period;
    ELSE
        RAISE EXCEPTION 'Invalid group_by value: %. Must be day, week, or month', p_group_by;
    END IF;
END;
$$;

-- ตัวอย่างที่ 20: Procedure พร้อม Transaction
CREATE OR REPLACE PROCEDURE sp_bulk_update_prices(
    IN p_category_id INT,
    IN p_percentage DECIMAL(5,2)  -- % เพิ่มหรือลด (บวก=เพิ่ม, ลบ=ลด)
)
LANGUAGE plpgsql
AS $$
DECLARE
    v_count INT;
    v_min_price DECIMAL(10,2) := 1.00;  -- ราคาขั้นต่ำ
BEGIN
    -- เริ่ม transaction
    UPDATE products
    SET 
        unit_price = GREATEST(unit_price * (1 + p_percentage/100), v_min_price),
        updated_at = NOW()
    WHERE category_id = p_category_id
      AND discontinued = 0;
    
    GET DIAGNOSTICS v_count = ROW_COUNT;
    
    IF v_count = 0 THEN
        RAISE WARNING 'No products found in category %', p_category_id;
    ELSE
        RAISE NOTICE 'Updated % products in category % by %', v_count, p_category_id, p_percentage || '%';
    END IF;
    
    COMMIT;
END;
$$;
```

---

## 7. SQLite Limitations

```sql
-- SQLite ไม่รองรับ Stored Procedures!
-- SQLite มีเพียง:
-- 1. User-Defined Functions (UDFs) ผ่าน Application Code
-- 2. Triggers
-- 3. Views

-- ตัวอย่างที่ 21: Workaround สำหรับ SQLite - ใช้ Application Code
-- Python example:
-- import sqlite3
-- 
-- def sp_get_customer_orders(conn, customer_id):
--     cursor = conn.cursor()
--     cursor.execute("""
--         SELECT o.order_id, o.order_date, SUM(od.quantity * od.unit_price) as total
--         FROM orders o
--         JOIN order_details od ON o.order_id = od.order_id
--         WHERE o.customer_id = ?
--         GROUP BY o.order_id
--     """, (customer_id,))
--     return cursor.fetchall()

-- SQLite CTE เป็นทางเลือก
WITH order_summary AS (
    SELECT 
        customer_id,
        COUNT(*) AS total_orders,
        SUM(total_amount) AS lifetime_value
    FROM orders
    WHERE status = 'completed'
    GROUP BY customer_id
)
SELECT c.first_name, c.last_name, os.total_orders, os.lifetime_value
FROM customers c
JOIN order_summary os ON c.customer_id = os.customer_id
WHERE c.customer_id = 1001;
```

---

## แบบฝึกหัด (Exercises)

**ข้อ 1:** สร้าง Stored Procedure `sp_get_products_by_category` ที่รับ category_id เป็น IN parameter และ return รายการสินค้า

**คำตอบข้อ 1:**
```sql
-- PostgreSQL
CREATE OR REPLACE PROCEDURE sp_get_products_by_category(IN p_category_id INT)
LANGUAGE plpgsql AS $$
BEGIN
    SELECT product_id, product_name, unit_price, units_in_stock
    FROM products WHERE category_id = p_category_id AND discontinued = 0
    ORDER BY product_name;
END; $$;
CALL sp_get_products_by_category(1);

-- MySQL
DELIMITER //
CREATE PROCEDURE sp_get_products_by_category(IN p_category_id INT)
BEGIN
    SELECT product_id, product_name, unit_price FROM products
    WHERE category_id = p_category_id AND discontinued = 0;
END //
DELIMITER ;
CALL sp_get_products_by_category(1);
```

**ข้อ 2:** สร้าง Procedure ที่มี OUT parameter ส่งคืน total revenue ของ customer

**คำตอบข้อ 2:**
```sql
CREATE OR REPLACE PROCEDURE sp_get_customer_revenue(
    IN p_customer_id INT,
    OUT p_total_revenue DECIMAL(15,2)
)
LANGUAGE plpgsql AS $$
BEGIN
    SELECT COALESCE(SUM(total_amount), 0) INTO p_total_revenue
    FROM orders WHERE customer_id = p_customer_id AND status = 'completed';
END; $$;

DO $$
DECLARE v_revenue DECIMAL;
BEGIN
    CALL sp_get_customer_revenue(1001, v_revenue);
    RAISE NOTICE 'Revenue: %', v_revenue;
END; $$;
```

**ข้อ 3:** สร้าง Procedure สำหรับสร้าง Order ใหม่พร้อม transaction handling (PostgreSQL)

**คำตอบข้อ 3:**
```sql
CREATE OR REPLACE PROCEDURE sp_create_order(
    IN p_customer_id INT,
    IN p_product_id INT,
    IN p_quantity INT,
    OUT p_order_id INT
)
LANGUAGE plpgsql AS $$
DECLARE v_price DECIMAL; v_stock INT;
BEGIN
    SELECT unit_price, units_in_stock INTO v_price, v_stock FROM products WHERE product_id = p_product_id;
    IF v_stock < p_quantity THEN
        RAISE EXCEPTION 'Insufficient stock';
    END IF;
    INSERT INTO orders (customer_id, order_date, status, total_amount)
    VALUES (p_customer_id, NOW(), 'pending', v_price * p_quantity)
    RETURNING order_id INTO p_order_id;
    INSERT INTO order_details (order_id, product_id, quantity, unit_price) VALUES (p_order_id, p_product_id, p_quantity, v_price);
    UPDATE products SET units_in_stock = units_in_stock - p_quantity WHERE product_id = p_product_id;
END; $$;
```

**ข้อ 4:** เขียน SQL Server Procedure ที่ค้นหา customers พร้อม optional parameters

**คำตอบข้อ 4:**
```sql
CREATE PROCEDURE sp_search_customers
    @Name NVARCHAR(100) = NULL,
    @City NVARCHAR(100) = NULL,
    @Country NVARCHAR(100) = NULL
AS
BEGIN
    SELECT customer_id, first_name, last_name, city, country
    FROM customers
    WHERE (@Name IS NULL OR first_name LIKE '%' + @Name + '%' OR last_name LIKE '%' + @Name + '%')
      AND (@City IS NULL OR city = @City)
      AND (@Country IS NULL OR country = @Country);
END;
```

**ข้อ 5:** สร้าง MySQL Procedure พร้อม error handling

**คำตอบข้อ 5:**
```sql
DELIMITER //
CREATE PROCEDURE sp_delete_customer(IN p_customer_id INT)
BEGIN
    DECLARE EXIT HANDLER FOR SQLEXCEPTION
    BEGIN
        ROLLBACK;
        RESIGNAL;
    END;
    START TRANSACTION;
    DELETE FROM orders WHERE customer_id = p_customer_id AND status = 'pending';
    DELETE FROM customers WHERE customer_id = p_customer_id;
    IF ROW_COUNT() = 0 THEN
        SIGNAL SQLSTATE '45000' SET MESSAGE_TEXT = 'Customer not found';
    END IF;
    COMMIT;
END //
DELIMITER ;
```

**ข้อ 6:** ดูรายการ Stored Procedures ทั้งหมดในฐานข้อมูล (MySQL)

**คำตอบข้อ 6:**
```sql
SELECT ROUTINE_NAME, ROUTINE_TYPE, CREATED, LAST_ALTERED
FROM information_schema.ROUTINES
WHERE ROUTINE_SCHEMA = DATABASE() AND ROUTINE_TYPE = 'PROCEDURE'
ORDER BY ROUTINE_NAME;
```

**ข้อ 7:** สร้าง Procedure ที่ใช้ INOUT parameter สำหรับ pagination (page number เข้า, ข้อมูล offset ออก)

**คำตอบข้อ 7:**
```sql
CREATE OR REPLACE PROCEDURE sp_paginate_products(
    INOUT p_page INT,  -- page number (1-based)
    IN p_page_size INT DEFAULT 10,
    OUT p_total_pages INT
)
LANGUAGE plpgsql AS $$
DECLARE v_total_rows INT; v_offset INT;
BEGIN
    SELECT COUNT(*) INTO v_total_rows FROM products WHERE discontinued = 0;
    p_total_pages := CEIL(v_total_rows::FLOAT / p_page_size);
    p_page := LEAST(GREATEST(p_page, 1), p_total_pages);
    v_offset := (p_page - 1) * p_page_size;
    SELECT product_id, product_name, unit_price FROM products
    WHERE discontinued = 0 ORDER BY product_name LIMIT p_page_size OFFSET v_offset;
END; $$;
```

**ข้อ 8:** Drop procedure และตรวจสอบว่าถูกลบแล้ว

**คำตอบข้อ 8:**
```sql
DROP PROCEDURE IF EXISTS sp_hello_world;
-- ตรวจสอบ (PostgreSQL)
SELECT routine_name FROM information_schema.routines
WHERE routine_type = 'PROCEDURE' AND routine_name = 'sp_hello_world';
-- ถ้าไม่มีผลลัพธ์ = ลบสำเร็จ
```

**ข้อ 9:** สร้าง Procedure ที่ return result set (PostgreSQL)

**คำตอบข้อ 9:**
```sql
CREATE OR REPLACE FUNCTION fn_get_top_customers(p_limit INT DEFAULT 10)
RETURNS TABLE(customer_id INT, customer_name TEXT, total_spent DECIMAL)
LANGUAGE plpgsql AS $$
BEGIN
    RETURN QUERY
    SELECT c.customer_id, c.first_name || ' ' || c.last_name,
           SUM(o.total_amount)
    FROM customers c JOIN orders o ON c.customer_id = o.customer_id
    WHERE o.status = 'completed'
    GROUP BY c.customer_id, c.first_name, c.last_name
    ORDER BY SUM(o.total_amount) DESC LIMIT p_limit;
END; $$;
SELECT * FROM fn_get_top_customers(5);
```

**ข้อ 10:** สร้าง Procedure สำหรับ Data Cleanup ที่ archive ข้อมูลเก่าก่อน delete

**คำตอบข้อ 10:**
```sql
CREATE OR REPLACE PROCEDURE sp_cleanup_old_logs(IN p_days_to_keep INT DEFAULT 90)
LANGUAGE plpgsql AS $$
DECLARE v_cutoff_date TIMESTAMP; v_archived INT; v_deleted INT;
BEGIN
    v_cutoff_date := NOW() - (p_days_to_keep || ' days')::INTERVAL;
    INSERT INTO audit_log_archive SELECT * FROM audit_log WHERE created_at < v_cutoff_date;
    GET DIAGNOSTICS v_archived = ROW_COUNT;
    DELETE FROM audit_log WHERE created_at < v_cutoff_date;
    GET DIAGNOSTICS v_deleted = ROW_COUNT;
    RAISE NOTICE 'Archived: %, Deleted: %', v_archived, v_deleted;
END; $$;
CALL sp_cleanup_old_logs(30);
```

---

*จบ Part 084: Stored Procedures - Introduction*
