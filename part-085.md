# Part 085: PL/pgSQL - PostgreSQL Stored Procedures

## บทนำ

PL/pgSQL (Procedural Language/PostgreSQL) คือภาษา procedural ของ PostgreSQL ที่รวมความสามารถของ SQL เข้ากับ flow control, variables, loops และ exception handling ทำให้สามารถเขียนโปรแกรมที่ซับซ้อนได้ภายในฐานข้อมูล

---

## 1. PL/pgSQL Block Structure

```sql
-- โครงสร้างพื้นฐานของ PL/pgSQL block
DO $$
[<<label>>]
[DECLARE
    variable_name data_type [:= initial_value];
    ...]
BEGIN
    -- executable statements
    [EXCEPTION
        WHEN exception_condition THEN
            -- error handling statements
    ]
END [label];
$$;

-- ตัวอย่างที่ 1: Anonymous Block พื้นฐาน
DO $$
DECLARE
    v_message TEXT := 'Hello from PL/pgSQL!';
    v_count INT;
BEGIN
    SELECT COUNT(*) INTO v_count FROM employees;
    RAISE NOTICE '%  Total employees: %', v_message, v_count;
END;
$$;

-- ตัวอย่างที่ 2: Function พื้นฐาน
CREATE OR REPLACE FUNCTION fn_get_greeting(p_name TEXT)
RETURNS TEXT
LANGUAGE plpgsql
AS $$
DECLARE
    v_result TEXT;
BEGIN
    v_result := 'สวัสดีคุณ ' || p_name || '!';
    RETURN v_result;
END;
$$;

SELECT fn_get_greeting('สมชาย');

-- ตัวอย่างที่ 3: Procedure พื้นฐาน
CREATE OR REPLACE PROCEDURE sp_log_event(
    p_event_type TEXT,
    p_message TEXT
)
LANGUAGE plpgsql
AS $$
BEGIN
    INSERT INTO event_log (event_type, message, logged_at, logged_by)
    VALUES (p_event_type, p_message, NOW(), current_user);
    
    RAISE NOTICE '[%] %', p_event_type, p_message;
END;
$$;

CALL sp_log_event('INFO', 'System started');
```

---

## 2. Variables and Data Types

```sql
-- ตัวอย่างที่ 4: ตัวแปรประเภทต่างๆ
DO $$
DECLARE
    -- Scalar types
    v_integer INT := 0;
    v_bigint BIGINT;
    v_numeric NUMERIC(10, 2) := 0.00;
    v_text TEXT := '';
    v_varchar VARCHAR(100);
    v_boolean BOOLEAN := FALSE;
    v_date DATE := CURRENT_DATE;
    v_timestamp TIMESTAMP := NOW();
    v_timestamptz TIMESTAMPTZ;
    v_interval INTERVAL;
    v_uuid UUID;
    
    -- Using %TYPE - ใช้ type จาก column
    v_salary employees.salary%TYPE;
    v_name employees.first_name%TYPE;
    
    -- Using %ROWTYPE - ใช้ type ทั้ง row
    v_employee employees%ROWTYPE;
    v_product products%ROWTYPE;
    
    -- Record type (flexible)
    v_record RECORD;
    
    -- Array
    v_ids INT[];
    v_names TEXT[];
    
    -- Constants
    c_tax_rate CONSTANT DECIMAL := 0.07;
    c_max_retries CONSTANT INT := 3;
BEGIN
    -- การใช้ %TYPE
    SELECT salary INTO v_salary FROM employees WHERE employee_id = 1;
    
    -- การใช้ %ROWTYPE
    SELECT * INTO v_employee FROM employees WHERE employee_id = 1;
    RAISE NOTICE 'Employee: % %', v_employee.first_name, v_employee.last_name;
    
    -- Array
    v_ids := ARRAY[1, 2, 3, 4, 5];
    RAISE NOTICE 'First ID: %', v_ids[1];
    
    -- Interval arithmetic
    v_interval := INTERVAL '1 day' * 30;
    RAISE NOTICE '30 days from now: %', CURRENT_DATE + v_interval;
END;
$$;

-- ตัวอย่างที่ 5: Variable Assignment
DO $$
DECLARE
    v_x INT;
    v_y INT;
    v_result TEXT;
BEGIN
    -- := operator
    v_x := 10;
    
    -- = (same as :=)
    v_y = 20;
    
    -- SELECT INTO
    SELECT COUNT(*) INTO v_x FROM orders WHERE status = 'pending';
    
    -- CASE expression
    v_result := CASE 
        WHEN v_x = 0 THEN 'No pending orders'
        WHEN v_x < 10 THEN 'Few pending orders'
        ELSE 'Many pending orders: ' || v_x
    END;
    
    RAISE NOTICE '%', v_result;
END;
$$;
```

---

## 3. IF/ELSIF/ELSE Statements

```sql
-- ตัวอย่างที่ 6: Basic IF statement
CREATE OR REPLACE FUNCTION fn_classify_salary(p_salary DECIMAL)
RETURNS TEXT
LANGUAGE plpgsql
AS $$
BEGIN
    IF p_salary < 25000 THEN
        RETURN 'Entry Level';
    ELSIF p_salary < 50000 THEN
        RETURN 'Junior';
    ELSIF p_salary < 80000 THEN
        RETURN 'Mid-Level';
    ELSIF p_salary < 120000 THEN
        RETURN 'Senior';
    ELSE
        RETURN 'Executive';
    END IF;
END;
$$;

SELECT first_name, salary, fn_classify_salary(salary) AS level
FROM employees;

-- ตัวอย่างที่ 7: IF with multiple conditions
CREATE OR REPLACE PROCEDURE sp_process_payment(
    p_order_id INT,
    p_payment_method TEXT,
    p_amount DECIMAL,
    OUT p_status TEXT,
    OUT p_transaction_ref TEXT
)
LANGUAGE plpgsql
AS $$
DECLARE
    v_order_total DECIMAL;
    v_order_status TEXT;
BEGIN
    -- ดึงข้อมูล order
    SELECT total_amount, status 
    INTO v_order_total, v_order_status
    FROM orders WHERE order_id = p_order_id;
    
    IF NOT FOUND THEN
        p_status := 'ERROR';
        p_transaction_ref := NULL;
        RAISE EXCEPTION 'Order % not found', p_order_id;
    END IF;
    
    -- ตรวจสอบสถานะ
    IF v_order_status != 'pending' THEN
        p_status := 'ERROR';
        p_transaction_ref := NULL;
        RAISE EXCEPTION 'Order % is not in pending status (current: %)', p_order_id, v_order_status;
    END IF;
    
    -- ตรวจสอบจำนวนเงิน
    IF p_amount < v_order_total THEN
        p_status := 'PARTIAL';
        RAISE WARNING 'Partial payment: % of %', p_amount, v_order_total;
    ELSIF p_amount > v_order_total THEN
        p_status := 'OVERPAYMENT';
        RAISE WARNING 'Overpayment: % vs %', p_amount, v_order_total;
    ELSE
        p_status := 'EXACT';
    END IF;
    
    -- ประมวลผลการชำระเงิน
    IF p_payment_method IN ('credit_card', 'debit_card') THEN
        p_transaction_ref := 'CC-' || gen_random_uuid()::TEXT;
    ELSIF p_payment_method = 'bank_transfer' THEN
        p_transaction_ref := 'BT-' || TO_CHAR(NOW(), 'YYYYMMDDHHMMSS');
    ELSIF p_payment_method = 'cash' THEN
        p_transaction_ref := 'CASH-' || p_order_id::TEXT;
    ELSE
        RAISE EXCEPTION 'Unknown payment method: %', p_payment_method;
    END IF;
    
    -- บันทึกการชำระเงิน
    INSERT INTO payments (order_id, amount, payment_method, transaction_ref, payment_date)
    VALUES (p_order_id, p_amount, p_payment_method, p_transaction_ref, NOW());
    
    -- อัพเดท order status
    UPDATE orders SET status = 'paid', paid_at = NOW() WHERE order_id = p_order_id;
END;
$$;
```

---

## 4. CASE Statement

```sql
-- ตัวอย่างที่ 8: CASE Statement
CREATE OR REPLACE FUNCTION fn_get_shipping_cost(
    p_weight DECIMAL,
    p_destination TEXT
)
RETURNS DECIMAL
LANGUAGE plpgsql
AS $$
DECLARE
    v_base_rate DECIMAL;
    v_weight_multiplier DECIMAL;
BEGIN
    -- CASE แบบ searched
    v_base_rate := CASE p_destination
        WHEN 'local' THEN 30.00
        WHEN 'domestic' THEN 50.00
        WHEN 'regional' THEN 150.00
        WHEN 'international' THEN 500.00
        ELSE 100.00
    END;
    
    -- CASE แบบ simple
    v_weight_multiplier := CASE
        WHEN p_weight <= 0.5 THEN 1.0
        WHEN p_weight <= 1.0 THEN 1.5
        WHEN p_weight <= 5.0 THEN 2.0
        WHEN p_weight <= 20.0 THEN 3.0
        ELSE 5.0
    END;
    
    RETURN v_base_rate * v_weight_multiplier;
END;
$$;

SELECT fn_get_shipping_cost(2.5, 'domestic');
```

---

## 5. LOOP, WHILE, FOR Loops

```sql
-- ตัวอย่างที่ 9: Basic LOOP
CREATE OR REPLACE PROCEDURE sp_generate_test_orders(p_count INT)
LANGUAGE plpgsql
AS $$
DECLARE
    v_i INT := 1;
    v_customer_id INT;
BEGIN
    LOOP
        EXIT WHEN v_i > p_count;
        
        -- สุ่ม customer
        SELECT customer_id INTO v_customer_id
        FROM customers ORDER BY RANDOM() LIMIT 1;
        
        INSERT INTO orders (customer_id, order_date, status, total_amount)
        VALUES (v_customer_id, NOW() - (RANDOM() * 365)::INT * INTERVAL '1 day', 'pending', RANDOM() * 1000);
        
        v_i := v_i + 1;
    END LOOP;
    
    RAISE NOTICE 'Generated % test orders', p_count;
END;
$$;

-- ตัวอย่างที่ 10: WHILE Loop
CREATE OR REPLACE FUNCTION fn_fibonacci(p_n INT)
RETURNS BIGINT
LANGUAGE plpgsql
AS $$
DECLARE
    v_a BIGINT := 0;
    v_b BIGINT := 1;
    v_temp BIGINT;
    v_i INT := 0;
BEGIN
    IF p_n <= 0 THEN RETURN 0; END IF;
    IF p_n = 1 THEN RETURN 1; END IF;
    
    WHILE v_i < p_n - 1 LOOP
        v_temp := v_a + v_b;
        v_a := v_b;
        v_b := v_temp;
        v_i := v_i + 1;
    END LOOP;
    
    RETURN v_b;
END;
$$;

SELECT fn_fibonacci(10);  -- 55

-- ตัวอย่างที่ 11: FOR Loop - Integer range
CREATE OR REPLACE PROCEDURE sp_create_monthly_reports(p_year INT)
LANGUAGE plpgsql
AS $$
DECLARE
    v_month INT;
    v_report_id INT;
BEGIN
    FOR v_month IN 1..12 LOOP
        -- สร้าง report สำหรับแต่ละเดือน
        INSERT INTO monthly_reports (year, month, created_at, status)
        VALUES (p_year, v_month, NOW(), 'pending')
        RETURNING report_id INTO v_report_id;
        
        RAISE NOTICE 'Created report for %/% (ID: %)', p_year, v_month, v_report_id;
    END LOOP;
END;
$$;

-- ตัวอย่างที่ 12: FOR Loop - Reverse
DO $$
BEGIN
    FOR i IN REVERSE 10..1 LOOP
        RAISE NOTICE 'Countdown: %', i;
    END LOOP;
    RAISE NOTICE 'BLAST OFF!';
END;
$$;

-- ตัวอย่างที่ 13: FOR Loop - Query results
CREATE OR REPLACE PROCEDURE sp_send_overdue_reminders()
LANGUAGE plpgsql
AS $$
DECLARE
    v_customer RECORD;
    v_overdue_count INT;
BEGIN
    FOR v_customer IN
        SELECT 
            c.customer_id,
            c.first_name,
            c.last_name,
            c.email,
            COUNT(o.order_id) AS overdue_count,
            SUM(o.total_amount) AS overdue_total
        FROM customers c
        JOIN orders o ON c.customer_id = o.customer_id
        WHERE o.status = 'pending'
          AND o.due_date < CURRENT_DATE
        GROUP BY c.customer_id, c.first_name, c.last_name, c.email
    LOOP
        -- ส่ง email reminder (simulated)
        INSERT INTO email_queue (
            recipient_email,
            subject,
            body,
            created_at
        ) VALUES (
            v_customer.email,
            'Payment Reminder - ' || v_customer.overdue_count || ' overdue orders',
            'Dear ' || v_customer.first_name || ', you have ' || v_customer.overdue_count || ' overdue orders totaling ' || v_customer.overdue_total,
            NOW()
        );
        
        RAISE NOTICE 'Queued reminder for % % (% orders)', v_customer.first_name, v_customer.last_name, v_customer.overdue_count;
    END LOOP;
END;
$$;

-- ตัวอย่างที่ 14: FOR Loop กับ Arrays
CREATE OR REPLACE PROCEDURE sp_process_product_ids(p_product_ids INT[])
LANGUAGE plpgsql
AS $$
DECLARE
    v_product_id INT;
    v_name TEXT;
    v_price DECIMAL;
BEGIN
    FOREACH v_product_id IN ARRAY p_product_ids
    LOOP
        SELECT product_name, unit_price INTO v_name, v_price
        FROM products WHERE product_id = v_product_id;
        
        IF FOUND THEN
            RAISE NOTICE 'Product %: % - $%', v_product_id, v_name, v_price;
        ELSE
            RAISE WARNING 'Product % not found', v_product_id;
        END IF;
    END LOOP;
END;
$$;

CALL sp_process_product_ids(ARRAY[1, 2, 3, 99, 5]);
```

---

## 6. Cursor-based Loops

```sql
-- ตัวอย่างที่ 15: Explicit Cursor
CREATE OR REPLACE PROCEDURE sp_batch_update_salaries(p_increase_pct DECIMAL)
LANGUAGE plpgsql
AS $$
DECLARE
    -- Declare cursor
    emp_cursor CURSOR FOR
        SELECT employee_id, first_name, last_name, salary, department_id
        FROM employees
        WHERE status = 'active'
        ORDER BY employee_id;
    
    v_emp employees%ROWTYPE;
    v_new_salary DECIMAL;
    v_count INT := 0;
BEGIN
    OPEN emp_cursor;
    
    LOOP
        FETCH emp_cursor INTO v_emp;
        EXIT WHEN NOT FOUND;
        
        v_new_salary := v_emp.salary * (1 + p_increase_pct / 100);
        
        -- Cap salary increase at certain levels
        IF v_emp.department_id IN (1, 2) THEN  -- Executive
            v_new_salary := LEAST(v_new_salary, 200000);
        END IF;
        
        UPDATE employees
        SET salary = v_new_salary,
            last_salary_review = CURRENT_DATE
        WHERE employee_id = v_emp.employee_id;
        
        v_count := v_count + 1;
    END LOOP;
    
    CLOSE emp_cursor;
    
    RAISE NOTICE 'Updated % employee salaries by %', v_count, p_increase_pct || '%';
END;
$$;

-- ตัวอย่างที่ 16: Parameterized Cursor
CREATE OR REPLACE PROCEDURE sp_process_dept_employees(p_dept_id INT)
LANGUAGE plpgsql
AS $$
DECLARE
    dept_emp_cursor CURSOR(p_dept INT) FOR
        SELECT employee_id, first_name, last_name, hire_date
        FROM employees
        WHERE department_id = p_dept
          AND status = 'active';
    
    v_emp RECORD;
    v_years_service DECIMAL;
BEGIN
    FOR v_emp IN dept_emp_cursor(p_dept_id)
    LOOP
        v_years_service := EXTRACT(EPOCH FROM NOW() - v_emp.hire_date) / (365.25 * 24 * 3600);
        
        RAISE NOTICE 'Employee: % % - % years', 
            v_emp.first_name, v_emp.last_name, ROUND(v_years_service::NUMERIC, 1);
    END LOOP;
END;
$$;

-- ตัวอย่างที่ 17: REF CURSOR (Cursor ที่ส่งผ่าน OUT parameter)
CREATE OR REPLACE PROCEDURE sp_get_department_employees(
    p_dept_id INT,
    INOUT p_result refcursor
)
LANGUAGE plpgsql
AS $$
BEGIN
    OPEN p_result FOR
        SELECT employee_id, first_name, last_name, salary
        FROM employees
        WHERE department_id = p_dept_id
          AND status = 'active'
        ORDER BY last_name;
END;
$$;

-- เรียกใช้
BEGIN;
CALL sp_get_department_employees(10, 'emp_cursor');
FETCH ALL FROM emp_cursor;
CLOSE emp_cursor;
COMMIT;
```

---

## 7. Exception Handling

```sql
-- ตัวอย่างที่ 18: Basic Exception Handling
CREATE OR REPLACE FUNCTION fn_safe_divide(p_numerator DECIMAL, p_denominator DECIMAL)
RETURNS DECIMAL
LANGUAGE plpgsql
AS $$
BEGIN
    IF p_denominator = 0 THEN
        RAISE EXCEPTION 'Division by zero';
    END IF;
    RETURN p_numerator / p_denominator;
EXCEPTION
    WHEN division_by_zero THEN
        RAISE WARNING 'Caught division by zero';
        RETURN NULL;
    WHEN OTHERS THEN
        RAISE WARNING 'Unexpected error: %', SQLERRM;
        RETURN NULL;
END;
$$;

-- ตัวอย่างที่ 19: Exception Handling ในขั้นสูง
CREATE OR REPLACE PROCEDURE sp_create_order_safe(
    p_customer_id INT,
    p_product_id INT,
    p_quantity INT,
    OUT p_order_id INT,
    OUT p_error_msg TEXT
)
LANGUAGE plpgsql
AS $$
DECLARE
    v_price DECIMAL;
    v_stock INT;
    v_customer_exists BOOLEAN;
BEGIN
    p_order_id := NULL;
    p_error_msg := NULL;
    
    -- Validate customer
    SELECT EXISTS(SELECT 1 FROM customers WHERE customer_id = p_customer_id AND is_active = TRUE)
    INTO v_customer_exists;
    
    IF NOT v_customer_exists THEN
        RAISE EXCEPTION USING 
            ERRCODE = 'P0001',
            MESSAGE = 'Customer not found or inactive',
            DETAIL = 'customer_id: ' || p_customer_id;
    END IF;
    
    -- Get product info
    SELECT unit_price, units_in_stock INTO v_price, v_stock
    FROM products WHERE product_id = p_product_id AND discontinued = 0;
    
    IF NOT FOUND THEN
        RAISE EXCEPTION USING
            ERRCODE = 'P0002',
            MESSAGE = 'Product not found or discontinued';
    END IF;
    
    IF v_stock < p_quantity THEN
        RAISE EXCEPTION USING
            ERRCODE = 'P0003',
            MESSAGE = 'Insufficient stock',
            DETAIL = 'Available: ' || v_stock || ', Requested: ' || p_quantity,
            HINT = 'Reduce quantity or check back later';
    END IF;
    
    -- Create order
    INSERT INTO orders (customer_id, order_date, status, total_amount)
    VALUES (p_customer_id, NOW(), 'pending', v_price * p_quantity)
    RETURNING order_id INTO p_order_id;
    
    INSERT INTO order_details (order_id, product_id, quantity, unit_price)
    VALUES (p_order_id, p_product_id, p_quantity, v_price);
    
    UPDATE products 
    SET units_in_stock = units_in_stock - p_quantity
    WHERE product_id = p_product_id;

EXCEPTION
    WHEN SQLSTATE 'P0001' THEN
        GET STACKED DIAGNOSTICS 
            p_error_msg = MESSAGE_TEXT;
        RAISE WARNING 'Customer error: %', p_error_msg;
        
    WHEN SQLSTATE 'P0002' THEN
        GET STACKED DIAGNOSTICS 
            p_error_msg = MESSAGE_TEXT;
        RAISE WARNING 'Product error: %', p_error_msg;
        
    WHEN SQLSTATE 'P0003' THEN
        DECLARE
            v_detail TEXT;
        BEGIN
            GET STACKED DIAGNOSTICS 
                p_error_msg = MESSAGE_TEXT,
                v_detail = PG_EXCEPTION_DETAIL;
            RAISE WARNING 'Stock error: % (%)', p_error_msg, v_detail;
        END;
        
    WHEN unique_violation THEN
        p_error_msg := 'Duplicate order detected';
        RAISE WARNING '%', p_error_msg;
        
    WHEN OTHERS THEN
        GET STACKED DIAGNOSTICS
            p_error_msg = MESSAGE_TEXT;
        RAISE WARNING 'Unexpected error: %', p_error_msg;
        -- Log to error table
        INSERT INTO error_log (error_code, error_message, procedure_name, occurred_at)
        VALUES (SQLSTATE, SQLERRM, 'sp_create_order_safe', NOW());
END;
$$;
```

---

## 8. RAISE for Logging

```sql
-- ตัวอย่างที่ 20: RAISE สำหรับ Logging ระดับต่างๆ
CREATE OR REPLACE PROCEDURE sp_complex_operation(p_id INT)
LANGUAGE plpgsql
AS $$
BEGIN
    -- DEBUG level (ต้องตั้ง log_min_messages)
    RAISE DEBUG 'Starting operation for ID: %', p_id;
    
    -- INFO level
    RAISE INFO 'Processing record %', p_id;
    
    -- NOTICE level (default - แสดงใน psql)
    RAISE NOTICE 'Record % processed successfully', p_id;
    
    -- WARNING level
    RAISE WARNING 'Performance concern: operation took too long for ID %', p_id;
    
    -- ERROR level - หยุดการทำงาน
    -- RAISE ERROR 'Critical error for ID %', p_id;
    
    -- EXCEPTION level - หยุดการทำงาน
    -- RAISE EXCEPTION 'Cannot continue: %', 'some error';
    
    -- Custom error codes
    RAISE EXCEPTION USING
        ERRCODE = '23503',  -- foreign_key_violation
        MESSAGE = 'Foreign key constraint failed',
        DETAIL = 'Referenced record does not exist',
        HINT = 'Check that the referenced ID exists';
END;
$$;

-- ตัวอย่างที่ 21: Structured Logging
CREATE TABLE application_logs (
    log_id SERIAL PRIMARY KEY,
    log_level VARCHAR(10),
    message TEXT,
    details JSONB,
    procedure_name TEXT,
    user_name TEXT,
    logged_at TIMESTAMP DEFAULT NOW()
);

CREATE OR REPLACE PROCEDURE log_message(
    p_level VARCHAR(10),
    p_message TEXT,
    p_details JSONB DEFAULT NULL,
    p_procedure TEXT DEFAULT NULL
)
LANGUAGE plpgsql
AS $$
BEGIN
    INSERT INTO application_logs (log_level, message, details, procedure_name, user_name)
    VALUES (p_level, p_message, p_details, COALESCE(p_procedure, current_query()), current_user);
    
    -- Also RAISE for real-time visibility
    IF p_level = 'ERROR' THEN
        RAISE WARNING '[%] %', p_level, p_message;
    ELSE
        RAISE NOTICE '[%] %', p_level, p_message;
    END IF;
END;
$$;

-- ใช้งาน
CALL log_message('INFO', 'Order processed', '{"order_id": 1001, "amount": 500}'::JSONB);
```

---

## 9. Returning Values

```sql
-- ตัวอย่างที่ 22: Return Scalar Value
CREATE OR REPLACE FUNCTION fn_get_total_revenue(p_year INT)
RETURNS DECIMAL
LANGUAGE plpgsql
AS $$
DECLARE
    v_total DECIMAL;
BEGIN
    SELECT SUM(total_amount) INTO v_total
    FROM orders
    WHERE EXTRACT(YEAR FROM order_date) = p_year
      AND status = 'completed';
    
    RETURN COALESCE(v_total, 0);
END;
$$;

SELECT fn_get_total_revenue(2024);

-- ตัวอย่างที่ 23: Return SETOF (Multiple rows)
CREATE OR REPLACE FUNCTION fn_get_customer_orders(p_customer_id INT)
RETURNS SETOF orders
LANGUAGE plpgsql
AS $$
BEGIN
    RETURN QUERY
        SELECT * FROM orders
        WHERE customer_id = p_customer_id
        ORDER BY order_date DESC;
END;
$$;

SELECT * FROM fn_get_customer_orders(1001);

-- ตัวอย่างที่ 24: Return TABLE
CREATE OR REPLACE FUNCTION fn_get_sales_by_month(p_year INT)
RETURNS TABLE(
    month_num INT,
    month_name TEXT,
    total_orders INT,
    total_revenue DECIMAL,
    avg_order_value DECIMAL
)
LANGUAGE plpgsql
AS $$
BEGIN
    RETURN QUERY
        SELECT 
            EXTRACT(MONTH FROM order_date)::INT,
            TO_CHAR(order_date, 'Month'),
            COUNT(*)::INT,
            SUM(total_amount),
            AVG(total_amount)
        FROM orders
        WHERE EXTRACT(YEAR FROM order_date) = p_year
          AND status = 'completed'
        GROUP BY EXTRACT(MONTH FROM order_date), TO_CHAR(order_date, 'Month')
        ORDER BY EXTRACT(MONTH FROM order_date);
END;
$$;

SELECT * FROM fn_get_sales_by_month(2024);

-- ตัวอย่างที่ 25: Return from Procedure with OUT
CREATE OR REPLACE PROCEDURE sp_get_customer_summary_v2(
    IN p_customer_id INT,
    OUT p_full_name TEXT,
    OUT p_total_orders INT,
    OUT p_lifetime_value DECIMAL,
    OUT p_customer_tier TEXT
)
LANGUAGE plpgsql
AS $$
BEGIN
    SELECT 
        first_name || ' ' || last_name,
        COUNT(o.order_id),
        COALESCE(SUM(o.total_amount), 0)
    INTO p_full_name, p_total_orders, p_lifetime_value
    FROM customers c
    LEFT JOIN orders o ON c.customer_id = o.customer_id AND o.status = 'completed'
    WHERE c.customer_id = p_customer_id
    GROUP BY c.first_name, c.last_name;
    
    p_customer_tier := CASE
        WHEN p_lifetime_value >= 100000 THEN 'Platinum'
        WHEN p_lifetime_value >= 50000 THEN 'Gold'
        WHEN p_lifetime_value >= 10000 THEN 'Silver'
        ELSE 'Bronze'
    END;
END;
$$;
```

---

## 10. Complex Procedures

```sql
-- ตัวอย่างที่ 26: Procedure ที่ดึงข้อมูลหลายตาราง
CREATE OR REPLACE FUNCTION fn_employee_report(p_dept_id INT DEFAULT NULL)
RETURNS TABLE(
    employee_id INT,
    full_name TEXT,
    department_name TEXT,
    manager_name TEXT,
    salary DECIMAL,
    years_of_service DECIMAL,
    salary_rank INT
)
LANGUAGE plpgsql
AS $$
BEGIN
    RETURN QUERY
        WITH emp_with_rank AS (
            SELECT 
                e.employee_id,
                e.first_name || ' ' || e.last_name AS full_name,
                d.department_name,
                m.first_name || ' ' || m.last_name AS manager_name,
                e.salary,
                ROUND(EXTRACT(EPOCH FROM NOW() - e.hire_date) / (365.25 * 86400)::NUMERIC, 1) AS years_of_service,
                RANK() OVER (PARTITION BY e.department_id ORDER BY e.salary DESC) AS dept_rank
            FROM employees e
            JOIN departments d ON e.department_id = d.department_id
            LEFT JOIN employees m ON e.manager_id = m.employee_id
            WHERE (p_dept_id IS NULL OR e.department_id = p_dept_id)
              AND e.status = 'active'
        )
        SELECT * FROM emp_with_rank;
END;
$$;

SELECT * FROM fn_employee_report(10);
SELECT * FROM fn_employee_report();  -- All departments

-- ตัวอย่างที่ 27: Recursive Function
CREATE OR REPLACE FUNCTION fn_get_org_hierarchy(p_employee_id INT, p_level INT DEFAULT 0)
RETURNS TABLE(
    employee_id INT,
    employee_name TEXT,
    level INT,
    path TEXT
)
LANGUAGE plpgsql
AS $$
DECLARE
    v_employee RECORD;
    v_subordinate RECORD;
BEGIN
    -- ดึงข้อมูล employee ปัจจุบัน
    SELECT e.employee_id, e.first_name || ' ' || e.last_name AS name
    INTO v_employee
    FROM employees e WHERE e.employee_id = p_employee_id;
    
    IF NOT FOUND THEN
        RETURN;
    END IF;
    
    -- Return current employee
    RETURN QUERY SELECT v_employee.employee_id, v_employee.name, p_level, REPEAT('  ', p_level) || v_employee.name;
    
    -- Recurse into subordinates
    FOR v_subordinate IN
        SELECT employee_id FROM employees WHERE manager_id = p_employee_id AND status = 'active'
    LOOP
        RETURN QUERY SELECT * FROM fn_get_org_hierarchy(v_subordinate.employee_id, p_level + 1);
    END LOOP;
END;
$$;

SELECT * FROM fn_get_org_hierarchy(1);  -- Start from top-level manager

-- ตัวอย่างที่ 28: Procedure สำหรับ Data Migration
CREATE OR REPLACE PROCEDURE sp_migrate_legacy_data(
    p_batch_size INT DEFAULT 1000,
    p_dry_run BOOLEAN DEFAULT TRUE
)
LANGUAGE plpgsql
AS $$
DECLARE
    v_batch_count INT := 0;
    v_total_migrated INT := 0;
    v_record RECORD;
    v_cursor CURSOR FOR
        SELECT * FROM legacy_customers
        WHERE migrated = FALSE
        ORDER BY legacy_id
        FOR UPDATE;
BEGIN
    RAISE NOTICE 'Starting migration. Dry run: %', p_dry_run;
    
    OPEN v_cursor;
    
    LOOP
        FETCH v_cursor INTO v_record;
        EXIT WHEN NOT FOUND;
        
        -- Transform and insert
        IF NOT p_dry_run THEN
            INSERT INTO customers (
                first_name, last_name, email, phone,
                created_at, legacy_id
            ) VALUES (
                TRIM(SPLIT_PART(v_record.full_name, ' ', 1)),
                TRIM(SUBSTRING(v_record.full_name FROM POSITION(' ' IN v_record.full_name))),
                LOWER(v_record.email_address),
                REGEXP_REPLACE(v_record.phone_number, '[^0-9]', '', 'g'),
                v_record.registration_date,
                v_record.legacy_id
            ) ON CONFLICT (email) DO NOTHING;
            
            -- Mark as migrated
            UPDATE legacy_customers SET migrated = TRUE WHERE legacy_id = v_record.legacy_id;
        END IF;
        
        v_total_migrated := v_total_migrated + 1;
        v_batch_count := v_batch_count + 1;
        
        -- Commit in batches
        IF v_batch_count >= p_batch_size THEN
            IF NOT p_dry_run THEN
                COMMIT;
            END IF;
            RAISE NOTICE 'Progress: % records migrated', v_total_migrated;
            v_batch_count := 0;
        END IF;
    END LOOP;
    
    CLOSE v_cursor;
    
    IF NOT p_dry_run THEN
        COMMIT;
    END IF;
    
    RAISE NOTICE 'Migration complete. Total: % records', v_total_migrated;
END;
$$;

-- ตัวอย่างที่ 29: Dynamic Query ใน PL/pgSQL
CREATE OR REPLACE FUNCTION fn_get_table_count(p_table_name TEXT)
RETURNS BIGINT
LANGUAGE plpgsql
AS $$
DECLARE
    v_count BIGINT;
    v_sql TEXT;
BEGIN
    -- Validate table name to prevent SQL injection
    IF NOT EXISTS (
        SELECT 1 FROM information_schema.tables
        WHERE table_schema = 'public'
          AND table_name = p_table_name
    ) THEN
        RAISE EXCEPTION 'Table % does not exist', p_table_name;
    END IF;
    
    -- Dynamic query
    v_sql := 'SELECT COUNT(*) FROM ' || quote_ident(p_table_name);
    EXECUTE v_sql INTO v_count;
    
    RETURN v_count;
END;
$$;

SELECT fn_get_table_count('orders');

-- ตัวอย่างที่ 30: Procedure ที่ generate JSON output
CREATE OR REPLACE FUNCTION fn_order_to_json(p_order_id INT)
RETURNS JSONB
LANGUAGE plpgsql
AS $$
DECLARE
    v_result JSONB;
BEGIN
    SELECT jsonb_build_object(
        'order_id', o.order_id,
        'order_date', o.order_date,
        'status', o.status,
        'customer', jsonb_build_object(
            'id', c.customer_id,
            'name', c.first_name || ' ' || c.last_name,
            'email', c.email
        ),
        'items', (
            SELECT jsonb_agg(
                jsonb_build_object(
                    'product_id', od.product_id,
                    'product_name', p.product_name,
                    'quantity', od.quantity,
                    'unit_price', od.unit_price,
                    'line_total', od.quantity * od.unit_price
                )
            )
            FROM order_details od
            JOIN products p ON od.product_id = p.product_id
            WHERE od.order_id = o.order_id
        ),
        'total_amount', o.total_amount
    ) INTO v_result
    FROM orders o
    JOIN customers c ON o.customer_id = c.customer_id
    WHERE o.order_id = p_order_id;
    
    IF NOT FOUND THEN
        RAISE EXCEPTION 'Order % not found', p_order_id;
    END IF;
    
    RETURN v_result;
END;
$$;

SELECT fn_order_to_json(1001);
```

---

## แบบฝึกหัด (Exercises)

**ข้อ 1:** สร้าง PL/pgSQL function `fn_calculate_age` ที่รับ birth_date และ return อายุเป็นปี

**คำตอบข้อ 1:**
```sql
CREATE OR REPLACE FUNCTION fn_calculate_age(p_birth_date DATE)
RETURNS INT LANGUAGE plpgsql AS $$
BEGIN
    RETURN EXTRACT(YEAR FROM AGE(CURRENT_DATE, p_birth_date))::INT;
END; $$;
SELECT fn_calculate_age('1990-05-15');
```

**ข้อ 2:** สร้าง Function ที่วน loop ผ่าน orders และนับจำนวนแต่ละ status

**คำตอบข้อ 2:**
```sql
CREATE OR REPLACE FUNCTION fn_count_orders_by_status()
RETURNS TABLE(status TEXT, count BIGINT) LANGUAGE plpgsql AS $$
BEGIN
    RETURN QUERY SELECT o.status, COUNT(*) FROM orders o GROUP BY o.status ORDER BY o.status;
END; $$;
SELECT * FROM fn_count_orders_by_status();
```

**ข้อ 3:** สร้าง Procedure ที่มี EXCEPTION handling สำหรับ foreign key violation

**คำตอบข้อ 3:**
```sql
CREATE OR REPLACE PROCEDURE sp_safe_insert_order_detail(
    p_order_id INT, p_product_id INT, p_qty INT, OUT p_success BOOLEAN)
LANGUAGE plpgsql AS $$
BEGIN
    p_success := FALSE;
    INSERT INTO order_details(order_id, product_id, quantity, unit_price)
    SELECT p_order_id, p_product_id, p_qty, unit_price FROM products WHERE product_id = p_product_id;
    p_success := TRUE;
EXCEPTION WHEN foreign_key_violation THEN
    RAISE WARNING 'Invalid order_id % or product_id %', p_order_id, p_product_id;
WHEN OTHERS THEN RAISE WARNING 'Error: %', SQLERRM;
END; $$;
```

**ข้อ 4:** สร้าง Function ที่ return TABLE ของ top-selling products

**คำตอบข้อ 4:**
```sql
CREATE OR REPLACE FUNCTION fn_top_products(p_limit INT DEFAULT 10)
RETURNS TABLE(product_id INT, product_name TEXT, total_sold BIGINT, revenue DECIMAL)
LANGUAGE plpgsql AS $$
BEGIN
    RETURN QUERY SELECT p.product_id, p.product_name, SUM(od.quantity)::BIGINT, SUM(od.quantity*od.unit_price)
    FROM products p JOIN order_details od ON p.product_id = od.product_id
    JOIN orders o ON od.order_id = o.order_id WHERE o.status = 'completed'
    GROUP BY p.product_id, p.product_name ORDER BY SUM(od.quantity) DESC LIMIT p_limit;
END; $$;
```

**ข้อ 5:** สร้าง Procedure ที่ใช้ Cursor วน loop ผ่าน customers และ update customer tier

**คำตอบข้อ 5:**
```sql
CREATE OR REPLACE PROCEDURE sp_update_customer_tiers()
LANGUAGE plpgsql AS $$
DECLARE
    v_cust RECORD; v_total DECIMAL;
BEGIN
    FOR v_cust IN SELECT customer_id FROM customers LOOP
        SELECT COALESCE(SUM(total_amount), 0) INTO v_total FROM orders
        WHERE customer_id = v_cust.customer_id AND status = 'completed';
        UPDATE customers SET tier = CASE WHEN v_total >= 50000 THEN 'Gold' WHEN v_total >= 10000 THEN 'Silver' ELSE 'Bronze' END
        WHERE customer_id = v_cust.customer_id;
    END LOOP;
END; $$;
```

**ข้อ 6:** สร้าง Function ที่ใช้ RAISE NOTICE บันทึก log ในทุก step

**คำตอบข้อ 6:**
```sql
CREATE OR REPLACE FUNCTION fn_process_with_log(p_id INT)
RETURNS TEXT LANGUAGE plpgsql AS $$
DECLARE v_result TEXT;
BEGIN
    RAISE NOTICE '[%] Step 1: Starting for id=%', clock_timestamp(), p_id;
    SELECT product_name INTO v_result FROM products WHERE product_id = p_id;
    RAISE NOTICE '[%] Step 2: Found product=%', clock_timestamp(), v_result;
    IF v_result IS NULL THEN RAISE EXCEPTION 'Product % not found', p_id; END IF;
    RAISE NOTICE '[%] Step 3: Complete', clock_timestamp();
    RETURN v_result;
END; $$;
```

**ข้อ 7:** สร้าง Function ที่รับ Array of IDs และ return aggregated results

**คำตอบข้อ 7:**
```sql
CREATE OR REPLACE FUNCTION fn_orders_total_for_ids(p_order_ids INT[])
RETURNS DECIMAL LANGUAGE plpgsql AS $$
DECLARE v_total DECIMAL := 0; v_id INT;
BEGIN
    FOREACH v_id IN ARRAY p_order_ids LOOP
        SELECT v_total + COALESCE(total_amount, 0) INTO v_total FROM orders WHERE order_id = v_id;
    END LOOP;
    RETURN v_total;
END; $$;
SELECT fn_orders_total_for_ids(ARRAY[1,2,3,4,5]);
```

**ข้อ 8:** สร้าง Recursive Function สำหรับ category tree (parent-child)

**คำตอบข้อ 8:**
```sql
CREATE OR REPLACE FUNCTION fn_category_tree(p_parent_id INT DEFAULT NULL)
RETURNS TABLE(id INT, name TEXT, level INT) LANGUAGE plpgsql AS $$
BEGIN
    RETURN QUERY
    WITH RECURSIVE cat_tree AS (
        SELECT category_id, category_name, 0 AS level FROM categories WHERE parent_id IS NOT DISTINCT FROM p_parent_id
        UNION ALL
        SELECT c.category_id, c.category_name, ct.level + 1
        FROM categories c JOIN cat_tree ct ON c.parent_id = ct.category_id
    )
    SELECT category_id, category_name, level FROM cat_tree;
END; $$;
```

**ข้อ 9:** สร้าง Function ที่ return JSONB ของ customer profile

**คำตอบข้อ 9:**
```sql
CREATE OR REPLACE FUNCTION fn_customer_profile(p_customer_id INT)
RETURNS JSONB LANGUAGE plpgsql AS $$
DECLARE v_result JSONB;
BEGIN
    SELECT jsonb_build_object('id', c.customer_id, 'name', c.first_name||' '||c.last_name,
        'email', c.email, 'stats', jsonb_build_object('orders', COUNT(o.order_id), 'total', COALESCE(SUM(o.total_amount),0)))
    INTO v_result FROM customers c LEFT JOIN orders o ON c.customer_id = o.customer_id AND o.status = 'completed'
    WHERE c.customer_id = p_customer_id GROUP BY c.customer_id, c.first_name, c.last_name, c.email;
    RETURN v_result;
END; $$;
```

**ข้อ 10:** สร้าง Procedure สำหรับ cleanup expired sessions พร้อม batch commit

**คำตอบข้อ 10:**
```sql
CREATE OR REPLACE PROCEDURE sp_cleanup_expired_sessions(p_batch INT DEFAULT 100)
LANGUAGE plpgsql AS $$
DECLARE v_deleted INT := 0; v_batch_deleted INT;
BEGIN
    LOOP
        DELETE FROM user_sessions WHERE session_id IN (
            SELECT session_id FROM user_sessions WHERE expires_at < NOW() LIMIT p_batch);
        GET DIAGNOSTICS v_batch_deleted = ROW_COUNT;
        v_deleted := v_deleted + v_batch_deleted;
        EXIT WHEN v_batch_deleted = 0;
        COMMIT;
        RAISE NOTICE 'Deleted % sessions so far', v_deleted;
    END LOOP;
    RAISE NOTICE 'Total sessions cleaned: %', v_deleted;
END; $$;
```

---

*จบ Part 085: PL/pgSQL - PostgreSQL Stored Procedures*
