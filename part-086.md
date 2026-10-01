# Part 086: MySQL Stored Procedures and Functions

## บทนำ

MySQL มีระบบ Stored Procedures และ Functions ที่ทรงพลัง แต่มีไวยากรณ์และข้อกำหนดที่แตกต่างจาก PostgreSQL ในบทนี้เราจะเรียนรู้วิธีการสร้างและใช้งาน Stored Procedures ใน MySQL อย่างละเอียด

---

## 1. MySQL Procedure Syntax และ DELIMITER

```sql
-- MySQL ต้องใช้ DELIMITER เพราะ ; ใช้สำหรับ statement terminator
-- ต้องเปลี่ยน delimiter ชั่วคราวเพื่อให้ MySQL รู้ว่า procedure จบที่ไหน

-- ตัวอย่างที่ 1: การใช้ DELIMITER
DELIMITER //

CREATE PROCEDURE sp_first_example()
BEGIN
    SELECT 'Hello from MySQL!' AS message;
    SELECT NOW() AS current_time;
END //

DELIMITER ;  -- เปลี่ยนกลับ

CALL sp_first_example();

-- ตัวอย่างที่ 2: DELIMITER อื่นๆ ที่ใช้ได้
DELIMITER $$

CREATE PROCEDURE sp_another_example()
BEGIN
    SELECT 'Using $$ as delimiter' AS message;
END $$

DELIMITER ;

-- ตัวอย่างที่ 3: Procedure พื้นฐานพร้อม Comments
DELIMITER //

CREATE PROCEDURE sp_get_active_products()
COMMENT 'Returns all active products with category info'
BEGIN
    SELECT 
        p.product_id,
        p.product_name,
        c.category_name,
        p.unit_price,
        p.units_in_stock
    FROM products p
    JOIN categories c ON p.category_id = c.category_id
    WHERE p.discontinued = 0
    ORDER BY c.category_name, p.product_name;
END //

DELIMITER ;
```

---

## 2. Variables (DECLARE, SET)

```sql
-- ตัวอย่างที่ 4: DECLARE Variables
DELIMITER //

CREATE PROCEDURE sp_variable_demo()
BEGIN
    -- DECLARE ตัวแปร
    DECLARE v_message VARCHAR(100);
    DECLARE v_count INT DEFAULT 0;
    DECLARE v_total DECIMAL(15,2) DEFAULT 0.00;
    DECLARE v_is_active BOOLEAN DEFAULT FALSE;
    DECLARE v_created_date DATE;
    
    -- SET values
    SET v_message = 'Hello MySQL Variables';
    SET v_count = 10;
    SET v_total = 1234.56;
    
    -- SET with expression
    SET v_count = v_count + 5;
    
    -- SELECT INTO
    SELECT COUNT(*), SUM(total_amount)
    INTO v_count, v_total
    FROM orders
    WHERE status = 'completed';
    
    SELECT v_message AS message, v_count AS order_count, v_total AS total_revenue;
END //

DELIMITER ;

-- ตัวอย่างที่ 5: User-defined Variables (@variable)
-- MySQL User-defined variables (session-level)
SET @customer_id = 1001;
SET @min_amount = 500.00;

CALL sp_get_customer_orders(@customer_id);

-- ตัวอย่างที่ 6: Variables ใน Procedure พร้อม logic
DELIMITER //

CREATE PROCEDURE sp_calculate_bonus(
    IN p_employee_id INT,
    OUT p_bonus_amount DECIMAL(10,2)
)
BEGIN
    DECLARE v_salary DECIMAL(10,2);
    DECLARE v_performance_score INT;
    DECLARE v_years_service INT;
    DECLARE v_bonus_pct DECIMAL(5,2) DEFAULT 0;
    
    -- ดึงข้อมูลพนักงาน
    SELECT salary, performance_score, 
           TIMESTAMPDIFF(YEAR, hire_date, CURDATE())
    INTO v_salary, v_performance_score, v_years_service
    FROM employees
    WHERE employee_id = p_employee_id;
    
    -- คำนวณ bonus %
    IF v_performance_score >= 90 THEN
        SET v_bonus_pct = 20.0;
    ELSEIF v_performance_score >= 75 THEN
        SET v_bonus_pct = 15.0;
    ELSEIF v_performance_score >= 60 THEN
        SET v_bonus_pct = 10.0;
    ELSE
        SET v_bonus_pct = 5.0;
    END IF;
    
    -- เพิ่ม bonus สำหรับพนักงานอาวุโส
    IF v_years_service >= 10 THEN
        SET v_bonus_pct = v_bonus_pct + 5.0;
    ELSEIF v_years_service >= 5 THEN
        SET v_bonus_pct = v_bonus_pct + 2.5;
    END IF;
    
    SET p_bonus_amount = v_salary * (v_bonus_pct / 100);
END //

DELIMITER ;

CALL sp_calculate_bonus(1001, @bonus);
SELECT @bonus AS bonus_amount;
```

---

## 3. Control Flow

```sql
-- ตัวอย่างที่ 7: IF/ELSEIF/ELSE
DELIMITER //

CREATE PROCEDURE sp_grade_student(
    IN p_student_id INT,
    IN p_score DECIMAL(5,2),
    OUT p_grade CHAR(1),
    OUT p_pass_fail VARCHAR(10)
)
BEGIN
    IF p_score >= 90 THEN
        SET p_grade = 'A';
    ELSEIF p_score >= 80 THEN
        SET p_grade = 'B';
    ELSEIF p_score >= 70 THEN
        SET p_grade = 'C';
    ELSEIF p_score >= 60 THEN
        SET p_grade = 'D';
    ELSE
        SET p_grade = 'F';
    END IF;
    
    IF p_score >= 60 THEN
        SET p_pass_fail = 'PASS';
    ELSE
        SET p_pass_fail = 'FAIL';
    END IF;
    
    UPDATE student_scores 
    SET grade = p_grade, pass_fail = p_pass_fail
    WHERE student_id = p_student_id AND score = p_score;
END //

DELIMITER ;

-- ตัวอย่างที่ 8: CASE Statement
DELIMITER //

CREATE PROCEDURE sp_apply_discount(
    IN p_order_id INT,
    IN p_customer_tier ENUM('Bronze', 'Silver', 'Gold', 'Platinum')
)
BEGIN
    DECLARE v_discount_pct DECIMAL(5,2);
    
    CASE p_customer_tier
        WHEN 'Bronze' THEN SET v_discount_pct = 0;
        WHEN 'Silver' THEN SET v_discount_pct = 5.0;
        WHEN 'Gold' THEN SET v_discount_pct = 10.0;
        WHEN 'Platinum' THEN SET v_discount_pct = 15.0;
        ELSE SET v_discount_pct = 0;
    END CASE;
    
    UPDATE orders 
    SET discount_pct = v_discount_pct,
        final_amount = total_amount * (1 - v_discount_pct/100)
    WHERE order_id = p_order_id;
    
    SELECT p_order_id AS order_id, p_customer_tier AS tier, 
           v_discount_pct AS discount_applied;
END //

DELIMITER ;

-- ตัวอย่างที่ 9: WHILE Loop
DELIMITER //

CREATE PROCEDURE sp_generate_order_numbers(
    IN p_year INT,
    IN p_count INT
)
BEGIN
    DECLARE v_i INT DEFAULT 1;
    DECLARE v_order_num VARCHAR(20);
    
    WHILE v_i <= p_count DO
        SET v_order_num = CONCAT('ORD-', p_year, '-', LPAD(v_i, 6, '0'));
        
        INSERT INTO order_numbers (order_number, year, created_at)
        VALUES (v_order_num, p_year, NOW());
        
        SET v_i = v_i + 1;
    END WHILE;
    
    SELECT CONCAT('Generated ', p_count, ' order numbers for ', p_year) AS result;
END //

DELIMITER ;

-- ตัวอย่างที่ 10: REPEAT Loop (like do-while)
DELIMITER //

CREATE PROCEDURE sp_retry_operation(
    IN p_max_retries INT
)
BEGIN
    DECLARE v_retry INT DEFAULT 0;
    DECLARE v_success BOOLEAN DEFAULT FALSE;
    
    REPEAT
        SET v_retry = v_retry + 1;
        
        -- Simulate operation (replace with real logic)
        IF RAND() > 0.7 THEN
            SET v_success = TRUE;
        END IF;
        
        IF NOT v_success THEN
            SELECT CONCAT('Attempt ', v_retry, ' failed, retrying...') AS status;
        END IF;
        
    UNTIL v_success OR v_retry >= p_max_retries END REPEAT;
    
    IF v_success THEN
        SELECT CONCAT('Success after ', v_retry, ' attempt(s)') AS result;
    ELSE
        SELECT CONCAT('Failed after ', p_max_retries, ' attempts') AS result;
    END IF;
END //

DELIMITER ;

-- ตัวอย่างที่ 11: LOOP with LEAVE (for early exit)
DELIMITER //

CREATE PROCEDURE sp_find_first_available(
    IN p_start_date DATE,
    IN p_days_to_check INT,
    OUT p_available_date DATE
)
BEGIN
    DECLARE v_current_date DATE;
    DECLARE v_bookings INT;
    DECLARE v_i INT DEFAULT 0;
    
    SET v_current_date = p_start_date;
    SET p_available_date = NULL;
    
    check_dates: LOOP
        IF v_i >= p_days_to_check THEN
            LEAVE check_dates;
        END IF;
        
        -- ตรวจสอบ bookings ในวันนั้น
        SELECT COUNT(*) INTO v_bookings
        FROM bookings
        WHERE booking_date = v_current_date
          AND status = 'confirmed';
        
        IF v_bookings < 10 THEN  -- max 10 bookings per day
            SET p_available_date = v_current_date;
            LEAVE check_dates;  -- Exit loop เมื่อหาเจอ
        END IF;
        
        SET v_current_date = DATE_ADD(v_current_date, INTERVAL 1 DAY);
        SET v_i = v_i + 1;
    END LOOP check_dates;
END //

DELIMITER ;
```

---

## 4. Cursors in MySQL

```sql
-- ตัวอย่างที่ 12: Cursor พื้นฐาน
DELIMITER //

CREATE PROCEDURE sp_process_pending_orders()
BEGIN
    DECLARE v_done INT DEFAULT FALSE;
    DECLARE v_order_id INT;
    DECLARE v_customer_id INT;
    DECLARE v_total DECIMAL(10,2);
    
    -- Declare cursor
    DECLARE order_cursor CURSOR FOR
        SELECT order_id, customer_id, total_amount
        FROM orders
        WHERE status = 'pending'
        ORDER BY order_date;
    
    -- Declare handler สำหรับ end of cursor
    DECLARE CONTINUE HANDLER FOR NOT FOUND SET v_done = TRUE;
    
    OPEN order_cursor;
    
    process_loop: LOOP
        FETCH order_cursor INTO v_order_id, v_customer_id, v_total;
        
        IF v_done THEN
            LEAVE process_loop;
        END IF;
        
        -- ประมวลผล order แต่ละรายการ
        UPDATE orders 
        SET status = 'processing', processed_at = NOW()
        WHERE order_id = v_order_id;
        
        -- Log การเปลี่ยนแปลง
        INSERT INTO order_log (order_id, action, action_time)
        VALUES (v_order_id, 'processing_started', NOW());
        
    END LOOP process_loop;
    
    CLOSE order_cursor;
    
    SELECT 'Pending orders processed' AS result;
END //

DELIMITER ;

-- ตัวอย่างที่ 13: Multiple Cursors
DELIMITER //

CREATE PROCEDURE sp_match_customers_to_orders()
BEGIN
    DECLARE v_done1 INT DEFAULT FALSE;
    DECLARE v_done2 INT DEFAULT FALSE;
    DECLARE v_customer_id INT;
    DECLARE v_order_count INT;
    
    DECLARE customer_cursor CURSOR FOR
        SELECT customer_id FROM customers WHERE is_active = TRUE;
    
    DECLARE CONTINUE HANDLER FOR NOT FOUND SET v_done1 = TRUE;
    
    OPEN customer_cursor;
    
    cust_loop: LOOP
        FETCH customer_cursor INTO v_customer_id;
        IF v_done1 THEN LEAVE cust_loop; END IF;
        
        -- นับ orders ของแต่ละ customer
        SELECT COUNT(*) INTO v_order_count
        FROM orders WHERE customer_id = v_customer_id AND status = 'completed';
        
        UPDATE customers
        SET total_orders = v_order_count,
            last_updated = NOW()
        WHERE customer_id = v_customer_id;
        
    END LOOP cust_loop;
    
    CLOSE customer_cursor;
END //

DELIMITER ;

-- ตัวอย่างที่ 14: Cursor กับ Error Handling
DELIMITER //

CREATE PROCEDURE sp_safe_cursor_operation()
BEGIN
    DECLARE v_done INT DEFAULT FALSE;
    DECLARE v_product_id INT;
    DECLARE v_price DECIMAL(10,2);
    DECLARE v_error_msg VARCHAR(255);
    DECLARE v_processed INT DEFAULT 0;
    DECLARE v_failed INT DEFAULT 0;
    
    DECLARE product_cursor CURSOR FOR
        SELECT product_id, unit_price FROM products WHERE discontinued = 0;
    
    DECLARE CONTINUE HANDLER FOR NOT FOUND SET v_done = TRUE;
    DECLARE CONTINUE HANDLER FOR SQLEXCEPTION
    BEGIN
        GET DIAGNOSTICS CONDITION 1 v_error_msg = MESSAGE_TEXT;
        SET v_failed = v_failed + 1;
        INSERT INTO error_log (message, logged_at) VALUES (v_error_msg, NOW());
    END;
    
    OPEN product_cursor;
    
    prod_loop: LOOP
        FETCH product_cursor INTO v_product_id, v_price;
        IF v_done THEN LEAVE prod_loop; END IF;
        
        -- ทำ operation ที่อาจ fail
        UPDATE products 
        SET discounted_price = v_price * 0.9
        WHERE product_id = v_product_id;
        
        SET v_processed = v_processed + 1;
    END LOOP prod_loop;
    
    CLOSE product_cursor;
    
    SELECT v_processed AS processed, v_failed AS failed;
END //

DELIMITER ;
```

---

## 5. Error Handlers (DECLARE ... HANDLER)

```sql
-- ตัวอย่างที่ 15: Error Handler Types
DELIMITER //

CREATE PROCEDURE sp_handler_examples()
BEGIN
    -- EXIT HANDLER: หยุด procedure เมื่อ error
    DECLARE EXIT HANDLER FOR SQLEXCEPTION
    BEGIN
        ROLLBACK;
        SELECT 'Transaction rolled back due to error' AS result;
    END;
    
    START TRANSACTION;
    
    INSERT INTO orders (customer_id, order_date, total_amount, status)
    VALUES (9999, NOW(), 100.00, 'pending');  -- อาจ fail ถ้า customer ไม่มี
    
    COMMIT;
END //

DELIMITER ;

-- ตัวอย่างที่ 16: CONTINUE vs EXIT Handler
DELIMITER //

CREATE PROCEDURE sp_batch_insert_products(
    IN p_names TEXT  -- comma-separated product names
)
BEGIN
    DECLARE v_done INT DEFAULT FALSE;
    DECLARE v_name VARCHAR(100);
    DECLARE v_success INT DEFAULT 0;
    DECLARE v_failed INT DEFAULT 0;
    
    -- CONTINUE handler - ไม่หยุด ทำต่อไป
    DECLARE CONTINUE HANDLER FOR SQLEXCEPTION
    BEGIN
        SET v_failed = v_failed + 1;
    END;
    
    -- Loop through names (simplified)
    -- ในความเป็นจริงใช้ String split logic
    INSERT INTO products (product_name, unit_price, discontinued)
    VALUES ('Test Product', 9.99, 0);
    SET v_success = v_success + 1;
    
    SELECT v_success AS inserted, v_failed AS failed;
END //

DELIMITER ;

-- ตัวอย่างที่ 17: Handler สำหรับ Specific Error Codes
DELIMITER //

CREATE PROCEDURE sp_create_unique_product(
    IN p_product_name VARCHAR(200),
    IN p_price DECIMAL(10,2),
    OUT p_result VARCHAR(100)
)
BEGIN
    DECLARE v_duplicate INT DEFAULT FALSE;
    
    -- Handler สำหรับ duplicate key (error code 1062)
    DECLARE CONTINUE HANDLER FOR 1062
    BEGIN
        SET v_duplicate = TRUE;
    END;
    
    INSERT INTO products (product_name, unit_price, discontinued)
    VALUES (p_product_name, p_price, 0);
    
    IF v_duplicate THEN
        SET p_result = CONCAT('Product "', p_product_name, '" already exists');
    ELSE
        SET p_result = CONCAT('Product "', p_product_name, '" created successfully');
    END IF;
END //

DELIMITER ;

CALL sp_create_unique_product('Widget Pro', 29.99, @result);
SELECT @result;

-- ตัวอย่างที่ 18: Transaction กับ Error Handling
DELIMITER //

CREATE PROCEDURE sp_transfer_stock(
    IN p_from_product INT,
    IN p_to_product INT,
    IN p_quantity INT,
    OUT p_status VARCHAR(100)
)
BEGIN
    DECLARE v_from_stock INT;
    DECLARE v_to_stock INT;
    DECLARE v_error INT DEFAULT FALSE;
    
    DECLARE EXIT HANDLER FOR SQLEXCEPTION
    BEGIN
        SET v_error = TRUE;
        SET p_status = 'ERROR: Transaction rolled back';
        ROLLBACK;
    END;
    
    START TRANSACTION;
    
    -- ล็อค rows
    SELECT units_in_stock INTO v_from_stock
    FROM products WHERE product_id = p_from_product FOR UPDATE;
    
    SELECT units_in_stock INTO v_to_stock
    FROM products WHERE product_id = p_to_product FOR UPDATE;
    
    -- ตรวจสอบ stock
    IF v_from_stock < p_quantity THEN
        SET p_status = 'ERROR: Insufficient stock';
        ROLLBACK;
        LEAVE sp_transfer_stock;  -- Exit procedure
    END IF;
    
    -- ทำการโอน stock
    UPDATE products SET units_in_stock = units_in_stock - p_quantity WHERE product_id = p_from_product;
    UPDATE products SET units_in_stock = units_in_stock + p_quantity WHERE product_id = p_to_product;
    
    COMMIT;
    SET p_status = CONCAT('Transferred ', p_quantity, ' units successfully');
END //

DELIMITER ;
```

---

## 6. Prepared Statements in Procedures

```sql
-- ตัวอย่างที่ 19: Prepared Statements ใน MySQL Procedure
DELIMITER //

CREATE PROCEDURE sp_dynamic_search(
    IN p_table VARCHAR(64),
    IN p_column VARCHAR(64),
    IN p_value VARCHAR(255)
)
BEGIN
    DECLARE v_sql TEXT;
    
    -- Validate table and column names (security check)
    IF p_table NOT IN ('products', 'customers', 'orders') THEN
        SIGNAL SQLSTATE '45000' 
        SET MESSAGE_TEXT = 'Invalid table name';
    END IF;
    
    -- สร้าง dynamic SQL
    SET v_sql = CONCAT('SELECT * FROM `', p_table, '` WHERE `', p_column, '` = ?');
    
    -- Prepare statement
    SET @sql = v_sql;
    SET @val = p_value;
    
    PREPARE stmt FROM @sql;
    EXECUTE stmt USING @val;
    DEALLOCATE PREPARE stmt;
END //

DELIMITER ;

CALL sp_dynamic_search('products', 'category_id', '1');

-- ตัวอย่างที่ 20: Dynamic ORDER BY
DELIMITER //

CREATE PROCEDURE sp_get_sorted_products(
    IN p_sort_column VARCHAR(64),
    IN p_sort_direction VARCHAR(4),  -- ASC or DESC
    IN p_limit INT
)
BEGIN
    DECLARE v_sql TEXT;
    
    -- Validate sort direction
    IF UPPER(p_sort_direction) NOT IN ('ASC', 'DESC') THEN
        SET p_sort_direction = 'ASC';
    END IF;
    
    -- Validate column name
    IF p_sort_column NOT IN ('product_name', 'unit_price', 'units_in_stock', 'product_id') THEN
        SET p_sort_column = 'product_name';
    END IF;
    
    SET v_sql = CONCAT(
        'SELECT product_id, product_name, unit_price, units_in_stock ',
        'FROM products ',
        'WHERE discontinued = 0 ',
        'ORDER BY ', p_sort_column, ' ', p_sort_direction, ' ',
        'LIMIT ?'
    );
    
    SET @sql = v_sql;
    SET @lim = p_limit;
    
    PREPARE stmt FROM @sql;
    EXECUTE stmt USING @lim;
    DEALLOCATE PREPARE stmt;
END //

DELIMITER ;

CALL sp_get_sorted_products('unit_price', 'DESC', 10);
```

---

## 7. MySQL Functions vs Procedures

```sql
-- ตัวอย่างที่ 21: MySQL FUNCTION (return value, ใช้ใน SELECT ได้)
DELIMITER //

CREATE FUNCTION fn_format_price(p_price DECIMAL(10,2))
RETURNS VARCHAR(20)
DETERMINISTIC
READS SQL DATA
BEGIN
    RETURN CONCAT('฿', FORMAT(p_price, 2));
END //

DELIMITER ;

-- ใช้ใน SELECT
SELECT product_name, fn_format_price(unit_price) AS formatted_price
FROM products LIMIT 5;

-- ตัวอย่างที่ 22: Function ที่ Query ข้อมูล
DELIMITER //

CREATE FUNCTION fn_get_customer_total_orders(p_customer_id INT)
RETURNS INT
READS SQL DATA
BEGIN
    DECLARE v_count INT;
    
    SELECT COUNT(*) INTO v_count
    FROM orders
    WHERE customer_id = p_customer_id
      AND status = 'completed';
    
    RETURN COALESCE(v_count, 0);
END //

DELIMITER ;

SELECT customer_id, first_name, fn_get_customer_total_orders(customer_id) AS order_count
FROM customers LIMIT 10;

-- ตัวอย่างที่ 23: ความแตกต่างหลักระหว่าง Function และ Procedure
-- FUNCTION:
-- - RETURN value เดียว
-- - ใช้ใน SELECT, WHERE, etc. ได้
-- - ต้องมี RETURN statement
-- - ไม่รองรับ Transaction (BEGIN/COMMIT/ROLLBACK)
-- - ต้องระบุ DETERMINISTIC หรือ NOT DETERMINISTIC

-- PROCEDURE:
-- - ไม่ต้อง RETURN (optional OUT parameters)
-- - เรียกด้วย CALL statement เท่านั้น
-- - รองรับ Transaction
-- - สามารถ CALL อีก Procedure ได้
-- - สามารถ return หลาย result sets ได้

-- ตัวอย่างที่ 24: Stored Function ที่ซับซ้อน
DELIMITER //

CREATE FUNCTION fn_calculate_shipping(
    p_weight DECIMAL(8,3),
    p_distance INT,
    p_express TINYINT
)
RETURNS DECIMAL(10,2)
DETERMINISTIC
NO SQL
BEGIN
    DECLARE v_base_cost DECIMAL(10,2);
    DECLARE v_weight_cost DECIMAL(10,2);
    DECLARE v_distance_cost DECIMAL(10,2);
    DECLARE v_express_multiplier DECIMAL(3,2);
    
    -- Base cost
    SET v_base_cost = 30.00;
    
    -- Weight cost
    SET v_weight_cost = CASE
        WHEN p_weight <= 0.5 THEN 0
        WHEN p_weight <= 1.0 THEN 10.00
        WHEN p_weight <= 5.0 THEN p_weight * 5.00
        ELSE p_weight * 8.00
    END;
    
    -- Distance cost
    SET v_distance_cost = CASE
        WHEN p_distance <= 50 THEN 0
        WHEN p_distance <= 200 THEN 20.00
        WHEN p_distance <= 500 THEN 50.00
        ELSE 100.00
    END;
    
    -- Express multiplier
    SET v_express_multiplier = IF(p_express = 1, 1.5, 1.0);
    
    RETURN (v_base_cost + v_weight_cost + v_distance_cost) * v_express_multiplier;
END //

DELIMITER ;

SELECT fn_calculate_shipping(2.5, 150, 0) AS normal_shipping,
       fn_calculate_shipping(2.5, 150, 1) AS express_shipping;
```

---

## 8. Advanced MySQL Procedures

```sql
-- ตัวอย่างที่ 25: Procedure ที่ return หลาย Result Sets
DELIMITER //

CREATE PROCEDURE sp_customer_dashboard(IN p_customer_id INT)
BEGIN
    -- Result set 1: Customer info
    SELECT customer_id, first_name, last_name, email, created_at
    FROM customers WHERE customer_id = p_customer_id;
    
    -- Result set 2: Recent orders
    SELECT order_id, order_date, total_amount, status
    FROM orders
    WHERE customer_id = p_customer_id
    ORDER BY order_date DESC
    LIMIT 5;
    
    -- Result set 3: Total stats
    SELECT 
        COUNT(*) AS total_orders,
        SUM(total_amount) AS lifetime_value,
        MAX(order_date) AS last_order
    FROM orders
    WHERE customer_id = p_customer_id
      AND status = 'completed';
END //

DELIMITER ;

CALL sp_customer_dashboard(1001);
-- Application code จะได้ 3 result sets

-- ตัวอย่างที่ 26: Stored Procedure for Reporting
DELIMITER //

CREATE PROCEDURE sp_sales_report(
    IN p_start_date DATE,
    IN p_end_date DATE,
    IN p_group_by ENUM('day', 'week', 'month', 'quarter')
)
BEGIN
    IF p_group_by = 'day' THEN
        SELECT DATE(order_date) AS period, COUNT(*) AS orders, SUM(total_amount) AS revenue
        FROM orders WHERE order_date BETWEEN p_start_date AND p_end_date AND status = 'completed'
        GROUP BY DATE(order_date) ORDER BY period;
        
    ELSEIF p_group_by = 'week' THEN
        SELECT YEARWEEK(order_date) AS period, COUNT(*) AS orders, SUM(total_amount) AS revenue
        FROM orders WHERE order_date BETWEEN p_start_date AND p_end_date AND status = 'completed'
        GROUP BY YEARWEEK(order_date) ORDER BY period;
        
    ELSEIF p_group_by = 'month' THEN
        SELECT DATE_FORMAT(order_date, '%Y-%m') AS period, COUNT(*) AS orders, SUM(total_amount) AS revenue
        FROM orders WHERE order_date BETWEEN p_start_date AND p_end_date AND status = 'completed'
        GROUP BY DATE_FORMAT(order_date, '%Y-%m') ORDER BY period;
        
    ELSEIF p_group_by = 'quarter' THEN
        SELECT CONCAT(YEAR(order_date), '-Q', QUARTER(order_date)) AS period, 
               COUNT(*) AS orders, SUM(total_amount) AS revenue
        FROM orders WHERE order_date BETWEEN p_start_date AND p_end_date AND status = 'completed'
        GROUP BY YEAR(order_date), QUARTER(order_date) ORDER BY YEAR(order_date), QUARTER(order_date);
    END IF;
END //

DELIMITER ;

CALL sp_sales_report('2024-01-01', '2024-12-31', 'month');
```

---

## 9. Calling Procedures และ Nested Calls

```sql
-- ตัวอย่างที่ 27: Calling a Procedure from Another Procedure
DELIMITER //

CREATE PROCEDURE sp_validate_order(
    IN p_customer_id INT,
    IN p_product_id INT,
    IN p_quantity INT,
    OUT p_is_valid BOOLEAN,
    OUT p_error_msg VARCHAR(255)
)
BEGIN
    DECLARE v_customer_active BOOLEAN;
    DECLARE v_stock INT;
    
    SET p_is_valid = FALSE;
    SET p_error_msg = '';
    
    SELECT is_active INTO v_customer_active FROM customers WHERE customer_id = p_customer_id;
    IF NOT v_customer_active THEN
        SET p_error_msg = 'Customer is not active';
        LEAVE sp_validate_order;
    END IF;
    
    SELECT units_in_stock INTO v_stock FROM products WHERE product_id = p_product_id;
    IF v_stock < p_quantity THEN
        SET p_error_msg = CONCAT('Insufficient stock: ', v_stock, ' available');
        LEAVE sp_validate_order;
    END IF;
    
    SET p_is_valid = TRUE;
END //

CREATE PROCEDURE sp_place_order(
    IN p_customer_id INT,
    IN p_product_id INT,
    IN p_quantity INT,
    OUT p_order_id INT,
    OUT p_result_msg VARCHAR(255)
)
BEGIN
    DECLARE v_is_valid BOOLEAN;
    DECLARE v_error VARCHAR(255);
    DECLARE v_price DECIMAL(10,2);
    
    -- เรียก validation procedure
    CALL sp_validate_order(p_customer_id, p_product_id, p_quantity, v_is_valid, v_error);
    
    IF NOT v_is_valid THEN
        SET p_order_id = NULL;
        SET p_result_msg = CONCAT('Validation failed: ', v_error);
    ELSE
        SELECT unit_price INTO v_price FROM products WHERE product_id = p_product_id;
        
        INSERT INTO orders (customer_id, order_date, total_amount, status)
        VALUES (p_customer_id, NOW(), v_price * p_quantity, 'pending');
        
        SET p_order_id = LAST_INSERT_ID();
        
        INSERT INTO order_details (order_id, product_id, quantity, unit_price)
        VALUES (p_order_id, p_product_id, p_quantity, v_price);
        
        UPDATE products SET units_in_stock = units_in_stock - p_quantity
        WHERE product_id = p_product_id;
        
        SET p_result_msg = CONCAT('Order ', p_order_id, ' created successfully');
    END IF;
END //

DELIMITER ;

CALL sp_place_order(1001, 5, 2, @order_id, @msg);
SELECT @order_id, @msg;
```

---

## แบบฝึกหัด (Exercises)

**ข้อ 1:** สร้าง MySQL Procedure ที่รับ category_id และ return จำนวนสินค้าและราคาเฉลี่ย

**คำตอบข้อ 1:**
```sql
DELIMITER //
CREATE PROCEDURE sp_category_stats(IN p_cat_id INT, OUT p_count INT, OUT p_avg_price DECIMAL(10,2))
BEGIN
    SELECT COUNT(*), AVG(unit_price) INTO p_count, p_avg_price
    FROM products WHERE category_id = p_cat_id AND discontinued = 0;
END //
DELIMITER ;
CALL sp_category_stats(1, @cnt, @avg); SELECT @cnt, @avg;
```

**ข้อ 2:** สร้าง MySQL Function ที่ return customer tier จาก total spending

**คำตอบข้อ 2:**
```sql
DELIMITER //
CREATE FUNCTION fn_customer_tier(p_customer_id INT) RETURNS VARCHAR(20) READS SQL DATA
BEGIN
    DECLARE v_total DECIMAL(15,2);
    SELECT COALESCE(SUM(total_amount),0) INTO v_total FROM orders
    WHERE customer_id = p_customer_id AND status = 'completed';
    RETURN CASE WHEN v_total >= 100000 THEN 'Platinum' WHEN v_total >= 50000 THEN 'Gold'
                WHEN v_total >= 10000 THEN 'Silver' ELSE 'Bronze' END;
END //
DELIMITER ;
SELECT customer_id, fn_customer_tier(customer_id) AS tier FROM customers LIMIT 5;
```

**ข้อ 3:** สร้าง Procedure ที่ใช้ Cursor วน loop อัพเดท inventory

**คำตอบข้อ 3:**
```sql
DELIMITER //
CREATE PROCEDURE sp_update_inventory()
BEGIN
    DECLARE v_done INT DEFAULT FALSE;
    DECLARE v_pid INT;
    DECLARE v_stock INT;
    DECLARE cur CURSOR FOR SELECT product_id, units_in_stock FROM products WHERE discontinued = 0;
    DECLARE CONTINUE HANDLER FOR NOT FOUND SET v_done = TRUE;
    OPEN cur;
    lp: LOOP
        FETCH cur INTO v_pid, v_stock;
        IF v_done THEN LEAVE lp; END IF;
        UPDATE products SET stock_status = IF(v_stock=0,'Out','In') WHERE product_id = v_pid;
    END LOOP lp;
    CLOSE cur;
END //
DELIMITER ;
```

**ข้อ 4:** สร้าง Procedure ที่มี EXIT HANDLER สำหรับ transaction rollback

**คำตอบข้อ 4:**
```sql
DELIMITER //
CREATE PROCEDURE sp_safe_transfer(IN p_from INT, IN p_to INT, IN p_amount DECIMAL(10,2))
BEGIN
    DECLARE EXIT HANDLER FOR SQLEXCEPTION BEGIN ROLLBACK; RESIGNAL; END;
    START TRANSACTION;
    UPDATE accounts SET balance = balance - p_amount WHERE account_id = p_from;
    UPDATE accounts SET balance = balance + p_amount WHERE account_id = p_to;
    COMMIT;
END //
DELIMITER ;
```

**ข้อ 5:** สร้าง Procedure ที่ใช้ Prepared Statement สำหรับ Dynamic table query

**คำตอบข้อ 5:**
```sql
DELIMITER //
CREATE PROCEDURE sp_table_row_count(IN p_table VARCHAR(64), OUT p_count INT)
BEGIN
    IF p_table NOT IN ('orders','customers','products') THEN
        SIGNAL SQLSTATE '45000' SET MESSAGE_TEXT = 'Invalid table';
    END IF;
    SET @sql = CONCAT('SELECT COUNT(*) INTO @cnt FROM `', p_table, '`');
    PREPARE stmt FROM @sql; EXECUTE stmt; DEALLOCATE PREPARE stmt;
    SET p_count = @cnt;
END //
DELIMITER ;
CALL sp_table_row_count('orders', @n); SELECT @n;
```

**ข้อ 6:** สร้าง WHILE loop Procedure สำหรับ generate test data

**คำตอบข้อ 6:**
```sql
DELIMITER //
CREATE PROCEDURE sp_generate_test_customers(IN p_count INT)
BEGIN
    DECLARE v_i INT DEFAULT 1;
    WHILE v_i <= p_count DO
        INSERT INTO customers (first_name, last_name, email, created_at)
        VALUES (CONCAT('First', v_i), CONCAT('Last', v_i), CONCAT('test', v_i, '@test.com'), NOW());
        SET v_i = v_i + 1;
    END WHILE;
    SELECT CONCAT('Created ', p_count, ' customers') AS result;
END //
DELIMITER ;
```

**ข้อ 7:** สร้าง Procedure ที่เรียกใช้ procedure อื่น (nested call)

**คำตอบข้อ 7:**
```sql
DELIMITER //
CREATE PROCEDURE sp_full_order_process(IN p_cust_id INT, IN p_prod_id INT, IN p_qty INT)
BEGIN
    DECLARE v_valid BOOLEAN; DECLARE v_err VARCHAR(255); DECLARE v_oid INT; DECLARE v_msg VARCHAR(255);
    CALL sp_validate_order(p_cust_id, p_prod_id, p_qty, v_valid, v_err);
    IF v_valid THEN
        CALL sp_place_order(p_cust_id, p_prod_id, p_qty, v_oid, v_msg);
        SELECT v_oid AS order_id, v_msg AS message;
    ELSE
        SELECT NULL AS order_id, v_err AS message;
    END IF;
END //
DELIMITER ;
```

**ข้อ 8:** ดูรายการ Procedures และ Functions ทั้งหมดใน Database

**คำตอบข้อ 8:**
```sql
SELECT ROUTINE_NAME, ROUTINE_TYPE, CREATED, LAST_ALTERED, DEFINER
FROM information_schema.ROUTINES
WHERE ROUTINE_SCHEMA = DATABASE()
ORDER BY ROUTINE_TYPE, ROUTINE_NAME;

-- ดู definition
SHOW CREATE PROCEDURE sp_sales_report;
SHOW CREATE FUNCTION fn_customer_tier;
```

**ข้อ 9:** สร้าง Function ที่ตรวจสอบว่า email valid หรือไม่

**คำตอบข้อ 9:**
```sql
DELIMITER //
CREATE FUNCTION fn_is_valid_email(p_email VARCHAR(255)) RETURNS TINYINT(1) DETERMINISTIC NO SQL
BEGIN
    RETURN p_email REGEXP '^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\\.[A-Za-z]{2,}$';
END //
DELIMITER ;
SELECT fn_is_valid_email('test@example.com') AS valid1, fn_is_valid_email('invalid') AS valid2;
```

**ข้อ 10:** สร้าง Procedure สำหรับ PIVOT report (rows to columns)

**คำตอบข้อ 10:**
```sql
DELIMITER //
CREATE PROCEDURE sp_pivot_monthly_sales(IN p_year INT)
BEGIN
    SET @sql = CONCAT(
        'SELECT product_id, ',
        GROUP_CONCAT(DISTINCT CONCAT('SUM(CASE WHEN MONTH(o.order_date)=', m, ' THEN od.quantity ELSE 0 END) AS m', m)
        ORDER BY m SEPARATOR ', ')
    );
    -- Simplified version using static months
    SELECT product_id,
        SUM(CASE WHEN MONTH(o.order_date)=1 THEN od.quantity ELSE 0 END) AS jan,
        SUM(CASE WHEN MONTH(o.order_date)=2 THEN od.quantity ELSE 0 END) AS feb,
        SUM(CASE WHEN MONTH(o.order_date)=3 THEN od.quantity ELSE 0 END) AS mar,
        SUM(CASE WHEN MONTH(o.order_date)=12 THEN od.quantity ELSE 0 END) AS dec_month
    FROM order_details od JOIN orders o ON od.order_id = o.order_id
    WHERE YEAR(o.order_date) = p_year GROUP BY product_id;
END //
DELIMITER ;
CALL sp_pivot_monthly_sales(2024);
```

---

*จบ Part 086: MySQL Stored Procedures and Functions*
