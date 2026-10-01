# Part 088: Triggers (Triggers - ตัวกระตุ้น)

## บทนำ

Triggers คือ code ที่ถูก execute อัตโนมัติเมื่อมีเหตุการณ์ที่กำหนดเกิดขึ้นในฐานข้อมูล เช่น การ INSERT, UPDATE, DELETE บน table Triggers ช่วยให้สามารถ enforce business rules, maintain audit logs และ maintain data integrity ได้โดยอัตโนมัติ

---

## 1. What Triggers Do (Triggers ทำอะไร)

```sql
-- Triggers ใช้สำหรับ:
-- 1. Audit Logging - บันทึกการเปลี่ยนแปลงข้อมูล
-- 2. Data Validation - ตรวจสอบข้อมูลที่ซับซ้อน
-- 3. Automatic Calculations - คำนวณค่าอัตโนมัติ
-- 4. Cascading Changes - เปลี่ยนแปลงข้อมูลในตารางอื่น
-- 5. Enforcing Business Rules - บังคับกฎทางธุรกิจ

-- ตัวอย่างที่ 1: Trigger สำหรับ Audit Logging
-- สร้างตาราง audit ก่อน
CREATE TABLE employee_audit (
    audit_id SERIAL PRIMARY KEY,
    employee_id INT,
    action VARCHAR(10),  -- INSERT, UPDATE, DELETE
    changed_by TEXT,
    changed_at TIMESTAMP DEFAULT NOW(),
    old_first_name VARCHAR(50),
    new_first_name VARCHAR(50),
    old_salary DECIMAL(10,2),
    new_salary DECIMAL(10,2),
    old_department_id INT,
    new_department_id INT
);
```

---

## 2. BEFORE/AFTER/INSTEAD OF Triggers

```sql
-- ตัวอย่างที่ 2: BEFORE Trigger - รัน ก่อน การเปลี่ยนแปลงข้อมูล
-- ใช้สำหรับ: validation, data transformation ก่อน insert/update

-- PostgreSQL
CREATE OR REPLACE FUNCTION fn_before_employee_insert()
RETURNS TRIGGER
LANGUAGE plpgsql
AS $$
BEGIN
    -- Validate email
    IF NEW.email !~ '^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}$' THEN
        RAISE EXCEPTION 'Invalid email format: %', NEW.email;
    END IF;
    
    -- Auto-format name
    NEW.first_name := TRIM(INITCAP(NEW.first_name));
    NEW.last_name := TRIM(INITCAP(NEW.last_name));
    
    -- Set defaults
    NEW.created_at := NOW();
    NEW.status := COALESCE(NEW.status, 'active');
    
    RETURN NEW;  -- ต้อง RETURN NEW ใน BEFORE trigger
END;
$$;

CREATE TRIGGER trg_before_employee_insert
BEFORE INSERT ON employees
FOR EACH ROW
EXECUTE FUNCTION fn_before_employee_insert();

-- ตัวอย่างที่ 3: AFTER Trigger - รัน หลัง การเปลี่ยนแปลง
-- ใช้สำหรับ: logging, cascading changes, sending notifications

CREATE OR REPLACE FUNCTION fn_after_order_update()
RETURNS TRIGGER
LANGUAGE plpgsql
AS $$
BEGIN
    -- Log status changes
    IF OLD.status != NEW.status THEN
        INSERT INTO order_status_history (order_id, old_status, new_status, changed_at, changed_by)
        VALUES (NEW.order_id, OLD.status, NEW.status, NOW(), current_user);
        
        -- ถ้า order ถูก complete ให้คำนวณ rewards points
        IF NEW.status = 'completed' THEN
            UPDATE customers
            SET reward_points = reward_points + FLOOR(NEW.total_amount / 100)
            WHERE customer_id = NEW.customer_id;
        END IF;
        
        -- ถ้า order ถูก cancel คืน stock
        IF NEW.status = 'cancelled' AND OLD.status != 'cancelled' THEN
            UPDATE products p
            SET p.units_in_stock = p.units_in_stock + od.quantity
            FROM order_details od
            WHERE od.order_id = NEW.order_id
              AND p.product_id = od.product_id;
        END IF;
    END IF;
    
    RETURN NEW;
END;
$$;

CREATE TRIGGER trg_after_order_update
AFTER UPDATE ON orders
FOR EACH ROW
EXECUTE FUNCTION fn_after_order_update();

-- ตัวอย่างที่ 4: INSTEAD OF Trigger (บน View)
-- ดูตัวอย่างใน Part 082
```

---

## 3. Row-Level vs Statement-Level Triggers

```sql
-- ตัวอย่างที่ 5: Row-Level Trigger (FOR EACH ROW)
-- รันสำหรับทุก row ที่ถูก affect

CREATE OR REPLACE FUNCTION fn_track_price_change()
RETURNS TRIGGER
LANGUAGE plpgsql
AS $$
BEGIN
    IF OLD.unit_price != NEW.unit_price THEN
        INSERT INTO price_history (product_id, old_price, new_price, changed_at, changed_by)
        VALUES (NEW.product_id, OLD.unit_price, NEW.unit_price, NOW(), current_user);
    END IF;
    RETURN NEW;
END;
$$;

CREATE TRIGGER trg_track_price_change
AFTER UPDATE OF unit_price ON products
FOR EACH ROW  -- Row-level
EXECUTE FUNCTION fn_track_price_change();

-- ตัวอย่างที่ 6: Statement-Level Trigger (FOR EACH STATEMENT)
-- รันครั้งเดียวต่อ SQL statement (แม้จะ affect หลาย rows)

CREATE OR REPLACE FUNCTION fn_log_bulk_insert()
RETURNS TRIGGER
LANGUAGE plpgsql
AS $$
BEGIN
    INSERT INTO operation_log (table_name, operation, affected_at, performed_by)
    VALUES (TG_TABLE_NAME, TG_OP, NOW(), current_user);
    
    RETURN NULL;  -- Statement-level trigger return NULL
END;
$$;

CREATE TRIGGER trg_log_bulk_order_insert
AFTER INSERT ON orders
FOR EACH STATEMENT  -- Statement-level
EXECUTE FUNCTION fn_log_bulk_insert();
```

---

## 4. CREATE TRIGGER Syntax (PostgreSQL)

```sql
-- ตัวอย่างที่ 7: Trigger Syntax ทุกแบบ
-- BEFORE INSERT
CREATE TRIGGER trg_name
BEFORE INSERT ON table_name
FOR EACH ROW
EXECUTE FUNCTION fn_function_name();

-- AFTER UPDATE on specific columns
CREATE TRIGGER trg_name
AFTER UPDATE OF column1, column2 ON table_name
FOR EACH ROW
WHEN (OLD.column1 IS DISTINCT FROM NEW.column1)  -- Condition
EXECUTE FUNCTION fn_function_name();

-- BEFORE DELETE
CREATE TRIGGER trg_name
BEFORE DELETE ON table_name
FOR EACH ROW
EXECUTE FUNCTION fn_function_name();

-- ตัวอย่างที่ 8: ใช้ WHEN Clause เพื่อจำกัดเงื่อนไข
CREATE OR REPLACE FUNCTION fn_audit_salary_change()
RETURNS TRIGGER
LANGUAGE plpgsql
AS $$
BEGIN
    INSERT INTO salary_audit (employee_id, old_salary, new_salary, pct_change, changed_at)
    VALUES (
        NEW.employee_id,
        OLD.salary,
        NEW.salary,
        ((NEW.salary - OLD.salary) / OLD.salary * 100),
        NOW()
    );
    RETURN NEW;
END;
$$;

-- Trigger ที่รันเฉพาะเมื่อ salary เปลี่ยนแปลงมากกว่า 10%
CREATE TRIGGER trg_significant_salary_change
AFTER UPDATE OF salary ON employees
FOR EACH ROW
WHEN (ABS(NEW.salary - OLD.salary) / OLD.salary > 0.1)  -- มากกว่า 10%
EXECUTE FUNCTION fn_audit_salary_change();
```

---

## 5. Trigger Functions in PostgreSQL

```sql
-- ตัวอย่างที่ 9: Special Variables ใน Trigger Functions
CREATE OR REPLACE FUNCTION fn_demo_trigger_variables()
RETURNS TRIGGER
LANGUAGE plpgsql
AS $$
BEGIN
    -- TG_NAME: ชื่อ trigger
    RAISE NOTICE 'Trigger name: %', TG_NAME;
    
    -- TG_WHEN: BEFORE, AFTER, INSTEAD OF
    RAISE NOTICE 'When: %', TG_WHEN;
    
    -- TG_OP: INSERT, UPDATE, DELETE, TRUNCATE
    RAISE NOTICE 'Operation: %', TG_OP;
    
    -- TG_TABLE_NAME: ชื่อตาราง
    RAISE NOTICE 'Table: %', TG_TABLE_NAME;
    
    -- TG_TABLE_SCHEMA: schema ของตาราง
    RAISE NOTICE 'Schema: %', TG_TABLE_SCHEMA;
    
    -- TG_LEVEL: ROW หรือ STATEMENT
    RAISE NOTICE 'Level: %', TG_LEVEL;
    
    -- NEW: new row (สำหรับ INSERT/UPDATE)
    -- OLD: old row (สำหรับ UPDATE/DELETE)
    IF TG_OP = 'INSERT' THEN
        RAISE NOTICE 'Inserted: %', NEW.*;
        RETURN NEW;
    ELSIF TG_OP = 'UPDATE' THEN
        RAISE NOTICE 'Updated: % -> %', OLD.*, NEW.*;
        RETURN NEW;
    ELSIF TG_OP = 'DELETE' THEN
        RAISE NOTICE 'Deleted: %', OLD.*;
        RETURN OLD;
    END IF;
    
    RETURN NULL;
END;
$$;
```

---

## 6. MySQL Trigger Syntax

```sql
-- ตัวอย่างที่ 10: MySQL BEFORE INSERT Trigger
DELIMITER //

CREATE TRIGGER trg_before_product_insert
BEFORE INSERT ON products
FOR EACH ROW
BEGIN
    -- Auto-set created_at
    SET NEW.created_at = NOW();
    
    -- Validate price
    IF NEW.unit_price <= 0 THEN
        SIGNAL SQLSTATE '45000'
        SET MESSAGE_TEXT = 'Unit price must be greater than 0';
    END IF;
    
    -- Default discontinued to 0
    SET NEW.discontinued = COALESCE(NEW.discontinued, 0);
END //

DELIMITER ;

-- ตัวอย่างที่ 11: MySQL AFTER INSERT Trigger
DELIMITER //

CREATE TRIGGER trg_after_order_insert
AFTER INSERT ON orders
FOR EACH ROW
BEGIN
    -- Update customer's last order date
    UPDATE customers
    SET last_order_date = NEW.order_date,
        total_orders = total_orders + 1
    WHERE customer_id = NEW.customer_id;
    
    -- Insert into order tracking
    INSERT INTO order_tracking (order_id, status, updated_at)
    VALUES (NEW.order_id, 'created', NOW());
END //

DELIMITER ;

-- ตัวอย่างที่ 12: MySQL BEFORE UPDATE Trigger
DELIMITER //

CREATE TRIGGER trg_before_employee_update
BEFORE UPDATE ON employees
FOR EACH ROW
BEGIN
    -- ป้องกัน salary ลดลงมากกว่า 20%
    IF NEW.salary < OLD.salary * 0.8 THEN
        SIGNAL SQLSTATE '45000'
        SET MESSAGE_TEXT = 'Cannot reduce salary by more than 20%';
    END IF;
    
    -- Auto-update modified_at
    SET NEW.updated_at = NOW();
    SET NEW.updated_by = USER();
END //

DELIMITER ;

-- ตัวอย่างที่ 13: MySQL AFTER DELETE Trigger
DELIMITER //

CREATE TRIGGER trg_after_product_delete
AFTER DELETE ON products
FOR EACH ROW
BEGIN
    -- Archive deleted product
    INSERT INTO products_deleted (
        product_id, product_name, unit_price, deleted_at, deleted_by
    ) VALUES (
        OLD.product_id, OLD.product_name, OLD.unit_price, NOW(), USER()
    );
END //

DELIMITER ;
```

---

## 7. Audit Logging with Triggers

```sql
-- ตัวอย่างที่ 14: Comprehensive Audit Table
CREATE TABLE data_audit_log (
    audit_id BIGSERIAL PRIMARY KEY,
    table_name TEXT NOT NULL,
    record_id TEXT,  -- Primary key value as text
    operation CHAR(1) NOT NULL,  -- I=Insert, U=Update, D=Delete
    old_data JSONB,  -- OLD row as JSON
    new_data JSONB,  -- NEW row as JSON
    changed_fields TEXT[],  -- list of changed column names
    changed_by TEXT,
    changed_at TIMESTAMP DEFAULT NOW(),
    session_id TEXT,
    application_name TEXT
);

CREATE INDEX idx_audit_table_name ON data_audit_log(table_name);
CREATE INDEX idx_audit_record_id ON data_audit_log(record_id);
CREATE INDEX idx_audit_changed_at ON data_audit_log(changed_at);

-- ตัวอย่างที่ 15: Generic Audit Trigger Function
CREATE OR REPLACE FUNCTION fn_generic_audit()
RETURNS TRIGGER
LANGUAGE plpgsql
SECURITY DEFINER  -- รันด้วยสิทธิ์ของ function creator
AS $$
DECLARE
    v_record_id TEXT;
    v_old_data JSONB;
    v_new_data JSONB;
    v_changed_fields TEXT[];
    v_key TEXT;
BEGIN
    -- Get primary key value
    IF TG_OP = 'DELETE' THEN
        v_record_id := row_to_json(OLD)->>'id';
        v_old_data := row_to_json(OLD)::JSONB;
        v_new_data := NULL;
    ELSIF TG_OP = 'INSERT' THEN
        v_record_id := row_to_json(NEW)->>'id';
        v_old_data := NULL;
        v_new_data := row_to_json(NEW)::JSONB;
    ELSE  -- UPDATE
        v_record_id := row_to_json(NEW)->>'id';
        v_old_data := row_to_json(OLD)::JSONB;
        v_new_data := row_to_json(NEW)::JSONB;
        
        -- หา columns ที่เปลี่ยน
        SELECT ARRAY_AGG(key) INTO v_changed_fields
        FROM jsonb_each(v_new_data) AS new_row(key, value)
        WHERE value IS DISTINCT FROM v_old_data->key;
    END IF;
    
    INSERT INTO data_audit_log (
        table_name, record_id, operation,
        old_data, new_data, changed_fields,
        changed_by, session_id, application_name
    ) VALUES (
        TG_TABLE_NAME,
        v_record_id,
        LEFT(TG_OP, 1),  -- I, U, D
        v_old_data,
        v_new_data,
        v_changed_fields,
        current_user,
        pg_backend_pid()::TEXT,
        current_setting('application_name', TRUE)
    );
    
    RETURN CASE WHEN TG_OP = 'DELETE' THEN OLD ELSE NEW END;
END;
$$;

-- Apply audit trigger to multiple tables
CREATE TRIGGER trg_audit_employees
AFTER INSERT OR UPDATE OR DELETE ON employees
FOR EACH ROW EXECUTE FUNCTION fn_generic_audit();

CREATE TRIGGER trg_audit_orders
AFTER INSERT OR UPDATE OR DELETE ON orders
FOR EACH ROW EXECUTE FUNCTION fn_generic_audit();

CREATE TRIGGER trg_audit_customers
AFTER INSERT OR UPDATE OR DELETE ON customers
FOR EACH ROW EXECUTE FUNCTION fn_generic_audit();
```

---

## 8. Automatic Timestamp Updates

```sql
-- ตัวอย่างที่ 16: Auto-update updated_at (PostgreSQL)
CREATE OR REPLACE FUNCTION fn_update_timestamp()
RETURNS TRIGGER
LANGUAGE plpgsql
AS $$
BEGIN
    NEW.updated_at := NOW();
    RETURN NEW;
END;
$$;

-- Apply to multiple tables
CREATE TRIGGER trg_update_timestamp_products
BEFORE UPDATE ON products
FOR EACH ROW EXECUTE FUNCTION fn_update_timestamp();

CREATE TRIGGER trg_update_timestamp_customers
BEFORE UPDATE ON customers
FOR EACH ROW EXECUTE FUNCTION fn_update_timestamp();

CREATE TRIGGER trg_update_timestamp_orders
BEFORE UPDATE ON orders
FOR EACH ROW EXECUTE FUNCTION fn_update_timestamp();

-- ตัวอย่างที่ 17: Automatic created_at and updated_at
CREATE OR REPLACE FUNCTION fn_set_timestamps()
RETURNS TRIGGER
LANGUAGE plpgsql
AS $$
BEGIN
    IF TG_OP = 'INSERT' THEN
        NEW.created_at := NOW();
        NEW.updated_at := NOW();
    ELSIF TG_OP = 'UPDATE' THEN
        -- ป้องกัน overwrite created_at
        NEW.created_at := OLD.created_at;
        NEW.updated_at := NOW();
    END IF;
    RETURN NEW;
END;
$$;
```

---

## 9. Enforcing Complex Constraints

```sql
-- ตัวอย่างที่ 18: Trigger ตรวจสอบ Business Rules
CREATE OR REPLACE FUNCTION fn_validate_order_constraint()
RETURNS TRIGGER
LANGUAGE plpgsql
AS $$
DECLARE
    v_customer_credit_limit DECIMAL;
    v_customer_outstanding DECIMAL;
BEGIN
    -- ตรวจสอบ credit limit
    SELECT credit_limit INTO v_customer_credit_limit
    FROM customers WHERE customer_id = NEW.customer_id;
    
    SELECT COALESCE(SUM(total_amount), 0) INTO v_customer_outstanding
    FROM orders
    WHERE customer_id = NEW.customer_id
      AND status IN ('pending', 'processing')
      AND order_id != COALESCE(NEW.order_id, -1);  -- ไม่นับ current order
    
    IF (v_customer_outstanding + NEW.total_amount) > v_customer_credit_limit THEN
        RAISE EXCEPTION 'Credit limit exceeded. Limit: %, Outstanding: %, New Order: %',
            v_customer_credit_limit, v_customer_outstanding, NEW.total_amount;
    END IF;
    
    -- ตรวจสอบ minimum order amount
    IF NEW.total_amount < 100 THEN
        RAISE EXCEPTION 'Minimum order amount is 100, got: %', NEW.total_amount;
    END IF;
    
    RETURN NEW;
END;
$$;

CREATE TRIGGER trg_validate_order
BEFORE INSERT OR UPDATE ON orders
FOR EACH ROW
EXECUTE FUNCTION fn_validate_order_constraint();

-- ตัวอย่างที่ 19: Trigger ป้องกัน Circular References
CREATE OR REPLACE FUNCTION fn_prevent_circular_manager()
RETURNS TRIGGER
LANGUAGE plpgsql
AS $$
DECLARE
    v_current_manager INT;
    v_depth INT := 0;
    v_max_depth CONSTANT INT := 20;
BEGIN
    -- ตรวจสอบ circular reference ใน manager hierarchy
    v_current_manager := NEW.manager_id;
    
    WHILE v_current_manager IS NOT NULL AND v_depth < v_max_depth LOOP
        IF v_current_manager = NEW.employee_id THEN
            RAISE EXCEPTION 'Circular reference detected in manager hierarchy for employee %', NEW.employee_id;
        END IF;
        
        SELECT manager_id INTO v_current_manager
        FROM employees WHERE employee_id = v_current_manager;
        
        v_depth := v_depth + 1;
    END LOOP;
    
    RETURN NEW;
END;
$$;

CREATE TRIGGER trg_prevent_circular_manager
BEFORE INSERT OR UPDATE OF manager_id ON employees
FOR EACH ROW
WHEN (NEW.manager_id IS NOT NULL)
EXECUTE FUNCTION fn_prevent_circular_manager();

-- ตัวอย่างที่ 20: Trigger สำหรับ Derived Values
CREATE OR REPLACE FUNCTION fn_calculate_order_total()
RETURNS TRIGGER
LANGUAGE plpgsql
AS $$
BEGIN
    -- Update order total เมื่อ order_details เปลี่ยน
    UPDATE orders
    SET 
        total_amount = (
            SELECT COALESCE(SUM(quantity * unit_price * (1 - COALESCE(discount, 0))), 0)
            FROM order_details
            WHERE order_id = COALESCE(NEW.order_id, OLD.order_id)
        ),
        updated_at = NOW()
    WHERE order_id = COALESCE(NEW.order_id, OLD.order_id);
    
    RETURN COALESCE(NEW, OLD);
END;
$$;

CREATE TRIGGER trg_update_order_total
AFTER INSERT OR UPDATE OR DELETE ON order_details
FOR EACH ROW
EXECUTE FUNCTION fn_calculate_order_total();
```

---

## 10. More Complex Trigger Examples

```sql
-- ตัวอย่างที่ 21: Trigger สำหรับ Inventory Management
CREATE OR REPLACE FUNCTION fn_manage_inventory()
RETURNS TRIGGER
LANGUAGE plpgsql
AS $$
BEGIN
    IF TG_OP = 'INSERT' THEN
        -- ลด stock เมื่อ order detail ถูกเพิ่ม
        UPDATE products
        SET units_in_stock = units_in_stock - NEW.quantity
        WHERE product_id = NEW.product_id;
        
        -- ตรวจสอบ reorder level
        IF (SELECT units_in_stock FROM products WHERE product_id = NEW.product_id) 
           <= (SELECT reorder_level FROM products WHERE product_id = NEW.product_id) THEN
            INSERT INTO reorder_alerts (product_id, alert_date, current_stock)
            SELECT product_id, NOW(), units_in_stock
            FROM products WHERE product_id = NEW.product_id;
        END IF;
        
    ELSIF TG_OP = 'UPDATE' THEN
        -- ปรับ stock ตาม quantity ที่เปลี่ยน
        UPDATE products
        SET units_in_stock = units_in_stock - (NEW.quantity - OLD.quantity)
        WHERE product_id = NEW.product_id;
        
    ELSIF TG_OP = 'DELETE' THEN
        -- คืน stock เมื่อ order detail ถูกลบ
        UPDATE products
        SET units_in_stock = units_in_stock + OLD.quantity
        WHERE product_id = OLD.product_id;
    END IF;
    
    RETURN COALESCE(NEW, OLD);
END;
$$;

CREATE TRIGGER trg_manage_inventory
AFTER INSERT OR UPDATE OR DELETE ON order_details
FOR EACH ROW
EXECUTE FUNCTION fn_manage_inventory();

-- ตัวอย่างที่ 22: Trigger สำหรับ Version Control
CREATE TABLE product_versions (
    version_id SERIAL PRIMARY KEY,
    product_id INT,
    version_number INT,
    product_name TEXT,
    unit_price DECIMAL,
    description TEXT,
    effective_from TIMESTAMP,
    effective_to TIMESTAMP,
    is_current BOOLEAN DEFAULT TRUE
);

CREATE OR REPLACE FUNCTION fn_version_product()
RETURNS TRIGGER
LANGUAGE plpgsql
AS $$
DECLARE
    v_version_num INT;
BEGIN
    -- Get next version number
    SELECT COALESCE(MAX(version_number), 0) + 1 INTO v_version_num
    FROM product_versions WHERE product_id = NEW.product_id;
    
    -- Close previous version
    UPDATE product_versions
    SET effective_to = NOW(), is_current = FALSE
    WHERE product_id = NEW.product_id AND is_current = TRUE;
    
    -- Create new version
    INSERT INTO product_versions (product_id, version_number, product_name, unit_price, description, effective_from)
    VALUES (NEW.product_id, v_version_num, NEW.product_name, NEW.unit_price, NEW.description, NOW());
    
    RETURN NEW;
END;
$$;

CREATE TRIGGER trg_version_product
AFTER UPDATE ON products
FOR EACH ROW
WHEN (OLD.product_name IS DISTINCT FROM NEW.product_name 
   OR OLD.unit_price IS DISTINCT FROM NEW.unit_price
   OR OLD.description IS DISTINCT FROM NEW.description)
EXECUTE FUNCTION fn_version_product();

-- ตัวอย่างที่ 23: Soft Delete Trigger
CREATE OR REPLACE FUNCTION fn_soft_delete()
RETURNS TRIGGER
LANGUAGE plpgsql
AS $$
BEGIN
    -- แทน DELETE ให้ mark เป็น deleted
    IF TG_OP = 'DELETE' THEN
        EXECUTE format('UPDATE %I SET deleted_at = NOW(), deleted_by = current_user WHERE %I = $1',
            TG_TABLE_NAME, 'id')
        USING OLD.id;
        RETURN NULL;  -- ป้องกันไม่ให้ DELETE จริงๆ
    END IF;
    RETURN OLD;
END;
$$;

-- ตัวอย่างที่ 24: Trigger สำหรับ Denormalization
CREATE OR REPLACE FUNCTION fn_maintain_customer_stats()
RETURNS TRIGGER
LANGUAGE plpgsql
AS $$
DECLARE
    v_customer_id INT;
BEGIN
    v_customer_id := COALESCE(NEW.customer_id, OLD.customer_id);
    
    UPDATE customers
    SET 
        total_orders = (SELECT COUNT(*) FROM orders WHERE customer_id = v_customer_id AND status != 'cancelled'),
        total_spent = (SELECT COALESCE(SUM(total_amount), 0) FROM orders WHERE customer_id = v_customer_id AND status = 'completed'),
        last_order_date = (SELECT MAX(order_date) FROM orders WHERE customer_id = v_customer_id),
        stats_updated_at = NOW()
    WHERE customer_id = v_customer_id;
    
    RETURN COALESCE(NEW, OLD);
END;
$$;

CREATE TRIGGER trg_maintain_customer_stats
AFTER INSERT OR UPDATE OR DELETE ON orders
FOR EACH ROW
EXECUTE FUNCTION fn_maintain_customer_stats();
```

---

## 11. Trigger Performance Implications

```sql
-- ตัวอย่างที่ 25: ตรวจสอบ Triggers ทั้งหมดในระบบ (PostgreSQL)
SELECT 
    trigger_schema,
    trigger_name,
    event_object_table AS table_name,
    event_manipulation AS event,
    action_timing AS timing,
    action_orientation AS level  -- ROW or STATEMENT
FROM information_schema.triggers
WHERE trigger_schema = 'public'
ORDER BY event_object_table, action_timing, trigger_name;

-- ตัวอย่างที่ 26: ดู Trigger Definition
SELECT 
    tg.trigger_name,
    tg.event_object_table,
    tg.event_manipulation,
    tg.action_timing,
    p.prosrc AS function_code
FROM information_schema.triggers tg
JOIN pg_proc p ON p.proname = tg.action_orientation
WHERE tg.trigger_schema = 'public';

-- MySQL - ดู Triggers
SHOW TRIGGERS;
SHOW TRIGGERS FROM database_name;
SHOW TRIGGERS LIKE 'trg_%';

-- ตัวอย่างที่ 27: Disabling Triggers (PostgreSQL)
-- Disable trigger เดียว
ALTER TABLE employees DISABLE TRIGGER trg_before_employee_insert;

-- Enable กลับมา
ALTER TABLE employees ENABLE TRIGGER trg_before_employee_insert;

-- Disable ทุก triggers บน table
ALTER TABLE employees DISABLE TRIGGER ALL;
ALTER TABLE employees ENABLE TRIGGER ALL;

-- Disable trigger ในช่วง data migration (ต้องเป็น superuser)
SET session_replication_role = 'replica';  -- Disables triggers
-- ทำ data migration ที่นี่
SET session_replication_role = 'origin';   -- Re-enables triggers

-- ตัวอย่างที่ 28: DROP TRIGGER
-- PostgreSQL
DROP TRIGGER IF EXISTS trg_before_employee_insert ON employees;
DROP TRIGGER IF EXISTS trg_audit_employees ON employees;

-- MySQL
DROP TRIGGER IF EXISTS trg_before_product_insert;
DROP TRIGGER IF EXISTS trg_after_order_insert;

-- SQL Server
DROP TRIGGER IF EXISTS trg_employee_update;
```

---

## 12. SQL Server Triggers

```sql
-- ตัวอย่างที่ 29: SQL Server AFTER Trigger
CREATE TRIGGER trg_audit_employee_changes
ON employees
AFTER UPDATE
AS
BEGIN
    SET NOCOUNT ON;
    
    INSERT INTO employee_audit (
        employee_id,
        changed_by,
        changed_at,
        old_salary,
        new_salary
    )
    SELECT 
        i.employee_id,
        SYSTEM_USER,
        GETDATE(),
        d.salary,
        i.salary
    FROM inserted i
    JOIN deleted d ON i.employee_id = d.employee_id
    WHERE i.salary != d.salary;
END;

-- ตัวอย่างที่ 30: SQL Server INSTEAD OF Trigger
CREATE TRIGGER trg_soft_delete_customer
ON customers
INSTEAD OF DELETE
AS
BEGIN
    SET NOCOUNT ON;
    
    UPDATE c
    SET 
        c.is_deleted = 1,
        c.deleted_at = GETDATE(),
        c.deleted_by = SYSTEM_USER
    FROM customers c
    JOIN deleted d ON c.customer_id = d.customer_id;
END;
```

---

## แบบฝึกหัด (Exercises)

**ข้อ 1:** สร้าง BEFORE INSERT trigger ที่ auto-set created_at และ validate email

**คำตอบข้อ 1:**
```sql
CREATE OR REPLACE FUNCTION fn_before_user_insert() RETURNS TRIGGER LANGUAGE plpgsql AS $$
BEGIN
    NEW.created_at := NOW();
    IF NEW.email !~ '^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}$' THEN
        RAISE EXCEPTION 'Invalid email: %', NEW.email;
    END IF;
    NEW.email := LOWER(NEW.email);
    RETURN NEW;
END; $$;
CREATE TRIGGER trg_before_user_insert BEFORE INSERT ON users FOR EACH ROW EXECUTE FUNCTION fn_before_user_insert();
```

**ข้อ 2:** สร้าง AFTER UPDATE trigger บน orders เพื่อ log การเปลี่ยน status

**คำตอบข้อ 2:**
```sql
CREATE OR REPLACE FUNCTION fn_log_order_status() RETURNS TRIGGER LANGUAGE plpgsql AS $$
BEGIN
    IF OLD.status IS DISTINCT FROM NEW.status THEN
        INSERT INTO order_status_log(order_id, from_status, to_status, changed_at, changed_by)
        VALUES(NEW.order_id, OLD.status, NEW.status, NOW(), current_user);
    END IF;
    RETURN NEW;
END; $$;
CREATE TRIGGER trg_log_order_status AFTER UPDATE ON orders FOR EACH ROW EXECUTE FUNCTION fn_log_order_status();
```

**ข้อ 3:** สร้าง MySQL BEFORE INSERT trigger ที่ validate ราคาสินค้าต้องมากกว่า 0

**คำตอบข้อ 3:**
```sql
DELIMITER //
CREATE TRIGGER trg_validate_product_price BEFORE INSERT ON products FOR EACH ROW
BEGIN
    IF NEW.unit_price <= 0 THEN
        SIGNAL SQLSTATE '45000' SET MESSAGE_TEXT = 'Price must be > 0';
    END IF;
    SET NEW.created_at = NOW();
END //
DELIMITER ;
```

**ข้อ 4:** สร้าง Trigger ที่ auto-update order total เมื่อ order_details เปลี่ยน

**คำตอบข้อ 4:**
```sql
CREATE OR REPLACE FUNCTION fn_recalc_order_total() RETURNS TRIGGER LANGUAGE plpgsql AS $$
BEGIN
    UPDATE orders SET total_amount = (
        SELECT COALESCE(SUM(quantity * unit_price), 0) FROM order_details
        WHERE order_id = COALESCE(NEW.order_id, OLD.order_id)
    ) WHERE order_id = COALESCE(NEW.order_id, OLD.order_id);
    RETURN COALESCE(NEW, OLD);
END; $$;
CREATE TRIGGER trg_recalc_total AFTER INSERT OR UPDATE OR DELETE ON order_details FOR EACH ROW EXECUTE FUNCTION fn_recalc_order_total();
```

**ข้อ 5:** สร้าง Generic Audit Trigger ที่ log ทุก INSERT/UPDATE/DELETE

**คำตอบข้อ 5:**
```sql
CREATE OR REPLACE FUNCTION fn_audit_all() RETURNS TRIGGER LANGUAGE plpgsql AS $$
BEGIN
    INSERT INTO audit_log(table_name, operation, old_data, new_data, changed_at, changed_by)
    VALUES(TG_TABLE_NAME, TG_OP, CASE WHEN TG_OP != 'INSERT' THEN row_to_json(OLD)::JSONB END,
           CASE WHEN TG_OP != 'DELETE' THEN row_to_json(NEW)::JSONB END, NOW(), current_user);
    RETURN COALESCE(NEW, OLD);
END; $$;
CREATE TRIGGER trg_audit_products AFTER INSERT OR UPDATE OR DELETE ON products FOR EACH ROW EXECUTE FUNCTION fn_audit_all();
```

**ข้อ 6:** สร้าง Trigger ที่ prevent การ DELETE customer ที่มี pending orders

**คำตอบข้อ 6:**
```sql
CREATE OR REPLACE FUNCTION fn_prevent_customer_delete() RETURNS TRIGGER LANGUAGE plpgsql AS $$
DECLARE v_pending INT;
BEGIN
    SELECT COUNT(*) INTO v_pending FROM orders WHERE customer_id = OLD.customer_id AND status = 'pending';
    IF v_pending > 0 THEN
        RAISE EXCEPTION 'Cannot delete customer with % pending orders', v_pending;
    END IF;
    RETURN OLD;
END; $$;
CREATE TRIGGER trg_prevent_customer_delete BEFORE DELETE ON customers FOR EACH ROW EXECUTE FUNCTION fn_prevent_customer_delete();
```

**ข้อ 7:** ดูรายการ Triggers ทั้งหมดในฐานข้อมูล (PostgreSQL)

**คำตอบข้อ 7:**
```sql
SELECT trigger_name, event_object_table, event_manipulation, action_timing, action_orientation
FROM information_schema.triggers WHERE trigger_schema = 'public'
ORDER BY event_object_table, trigger_name;
```

**ข้อ 8:** Disable trigger ชั่วคราวระหว่าง bulk data import

**คำตอบข้อ 8:**
```sql
-- PostgreSQL
ALTER TABLE products DISABLE TRIGGER ALL;
-- Bulk import here
COPY products FROM '/tmp/products.csv' CSV HEADER;
ALTER TABLE products ENABLE TRIGGER ALL;

-- MySQL: ใช้ SET FOREIGN_KEY_CHECKS = 0; (triggers ยังทำงาน)
-- ต้อง drop trigger แล้วสร้างใหม่ใน MySQL
```

**ข้อ 9:** สร้าง Trigger ที่ prevent UPDATE บน archived records

**คำตอบข้อ 9:**
```sql
CREATE OR REPLACE FUNCTION fn_prevent_archived_update() RETURNS TRIGGER LANGUAGE plpgsql AS $$
BEGIN
    IF OLD.is_archived = TRUE THEN
        RAISE EXCEPTION 'Cannot modify archived record (id=%)', OLD.id;
    END IF;
    RETURN NEW;
END; $$;
CREATE TRIGGER trg_prevent_archived_update BEFORE UPDATE ON orders FOR EACH ROW EXECUTE FUNCTION fn_prevent_archived_update();
```

**ข้อ 10:** สร้าง Trigger สำหรับ Soft Delete พร้อม undelete support

**คำตอบข้อ 10:**
```sql
CREATE OR REPLACE FUNCTION fn_soft_delete_handler() RETURNS TRIGGER LANGUAGE plpgsql AS $$
BEGIN
    IF TG_OP = 'DELETE' THEN
        UPDATE customers SET deleted_at = NOW(), deleted_by = current_user WHERE customer_id = OLD.customer_id;
        RETURN NULL;  -- Cancel actual DELETE
    END IF;
    RETURN NEW;
END; $$;
CREATE TRIGGER trg_soft_delete BEFORE DELETE ON customers FOR EACH ROW EXECUTE FUNCTION fn_soft_delete_handler();
-- Undelete: UPDATE customers SET deleted_at = NULL, deleted_by = NULL WHERE customer_id = 100;
```

---

*จบ Part 088: Triggers*
