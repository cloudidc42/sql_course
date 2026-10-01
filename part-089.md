# Part 089: Dynamic SQL (Dynamic SQL - SQL แบบไดนามิก)

## บทนำ

Dynamic SQL คือการสร้าง SQL statements ขณะ runtime แทนที่จะเขียนตายตัวใน code ช่วยให้สามารถสร้าง query ที่ยืดหยุ่นตาม parameter ที่รับมา เช่น ชื่อตาราง ชื่อ column หรือ conditions ที่ไม่ทราบล่วงหน้า

---

## ⚠️ คำเตือนด้านความปลอดภัย (CRITICAL SECURITY WARNING)

```
=====================================================================
WARNING: SQL INJECTION RISK
=====================================================================
Dynamic SQL มีความเสี่ยงสูงมากต่อ SQL Injection Attack
ถ้าใช้ user input โดยตรงใน Dynamic SQL โดยไม่ผ่าน sanitization
ผู้โจมตีสามารถ:
1. ดึงข้อมูลที่ไม่ได้รับอนุญาต
2. แก้ไขหรือลบข้อมูล
3. Execute arbitrary SQL commands
4. เข้าถึง system-level operations

RULE: ALWAYS use parameterized queries / bind variables
NEVER concatenate user input directly into SQL strings
=====================================================================
```

---

## 1. EXECUTE / EXEC Statement

```sql
-- ตัวอย่างที่ 1: PostgreSQL EXECUTE (Dynamic SQL)
DO $$
DECLARE
    v_table_name TEXT := 'employees';
    v_sql TEXT;
    v_count INT;
BEGIN
    v_sql := 'SELECT COUNT(*) FROM ' || quote_ident(v_table_name);
    EXECUTE v_sql INTO v_count;
    RAISE NOTICE 'Count: %', v_count;
END;
$$;

-- ตัวอย่างที่ 2: EXECUTE พร้อม USING clause (safe parameterization)
DO $$
DECLARE
    v_dept_id INT := 10;
    v_sql TEXT;
    v_count INT;
BEGIN
    v_sql := 'SELECT COUNT(*) FROM employees WHERE department_id = $1';
    EXECUTE v_sql INTO v_count USING v_dept_id;  -- Safe! ใช้ bind parameter
    RAISE NOTICE 'Count: %', v_count;
END;
$$;

-- ตัวอย่างที่ 3: EXECUTE ใน Function
CREATE OR REPLACE FUNCTION fn_count_rows(p_table_name TEXT)
RETURNS BIGINT
LANGUAGE plpgsql
AS $$
DECLARE
    v_count BIGINT;
BEGIN
    -- ตรวจสอบว่า table มีอยู่จริง (security check)
    IF NOT EXISTS (
        SELECT 1 FROM information_schema.tables
        WHERE table_schema = 'public'
          AND table_name = p_table_name
    ) THEN
        RAISE EXCEPTION 'Table % does not exist or is not accessible', p_table_name;
    END IF;
    
    -- quote_ident ป้องกัน SQL injection สำหรับ identifiers
    EXECUTE 'SELECT COUNT(*) FROM ' || quote_ident(p_table_name) INTO v_count;
    RETURN v_count;
END;
$$;

SELECT fn_count_rows('orders');
```

---

## 2. sp_executesql (SQL Server)

```sql
-- ตัวอย่างที่ 4: SQL Server sp_executesql (SAFE - parameterized)
DECLARE @sql NVARCHAR(MAX);
DECLARE @dept_id INT = 10;
DECLARE @count INT;

SET @sql = N'SELECT @count = COUNT(*) FROM employees WHERE department_id = @dept_id';

EXEC sp_executesql 
    @sql, 
    N'@dept_id INT, @count INT OUTPUT',  -- parameter definitions
    @dept_id = @dept_id,                 -- input parameter
    @count = @count OUTPUT;              -- output parameter

SELECT @count AS employee_count;

-- ตัวอย่างที่ 5: SQL Server sp_executesql กับ Multiple Parameters
DECLARE @sql NVARCHAR(MAX);
DECLARE @start_date DATE = '2024-01-01';
DECLARE @end_date DATE = '2024-12-31';
DECLARE @status NVARCHAR(20) = 'completed';

SET @sql = N'
    SELECT 
        COUNT(*) AS total_orders,
        SUM(total_amount) AS total_revenue
    FROM orders
    WHERE order_date BETWEEN @start AND @end
      AND status = @status';

EXEC sp_executesql 
    @sql,
    N'@start DATE, @end DATE, @status NVARCHAR(20)',
    @start = @start_date,
    @end = @end_date,
    @status = @status;
```

---

## 3. 🔴 UNSAFE Dynamic SQL - SQL Injection Examples

```sql
-- =====================================================================
-- UNSAFE PATTERNS - อย่าทำแบบนี้!
-- =====================================================================

-- ตัวอย่างที่ 6: 🔴 UNSAFE - String Concatenation โดยตรง (DANGEROUS!)
-- ตัวอย่างการ SQL Injection:
CREATE OR REPLACE FUNCTION fn_UNSAFE_search_products(p_name TEXT)
RETURNS TABLE(product_id INT, product_name TEXT)
LANGUAGE plpgsql
AS $$
DECLARE
    v_sql TEXT;
BEGIN
    -- 🔴 DANGEROUS! อย่าทำแบบนี้!
    v_sql := 'SELECT product_id, product_name FROM products WHERE product_name = ''' || p_name || '''';
    RETURN QUERY EXECUTE v_sql;
END;
$$;

-- ผู้โจมตีสามารถ call ด้วย:
-- fn_UNSAFE_search_products($$' OR '1'='1$$)
-- จะกลายเป็น: WHERE product_name = '' OR '1'='1'
-- ดึงข้อมูลทุก row!

-- ยิ่งกว่านั้น:
-- fn_UNSAFE_search_products($$'; DROP TABLE products; --$$)
-- จะกลายเป็น: WHERE product_name = ''; DROP TABLE products; --'
-- ลบตารางทั้งตาราง!

-- =====================================================================
-- SAFE PATTERNS - ควรทำแบบนี้!
-- =====================================================================

-- ตัวอย่างที่ 7: 🟢 SAFE - ใช้ USING clause (PostgreSQL)
CREATE OR REPLACE FUNCTION fn_SAFE_search_products(p_name TEXT)
RETURNS TABLE(product_id INT, product_name TEXT)
LANGUAGE plpgsql
AS $$
BEGIN
    -- 🟢 SAFE! ใช้ $1 พร้อม USING clause
    RETURN QUERY EXECUTE 
        'SELECT product_id, product_name FROM products WHERE product_name ILIKE $1'
        USING '%' || p_name || '%';
END;
$$;

-- ตัวอย่างที่ 8: 🟢 SAFE - ใช้ format() พร้อม %L สำหรับ literals
DO $$
DECLARE
    v_search TEXT := 'Widget';
    v_sql TEXT;
BEGIN
    -- %L = quote เป็น SQL literal (safe สำหรับ values)
    v_sql := format('SELECT * FROM products WHERE product_name ILIKE %L', '%' || v_search || '%');
    RAISE NOTICE 'SQL: %', v_sql;
END;
$$;
```

---

## 4. Safe Parameterization

```sql
-- ตัวอย่างที่ 9: 🟢 SAFE - Whitelist สำหรับ Column Names
CREATE OR REPLACE FUNCTION fn_safe_sort_products(p_sort_column TEXT, p_sort_dir TEXT DEFAULT 'ASC')
RETURNS TABLE(product_id INT, product_name TEXT, unit_price DECIMAL)
LANGUAGE plpgsql
AS $$
DECLARE
    v_allowed_columns TEXT[] := ARRAY['product_id', 'product_name', 'unit_price', 'units_in_stock'];
    v_allowed_dirs TEXT[] := ARRAY['ASC', 'DESC'];
    v_sql TEXT;
BEGIN
    -- Whitelist check - ป้องกัน injection ผ่าน column name
    IF NOT (p_sort_column = ANY(v_allowed_columns)) THEN
        RAISE EXCEPTION 'Invalid sort column: %. Allowed: %', 
            p_sort_column, array_to_string(v_allowed_columns, ', ');
    END IF;
    
    IF NOT (UPPER(p_sort_dir) = ANY(v_allowed_dirs)) THEN
        RAISE EXCEPTION 'Invalid sort direction: %. Use ASC or DESC', p_sort_dir;
    END IF;
    
    -- ตอนนี้ safe แล้ว ใช้ quote_ident สำหรับ column name
    v_sql := format(
        'SELECT product_id, product_name, unit_price FROM products WHERE discontinued = 0 ORDER BY %I %s',
        p_sort_column,    -- %I = quote_ident (safe for identifiers)
        UPPER(p_sort_dir) -- direction ผ่าน whitelist แล้ว
    );
    
    RETURN QUERY EXECUTE v_sql;
END;
$$;

-- Safe usage
SELECT * FROM fn_safe_sort_products('unit_price', 'DESC');

-- จะ ERROR เพราะ column ไม่อยู่ใน whitelist
-- SELECT * FROM fn_safe_sort_products('1; DROP TABLE products; --', 'ASC');

-- ตัวอย่างที่ 10: 🟢 SAFE - Parameterized Search (PostgreSQL)
CREATE OR REPLACE PROCEDURE sp_safe_search(
    p_table TEXT,
    p_column TEXT,
    p_value TEXT
)
LANGUAGE plpgsql
AS $$
DECLARE
    v_allowed_tables TEXT[] := ARRAY['products', 'customers', 'orders'];
    v_col_exists BOOLEAN;
BEGIN
    -- Validate table
    IF NOT (p_table = ANY(v_allowed_tables)) THEN
        RAISE EXCEPTION 'Access denied to table: %', p_table;
    END IF;
    
    -- Validate column exists in table
    SELECT EXISTS(
        SELECT 1 FROM information_schema.columns
        WHERE table_schema = 'public'
          AND table_name = p_table
          AND column_name = p_column
    ) INTO v_col_exists;
    
    IF NOT v_col_exists THEN
        RAISE EXCEPTION 'Column % does not exist in table %', p_column, p_table;
    END IF;
    
    -- Safe query using identifiers (not concatenation of user input into SQL directly)
    EXECUTE format(
        'SELECT * FROM %I WHERE CAST(%I AS TEXT) ILIKE $1',
        p_table,
        p_column
    ) USING '%' || p_value || '%';
END;
$$;

-- ตัวอย่างที่ 11: 🟢 SAFE - MySQL Prepared Statements
DELIMITER //

CREATE PROCEDURE sp_mysql_safe_search(
    IN p_column VARCHAR(64),
    IN p_value VARCHAR(255)
)
BEGIN
    DECLARE v_sql TEXT;
    
    -- Whitelist column names
    IF p_column NOT IN ('product_name', 'description', 'category_id') THEN
        SIGNAL SQLSTATE '45000' SET MESSAGE_TEXT = 'Invalid column name';
    END IF;
    
    -- ใช้ prepared statement กับ ? placeholder
    SET v_sql = CONCAT('SELECT product_id, product_name FROM products WHERE ', p_column, ' LIKE ?');
    
    SET @sql = v_sql;
    SET @val = CONCAT('%', p_value, '%');
    
    PREPARE stmt FROM @sql;
    EXECUTE stmt USING @val;  -- Safe! ค่าถูก escape อัตโนมัติ
    DEALLOCATE PREPARE stmt;
END //

DELIMITER ;
```

---

## 5. Building Dynamic Pivot Queries

```sql
-- ตัวอย่างที่ 12: Dynamic PIVOT (PostgreSQL)
-- ปัญหา: ต้องการ PIVOT data แต่ไม่รู้ล่วงหน้าว่ามีกี่ columns

CREATE OR REPLACE FUNCTION fn_dynamic_pivot(
    p_source_table TEXT,
    p_row_column TEXT,
    p_pivot_column TEXT,
    p_value_column TEXT
)
RETURNS TEXT  -- return SQL statement ที่สร้าง
LANGUAGE plpgsql
STABLE
AS $$
DECLARE
    v_pivot_values TEXT;
    v_sql TEXT;
    v_pivot_val TEXT;
BEGIN
    -- Validate inputs (security)
    IF p_source_table !~ '^[a-zA-Z_][a-zA-Z0-9_]*$' THEN
        RAISE EXCEPTION 'Invalid table name: %', p_source_table;
    END IF;
    
    -- Get distinct values for pivot column
    EXECUTE format(
        'SELECT string_agg(DISTINCT %I::TEXT, '','') FROM %I',
        p_pivot_column,
        p_source_table
    ) INTO v_pivot_values;
    
    -- Build column list
    SELECT string_agg(
        format('SUM(CASE WHEN %I = %L THEN %I END) AS %I',
            p_pivot_column,
            val,
            p_value_column,
            'col_' || val
        ),
        ', '
    )
    INTO v_sql
    FROM unnest(string_to_array(v_pivot_values, ',')) AS val;
    
    -- Return complete query
    RETURN format(
        'SELECT %I, %s FROM %I GROUP BY %I',
        p_row_column,
        v_sql,
        p_source_table,
        p_row_column
    );
END;
$$;

-- ตัวอย่างที่ 13: Dynamic PIVOT - PostgreSQL Crosstab
-- ใช้ tablefunc extension
CREATE EXTENSION IF NOT EXISTS tablefunc;

-- Dynamic category by month pivot
CREATE OR REPLACE FUNCTION fn_category_month_pivot(p_year INT)
RETURNS VOID
LANGUAGE plpgsql
AS $$
DECLARE
    v_categories TEXT;
    v_sql TEXT;
BEGIN
    -- Get category names
    SELECT string_agg(quote_ident(category_name), ',' ORDER BY category_name)
    INTO v_categories
    FROM categories;
    
    -- Build crosstab query
    v_sql := format($q$
        SELECT * FROM crosstab(
            'SELECT MONTH(o.order_date), c.category_name, SUM(od.quantity * od.unit_price)::DECIMAL
             FROM orders o
             JOIN order_details od ON o.order_id = od.order_id
             JOIN products p ON od.product_id = p.product_id
             JOIN categories c ON p.category_id = c.category_id
             WHERE YEAR(o.order_date) = %s AND o.status = ''completed''
             GROUP BY MONTH(o.order_date), c.category_name
             ORDER BY 1, 2',
            'SELECT DISTINCT category_name FROM categories ORDER BY 1'
        ) AS t(month INT, %s)
    $q$, p_year, v_categories);
    
    RAISE NOTICE 'Generated SQL: %', v_sql;
END;
$$;

-- ตัวอย่างที่ 14: SQL Server Dynamic PIVOT
CREATE PROCEDURE sp_dynamic_pivot_sales @Year INT
AS
BEGIN
    DECLARE @months NVARCHAR(MAX) = '';
    DECLARE @sql NVARCHAR(MAX);
    
    -- สร้าง column list
    SELECT @months += '[' + CAST(month_num AS NVARCHAR) + '],'
    FROM (SELECT DISTINCT MONTH(order_date) AS month_num FROM orders WHERE YEAR(order_date) = @Year) t
    ORDER BY month_num;
    
    SET @months = LEFT(@months, LEN(@months) - 1);  -- Remove last comma
    
    SET @sql = N'
        SELECT product_id, ' + @months + '
        FROM (
            SELECT od.product_id, MONTH(o.order_date) AS month_num, od.quantity
            FROM order_details od
            JOIN orders o ON od.order_id = o.order_id
            WHERE YEAR(o.order_date) = ' + CAST(@Year AS NVARCHAR) + '
        ) src
        PIVOT (SUM(quantity) FOR month_num IN (' + @months + ')) AS pvt';
    
    EXEC sp_executesql @sql;
END;
```

---

## 6. Dynamic Table/Column Names

```sql
-- ตัวอย่างที่ 15: 🟢 SAFE - Dynamic Table Names
CREATE OR REPLACE FUNCTION fn_get_table_info(p_table_name TEXT)
RETURNS TABLE(
    column_name TEXT,
    data_type TEXT,
    is_nullable TEXT
)
LANGUAGE plpgsql
AS $$
BEGIN
    -- Validate table exists (security check)
    IF NOT EXISTS (
        SELECT 1 FROM information_schema.tables
        WHERE table_schema = 'public'
          AND table_name = p_table_name
    ) THEN
        RAISE EXCEPTION 'Table % not found', p_table_name;
    END IF;
    
    RETURN QUERY
    SELECT c.column_name::TEXT, c.data_type::TEXT, c.is_nullable::TEXT
    FROM information_schema.columns c
    WHERE c.table_schema = 'public'
      AND c.table_name = p_table_name
    ORDER BY c.ordinal_position;
END;
$$;

-- ตัวอย่างที่ 16: Dynamic Schema-based Routing
CREATE OR REPLACE FUNCTION fn_route_query(
    p_client_id INT,
    p_query TEXT
)
RETURNS VOID
LANGUAGE plpgsql
AS $$
DECLARE
    v_schema TEXT;
BEGIN
    -- หา schema ของ client
    SELECT schema_name INTO v_schema
    FROM client_schemas
    WHERE client_id = p_client_id;
    
    IF v_schema IS NULL THEN
        RAISE EXCEPTION 'Unknown client: %', p_client_id;
    END IF;
    
    -- Set search path และรัน query
    EXECUTE 'SET LOCAL search_path TO ' || quote_ident(v_schema);
    EXECUTE p_query;
END;
$$;

-- ตัวอย่างที่ 17: Dynamic ALTER TABLE (Database Migration)
CREATE OR REPLACE PROCEDURE sp_add_column_if_not_exists(
    p_table TEXT,
    p_column TEXT,
    p_data_type TEXT
)
LANGUAGE plpgsql
AS $$
BEGIN
    -- ตรวจสอบว่า column มีอยู่แล้วหรือไม่
    IF NOT EXISTS (
        SELECT 1 FROM information_schema.columns
        WHERE table_schema = 'public'
          AND table_name = p_table
          AND column_name = p_column
    ) THEN
        EXECUTE format(
            'ALTER TABLE %I ADD COLUMN %I %s',
            p_table,
            p_column,
            p_data_type  -- data type ไม่ต้อง quote (ไม่ใช่ identifier หรือ literal)
        );
        RAISE NOTICE 'Added column % to table %', p_column, p_table;
    ELSE
        RAISE NOTICE 'Column % already exists in table %', p_column, p_table;
    END IF;
END;
$$;

CALL sp_add_column_if_not_exists('customers', 'loyalty_points', 'INT DEFAULT 0');
```

---

## 7. When to Use vs Avoid Dynamic SQL

```sql
-- ตัวอย่างที่ 18: 🟢 เหมาะกับ Dynamic SQL
-- 1. ไม่รู้ชื่อตาราง/คอลัมน์ล่วงหน้า
-- 2. สร้าง reports แบบ flexible
-- 3. Database migration scripts
-- 4. Multi-tenant queries
-- 5. Building query builders

-- ตัวอย่าง: Database maintenance across all tables
CREATE OR REPLACE PROCEDURE sp_analyze_all_tables()
LANGUAGE plpgsql
AS $$
DECLARE
    v_table RECORD;
BEGIN
    FOR v_table IN
        SELECT table_name
        FROM information_schema.tables
        WHERE table_schema = 'public'
          AND table_type = 'BASE TABLE'
    LOOP
        EXECUTE 'ANALYZE ' || quote_ident(v_table.table_name);
        RAISE NOTICE 'Analyzed table: %', v_table.table_name;
    END LOOP;
END;
$$;

-- ตัวอย่างที่ 19: 🔴 ไม่ควรใช้ Dynamic SQL เมื่อ
-- 1. สามารถเขียน Static SQL ได้
-- 2. WHERE condition ที่ value เปลี่ยนแปลง (ใช้ parameterized query แทน)
-- 3. ภายใน hot path (performance sensitive)

-- ❌ BAD: ไม่จำเป็นต้องใช้ Dynamic SQL
CREATE OR REPLACE FUNCTION fn_BAD_get_orders(p_customer_id INT)
RETURNS TABLE(order_id INT, total_amount DECIMAL)
LANGUAGE plpgsql
AS $$
DECLARE v_sql TEXT;
BEGIN
    -- ไม่จำเป็น! ใช้ static SQL ได้เลย
    v_sql := 'SELECT order_id, total_amount FROM orders WHERE customer_id = ' || p_customer_id;
    RETURN QUERY EXECUTE v_sql;
END;
$$;

-- ✅ BETTER: ใช้ Static SQL
CREATE OR REPLACE FUNCTION fn_GOOD_get_orders(p_customer_id INT)
RETURNS TABLE(order_id INT, total_amount DECIMAL)
LANGUAGE plpgsql
AS $$
BEGIN
    RETURN QUERY 
    SELECT o.order_id, o.total_amount 
    FROM orders o 
    WHERE o.customer_id = p_customer_id;
END;
$$;

-- ตัวอย่างที่ 20: Dynamic Query Builder (Safe implementation)
CREATE OR REPLACE FUNCTION fn_build_product_query(
    p_category_id INT DEFAULT NULL,
    p_min_price DECIMAL DEFAULT NULL,
    p_max_price DECIMAL DEFAULT NULL,
    p_in_stock BOOLEAN DEFAULT NULL,
    p_keyword TEXT DEFAULT NULL
)
RETURNS TABLE(product_id INT, product_name TEXT, unit_price DECIMAL, units_in_stock INT)
LANGUAGE plpgsql
AS $$
DECLARE
    v_conditions TEXT[] := ARRAY['discontinued = 0'];
    v_params TEXT[] := ARRAY[]::TEXT[];
    v_param_count INT := 0;
    v_sql TEXT;
    v_where TEXT;
BEGIN
    -- สร้าง conditions อย่าง safe
    IF p_category_id IS NOT NULL THEN
        v_param_count := v_param_count + 1;
        v_conditions := v_conditions || ('category_id = $' || v_param_count)::TEXT;
        v_params := v_params || p_category_id::TEXT;
    END IF;
    
    IF p_min_price IS NOT NULL THEN
        v_param_count := v_param_count + 1;
        v_conditions := v_conditions || ('unit_price >= $' || v_param_count)::TEXT;
        v_params := v_params || p_min_price::TEXT;
    END IF;
    
    IF p_max_price IS NOT NULL THEN
        v_param_count := v_param_count + 1;
        v_conditions := v_conditions || ('unit_price <= $' || v_param_count)::TEXT;
        v_params := v_params || p_max_price::TEXT;
    END IF;
    
    IF p_in_stock IS NOT NULL AND p_in_stock THEN
        v_conditions := v_conditions || 'units_in_stock > 0';
    END IF;
    
    IF p_keyword IS NOT NULL THEN
        v_param_count := v_param_count + 1;
        v_conditions := v_conditions || ('product_name ILIKE $' || v_param_count)::TEXT;
        v_params := v_params || ('%' || p_keyword || '%');
    END IF;
    
    v_where := array_to_string(v_conditions, ' AND ');
    v_sql := 'SELECT product_id, product_name, unit_price, units_in_stock FROM products WHERE ' || v_where;
    
    -- Execute with parameters
    IF v_param_count = 0 THEN
        RETURN QUERY EXECUTE v_sql;
    ELSIF v_param_count = 1 THEN
        RETURN QUERY EXECUTE v_sql USING v_params[1]::TEXT::ANYELEMENT;
    END IF;
    -- Note: ใน production ใช้ format() กับ USING ที่รองรับหลาย params
    
    RAISE NOTICE 'Executed: %', v_sql;
END;
$$;
```

---

## 8. Safe Pattern Summary

```sql
-- ตัวอย่างที่ 21: Summary ของ Safe vs Unsafe Patterns
-- =====================================================================
-- SAFE PATTERNS
-- =====================================================================

-- Pattern 1: ใช้ USING clause สำหรับ values
EXECUTE 'SELECT * FROM products WHERE category_id = $1' USING p_category_id;

-- Pattern 2: ใช้ quote_ident() สำหรับ identifiers (table/column names)
EXECUTE 'SELECT * FROM ' || quote_ident(p_table_name);

-- Pattern 3: ใช้ quote_literal() สำหรับ string literals
EXECUTE 'SELECT * FROM products WHERE name = ' || quote_literal(p_name);

-- Pattern 4: ใช้ format() function
EXECUTE format('SELECT * FROM %I WHERE %I = %L', p_table, p_column, p_value);
-- %I = quote_ident (identifiers)
-- %L = quote_literal (literals)
-- %s = no quoting (use only for known-safe values)

-- Pattern 5: Whitelist validation
IF p_column NOT IN ('id', 'name', 'price') THEN
    RAISE EXCEPTION 'Invalid column: %', p_column;
END IF;

-- =====================================================================
-- UNSAFE PATTERNS - NEVER USE WITH USER INPUT
-- =====================================================================

-- ❌ Direct concatenation
EXECUTE 'SELECT * FROM ' || p_table;  -- UNSAFE if p_table is user input

-- ❌ String interpolation without validation
v_sql := 'WHERE name = ''' || user_input || '''';  -- UNSAFE

-- ❌ MySQL string concat
SET @sql = CONCAT('SELECT * FROM ', p_table_name);  -- UNSAFE

-- ตัวอย่างที่ 22: PostgreSQL format() Reference
-- %s = เปลี่ยนเป็น string ไม่มี quoting
-- %I = identifier (table/column name) - จะถูก double-quote
-- %L = literal value - จะถูก single-quote และ escape
-- %% = literal percent sign

SELECT format('Hello %s!', 'World');          -- Hello World!
SELECT format('SELECT * FROM %I', 'my table'); -- SELECT * FROM "my table"
SELECT format('WHERE name = %L', "it's");      -- WHERE name = 'it''s'

-- ตัวอย่างที่ 23: SQL Server Safe Dynamic SQL
DECLARE @sql NVARCHAR(MAX);
DECLARE @params NVARCHAR(500);
DECLARE @dept INT = 10;
DECLARE @salary DECIMAL(10,2) = 50000;

-- ✅ SAFE: ใช้ parameterized query
SET @sql = N'SELECT * FROM employees WHERE department_id = @dept AND salary > @salary';
SET @params = N'@dept INT, @salary DECIMAL(10,2)';

EXEC sp_executesql @sql, @params, @dept = @dept, @salary = @salary;

-- ❌ UNSAFE: string concatenation
-- SET @sql = 'SELECT * FROM employees WHERE department_id = ' + CAST(@dept AS VARCHAR);
-- EXEC (@sql);  -- UNSAFE!
```

---

## 9. Advanced Dynamic SQL Examples

```sql
-- ตัวอย่างที่ 24: Dynamic Index Creation
CREATE OR REPLACE PROCEDURE sp_create_index_if_not_exists(
    p_table TEXT,
    p_columns TEXT[],  -- array of column names
    p_unique BOOLEAN DEFAULT FALSE
)
LANGUAGE plpgsql
AS $$
DECLARE
    v_index_name TEXT;
    v_col_list TEXT;
    v_sql TEXT;
BEGIN
    -- สร้างชื่อ index
    v_index_name := 'idx_' || p_table || '_' || array_to_string(p_columns, '_');
    
    -- ตรวจสอบว่ามี index แล้วหรือไม่
    IF EXISTS (SELECT 1 FROM pg_indexes WHERE tablename = p_table AND indexname = v_index_name) THEN
        RAISE NOTICE 'Index % already exists', v_index_name;
        RETURN;
    END IF;
    
    -- สร้าง column list (safe - ใช้ quote_ident)
    SELECT string_agg(quote_ident(col), ', ')
    INTO v_col_list
    FROM unnest(p_columns) AS col;
    
    -- สร้าง SQL
    v_sql := format(
        'CREATE %s INDEX %I ON %I (%s)',
        CASE WHEN p_unique THEN 'UNIQUE' ELSE '' END,
        v_index_name,
        p_table,
        v_col_list
    );
    
    EXECUTE v_sql;
    RAISE NOTICE 'Created index: %', v_index_name;
END;
$$;

CALL sp_create_index_if_not_exists('orders', ARRAY['customer_id', 'order_date'], FALSE);

-- ตัวอย่างที่ 25: Dynamic Statistics Query
CREATE OR REPLACE FUNCTION fn_table_statistics(p_table TEXT)
RETURNS TABLE(
    column_name TEXT,
    min_value TEXT,
    max_value TEXT,
    null_count BIGINT,
    distinct_count BIGINT
)
LANGUAGE plpgsql
AS $$
DECLARE
    v_col RECORD;
    v_sql TEXT;
    v_min TEXT;
    v_max TEXT;
    v_nulls BIGINT;
    v_distinct BIGINT;
BEGIN
    FOR v_col IN
        SELECT column_name, data_type
        FROM information_schema.columns
        WHERE table_schema = 'public'
          AND table_name = p_table
        ORDER BY ordinal_position
    LOOP
        -- Get stats for each column
        v_sql := format(
            'SELECT MIN(%I)::TEXT, MAX(%I)::TEXT, COUNT(*) FILTER (WHERE %I IS NULL), COUNT(DISTINCT %I) FROM %I',
            v_col.column_name, v_col.column_name, v_col.column_name, v_col.column_name, p_table
        );
        
        EXECUTE v_sql INTO v_min, v_max, v_nulls, v_distinct;
        
        column_name := v_col.column_name;
        min_value := v_min;
        max_value := v_max;
        null_count := v_nulls;
        distinct_count := v_distinct;
        RETURN NEXT;
    END LOOP;
END;
$$;

SELECT * FROM fn_table_statistics('products');
```

---

## แบบฝึกหัด (Exercises)

**ข้อ 1:** เขียน Safe dynamic SQL ที่รับชื่อ table และ return row count

**คำตอบข้อ 1:**
```sql
CREATE OR REPLACE FUNCTION fn_safe_row_count(p_table TEXT) RETURNS BIGINT LANGUAGE plpgsql AS $$
DECLARE v_count BIGINT;
BEGIN
    IF NOT EXISTS(SELECT 1 FROM information_schema.tables WHERE table_schema='public' AND table_name=p_table) THEN
        RAISE EXCEPTION 'Table % not found', p_table;
    END IF;
    EXECUTE 'SELECT COUNT(*) FROM ' || quote_ident(p_table) INTO v_count;
    RETURN v_count;
END; $$;
SELECT fn_safe_row_count('orders');
```

**ข้อ 2:** สร้าง SQL Server stored procedure ที่ใช้ sp_executesql อย่างปลอดภัย

**คำตอบข้อ 2:**
```sql
CREATE PROCEDURE sp_search_by_date @StartDate DATE, @EndDate DATE
AS BEGIN
    DECLARE @sql NVARCHAR(MAX) = N'SELECT * FROM orders WHERE order_date BETWEEN @s AND @e AND status = ''completed''';
    EXEC sp_executesql @sql, N'@s DATE, @e DATE', @s=@StartDate, @e=@EndDate;
END;
EXEC sp_search_by_date '2024-01-01', '2024-12-31';
```

**ข้อ 3:** สร้าง MySQL Procedure ที่ใช้ prepared statement กับ dynamic WHERE clause

**คำตอบข้อ 3:**
```sql
DELIMITER //
CREATE PROCEDURE sp_dynamic_filter(IN p_status VARCHAR(20), IN p_min_amount DECIMAL(10,2))
BEGIN
    SET @sql = 'SELECT order_id, total_amount, status FROM orders WHERE status = ? AND total_amount >= ?';
    PREPARE stmt FROM @sql;
    SET @s = p_status; SET @a = p_min_amount;
    EXECUTE stmt USING @s, @a;
    DEALLOCATE PREPARE stmt;
END //
DELIMITER ;
```

**ข้อ 4:** สร้าง Function ที่ generate Dynamic PIVOT query สำหรับ sales by month

**คำตอบข้อ 4:**
```sql
CREATE OR REPLACE FUNCTION fn_pivot_sales_sql(p_year INT) RETURNS TEXT LANGUAGE plpgsql AS $$
DECLARE v_cols TEXT; v_sql TEXT;
BEGIN
    SELECT string_agg(format('SUM(CASE WHEN m=%s THEN rev END) AS "Month_%s"', m, m), ', ')
    INTO v_cols FROM generate_series(1,12) m;
    v_sql := format('SELECT product_id, %s FROM (SELECT product_id, EXTRACT(MONTH FROM order_date)::INT AS m, SUM(total_amount) AS rev FROM orders JOIN order_details USING(order_id) WHERE EXTRACT(YEAR FROM order_date)=%s GROUP BY product_id, m) t GROUP BY product_id', v_cols, p_year);
    RETURN v_sql;
END; $$;
SELECT fn_pivot_sales_sql(2024);
```

**ข้อ 5:** อธิบายความแตกต่างระหว่าง quote_ident() และ quote_literal() พร้อมตัวอย่าง

**คำตอบข้อ 5:**
```sql
-- quote_ident: ใช้สำหรับ identifiers (table, column names) - double quoted
SELECT quote_ident('my table') AS result;       -- "my table"
SELECT quote_ident('MyColumn') AS result;       -- "MyColumn"

-- quote_literal: ใช้สำหรับ string values - single quoted with escaping
SELECT quote_literal('it''s fine') AS result;  -- 'it''s fine'
SELECT quote_literal('O''Brien') AS result;    -- 'O''Brien'

-- %I ใน format() = quote_ident
-- %L ใน format() = quote_literal
SELECT format('SELECT * FROM %I WHERE name = %L', 'my_table', "it's");
-- Result: SELECT * FROM my_table WHERE name = 'it''s'
```

**ข้อ 6:** สร้าง Procedure ที่ build dynamic ORDER BY clause อย่างปลอดภัย

**คำตอบข้อ 6:**
```sql
CREATE OR REPLACE FUNCTION fn_sorted_products(p_sort TEXT DEFAULT 'product_name', p_dir TEXT DEFAULT 'ASC')
RETURNS TABLE(product_id INT, product_name TEXT, unit_price DECIMAL) LANGUAGE plpgsql AS $$
DECLARE v_allowed TEXT[] := ARRAY['product_id','product_name','unit_price','units_in_stock'];
BEGIN
    IF NOT (p_sort = ANY(v_allowed)) THEN RAISE EXCEPTION 'Invalid column: %', p_sort; END IF;
    IF UPPER(p_dir) NOT IN ('ASC','DESC') THEN RAISE EXCEPTION 'Invalid direction: %', p_dir; END IF;
    RETURN QUERY EXECUTE format('SELECT product_id, product_name, unit_price FROM products ORDER BY %I %s', p_sort, p_dir);
END; $$;
```

**ข้อ 7:** สร้าง Procedure สำหรับ Database Schema Inspection

**คำตอบข้อ 7:**
```sql
CREATE OR REPLACE FUNCTION fn_describe_table(p_table TEXT)
RETURNS TABLE(column_name TEXT, data_type TEXT, nullable TEXT, default_val TEXT) LANGUAGE plpgsql AS $$
BEGIN
    RETURN QUERY SELECT c.column_name::TEXT, c.data_type::TEXT, c.is_nullable::TEXT, c.column_default::TEXT
    FROM information_schema.columns c WHERE table_schema='public' AND table_name=p_table ORDER BY ordinal_position;
END; $$;
SELECT * FROM fn_describe_table('orders');
```

**ข้อ 8:** อธิบายความเสี่ยงของ SQL Injection และวิธีป้องกัน

**คำตอบข้อ 8:**
```sql
-- ความเสี่ยง SQL Injection:
-- 1. Data breach - ดึงข้อมูลที่ไม่ได้รับอนุญาต
-- 2. Data manipulation - แก้ไข/ลบข้อมูล  
-- 3. System compromise - รัน system commands

-- วิธีป้องกัน:
-- 1. ใช้ parameterized queries (USING clause, ?, @param)
-- 2. ใช้ Whitelist validation สำหรับ identifiers
-- 3. ใช้ quote_ident()/quote_literal() ไม่ใช่ string concat
-- 4. Principle of least privilege
-- 5. Input validation และ sanitization

-- Safe example:
EXECUTE 'SELECT * FROM orders WHERE customer_id = $1' USING p_customer_id;
-- NOT: EXECUTE 'SELECT * FROM orders WHERE customer_id = ' || p_customer_id;
```

**ข้อ 9:** สร้าง Function ที่ copy structure ของตารางหนึ่งไปสร้างตารางใหม่

**คำตอบข้อ 9:**
```sql
CREATE OR REPLACE PROCEDURE sp_clone_table_structure(p_source TEXT, p_dest TEXT)
LANGUAGE plpgsql AS $$
BEGIN
    IF EXISTS(SELECT 1 FROM information_schema.tables WHERE table_name = p_dest AND table_schema = 'public') THEN
        RAISE EXCEPTION 'Table % already exists', p_dest;
    END IF;
    EXECUTE format('CREATE TABLE %I (LIKE %I INCLUDING DEFAULTS INCLUDING CONSTRAINTS)', p_dest, p_source);
    RAISE NOTICE 'Created table % from structure of %', p_dest, p_source;
END; $$;
CALL sp_clone_table_structure('orders', 'orders_2024');
```

**ข้อ 10:** สร้าง Dynamic WHERE clause builder ที่ safe จาก map ของ filters

**คำตอบข้อ 10:**
```sql
CREATE OR REPLACE FUNCTION fn_safe_filter_query(p_filters JSONB)
RETURNS TABLE(product_id INT, product_name TEXT, unit_price DECIMAL) LANGUAGE plpgsql AS $$
DECLARE v_where TEXT := 'discontinued = 0'; v_params TEXT[]; v_n INT := 0;
        v_key TEXT; v_val TEXT;
BEGIN
    FOR v_key, v_val IN SELECT * FROM jsonb_each_text(p_filters) LOOP
        IF v_key IN ('category_id','supplier_id') THEN
            v_n := v_n + 1;
            v_where := v_where || format(' AND %I = $%s', v_key, v_n);
            v_params := v_params || v_val;
        END IF;
    END LOOP;
    IF v_n = 0 THEN RETURN QUERY EXECUTE 'SELECT product_id,product_name,unit_price FROM products WHERE '||v_where;
    ELSIF v_n = 1 THEN RETURN QUERY EXECUTE 'SELECT product_id,product_name,unit_price FROM products WHERE '||v_where USING v_params[1]::INT;
    END IF;
END; $$;
SELECT * FROM fn_safe_filter_query('{"category_id": "1"}'::JSONB);
```

---

*จบ Part 089: Dynamic SQL*
