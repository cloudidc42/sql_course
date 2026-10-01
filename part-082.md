# Part 082: Updatable Views (View ที่แก้ไขข้อมูลได้)

## บทนำ

Updatable Views คือ Views ที่สามารถใช้ INSERT, UPDATE, DELETE ได้ผ่าน View เสมือนกับทำงานกับตารางต้นฉบับโดยตรง ฟีเจอร์นี้มีประโยชน์มากเพราะช่วยให้ผู้ใช้ที่เห็นเฉพาะบาง columns สามารถแก้ไขข้อมูลที่อนุญาตได้

---

## 1. What Makes a View Updatable (เงื่อนไข View ที่ Update ได้)

```sql
-- View สามารถ UPDATE/INSERT/DELETE ได้เมื่อ:
-- 1. เป็น SELECT จากตารางเดียว (single base table)
-- 2. ไม่มี DISTINCT
-- 3. ไม่มี Aggregate Functions (SUM, COUNT, AVG, ฯลฯ)
-- 4. ไม่มี GROUP BY หรือ HAVING
-- 5. ไม่มี UNION, INTERSECT, EXCEPT
-- 6. ไม่มี Subqueries ใน SELECT list
-- 7. ไม่มี Window Functions

-- ตัวอย่างที่ 1: View ที่ UPDATE ได้
CREATE VIEW v_employee_contact AS
SELECT 
    employee_id,
    first_name,
    last_name,
    email,
    phone
FROM employees
WHERE status = 'active';

-- UPDATE ผ่าน View ได้
UPDATE v_employee_contact 
SET email = 'new.email@company.com'
WHERE employee_id = 101;

-- ตัวอย่างที่ 2: View ที่ INSERT ได้
CREATE VIEW v_active_products AS
SELECT product_id, product_name, category_id, unit_price
FROM products
WHERE discontinued = 0;

-- INSERT ผ่าน View
INSERT INTO v_active_products (product_name, category_id, unit_price)
VALUES ('New Widget', 1, 29.99);
-- เทียบเท่ากับ INSERT INTO products (product_name, category_id, unit_price, discontinued) VALUES (...)

-- ตัวอย่างที่ 3: View ที่ DELETE ได้
DELETE FROM v_active_products WHERE product_id = 200;
-- ลบออกจากตาราง products จริงๆ
```

---

## 2. Simple Updatable Views

```sql
-- ตัวอย่างที่ 4: Updatable View สำหรับจัดการ Users
CREATE VIEW v_user_profiles AS
SELECT 
    user_id,
    username,
    first_name,
    last_name,
    email,
    phone,
    bio,
    avatar_url
FROM users
WHERE is_deleted = FALSE;

-- แก้ไข profile
UPDATE v_user_profiles
SET 
    first_name = 'สมชาย',
    last_name = 'ใจดี',
    phone = '081-234-5678'
WHERE user_id = 42;

-- ตัวอย่างที่ 5: Updatable View สำหรับ Inventory Management
CREATE VIEW v_current_inventory AS
SELECT 
    product_id,
    product_name,
    units_in_stock,
    units_on_order,
    reorder_level,
    unit_cost
FROM products
WHERE discontinued = 0;

-- อัพเดทสต็อก
UPDATE v_current_inventory
SET units_in_stock = units_in_stock + 100
WHERE product_id = 15;

-- ตัวอย่างที่ 6: Updatable View พร้อม WITH CHECK OPTION
CREATE VIEW v_active_employees AS
SELECT employee_id, first_name, last_name, department_id, status
FROM employees
WHERE status = 'active'
WITH CHECK OPTION;

-- UPDATE นี้จะสำเร็จ
UPDATE v_active_employees SET department_id = 20 WHERE employee_id = 50;

-- UPDATE นี้จะ FAIL เพราะจะทำให้ row หายออกจาก View
UPDATE v_active_employees SET status = 'terminated' WHERE employee_id = 50;
-- ERROR: CHECK OPTION failed for view 'v_active_employees'

-- ตัวอย่างที่ 7: Updatable View สำหรับ Address Management
CREATE VIEW v_customer_addresses AS
SELECT 
    customer_id,
    address_line1,
    address_line2,
    city,
    state,
    postal_code,
    country
FROM customer_addresses
WHERE address_type = 'shipping' AND is_primary = TRUE;

-- อัพเดท address
UPDATE v_customer_addresses
SET 
    address_line1 = '123 ถนนสุขุมวิท',
    city = 'กรุงเทพฯ',
    postal_code = '10110'
WHERE customer_id = 1001;
```

---

## 3. Limitations of Updatable Views

```sql
-- Views ที่ UPDATE ไม่ได้ - ตัวอย่างที่ทำให้ไม่ updatable

-- ตัวอย่างที่ 8: View ที่มี DISTINCT - ไม่ updatable
CREATE VIEW v_unique_cities AS
SELECT DISTINCT city, country FROM customers;
-- ไม่สามารถ UPDATE ผ่าน View นี้

-- ตัวอย่างที่ 9: View ที่มี GROUP BY - ไม่ updatable
CREATE VIEW v_orders_by_customer AS
SELECT customer_id, COUNT(*) AS total_orders, SUM(total_amount) AS total_spent
FROM orders
GROUP BY customer_id;
-- ไม่สามารถ UPDATE ผ่าน View นี้

-- ตัวอย่างที่ 10: View ที่มี UNION - ไม่ updatable
CREATE VIEW v_all_contacts AS
SELECT customer_id AS id, first_name, last_name, email, 'customer' AS type FROM customers
UNION ALL
SELECT employee_id, first_name, last_name, email, 'employee' AS type FROM employees;
-- ไม่สามารถ UPDATE ผ่าน View นี้

-- ตัวอย่างที่ 11: View ที่มี Subquery ใน SELECT - ไม่ updatable
CREATE VIEW v_products_with_sales AS
SELECT 
    p.product_id,
    p.product_name,
    (SELECT SUM(od.quantity) FROM order_details od WHERE od.product_id = p.product_id) AS total_sold
FROM products p;
-- ไม่สามารถ UPDATE ผ่าน View นี้

-- ตัวอย่างที่ 12: View ที่มี JOIN - ส่วนใหญ่ไม่ updatable (ยกเว้นบางกรณี)
CREATE VIEW v_order_with_customer AS
SELECT o.order_id, o.order_date, o.total_amount, c.company_name
FROM orders o
JOIN customers c ON o.customer_id = c.customer_id;
-- อาจ UPDATE orders columns ได้ใน PostgreSQL/MySQL แต่ไม่ได้เสมอไป
```

---

## 4. INSTEAD OF Triggers on Views (SQL Server/Oracle)

SQL Server และ Oracle รองรับ `INSTEAD OF` triggers ที่ทำให้ View ที่ปกติ update ไม่ได้สามารถ update ได้

```sql
-- ตัวอย่างที่ 13: INSTEAD OF INSERT trigger บน View (SQL Server)
-- View ที่ JOIN หลายตาราง
CREATE VIEW v_order_full AS
SELECT 
    o.order_id,
    o.order_date,
    c.customer_id,
    c.company_name,
    c.email AS customer_email,
    o.total_amount,
    o.status
FROM orders o
JOIN customers c ON o.customer_id = c.customer_id;

-- สร้าง INSTEAD OF INSERT trigger
CREATE TRIGGER trg_order_full_insert
ON v_order_full
INSTEAD OF INSERT
AS
BEGIN
    -- INSERT เฉพาะ columns ของ orders table
    INSERT INTO orders (order_date, customer_id, total_amount, status)
    SELECT 
        i.order_date,
        i.customer_id,
        i.total_amount,
        ISNULL(i.status, 'pending')
    FROM inserted i;
END;

-- ตอนนี้ INSERT ผ่าน View ได้
INSERT INTO v_order_full (order_date, customer_id, total_amount)
VALUES ('2024-01-15', 5, 1500.00);

-- ตัวอย่างที่ 14: INSTEAD OF UPDATE trigger (SQL Server)
CREATE TRIGGER trg_order_full_update
ON v_order_full
INSTEAD OF UPDATE
AS
BEGIN
    -- UPDATE เฉพาะ orders table
    UPDATE o
    SET 
        o.order_date = i.order_date,
        o.total_amount = i.total_amount,
        o.status = i.status
    FROM orders o
    JOIN inserted i ON o.order_id = i.order_id;
    
    -- UPDATE customers table ถ้าต้องการ
    UPDATE c
    SET c.company_name = i.company_name,
        c.email = i.customer_email
    FROM customers c
    JOIN inserted i ON c.customer_id = i.customer_id;
END;

-- ตัวอย่างที่ 15: INSTEAD OF DELETE trigger (SQL Server)
CREATE TRIGGER trg_order_full_delete
ON v_order_full
INSTEAD OF DELETE
AS
BEGIN
    -- แทนที่จะลบ ให้ mark ว่า cancelled
    UPDATE orders
    SET status = 'cancelled', cancelled_date = GETDATE()
    WHERE order_id IN (SELECT order_id FROM deleted);
END;

-- ตัวอย่างที่ 16: INSTEAD OF Trigger สำหรับ Complex View (SQL Server)
-- View สำหรับ Employee Full Info
CREATE VIEW v_employee_full AS
SELECT 
    e.employee_id,
    e.first_name,
    e.last_name,
    e.email,
    d.department_id,
    d.department_name,
    pos.position_id,
    pos.position_title,
    e.salary,
    e.hire_date
FROM employees e
JOIN departments d ON e.department_id = d.department_id
JOIN positions pos ON e.position_id = pos.position_id;

-- INSTEAD OF INSERT trigger
CREATE TRIGGER trg_employee_full_insert
ON v_employee_full
INSTEAD OF INSERT
AS
BEGIN
    SET NOCOUNT ON;
    
    DECLARE @employee_id INT;
    
    -- INSERT เฉพาะ employees table
    INSERT INTO employees (first_name, last_name, email, department_id, position_id, salary, hire_date)
    SELECT first_name, last_name, email, department_id, position_id, salary, hire_date
    FROM inserted;
END;
```

---

## 5. Rules on Views (PostgreSQL)

PostgreSQL มี Rule system ที่แตกต่างจาก triggers

```sql
-- ตัวอย่างที่ 17: PostgreSQL Rules สำหรับ INSERT บน View
CREATE VIEW v_active_users AS
SELECT user_id, username, email, created_at
FROM users
WHERE is_active = TRUE;

-- สร้าง Rule สำหรับ INSERT
CREATE RULE v_active_users_insert AS
ON INSERT TO v_active_users
DO INSTEAD
    INSERT INTO users (username, email, created_at, is_active)
    VALUES (NEW.username, NEW.email, COALESCE(NEW.created_at, NOW()), TRUE);

-- ตอนนี้ INSERT ผ่าน View ได้
INSERT INTO v_active_users (username, email) VALUES ('somchai', 'somchai@test.com');

-- ตัวอย่างที่ 18: PostgreSQL Rules สำหรับ UPDATE
CREATE RULE v_active_users_update AS
ON UPDATE TO v_active_users
DO INSTEAD
    UPDATE users
    SET 
        username = NEW.username,
        email = NEW.email
    WHERE user_id = OLD.user_id;

-- ตัวอย่างที่ 19: PostgreSQL Rules สำหรับ DELETE (Soft delete)
CREATE RULE v_active_users_delete AS
ON DELETE TO v_active_users
DO INSTEAD
    UPDATE users
    SET is_active = FALSE, deactivated_at = NOW()
    WHERE user_id = OLD.user_id;

-- ตัวอย่างที่ 20: PostgreSQL - ใช้ Trigger Function แทน Rule (แนะนำมากกว่า)
-- สร้าง Trigger Function
CREATE OR REPLACE FUNCTION fn_v_active_users_insert()
RETURNS TRIGGER
LANGUAGE plpgsql
AS $$
BEGIN
    INSERT INTO users (username, email, created_at, is_active)
    VALUES (NEW.username, NEW.email, COALESCE(NEW.created_at, NOW()), TRUE);
    
    RETURN NEW;
END;
$$;

-- สร้าง Trigger บน View
CREATE TRIGGER trg_v_active_users_insert
INSTEAD OF INSERT ON v_active_users
FOR EACH ROW EXECUTE FUNCTION fn_v_active_users_insert();
```

---

## 6. INSTEAD OF Triggers - Advanced Examples

```sql
-- ตัวอย่างที่ 21: INSTEAD OF trigger สำหรับ multi-table INSERT (PostgreSQL)
-- View รวมข้อมูล order กับ customer
CREATE VIEW v_new_order AS
SELECT 
    NULL::INT AS order_id,
    NULL::INT AS customer_id,
    NULL::VARCHAR AS customer_name,
    NULL::VARCHAR AS customer_email,
    NULL::TIMESTAMP AS order_date,
    NULL::DECIMAL AS total_amount,
    NULL::VARCHAR AS status;

-- Function สำหรับ INSERT
CREATE OR REPLACE FUNCTION fn_insert_new_order()
RETURNS TRIGGER
LANGUAGE plpgsql
AS $$
DECLARE
    v_customer_id INT;
    v_new_order_id INT;
BEGIN
    -- ตรวจสอบว่า customer มีอยู่หรือยัง
    SELECT customer_id INTO v_customer_id
    FROM customers
    WHERE email = NEW.customer_email;
    
    -- ถ้าไม่มี สร้าง customer ใหม่
    IF v_customer_id IS NULL THEN
        INSERT INTO customers (first_name, email)
        VALUES (NEW.customer_name, NEW.customer_email)
        RETURNING customer_id INTO v_customer_id;
    END IF;
    
    -- สร้าง order
    INSERT INTO orders (customer_id, order_date, total_amount, status)
    VALUES (
        v_customer_id,
        COALESCE(NEW.order_date, NOW()),
        COALESCE(NEW.total_amount, 0),
        COALESCE(NEW.status, 'pending')
    )
    RETURNING order_id INTO v_new_order_id;
    
    RAISE NOTICE 'Created order % for customer %', v_new_order_id, v_customer_id;
    
    RETURN NEW;
END;
$$;

CREATE TRIGGER trg_new_order_insert
INSTEAD OF INSERT ON v_new_order
FOR EACH ROW EXECUTE FUNCTION fn_insert_new_order();

-- ตัวอย่างที่ 22: INSTEAD OF trigger พร้อม Validation (PostgreSQL)
CREATE VIEW v_validated_products AS
SELECT product_id, product_name, unit_price, units_in_stock, category_id
FROM products
WHERE discontinued = 0;

CREATE OR REPLACE FUNCTION fn_validate_product_update()
RETURNS TRIGGER
LANGUAGE plpgsql
AS $$
BEGIN
    -- Validate price
    IF NEW.unit_price <= 0 THEN
        RAISE EXCEPTION 'Price must be greater than 0, got: %', NEW.unit_price;
    END IF;
    
    -- Validate stock
    IF NEW.units_in_stock < 0 THEN
        RAISE EXCEPTION 'Stock cannot be negative, got: %', NEW.units_in_stock;
    END IF;
    
    -- Validate category exists
    IF NOT EXISTS (SELECT 1 FROM categories WHERE category_id = NEW.category_id) THEN
        RAISE EXCEPTION 'Category % does not exist', NEW.category_id;
    END IF;
    
    -- Perform the actual UPDATE
    UPDATE products
    SET 
        product_name = NEW.product_name,
        unit_price = NEW.unit_price,
        units_in_stock = NEW.units_in_stock,
        category_id = NEW.category_id
    WHERE product_id = OLD.product_id;
    
    RETURN NEW;
END;
$$;

CREATE TRIGGER trg_validated_product_update
INSTEAD OF UPDATE ON v_validated_products
FOR EACH ROW EXECUTE FUNCTION fn_validate_product_update();

-- ตัวอย่างที่ 23: INSTEAD OF trigger สำหรับ Soft Delete (PostgreSQL)
CREATE VIEW v_active_customers AS
SELECT customer_id, first_name, last_name, email, phone, created_at
FROM customers
WHERE deleted_at IS NULL;

CREATE OR REPLACE FUNCTION fn_soft_delete_customer()
RETURNS TRIGGER
LANGUAGE plpgsql
AS $$
BEGIN
    UPDATE customers
    SET 
        deleted_at = NOW(),
        deleted_by = current_user
    WHERE customer_id = OLD.customer_id;
    
    RETURN OLD;
END;
$$;

CREATE TRIGGER trg_soft_delete_customer
INSTEAD OF DELETE ON v_active_customers
FOR EACH ROW EXECUTE FUNCTION fn_soft_delete_customer();

-- ตอนนี้ DELETE จาก View จะทำ Soft Delete
DELETE FROM v_active_customers WHERE customer_id = 100;
-- ลบจริงๆ ไม่ได้ แต่ set deleted_at แทน
```

---

## 7. Alternatives When Views Can't Be Updated

```sql
-- ตัวอย่างที่ 24: ใช้ Stored Procedure แทนการ Update ผ่าน View ที่ไม่ updatable
-- View ที่มี aggregate (ไม่ updatable)
CREATE VIEW v_customer_stats AS
SELECT 
    customer_id,
    COUNT(*) AS total_orders,
    SUM(total_amount) AS lifetime_value
FROM orders
GROUP BY customer_id;

-- สร้าง Stored Procedure สำหรับ Update แทน
-- PostgreSQL
CREATE OR REPLACE PROCEDURE update_customer_order(
    p_customer_id INT,
    p_order_id INT,
    p_new_amount DECIMAL
)
LANGUAGE plpgsql
AS $$
BEGIN
    UPDATE orders
    SET total_amount = p_new_amount
    WHERE order_id = p_order_id AND customer_id = p_customer_id;
    
    COMMIT;
END;
$$;

-- ตัวอย่างที่ 25: ใช้ Base Table แทน View สำหรับ Aggregated Data
-- แทนที่จะ update view ที่มี aggregate
-- ให้ update ตารางต้นฉบับโดยตรง

-- สร้าง Stored Procedure ที่ทำงานบน base tables
CREATE OR REPLACE PROCEDURE process_order_cancellation(
    p_order_id INT
)
LANGUAGE plpgsql
AS $$
BEGIN
    -- Update order status
    UPDATE orders SET status = 'cancelled' WHERE order_id = p_order_id;
    
    -- คืน stock
    UPDATE products p
    JOIN order_details od ON p.product_id = od.product_id
    SET p.units_in_stock = p.units_in_stock + od.quantity
    WHERE od.order_id = p_order_id;
    
    -- Log the cancellation
    INSERT INTO order_history (order_id, action, action_date, notes)
    VALUES (p_order_id, 'cancelled', NOW(), 'Cancelled via procedure');
    
    COMMIT;
END;
$$;

-- ตัวอย่างที่ 26: ใช้ Materialized View สำหรับ Aggregated Data
-- สำหรับ Views ที่มี aggregate ไม่ต้องการ update
-- แต่ต้องการประสิทธิภาพ ใช้ Materialized View แทน (Part 083)
CREATE MATERIALIZED VIEW mv_customer_stats AS
SELECT 
    customer_id,
    COUNT(*) AS total_orders,
    SUM(total_amount) AS lifetime_value,
    AVG(total_amount) AS avg_order_value
FROM orders
GROUP BY customer_id;

-- Refresh เมื่อต้องการข้อมูลล่าสุด
REFRESH MATERIALIZED VIEW mv_customer_stats;

-- ตัวอย่างที่ 27: Updatable View บน Single Table - ทุก Column
-- View ที่ updatable ได้ทุก column (safe)
CREATE VIEW v_product_management AS
SELECT 
    product_id,
    product_name,
    category_id,
    supplier_id,
    unit_price,
    units_in_stock,
    units_on_order,
    reorder_level,
    discontinued
FROM products;

-- สามารถทำได้ทุกอย่าง
UPDATE v_product_management SET unit_price = 49.99 WHERE product_id = 1;
INSERT INTO v_product_management (product_name, category_id, unit_price) VALUES ('Test', 1, 10.00);
DELETE FROM v_product_management WHERE product_id = 999;
```

---

## 8. Checking View Updatability

```sql
-- ตัวอย่างที่ 28: ตรวจสอบว่า View Updatable ได้หรือไม่ (MySQL)
SELECT 
    TABLE_NAME,
    IS_UPDATABLE,
    CHECK_OPTION,
    DEFINER,
    SECURITY_TYPE
FROM information_schema.VIEWS
WHERE TABLE_SCHEMA = DATABASE()
ORDER BY TABLE_NAME;

-- ตัวอย่างที่ 29: PostgreSQL - ตรวจสอบ View ที่ updatable ได้
SELECT 
    viewname,
    definition
FROM pg_views
WHERE schemaname = 'public'
ORDER BY viewname;

-- ดูข้อมูลเพิ่มเติม
SELECT 
    table_name,
    is_insertable_into,
    is_typed
FROM information_schema.tables
WHERE table_type = 'VIEW'
  AND table_schema = 'public';
```

---

## 9. WITH CHECK OPTION - Deep Dive

```sql
-- ตัวอย่างที่ 30: LOCAL vs CASCADED behavior
-- สร้าง Base View
CREATE VIEW v_dept_10 AS
SELECT employee_id, first_name, department_id, salary
FROM employees
WHERE department_id = 10;

-- View ซ้อน 1 ชั้น
CREATE VIEW v_dept_10_high_salary AS
SELECT * FROM v_dept_10
WHERE salary > 50000
WITH LOCAL CHECK OPTION;

-- ทดสอบ WITH LOCAL:
-- นี้จะสำเร็จ (salary OK แต่ dept ผิด - LOCAL ไม่ตรวจ dept)
-- ขึ้นอยู่กับ database engine
UPDATE v_dept_10_high_salary
SET department_id = 20
WHERE employee_id = 1;

-- สร้างด้วย CASCADED
CREATE VIEW v_dept_10_high_salary_strict AS
SELECT * FROM v_dept_10
WHERE salary > 50000
WITH CASCADED CHECK OPTION;

-- นี้จะ FAIL (department_id เปลี่ยนทำให้ base view เห็น row)
-- CASCADED ตรวจสอบทุกชั้น
UPDATE v_dept_10_high_salary_strict
SET department_id = 20
WHERE employee_id = 1;
```

---

## 10. Advanced Updatable View Patterns

```sql
-- ตัวอย่างที่ 31: Updatable View สำหรับ Multi-Tenant System
-- แต่ละ tenant เห็นและแก้ไขได้เฉพาะข้อมูลตัวเอง
CREATE VIEW v_tenant_orders AS
SELECT order_id, customer_id, order_date, total_amount, status
FROM orders
WHERE tenant_id = get_current_tenant_id()  -- function ที่ return tenant ปัจจุบัน
WITH CHECK OPTION;

-- ตัวอย่างที่ 32: View ที่ INSERT ไปยัง Archive Table
-- PostgreSQL INSTEAD OF trigger
CREATE TABLE orders_current (LIKE orders);
CREATE TABLE orders_archive (LIKE orders);

CREATE VIEW v_orders AS
SELECT * FROM orders_current
UNION ALL
SELECT * FROM orders_archive;

-- ไม่สามารถ update view ที่มี UNION
-- ใช้ trigger แทน
CREATE OR REPLACE FUNCTION fn_orders_view_insert()
RETURNS TRIGGER
LANGUAGE plpgsql
AS $$
BEGIN
    -- ถ้าเป็น order เก่า ใส่ archive ถ้าใหม่ใส่ current
    IF NEW.order_date < CURRENT_DATE - INTERVAL '1 year' THEN
        INSERT INTO orders_archive VALUES (NEW.*);
    ELSE
        INSERT INTO orders_current VALUES (NEW.*);
    END IF;
    RETURN NEW;
END;
$$;

CREATE TRIGGER trg_orders_view_insert
INSTEAD OF INSERT ON v_orders
FOR EACH ROW EXECUTE FUNCTION fn_orders_view_insert();

-- ตัวอย่างที่ 33: Updatable View สำหรับ Audit Logging อัตโนมัติ
CREATE OR REPLACE FUNCTION fn_employee_update_with_audit()
RETURNS TRIGGER
LANGUAGE plpgsql
AS $$
BEGIN
    -- Log การเปลี่ยนแปลง
    INSERT INTO employee_audit (
        employee_id,
        changed_by,
        changed_at,
        old_email,
        new_email,
        old_department,
        new_department,
        old_salary,
        new_salary
    ) VALUES (
        OLD.employee_id,
        current_user,
        NOW(),
        OLD.email,
        NEW.email,
        OLD.department_id,
        NEW.department_id,
        OLD.salary,
        NEW.salary
    );
    
    -- ทำการ UPDATE จริง
    UPDATE employees
    SET 
        first_name = NEW.first_name,
        last_name = NEW.last_name,
        email = NEW.email,
        department_id = NEW.department_id,
        salary = NEW.salary
    WHERE employee_id = OLD.employee_id;
    
    RETURN NEW;
END;
$$;

CREATE VIEW v_employee_details AS
SELECT employee_id, first_name, last_name, email, department_id, salary
FROM employees
WHERE status = 'active';

CREATE TRIGGER trg_employee_update_audit
INSTEAD OF UPDATE ON v_employee_details
FOR EACH ROW EXECUTE FUNCTION fn_employee_update_with_audit();
```

---

## แบบฝึกหัด (Exercises)

**ข้อ 1:** สร้าง Updatable View `v_products_editable` สำหรับ products ที่ยังขายอยู่ พร้อม WITH CHECK OPTION เพื่อป้องกันการ UPDATE discontinued = 1

**คำตอบข้อ 1:**
```sql
CREATE VIEW v_products_editable AS
SELECT product_id, product_name, category_id, unit_price, units_in_stock, discontinued
FROM products
WHERE discontinued = 0
WITH CHECK OPTION;

-- ทดสอบ: นี้จะ FAIL
UPDATE v_products_editable SET discontinued = 1 WHERE product_id = 1;
```

**ข้อ 2:** สร้าง INSTEAD OF INSERT trigger บน View ที่ JOIN employees กับ departments (PostgreSQL)

**คำตอบข้อ 2:**
```sql
CREATE VIEW v_emp_dept AS
SELECT e.employee_id, e.first_name, e.last_name, e.email,
       d.department_id, d.department_name
FROM employees e JOIN departments d ON e.department_id = d.department_id;

CREATE OR REPLACE FUNCTION fn_emp_dept_insert()
RETURNS TRIGGER LANGUAGE plpgsql AS $$
BEGIN
    INSERT INTO employees (first_name, last_name, email, department_id)
    VALUES (NEW.first_name, NEW.last_name, NEW.email, NEW.department_id);
    RETURN NEW;
END;
$$;

CREATE TRIGGER trg_emp_dept_insert
INSTEAD OF INSERT ON v_emp_dept
FOR EACH ROW EXECUTE FUNCTION fn_emp_dept_insert();
```

**ข้อ 3:** สร้าง View ที่ทำ Soft Delete เมื่อ DELETE ผ่าน View (PostgreSQL INSTEAD OF trigger)

**คำตอบข้อ 3:**
```sql
CREATE VIEW v_active_products AS
SELECT product_id, product_name, unit_price, deleted_at
FROM products WHERE deleted_at IS NULL;

CREATE OR REPLACE FUNCTION fn_soft_delete_product()
RETURNS TRIGGER LANGUAGE plpgsql AS $$
BEGIN
    UPDATE products SET deleted_at = NOW() WHERE product_id = OLD.product_id;
    RETURN OLD;
END;
$$;

CREATE TRIGGER trg_soft_delete_product
INSTEAD OF DELETE ON v_active_products
FOR EACH ROW EXECUTE FUNCTION fn_soft_delete_product();
```

**ข้อ 4:** ตรวจสอบว่า View ใดบ้างใน Database ที่ IS_UPDATABLE = YES (MySQL)

**คำตอบข้อ 4:**
```sql
SELECT TABLE_NAME, IS_UPDATABLE, CHECK_OPTION
FROM information_schema.VIEWS
WHERE TABLE_SCHEMA = DATABASE()
  AND IS_UPDATABLE = 'YES'
ORDER BY TABLE_NAME;
```

**ข้อ 5:** อธิบายความแตกต่างระหว่าง LOCAL และ CASCADED CHECK OPTION พร้อมตัวอย่าง

**คำตอบข้อ 5:**
```sql
-- LOCAL: ตรวจสอบเฉพาะ WHERE clause ของ View ปัจจุบัน
CREATE VIEW v_base AS SELECT * FROM t WHERE x > 0;
CREATE VIEW v_child_local AS SELECT * FROM v_base WHERE y > 0 WITH LOCAL CHECK OPTION;
-- LOCAL: ตรวจเฉพาะ y > 0 ไม่ตรวจ x > 0

-- CASCADED: ตรวจสอบ WHERE clause ของทุก View ในสาย
CREATE VIEW v_child_cascaded AS SELECT * FROM v_base WHERE y > 0 WITH CASCADED CHECK OPTION;
-- CASCADED: ตรวจทั้ง y > 0 AND x > 0
```

**ข้อ 6:** สร้าง Updatable View สำหรับ customer contact info ที่ผู้ใช้สามารถแก้ไขได้เฉพาะ email และ phone

**คำตอบข้อ 6:**
```sql
CREATE VIEW v_customer_contact_editable AS
SELECT customer_id, first_name, last_name, email, phone
FROM customers WHERE is_active = TRUE;

-- เฉพาะ email และ phone ที่ควรแก้ไข
-- ใช้ trigger เพื่อจำกัด columns ที่แก้ไขได้
CREATE OR REPLACE FUNCTION fn_customer_contact_update()
RETURNS TRIGGER LANGUAGE plpgsql AS $$
BEGIN
    UPDATE customers SET email = NEW.email, phone = NEW.phone
    WHERE customer_id = OLD.customer_id;
    RETURN NEW;
END;
$$;

CREATE TRIGGER trg_customer_contact_update
INSTEAD OF UPDATE ON v_customer_contact_editable
FOR EACH ROW EXECUTE FUNCTION fn_customer_contact_update();
```

**ข้อ 7:** สร้าง View ที่ INSERT อัตโนมัติ set created_at และ created_by

**คำตอบข้อ 7:**
```sql
CREATE VIEW v_orders_managed AS
SELECT order_id, customer_id, total_amount, status FROM orders;

CREATE OR REPLACE FUNCTION fn_orders_managed_insert()
RETURNS TRIGGER LANGUAGE plpgsql AS $$
BEGIN
    INSERT INTO orders (customer_id, total_amount, status, created_at, created_by)
    VALUES (NEW.customer_id, NEW.total_amount, 
            COALESCE(NEW.status,'pending'), NOW(), current_user);
    RETURN NEW;
END;
$$;

CREATE TRIGGER trg_orders_managed_insert
INSTEAD OF INSERT ON v_orders_managed
FOR EACH ROW EXECUTE FUNCTION fn_orders_managed_insert();
```

**ข้อ 8:** สร้าง View ที่ UPDATE ไปยังหลาย Tables พร้อมกัน (PostgreSQL)

**คำตอบข้อ 8:**
```sql
CREATE VIEW v_customer_order_update AS
SELECT c.customer_id, c.email AS customer_email,
       o.order_id, o.status AS order_status
FROM customers c JOIN orders o ON c.customer_id = o.customer_id;

CREATE OR REPLACE FUNCTION fn_customer_order_update()
RETURNS TRIGGER LANGUAGE plpgsql AS $$
BEGIN
    UPDATE customers SET email = NEW.customer_email WHERE customer_id = OLD.customer_id;
    UPDATE orders SET status = NEW.order_status WHERE order_id = OLD.order_id;
    RETURN NEW;
END;
$$;

CREATE TRIGGER trg_customer_order_update
INSTEAD OF UPDATE ON v_customer_order_update
FOR EACH ROW EXECUTE FUNCTION fn_customer_order_update();
```

**ข้อ 9:** ทดสอบ WITH CHECK OPTION กับ INSERT ผ่าน View

**คำตอบข้อ 9:**
```sql
CREATE VIEW v_year_2024_orders AS
SELECT order_id, customer_id, order_date, total_amount
FROM orders
WHERE YEAR(order_date) = 2024
WITH CHECK OPTION;

-- นี้จะสำเร็จ
INSERT INTO v_year_2024_orders (customer_id, order_date, total_amount)
VALUES (1, '2024-06-15', 500.00);

-- นี้จะ FAIL เพราะปีไม่ตรง
INSERT INTO v_year_2024_orders (customer_id, order_date, total_amount)
VALUES (1, '2023-12-01', 500.00);
```

**ข้อ 10:** สร้าง INSTEAD OF trigger ที่มี Error Handling และ Logging

**คำตอบข้อ 10:**
```sql
CREATE OR REPLACE FUNCTION fn_safe_product_insert()
RETURNS TRIGGER LANGUAGE plpgsql AS $$
DECLARE
    v_error TEXT;
BEGIN
    -- Validation
    IF NEW.unit_price <= 0 THEN
        RAISE EXCEPTION 'Invalid price: %', NEW.unit_price;
    END IF;
    
    BEGIN
        INSERT INTO products (product_name, category_id, unit_price, discontinued)
        VALUES (NEW.product_name, NEW.category_id, NEW.unit_price, 0);
    EXCEPTION WHEN OTHERS THEN
        GET STACKED DIAGNOSTICS v_error = MESSAGE_TEXT;
        INSERT INTO error_log (operation, error_message, occurred_at)
        VALUES ('product_insert', v_error, NOW());
        RAISE;
    END;
    
    RETURN NEW;
END;
$$;
```

---

*จบ Part 082: Updatable Views*
