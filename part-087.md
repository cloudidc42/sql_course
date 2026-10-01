# Part 087: User-Defined Functions (UDFs)

## บทนำ

User-Defined Functions (UDFs) คือ Functions ที่ผู้ใช้สร้างขึ้นเองเพื่อ extend ความสามารถของ SQL ต่างจาก Stored Procedures ตรงที่ Functions สามารถใช้ได้ใน SELECT, WHERE, ORDER BY และ expression อื่นๆ ได้โดยตรง

---

## 1. Scalar Functions vs Table-Valued Functions

```sql
-- Scalar Function: return ค่าเดียว
-- Table-Valued Function: return หลาย rows เหมือนตาราง

-- ตัวอย่างที่ 1: Scalar Function พื้นฐาน (PostgreSQL)
CREATE OR REPLACE FUNCTION fn_full_name(p_first TEXT, p_last TEXT)
RETURNS TEXT
LANGUAGE plpgsql
AS $$
BEGIN
    RETURN TRIM(p_first) || ' ' || TRIM(p_last);
END;
$$;

-- ใช้ใน SELECT
SELECT fn_full_name(first_name, last_name) AS full_name FROM employees;

-- ตัวอย่างที่ 2: Table-Valued Function (PostgreSQL)
CREATE OR REPLACE FUNCTION fn_get_employee_orders(p_employee_id INT)
RETURNS TABLE(
    order_id INT,
    order_date TIMESTAMP,
    customer_name TEXT,
    total_amount DECIMAL
)
LANGUAGE plpgsql
AS $$
BEGIN
    RETURN QUERY
    SELECT o.order_id, o.order_date, c.first_name || ' ' || c.last_name, o.total_amount
    FROM orders o JOIN customers c ON o.customer_id = c.customer_id
    WHERE o.employee_id = p_employee_id
    ORDER BY o.order_date DESC;
END;
$$;

-- ใช้เหมือนตาราง
SELECT * FROM fn_get_employee_orders(5);
SELECT ef.order_id, ef.customer_name FROM fn_get_employee_orders(5) ef WHERE ef.total_amount > 500;
```

---

## 2. Immutable / Stable / Volatile (PostgreSQL)

```sql
-- PostgreSQL มี 3 ระดับสำหรับ Function volatility
-- ส่งผลต่อ query optimization และ caching

-- ตัวอย่างที่ 3: IMMUTABLE - ผลลัพธ์เหมือนเดิมเสมอสำหรับ input เดิม
-- ไม่มีการ query DB, ผลลัพธ์ไม่เปลี่ยนแปลง
CREATE OR REPLACE FUNCTION fn_tax_rate(p_country TEXT)
RETURNS DECIMAL
LANGUAGE plpgsql
IMMUTABLE  -- PostgreSQL สามารถ cache ผลลัพธ์ได้
AS $$
BEGIN
    CASE p_country
        WHEN 'TH' THEN RETURN 0.07;
        WHEN 'US' THEN RETURN 0.08;
        WHEN 'UK' THEN RETURN 0.20;
        ELSE RETURN 0.00;
    END CASE;
END;
$$;

-- IMMUTABLE function สามารถใช้ใน Index ได้!
CREATE INDEX idx_products_tax ON products(fn_tax_rate(country));

-- ตัวอย่างที่ 4: STABLE - ผลลัพธ์เหมือนกันตลอด transaction เดียวกัน
CREATE OR REPLACE FUNCTION fn_get_exchange_rate(p_currency TEXT)
RETURNS DECIMAL
LANGUAGE plpgsql
STABLE  -- query DB แต่ไม่เปลี่ยนแปลงใน transaction เดียว
AS $$
DECLARE
    v_rate DECIMAL;
BEGIN
    SELECT rate INTO v_rate
    FROM exchange_rates
    WHERE currency = p_currency
      AND effective_date <= CURRENT_DATE
    ORDER BY effective_date DESC
    LIMIT 1;
    
    RETURN COALESCE(v_rate, 1.0);
END;
$$;

-- ตัวอย่างที่ 5: VOLATILE (default) - ผลลัพธ์อาจเปลี่ยนได้ทุก call
CREATE OR REPLACE FUNCTION fn_get_next_order_number()
RETURNS TEXT
LANGUAGE plpgsql
VOLATILE  -- ใช้ SEQUENCE ดังนั้นต้องเป็น VOLATILE
AS $$
DECLARE
    v_seq INT;
BEGIN
    SELECT nextval('order_number_seq') INTO v_seq;
    RETURN 'ORD-' || LPAD(v_seq::TEXT, 8, '0');
END;
$$;
```

---

## 3. DETERMINISTIC (MySQL)

```sql
-- ตัวอย่างที่ 6: DETERMINISTIC ใน MySQL
-- DETERMINISTIC = ผลลัพธ์เหมือนกันสำหรับ input เดิม (เหมือน IMMUTABLE ของ PostgreSQL)
DELIMITER //

CREATE FUNCTION fn_circle_area(p_radius DECIMAL(10,4))
RETURNS DECIMAL(15,6)
DETERMINISTIC
NO SQL
BEGIN
    RETURN PI() * p_radius * p_radius;
END //

DELIMITER ;

SELECT fn_circle_area(5.0) AS area;

-- ตัวอย่างที่ 7: NOT DETERMINISTIC ใน MySQL
DELIMITER //

CREATE FUNCTION fn_current_age(p_birth_date DATE)
RETURNS INT
NOT DETERMINISTIC
NO SQL
BEGIN
    RETURN TIMESTAMPDIFF(YEAR, p_birth_date, CURDATE());
END //

DELIMITER ;

SELECT first_name, fn_current_age(birth_date) AS age FROM employees;
```

---

## 4. Scalar UDFs in PostgreSQL

```sql
-- ตัวอย่างที่ 8: Scalar Function สำหรับ String Processing
CREATE OR REPLACE FUNCTION fn_clean_phone(p_phone TEXT)
RETURNS TEXT
LANGUAGE plpgsql
IMMUTABLE
AS $$
BEGIN
    -- ลบ characters ที่ไม่ใช่ตัวเลข
    RETURN REGEXP_REPLACE(p_phone, '[^0-9]', '', 'g');
END;
$$;

SELECT fn_clean_phone('(081) 234-5678') AS clean_phone;  -- '0812345678'

-- ตัวอย่างที่ 9: Scalar Function สำหรับ Business Logic
CREATE OR REPLACE FUNCTION fn_calculate_vat(
    p_amount DECIMAL(15,2),
    p_include_vat BOOLEAN DEFAULT FALSE
)
RETURNS DECIMAL(15,2)
LANGUAGE plpgsql
IMMUTABLE
AS $$
DECLARE
    v_vat_rate CONSTANT DECIMAL := 0.07;
BEGIN
    IF p_include_vat THEN
        -- คำนวณ VAT จากราคาที่รวม VAT แล้ว
        RETURN p_amount - (p_amount / (1 + v_vat_rate));
    ELSE
        -- คำนวณ VAT จากราคาก่อน VAT
        RETURN p_amount * v_vat_rate;
    END IF;
END;
$$;

SELECT 
    unit_price,
    fn_calculate_vat(unit_price) AS vat_amount,
    unit_price + fn_calculate_vat(unit_price) AS price_with_vat
FROM products LIMIT 5;

-- ตัวอย่างที่ 10: Scalar Function พร้อม Error Handling
CREATE OR REPLACE FUNCTION fn_safe_cast_int(p_value TEXT)
RETURNS INT
LANGUAGE plpgsql
IMMUTABLE
AS $$
BEGIN
    RETURN p_value::INT;
EXCEPTION
    WHEN invalid_text_representation THEN
        RETURN NULL;
    WHEN numeric_value_out_of_range THEN
        RETURN NULL;
END;
$$;

SELECT fn_safe_cast_int('123') AS valid,
       fn_safe_cast_int('abc') AS invalid,
       fn_safe_cast_int('999999999999') AS too_large;

-- ตัวอย่างที่ 11: Scalar Function สำหรับ Date Operations
CREATE OR REPLACE FUNCTION fn_working_days_between(
    p_start_date DATE,
    p_end_date DATE
)
RETURNS INT
LANGUAGE plpgsql
STABLE
AS $$
DECLARE
    v_days INT := 0;
    v_current DATE;
BEGIN
    v_current := p_start_date;
    
    WHILE v_current <= p_end_date LOOP
        -- ไม่นับ วันเสาร์ (6) และ วันอาทิตย์ (0)
        IF EXTRACT(DOW FROM v_current) NOT IN (0, 6) THEN
            -- ไม่นับวันหยุดนักขัตฤกษ์
            IF NOT EXISTS (SELECT 1 FROM holidays WHERE holiday_date = v_current) THEN
                v_days := v_days + 1;
            END IF;
        END IF;
        v_current := v_current + INTERVAL '1 day';
    END LOOP;
    
    RETURN v_days;
END;
$$;

SELECT fn_working_days_between('2024-01-01', '2024-01-31') AS working_days;

-- ตัวอย่างที่ 12: Overloaded Functions (PostgreSQL)
-- PostgreSQL รองรับการ overload function
CREATE OR REPLACE FUNCTION fn_format_currency(p_amount DECIMAL)
RETURNS TEXT
LANGUAGE plpgsql
IMMUTABLE
AS $$
BEGIN
    RETURN '฿' || TO_CHAR(p_amount, 'FM999,999,990.00');
END;
$$;

CREATE OR REPLACE FUNCTION fn_format_currency(p_amount DECIMAL, p_currency TEXT)
RETURNS TEXT
LANGUAGE plpgsql
IMMUTABLE
AS $$
BEGIN
    RETURN p_currency || ' ' || TO_CHAR(p_amount, 'FM999,999,990.00');
END;
$$;

SELECT fn_format_currency(1234.56) AS thb;
SELECT fn_format_currency(1234.56, 'USD') AS usd;
```

---

## 5. Scalar UDFs in MySQL

```sql
-- ตัวอย่างที่ 13: MySQL Scalar Functions
DELIMITER //

-- Function แปลง Thai score เป็น letter grade
CREATE FUNCTION fn_score_to_grade(p_score DECIMAL(5,2))
RETURNS VARCHAR(2)
DETERMINISTIC
NO SQL
BEGIN
    RETURN CASE
        WHEN p_score >= 80 THEN 'A'
        WHEN p_score >= 75 THEN 'B+'
        WHEN p_score >= 70 THEN 'B'
        WHEN p_score >= 65 THEN 'C+'
        WHEN p_score >= 60 THEN 'C'
        WHEN p_score >= 55 THEN 'D+'
        WHEN p_score >= 50 THEN 'D'
        ELSE 'F'
    END;
END //

DELIMITER ;

SELECT student_id, score, fn_score_to_grade(score) AS grade FROM student_scores;

-- ตัวอย่างที่ 14: MySQL Function สำหรับ String Operations
DELIMITER //

CREATE FUNCTION fn_title_case(p_str VARCHAR(255))
RETURNS VARCHAR(255)
DETERMINISTIC
NO SQL
BEGIN
    DECLARE v_result VARCHAR(255) DEFAULT '';
    DECLARE v_i INT DEFAULT 1;
    DECLARE v_char CHAR(1);
    DECLARE v_prev_space BOOLEAN DEFAULT TRUE;
    
    WHILE v_i <= CHAR_LENGTH(p_str) DO
        SET v_char = SUBSTRING(p_str, v_i, 1);
        
        IF v_prev_space THEN
            SET v_result = CONCAT(v_result, UPPER(v_char));
        ELSE
            SET v_result = CONCAT(v_result, LOWER(v_char));
        END IF;
        
        SET v_prev_space = (v_char = ' ');
        SET v_i = v_i + 1;
    END WHILE;
    
    RETURN v_result;
END //

DELIMITER ;

SELECT fn_title_case('hello world from mysql') AS title_case;

-- ตัวอย่างที่ 15: MySQL Function ที่ Query ข้อมูล
DELIMITER //

CREATE FUNCTION fn_is_vip_customer(p_customer_id INT)
RETURNS TINYINT(1)
READS SQL DATA
BEGIN
    DECLARE v_total DECIMAL(15,2);
    
    SELECT COALESCE(SUM(total_amount), 0) INTO v_total
    FROM orders
    WHERE customer_id = p_customer_id
      AND status = 'completed'
      AND order_date >= DATE_SUB(NOW(), INTERVAL 1 YEAR);
    
    RETURN v_total >= 50000;
END //

DELIMITER ;

SELECT customer_id, first_name,
       IF(fn_is_vip_customer(customer_id), 'VIP', 'Regular') AS status
FROM customers;
```

---

## 6. Table-Valued Functions (PostgreSQL RETURNS TABLE)

```sql
-- ตัวอย่างที่ 16: RETURNS TABLE
CREATE OR REPLACE FUNCTION fn_search_products(
    p_keyword TEXT,
    p_min_price DECIMAL DEFAULT NULL,
    p_max_price DECIMAL DEFAULT NULL,
    p_in_stock BOOLEAN DEFAULT TRUE
)
RETURNS TABLE(
    product_id INT,
    product_name TEXT,
    category_name TEXT,
    unit_price DECIMAL,
    units_in_stock INT,
    relevance_score FLOAT
)
LANGUAGE plpgsql
STABLE
AS $$
BEGIN
    RETURN QUERY
    SELECT 
        p.product_id,
        p.product_name,
        c.category_name,
        p.unit_price,
        p.units_in_stock,
        similarity(p.product_name, p_keyword) AS relevance
    FROM products p
    JOIN categories c ON p.category_id = c.category_id
    WHERE p.discontinued = 0
      AND (p_keyword IS NULL OR 
           p.product_name ILIKE '%' || p_keyword || '%' OR 
           c.category_name ILIKE '%' || p_keyword || '%')
      AND (p_min_price IS NULL OR p.unit_price >= p_min_price)
      AND (p_max_price IS NULL OR p.unit_price <= p_max_price)
      AND (NOT p_in_stock OR p.units_in_stock > 0)
    ORDER BY relevance DESC, p.product_name;
END;
$$;

-- ใช้งาน
SELECT * FROM fn_search_products('wireless', 50, 200, TRUE);
SELECT product_name, unit_price 
FROM fn_search_products('keyboard') 
WHERE unit_price < 100;

-- ตัวอย่างที่ 17: Set-returning function (SETOF)
CREATE OR REPLACE FUNCTION fn_get_customers_by_tier(p_tier TEXT)
RETURNS SETOF customers
LANGUAGE plpgsql
STABLE
AS $$
BEGIN
    RETURN QUERY
    SELECT c.* FROM customers c
    JOIN customer_tiers ct ON c.customer_id = ct.customer_id
    WHERE ct.tier = p_tier
    ORDER BY c.last_name, c.first_name;
END;
$$;

SELECT * FROM fn_get_customers_by_tier('Gold');

-- ตัวอย่างที่ 18: Table function กับ LATERAL JOIN
CREATE OR REPLACE FUNCTION fn_recent_orders_for_customer(
    p_customer_id INT,
    p_limit INT DEFAULT 5
)
RETURNS TABLE(
    order_id INT,
    order_date TIMESTAMP,
    total_amount DECIMAL,
    status TEXT
)
LANGUAGE plpgsql
STABLE
AS $$
BEGIN
    RETURN QUERY
    SELECT o.order_id, o.order_date, o.total_amount, o.status
    FROM orders o
    WHERE o.customer_id = p_customer_id
    ORDER BY o.order_date DESC
    LIMIT p_limit;
END;
$$;

-- ใช้กับ LATERAL JOIN
SELECT c.customer_id, c.first_name, ro.*
FROM customers c
CROSS JOIN LATERAL fn_recent_orders_for_customer(c.customer_id, 3) ro
WHERE c.tier = 'Gold';
```

---

## 7. Inline Table-Valued Functions (SQL Server)

```sql
-- ตัวอย่างที่ 19: Inline TVF (SQL Server) - ประสิทธิภาพสูงสุด
CREATE FUNCTION fn_orders_by_customer(@CustomerId INT)
RETURNS TABLE
AS
RETURN (
    SELECT 
        o.order_id,
        o.order_date,
        o.total_amount,
        o.status,
        COUNT(od.product_id) AS item_count
    FROM orders o
    LEFT JOIN order_details od ON o.order_id = od.order_id
    WHERE o.customer_id = @CustomerId
    GROUP BY o.order_id, o.order_date, o.total_amount, o.status
);

-- ใช้งาน
SELECT * FROM fn_orders_by_customer(1001);

-- ตัวอย่างที่ 20: Multi-statement TVF (SQL Server)
CREATE FUNCTION fn_product_sales_history(@ProductId INT)
RETURNS @result TABLE(
    year INT,
    month INT,
    total_sold INT,
    revenue DECIMAL(15,2)
)
AS
BEGIN
    INSERT @result
    SELECT 
        YEAR(o.order_date),
        MONTH(o.order_date),
        SUM(od.quantity),
        SUM(od.quantity * od.unit_price)
    FROM order_details od
    JOIN orders o ON od.order_id = o.order_id
    WHERE od.product_id = @ProductId
      AND o.status = 'completed'
    GROUP BY YEAR(o.order_date), MONTH(o.order_date);
    
    RETURN;
END;

SELECT * FROM fn_product_sales_history(1) ORDER BY year, month;

-- ตัวอย่างที่ 21: SQL Server Scalar Function
CREATE FUNCTION fn_age_in_years(@BirthDate DATE)
RETURNS INT
AS
BEGIN
    RETURN DATEDIFF(YEAR, @BirthDate, GETDATE()) -
           CASE WHEN (MONTH(@BirthDate) > MONTH(GETDATE())) OR 
                     (MONTH(@BirthDate) = MONTH(GETDATE()) AND DAY(@BirthDate) > DAY(GETDATE()))
                THEN 1 ELSE 0 END;
END;

SELECT employee_id, first_name, dbo.fn_age_in_years(birth_date) AS age
FROM employees;
```

---

## 8. Aggregate Functions (PostgreSQL)

```sql
-- ตัวอย่างที่ 22: Custom Aggregate Function
-- สร้าง Geometric Mean aggregate

-- Step 1: สร้าง state transition function
CREATE OR REPLACE FUNCTION geometric_mean_transition(
    p_state FLOAT[],
    p_value FLOAT
)
RETURNS FLOAT[]
LANGUAGE plpgsql
IMMUTABLE
AS $$
BEGIN
    IF p_value > 0 THEN
        RETURN ARRAY[p_state[1] + LN(p_value), p_state[2] + 1];
    ELSE
        RETURN p_state;
    END IF;
END;
$$;

-- Step 2: สร้าง final function
CREATE OR REPLACE FUNCTION geometric_mean_final(p_state FLOAT[])
RETURNS FLOAT
LANGUAGE plpgsql
IMMUTABLE
AS $$
BEGIN
    IF p_state[2] = 0 THEN
        RETURN NULL;
    END IF;
    RETURN EXP(p_state[1] / p_state[2]);
END;
$$;

-- Step 3: สร้าง aggregate
CREATE AGGREGATE geometric_mean(FLOAT) (
    SFUNC = geometric_mean_transition,
    STYPE = FLOAT[],
    INITCOND = '{0, 0}',
    FINALFUNC = geometric_mean_final
);

-- ใช้งาน
SELECT department_id, geometric_mean(salary::FLOAT) AS geo_mean_salary
FROM employees GROUP BY department_id;

-- ตัวอย่างที่ 23: Simple Custom Aggregate (Product/Multiplication)
CREATE OR REPLACE FUNCTION product_sfunc(state NUMERIC, next NUMERIC)
RETURNS NUMERIC
LANGUAGE plpgsql
IMMUTABLE
AS $$
BEGIN
    IF next IS NULL THEN RETURN state; END IF;
    RETURN state * next;
END;
$$;

CREATE AGGREGATE product_agg(NUMERIC) (
    SFUNC = product_sfunc,
    STYPE = NUMERIC,
    INITCOND = '1'
);

-- คำนวณผลคูณ
SELECT product_agg(quantity) AS total_product FROM order_details WHERE order_id = 1;
```

---

## 9. Advanced Function Examples

```sql
-- ตัวอย่างที่ 24: Function ที่ใช้ JSON
CREATE OR REPLACE FUNCTION fn_order_summary_json(p_order_id INT)
RETURNS JSONB
LANGUAGE plpgsql
STABLE
AS $$
DECLARE v_result JSONB;
BEGIN
    SELECT jsonb_build_object(
        'order_id', o.order_id,
        'total', o.total_amount,
        'items_count', (SELECT COUNT(*) FROM order_details WHERE order_id = o.order_id),
        'items', (
            SELECT jsonb_agg(jsonb_build_object('product', p.product_name, 'qty', od.quantity))
            FROM order_details od JOIN products p ON od.product_id = p.product_id
            WHERE od.order_id = o.order_id
        )
    ) INTO v_result FROM orders o WHERE o.order_id = p_order_id;
    RETURN v_result;
END;
$$;

-- ตัวอย่างที่ 25: Polymorphic Function (PostgreSQL)
CREATE OR REPLACE FUNCTION fn_first_element(p_arr ANYARRAY)
RETURNS ANYELEMENT
LANGUAGE plpgsql
IMMUTABLE
AS $$
BEGIN
    IF array_length(p_arr, 1) IS NULL THEN
        RETURN NULL;
    END IF;
    RETURN p_arr[1];
END;
$$;

SELECT fn_first_element(ARRAY[3,1,4,1,5]) AS first_int;
SELECT fn_first_element(ARRAY['a','b','c']) AS first_text;

-- ตัวอย่างที่ 26: Function สำหรับ Pagination
CREATE OR REPLACE FUNCTION fn_paginate_orders(
    p_page INT DEFAULT 1,
    p_page_size INT DEFAULT 10,
    p_status TEXT DEFAULT NULL,
    OUT total_count BIGINT,
    OUT total_pages INT
)
RETURNS TABLE(
    order_id INT,
    order_date TIMESTAMP,
    customer_name TEXT,
    total_amount DECIMAL,
    status TEXT,
    page_total_count BIGINT,
    current_page INT
)
LANGUAGE plpgsql
STABLE
AS $$
BEGIN
    -- Get total count
    SELECT COUNT(*) INTO total_count
    FROM orders
    WHERE (p_status IS NULL OR status = p_status);
    
    total_pages := CEIL(total_count::FLOAT / p_page_size);
    
    RETURN QUERY
    SELECT 
        o.order_id,
        o.order_date,
        c.first_name || ' ' || c.last_name,
        o.total_amount,
        o.status,
        total_count,
        p_page
    FROM orders o
    JOIN customers c ON o.customer_id = c.customer_id
    WHERE (p_status IS NULL OR o.status = p_status)
    ORDER BY o.order_date DESC
    LIMIT p_page_size
    OFFSET (p_page - 1) * p_page_size;
END;
$$;

SELECT * FROM fn_paginate_orders(1, 10, 'pending');

-- ตัวอย่างที่ 27: Function สำหรับ Text Search
CREATE OR REPLACE FUNCTION fn_search_customers(
    p_query TEXT,
    p_limit INT DEFAULT 20
)
RETURNS TABLE(
    customer_id INT,
    full_name TEXT,
    email TEXT,
    phone TEXT,
    match_score FLOAT
)
LANGUAGE plpgsql
STABLE
AS $$
BEGIN
    RETURN QUERY
    SELECT 
        c.customer_id,
        c.first_name || ' ' || c.last_name,
        c.email,
        c.phone,
        GREATEST(
            similarity(c.first_name || ' ' || c.last_name, p_query),
            similarity(c.email, p_query),
            similarity(COALESCE(c.company_name, ''), p_query)
        ) AS match_score
    FROM customers c
    WHERE c.first_name || ' ' || c.last_name ILIKE '%' || p_query || '%'
       OR c.email ILIKE '%' || p_query || '%'
       OR c.company_name ILIKE '%' || p_query || '%'
    ORDER BY match_score DESC
    LIMIT p_limit;
END;
$$;

-- ตัวอย่างที่ 28: Function สำหรับ Period Comparison
CREATE OR REPLACE FUNCTION fn_compare_periods(
    p_metric TEXT,  -- 'revenue', 'orders', 'customers'
    p_period1_start DATE,
    p_period1_end DATE,
    p_period2_start DATE,
    p_period2_end DATE
)
RETURNS TABLE(
    metric TEXT,
    period1_value DECIMAL,
    period2_value DECIMAL,
    change_amount DECIMAL,
    change_pct DECIMAL
)
LANGUAGE plpgsql
STABLE
AS $$
DECLARE
    v_p1_value DECIMAL;
    v_p2_value DECIMAL;
BEGIN
    IF p_metric = 'revenue' THEN
        SELECT COALESCE(SUM(total_amount), 0) INTO v_p1_value
        FROM orders WHERE order_date BETWEEN p_period1_start AND p_period1_end AND status = 'completed';
        
        SELECT COALESCE(SUM(total_amount), 0) INTO v_p2_value
        FROM orders WHERE order_date BETWEEN p_period2_start AND p_period2_end AND status = 'completed';
        
    ELSIF p_metric = 'orders' THEN
        SELECT COUNT(*) INTO v_p1_value
        FROM orders WHERE order_date BETWEEN p_period1_start AND p_period1_end;
        
        SELECT COUNT(*) INTO v_p2_value
        FROM orders WHERE order_date BETWEEN p_period2_start AND p_period2_end;
        
    ELSIF p_metric = 'customers' THEN
        SELECT COUNT(DISTINCT customer_id) INTO v_p1_value
        FROM orders WHERE order_date BETWEEN p_period1_start AND p_period1_end;
        
        SELECT COUNT(DISTINCT customer_id) INTO v_p2_value
        FROM orders WHERE order_date BETWEEN p_period2_start AND p_period2_end;
    END IF;
    
    RETURN QUERY SELECT 
        p_metric,
        v_p1_value,
        v_p2_value,
        v_p2_value - v_p1_value,
        CASE WHEN v_p1_value = 0 THEN NULL
             ELSE (v_p2_value - v_p1_value) / v_p1_value * 100
        END;
END;
$$;

SELECT * FROM fn_compare_periods('revenue', '2023-01-01', '2023-12-31', '2024-01-01', '2024-12-31');

-- ตัวอย่างที่ 29: Function ที่ใช้ pl/sql ใน SQL
CREATE OR REPLACE FUNCTION fn_running_total(
    p_customer_id INT
)
RETURNS TABLE(
    order_id INT,
    order_date TIMESTAMP,
    amount DECIMAL,
    running_total DECIMAL
)
LANGUAGE sql
STABLE
AS $$
    SELECT 
        order_id,
        order_date,
        total_amount,
        SUM(total_amount) OVER (ORDER BY order_date ROWS UNBOUNDED PRECEDING) AS running_total
    FROM orders
    WHERE customer_id = p_customer_id AND status = 'completed'
    ORDER BY order_date;
$$;

-- ตัวอย่างที่ 30: Function ที่ใช้ SQL Language (simpler)
CREATE OR REPLACE FUNCTION fn_active_product_count()
RETURNS BIGINT
LANGUAGE sql
STABLE
AS $$
    SELECT COUNT(*) FROM products WHERE discontinued = 0;
$$;

SELECT fn_active_product_count() AS active_products;
```

---

## แบบฝึกหัด (Exercises)

**ข้อ 1:** สร้าง IMMUTABLE PostgreSQL Function ที่แปลง Celsius เป็น Fahrenheit

**คำตอบข้อ 1:**
```sql
CREATE OR REPLACE FUNCTION fn_celsius_to_fahrenheit(p_celsius DECIMAL)
RETURNS DECIMAL LANGUAGE sql IMMUTABLE AS $$
    SELECT (p_celsius * 9.0/5.0) + 32;
$$;
SELECT fn_celsius_to_fahrenheit(100) AS fahrenheit;  -- 212
```

**ข้อ 2:** สร้าง MySQL DETERMINISTIC Function ที่ format เบอร์โทรศัพท์ไทย

**คำตอบข้อ 2:**
```sql
DELIMITER //
CREATE FUNCTION fn_format_thai_phone(p_phone VARCHAR(20)) RETURNS VARCHAR(20) DETERMINISTIC NO SQL
BEGIN
    DECLARE v_clean VARCHAR(20);
    SET v_clean = REGEXP_REPLACE(p_phone, '[^0-9]', '');
    IF LENGTH(v_clean) = 10 THEN
        RETURN CONCAT(SUBSTRING(v_clean,1,3), '-', SUBSTRING(v_clean,4,3), '-', SUBSTRING(v_clean,7,4));
    END IF;
    RETURN v_clean;
END //
DELIMITER ;
SELECT fn_format_thai_phone('0812345678');
```

**ข้อ 3:** สร้าง PostgreSQL Table-Valued Function ที่ return paginated results

**คำตอบข้อ 3:**
```sql
CREATE OR REPLACE FUNCTION fn_products_page(p_page INT DEFAULT 1, p_size INT DEFAULT 10)
RETURNS TABLE(product_id INT, product_name TEXT, unit_price DECIMAL, total_rows BIGINT)
LANGUAGE sql STABLE AS $$
    SELECT p.product_id, p.product_name, p.unit_price, COUNT(*) OVER() AS total_rows
    FROM products p WHERE discontinued = 0
    ORDER BY product_name LIMIT p_size OFFSET (p_page-1)*p_size;
$$;
SELECT * FROM fn_products_page(1, 5);
```

**ข้อ 4:** สร้าง SQL Server Inline TVF สำหรับ filter orders

**คำตอบข้อ 4:**
```sql
CREATE FUNCTION fn_filtered_orders(@Status NVARCHAR(20), @MinAmount DECIMAL(10,2) = 0)
RETURNS TABLE AS RETURN (
    SELECT order_id, customer_id, order_date, total_amount, status
    FROM orders WHERE status = @Status AND total_amount >= @MinAmount
);
SELECT * FROM fn_filtered_orders('pending', 100);
```

**ข้อ 5:** สร้าง PostgreSQL Aggregate Function สำหรับหา Mode (ค่าที่พบมากที่สุด)

**คำตอบข้อ 5:**
```sql
-- Mode aggregate (simplified version)
CREATE OR REPLACE FUNCTION fn_mode_final(state ANYARRAY)
RETURNS ANYELEMENT LANGUAGE sql IMMUTABLE AS $$
    SELECT val FROM (SELECT val, COUNT(*) AS cnt FROM unnest(state) AS val GROUP BY val ORDER BY cnt DESC LIMIT 1) t;
$$;
-- Note: PostgreSQL has built-in MODE() in newer versions
SELECT MODE() WITHIN GROUP (ORDER BY status) FROM orders;
```

**ข้อ 6:** สร้าง Function ที่ validate Thai national ID

**คำตอบข้อ 6:**
```sql
CREATE OR REPLACE FUNCTION fn_validate_thai_id(p_id TEXT) RETURNS BOOLEAN LANGUAGE plpgsql IMMUTABLE AS $$
DECLARE v_sum INT := 0; v_i INT; v_check INT;
BEGIN
    IF LENGTH(p_id) != 13 THEN RETURN FALSE; END IF;
    IF p_id !~ '^[0-9]{13}$' THEN RETURN FALSE; END IF;
    FOR v_i IN 1..12 LOOP v_sum := v_sum + SUBSTRING(p_id, v_i, 1)::INT * (13 - v_i); END LOOP;
    v_check := (11 - (v_sum % 11)) % 10;
    RETURN v_check = SUBSTRING(p_id, 13, 1)::INT;
END; $$;
SELECT fn_validate_thai_id('1234567890123');
```

**ข้อ 7:** สร้าง Function ที่ generate random password

**คำตอบข้อ 7:**
```sql
CREATE OR REPLACE FUNCTION fn_generate_password(p_length INT DEFAULT 12) RETURNS TEXT LANGUAGE sql VOLATILE AS $$
    SELECT string_agg(substr('ABCDEFGHJKLMNPQRSTUVWXYZabcdefghjkmnpqrstuvwxyz23456789!@#$%', ceil(random()*60)::int, 1), '')
    FROM generate_series(1, p_length);
$$;
SELECT fn_generate_password(16) AS new_password;
```

**ข้อ 8:** สร้าง STABLE Function ที่ look up current exchange rate

**คำตอบข้อ 8:**
```sql
CREATE OR REPLACE FUNCTION fn_current_rate(p_from TEXT, p_to TEXT) RETURNS DECIMAL
LANGUAGE plpgsql STABLE AS $$
DECLARE v_rate DECIMAL;
BEGIN
    SELECT rate INTO v_rate FROM exchange_rates
    WHERE from_currency = p_from AND to_currency = p_to
    ORDER BY rate_date DESC LIMIT 1;
    RETURN COALESCE(v_rate, 1.0);
END; $$;
SELECT fn_current_rate('USD', 'THB') AS usd_to_thb;
```

**ข้อ 9:** ดูรายการ Functions ทั้งหมดใน PostgreSQL

**คำตอบข้อ 9:**
```sql
SELECT routine_name, routine_type, data_type AS return_type,
       external_language AS language
FROM information_schema.routines
WHERE routine_schema = 'public' AND routine_type = 'FUNCTION'
ORDER BY routine_name;

-- ดู function arguments
SELECT proname AS name, pg_get_function_arguments(oid) AS args,
       pg_get_function_result(oid) AS returns
FROM pg_proc WHERE pronamespace = 'public'::regnamespace
AND prokind = 'f' ORDER BY proname;
```

**ข้อ 10:** สร้าง Function ที่ normalize string (remove accents, lowercase)

**คำตอบข้อ 10:**
```sql
CREATE OR REPLACE FUNCTION fn_normalize_string(p_input TEXT) RETURNS TEXT
LANGUAGE plpgsql IMMUTABLE AS $$
BEGIN
    RETURN LOWER(UNACCENT(TRIM(REGEXP_REPLACE(p_input, '\s+', ' ', 'g'))));
END; $$;
-- ต้องมี extension unaccent
-- CREATE EXTENSION IF NOT EXISTS unaccent;
SELECT fn_normalize_string('  Héllo   WORLD  ') AS normalized;
```

---

*จบ Part 087: User-Defined Functions (UDFs)*
