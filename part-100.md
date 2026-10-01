# ส่วนที่ 100: Temporal Tables และ Time-Travel Queries

## บทนำ

Temporal Database คือการเก็บข้อมูล "เวลา" ไว้ใน database เพื่อตอบคำถาม:
- ข้อมูลในอดีตเป็นอย่างไร? (ก่อน update/delete)
- ข้อมูลที่ valid ณ วันที่ใดวันหนึ่งคืออะไร?
- การเปลี่ยนแปลงเกิดขึ้นเมื่อไหร่?

SQL:2011 กำหนดมาตรฐาน temporal tables ไว้ 2 ประเภท:
1. **System-Time Temporal Tables** — ติดตาม transaction history อัตโนมัติ
2. **Application-Time Temporal Tables** — ติดตาม valid time ที่ defined โดย business
3. **Bitemporal Tables** — รวมทั้งสองอย่าง

---

## 100.1 System-Versioned Temporal Tables (SQL Server)

### ตัวอย่างที่ 1: สร้าง System-Versioned Temporal Table

```sql
-- SQL Server 2016+
-- System-versioned: SQL Server จัดการ history อัตโนมัติ
CREATE TABLE employees (
    employee_id     INT PRIMARY KEY,
    name            NVARCHAR(200),
    department      NVARCHAR(100),
    salary          DECIMAL(10,2),
    email           NVARCHAR(200),
    -- System-time columns (managed by SQL Server)
    valid_from      DATETIME2 GENERATED ALWAYS AS ROW START,
    valid_to        DATETIME2 GENERATED ALWAYS AS ROW END,
    PERIOD FOR SYSTEM_TIME (valid_from, valid_to)
)
WITH (SYSTEM_VERSIONING = ON (HISTORY_TABLE = dbo.employees_history));

-- ทดสอบ: insert และ update data
INSERT INTO employees VALUES (1, 'Alice Smith', 'Engineering', 75000, 'alice@company.com', DEFAULT, DEFAULT);
INSERT INTO employees VALUES (2, 'Bob Johnson', 'Marketing', 65000, 'bob@company.com', DEFAULT, DEFAULT);

-- Update salary
UPDATE employees SET salary = 80000 WHERE employee_id = 1;

-- Delete employee
DELETE FROM employees WHERE employee_id = 2;
```

### ตัวอย่างที่ 2: Time-Travel Queries

```sql
-- AS OF: ดูข้อมูล ณ เวลาหนึ่ง
SELECT * FROM employees
FOR SYSTEM_TIME AS OF '2024-01-15 12:00:00';

-- FROM...TO: ช่วงเวลา (exclusive end)
SELECT employee_id, name, salary, valid_from, valid_to
FROM employees
FOR SYSTEM_TIME FROM '2024-01-01' TO '2024-06-30';

-- BETWEEN...AND: ช่วงเวลา (inclusive end)
SELECT employee_id, name, salary, valid_from, valid_to
FROM employees
FOR SYSTEM_TIME BETWEEN '2024-01-01' AND '2024-06-30';

-- CONTAINED IN: rows ที่เกิดและสิ้นสุดในช่วงนี้
SELECT employee_id, name, salary, valid_from, valid_to
FROM employees
FOR SYSTEM_TIME CONTAINED IN ('2024-01-01', '2024-06-30');

-- ALL: ดูทั้ง current และ history
SELECT employee_id, name, salary, valid_from, valid_to
FROM employees
FOR SYSTEM_TIME ALL
ORDER BY employee_id, valid_from;
```

### ตัวอย่างที่ 3: History Table Analysis

```sql
-- ดู audit trail ของ employee
SELECT 
    employee_id,
    name,
    salary,
    department,
    valid_from AS changed_at,
    LEAD(valid_from) OVER (PARTITION BY employee_id ORDER BY valid_from) AS next_change,
    valid_to,
    CASE 
        WHEN valid_to = '9999-12-31 23:59:59' THEN 'Current'
        ELSE 'Historical'
    END AS record_status
FROM employees
FOR SYSTEM_TIME ALL
ORDER BY employee_id, valid_from;

-- ดู salary changes
SELECT 
    employee_id,
    name,
    salary,
    LAG(salary) OVER (PARTITION BY employee_id ORDER BY valid_from) AS prev_salary,
    salary - LAG(salary) OVER (PARTITION BY employee_id ORDER BY valid_from) AS salary_change,
    valid_from AS effective_date
FROM employees
FOR SYSTEM_TIME ALL
WHERE employee_id = 1
ORDER BY valid_from;
```

---

## 100.2 Application-Time Temporal Tables

### ตัวอย่างที่ 4: Application-Time Table

```sql
-- Application-time: track valid time ที่ business กำหนด
-- เช่น contract periods, price validity, employee positions

CREATE TABLE price_history (
    product_id      INT,
    product_name    VARCHAR(200),
    price           DECIMAL(10,2),
    -- Application-time period
    valid_from      DATE,
    valid_to        DATE,
    PERIOD FOR APPLICATION_TIME (valid_from, valid_to),
    PRIMARY KEY (product_id, valid_from)
);

INSERT INTO price_history VALUES
(1, 'Laptop Pro', 45000, '2024-01-01', '2024-03-31'),
(1, 'Laptop Pro', 43000, '2024-04-01', '2024-06-30'),
(1, 'Laptop Pro', 41000, '2024-07-01', '9999-12-31'),
(2, 'Phone X', 25000, '2024-01-01', '2024-05-31'),
(2, 'Phone X', 22000, '2024-06-01', '9999-12-31');

-- ดูราคา ณ วันที่ใดวันหนึ่ง
SELECT product_id, product_name, price
FROM price_history
WHERE '2024-05-15' BETWEEN valid_from AND valid_to - INTERVAL '1 day';
```

### ตัวอย่างที่ 5: Application-Time Queries

```sql
-- SQL:2011 FOR APPLICATION_TIME (รองรับใน SQL Server 2022+)
-- PostgreSQL ใช้ WHERE condition แทน

-- ราคาที่ valid ณ วันนี้
SELECT product_id, product_name, price
FROM price_history
WHERE CURRENT_DATE BETWEEN valid_from AND valid_to - INTERVAL '1 day'
ORDER BY product_id;

-- ราคาในอดีต (1 เมษา 2024)
SELECT product_id, product_name, price
FROM price_history
WHERE '2024-04-01' BETWEEN valid_from AND valid_to - INTERVAL '1 day';

-- ช่วงที่ราคาเปลี่ยน
SELECT 
    product_id,
    product_name,
    valid_from,
    valid_to,
    price,
    LAG(price) OVER (PARTITION BY product_id ORDER BY valid_from) AS previous_price,
    ROUND(
        100.0 * (price - LAG(price) OVER (PARTITION BY product_id ORDER BY valid_from)) /
        NULLIF(LAG(price) OVER (PARTITION BY product_id ORDER BY valid_from), 0),
        2
    ) AS price_change_pct
FROM price_history
ORDER BY product_id, valid_from;
```

---

## 100.3 Slowly Changing Dimensions (SCD)

### SCD Type 1 - Overwrite

```sql
-- SCD Type 1: เขียนทับ ไม่เก็บ history
CREATE TABLE dim_customer_type1 (
    customer_id     INT PRIMARY KEY,
    first_name      VARCHAR(100),
    last_name       VARCHAR(100),
    email           VARCHAR(200),
    city            VARCHAR(100),
    country         VARCHAR(100)
);

-- Update: เขียนทับ (ไม่มี history)
UPDATE dim_customer_type1
SET city = 'Bangkok', country = 'Thailand'
WHERE customer_id = 1;
```

### ตัวอย่างที่ 6: SCD Type 2 - Keep Full History

```sql
-- SCD Type 2: เก็บทุก version ของ record
CREATE TABLE dim_employee (
    surrogate_key   SERIAL PRIMARY KEY,     -- synthetic key
    employee_id     INT NOT NULL,           -- natural/business key
    name            VARCHAR(200),
    department      VARCHAR(100),
    salary          DECIMAL(10,2),
    manager_id      INT,
    -- SCD Type 2 metadata columns
    effective_date  DATE NOT NULL,          -- เริ่มต้น valid เมื่อ
    expiry_date     DATE NOT NULL           -- หมดอายุเมื่อ (9999-12-31 = current)
        DEFAULT DATE '9999-12-31',
    is_current      BOOLEAN NOT NULL DEFAULT TRUE,
    created_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Insert initial records
INSERT INTO dim_employee (employee_id, name, department, salary, manager_id, effective_date)
VALUES 
(101, 'Alice Smith', 'Engineering', 75000, NULL, '2020-01-01'),
(102, 'Bob Johnson', 'Marketing', 65000, NULL, '2021-03-15'),
(103, 'Charlie Brown', 'Engineering', 70000, 101, '2021-06-01');
```

### ตัวอย่างที่ 7: SCD Type 2 UPSERT Function

```sql
-- PostgreSQL function สำหรับ SCD Type 2 update
CREATE OR REPLACE FUNCTION upsert_dim_employee(
    p_employee_id   INT,
    p_name          VARCHAR,
    p_department    VARCHAR,
    p_salary        DECIMAL,
    p_manager_id    INT,
    p_effective_date DATE
)
RETURNS VOID AS $$
DECLARE
    v_changed BOOLEAN := FALSE;
BEGIN
    -- ตรวจสอบว่ามีการเปลี่ยนแปลงหรือไม่
    SELECT EXISTS (
        SELECT 1 FROM dim_employee
        WHERE employee_id = p_employee_id
          AND is_current = TRUE
          AND (name != p_name 
               OR department != p_department 
               OR salary != p_salary
               OR COALESCE(manager_id, -1) != COALESCE(p_manager_id, -1))
    ) INTO v_changed;
    
    IF v_changed THEN
        -- Expire ข้อมูลเก่า
        UPDATE dim_employee
        SET expiry_date = p_effective_date - INTERVAL '1 day',
            is_current = FALSE
        WHERE employee_id = p_employee_id
          AND is_current = TRUE;
        
        -- Insert ข้อมูลใหม่
        INSERT INTO dim_employee (employee_id, name, department, salary, manager_id, effective_date)
        VALUES (p_employee_id, p_name, p_department, p_salary, p_manager_id, p_effective_date);
        
    ELSIF NOT EXISTS (
        SELECT 1 FROM dim_employee WHERE employee_id = p_employee_id
    ) THEN
        -- New employee
        INSERT INTO dim_employee (employee_id, name, department, salary, manager_id, effective_date)
        VALUES (p_employee_id, p_name, p_department, p_salary, p_manager_id, p_effective_date);
    END IF;
END;
$$ LANGUAGE plpgsql;

-- ทดสอบ: update salary
SELECT upsert_dim_employee(101, 'Alice Smith', 'Engineering', 85000, NULL, '2024-01-01');

-- ดู history
SELECT employee_id, name, department, salary, effective_date, expiry_date, is_current
FROM dim_employee
WHERE employee_id = 101
ORDER BY effective_date;
```

### ตัวอย่างที่ 8: SCD Type 2 Queries

```sql
-- Query ข้อมูล current
SELECT * FROM dim_employee WHERE is_current = TRUE;

-- Query ข้อมูล ณ วันที่ใดวันหนึ่ง
SELECT * FROM dim_employee
WHERE '2023-06-15' BETWEEN effective_date AND expiry_date
  AND employee_id = 101;

-- ดู history ทั้งหมด
SELECT 
    employee_id,
    name,
    department,
    salary,
    effective_date,
    expiry_date,
    is_current,
    LEAD(salary) OVER (PARTITION BY employee_id ORDER BY effective_date) AS next_salary
FROM dim_employee
ORDER BY employee_id, effective_date;

-- คำนวณระยะเวลาที่อยู่ในแต่ละ department
SELECT 
    employee_id,
    name,
    department,
    effective_date,
    COALESCE(expiry_date, CURRENT_DATE) AS expiry_date,
    COALESCE(expiry_date, CURRENT_DATE) - effective_date AS days_in_dept
FROM dim_employee
ORDER BY employee_id, effective_date;
```

---

## 100.4 Bitemporal Tables

### ตัวอย่างที่ 9: Bitemporal Model

```sql
-- Bitemporal: both valid time AND transaction time
-- valid_from/valid_to: เวลาที่ข้อมูลถูกต้องตาม business
-- row_created_at/row_expired_at: เวลาที่ row ถูกสร้าง/expired ใน database

CREATE TABLE contract_bitemporal (
    contract_id         INT,
    employee_id         INT,
    position            VARCHAR(100),
    salary              DECIMAL(10,2),
    -- Application (valid) time
    valid_from          DATE NOT NULL,
    valid_to            DATE NOT NULL DEFAULT '9999-12-31',
    -- Transaction (system) time
    row_created_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    row_expired_at      TIMESTAMP DEFAULT '9999-12-31 23:59:59',
    PRIMARY KEY (contract_id, row_created_at)
);

-- Insert initial contract
INSERT INTO contract_bitemporal 
    (contract_id, employee_id, position, salary, valid_from, valid_to)
VALUES 
    (1001, 101, 'Engineer', 75000, '2023-01-01', '9999-12-31'),
    (1002, 102, 'Manager', 90000, '2023-01-01', '2023-12-31');

-- ต่อมา ค้นพบว่า position ของ contract 1001 ผิด (ควรเป็น Senior Engineer)
-- Correct with bitemporal:
-- 1. Expire ข้อมูลเก่าใน transaction time
UPDATE contract_bitemporal
SET row_expired_at = CURRENT_TIMESTAMP
WHERE contract_id = 1001 AND row_expired_at = '9999-12-31 23:59:59';

-- 2. Insert corrected record
INSERT INTO contract_bitemporal 
    (contract_id, employee_id, position, salary, valid_from, valid_to)
VALUES 
    (1001, 101, 'Senior Engineer', 80000, '2023-01-01', '9999-12-31');
```

### ตัวอย่างที่ 10: Bitemporal Queries

```sql
-- Current view (both times current)
SELECT * FROM contract_bitemporal
WHERE CURRENT_DATE BETWEEN valid_from AND valid_to
  AND CURRENT_TIMESTAMP BETWEEN row_created_at AND row_expired_at;

-- What we KNEW at time T (transaction time point)
-- ข้อมูลที่เราเชื่อ ณ เวลา T
SELECT * FROM contract_bitemporal
WHERE '2023-06-01' BETWEEN valid_from AND valid_to
  AND '2023-06-01'::TIMESTAMP BETWEEN row_created_at AND row_expired_at;

-- What was TRUE at time T (valid time point)  
-- ข้อมูลที่ถูกต้องตาม business ณ เวลา T
SELECT * FROM contract_bitemporal
WHERE '2023-06-01' BETWEEN valid_from AND valid_to
  AND row_expired_at = '9999-12-31 23:59:59';  -- current record version
```

---

## 100.5 PostgreSQL Temporal Patterns

### ตัวอย่างที่ 11: Audit Table Pattern

```sql
-- Audit trail ด้วย trigger
CREATE TABLE products (
    product_id      SERIAL PRIMARY KEY,
    name            VARCHAR(200),
    price           DECIMAL(10,2),
    stock           INT,
    updated_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE products_audit (
    audit_id        SERIAL PRIMARY KEY,
    product_id      INT,
    operation       VARCHAR(10),  -- INSERT, UPDATE, DELETE
    old_data        JSONB,
    new_data        JSONB,
    changed_by      TEXT DEFAULT CURRENT_USER,
    changed_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Audit trigger function
CREATE OR REPLACE FUNCTION audit_products()
RETURNS TRIGGER AS $$
BEGIN
    IF TG_OP = 'INSERT' THEN
        INSERT INTO products_audit (product_id, operation, new_data)
        VALUES (NEW.product_id, 'INSERT', row_to_json(NEW)::JSONB);
        RETURN NEW;
    ELSIF TG_OP = 'UPDATE' THEN
        INSERT INTO products_audit (product_id, operation, old_data, new_data)
        VALUES (NEW.product_id, 'UPDATE', row_to_json(OLD)::JSONB, row_to_json(NEW)::JSONB);
        RETURN NEW;
    ELSIF TG_OP = 'DELETE' THEN
        INSERT INTO products_audit (product_id, operation, old_data)
        VALUES (OLD.product_id, 'DELETE', row_to_json(OLD)::JSONB);
        RETURN OLD;
    END IF;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER products_audit_trigger
AFTER INSERT OR UPDATE OR DELETE ON products
FOR EACH ROW EXECUTE FUNCTION audit_products();

-- Test
INSERT INTO products (name, price, stock) VALUES ('Laptop Pro', 45000, 100);
UPDATE products SET price = 43000 WHERE product_id = 1;
UPDATE products SET stock = 95 WHERE product_id = 1;
DELETE FROM products WHERE product_id = 1;

-- View audit trail
SELECT 
    audit_id,
    product_id,
    operation,
    old_data->>'price' AS old_price,
    new_data->>'price' AS new_price,
    changed_at
FROM products_audit
ORDER BY changed_at;
```

### ตัวอย่างที่ 12: SCD Type 2 ด้วย trigger

```sql
-- Automatic SCD Type 2 ด้วย trigger
CREATE TABLE dim_product (
    surrogate_key   SERIAL PRIMARY KEY,
    product_id      INT NOT NULL,
    name            VARCHAR(200),
    category        VARCHAR(100),
    price           DECIMAL(10,2),
    effective_from  DATE DEFAULT CURRENT_DATE,
    effective_to    DATE DEFAULT '9999-12-31',
    is_current      BOOLEAN DEFAULT TRUE
);

CREATE OR REPLACE FUNCTION scd2_product()
RETURNS TRIGGER AS $$
BEGIN
    -- ถ้าเป็น UPDATE ที่เปลี่ยนข้อมูลจริง
    IF TG_OP = 'UPDATE' AND (
        OLD.name != NEW.name OR
        OLD.category != NEW.category OR
        OLD.price != NEW.price
    ) THEN
        -- Expire current record
        UPDATE dim_product
        SET effective_to = CURRENT_DATE - 1,
            is_current = FALSE
        WHERE product_id = OLD.product_id
          AND is_current = TRUE;
        
        -- Insert new version
        INSERT INTO dim_product (product_id, name, category, price)
        VALUES (NEW.product_id, NEW.name, NEW.category, NEW.price);
        
        RETURN NULL;  -- Skip the original UPDATE
    END IF;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;
```

---

## 100.6 MySQL Temporal Patterns

### ตัวอย่างที่ 13: MySQL ด้วย Stored Procedure

```sql
-- MySQL SCD Type 2 pattern
CREATE TABLE dim_employee_mysql (
    surrogate_key   INT AUTO_INCREMENT PRIMARY KEY,
    employee_id     INT NOT NULL,
    name            VARCHAR(200),
    department      VARCHAR(100),
    salary          DECIMAL(10,2),
    effective_date  DATE NOT NULL,
    expiry_date     DATE NOT NULL DEFAULT '9999-12-31',
    is_current      TINYINT(1) DEFAULT 1,
    INDEX idx_employee_id (employee_id),
    INDEX idx_current (is_current),
    INDEX idx_dates (effective_date, expiry_date)
);

DELIMITER $$

CREATE PROCEDURE upsert_employee_scd2(
    IN p_employee_id    INT,
    IN p_name           VARCHAR(200),
    IN p_department     VARCHAR(100),
    IN p_salary         DECIMAL(10,2),
    IN p_effective_date DATE
)
BEGIN
    DECLARE v_changed BOOLEAN DEFAULT FALSE;
    
    -- Check for changes
    SELECT (name != p_name OR department != p_department OR salary != p_salary)
    INTO v_changed
    FROM dim_employee_mysql
    WHERE employee_id = p_employee_id AND is_current = 1
    LIMIT 1;
    
    IF v_changed THEN
        -- Expire old record
        UPDATE dim_employee_mysql
        SET expiry_date = DATE_SUB(p_effective_date, INTERVAL 1 DAY),
            is_current = 0
        WHERE employee_id = p_employee_id AND is_current = 1;
        
        -- Insert new record
        INSERT INTO dim_employee_mysql (employee_id, name, department, salary, effective_date)
        VALUES (p_employee_id, p_name, p_department, p_salary, p_effective_date);
    ELSEIF NOT EXISTS (
        SELECT 1 FROM dim_employee_mysql WHERE employee_id = p_employee_id
    ) THEN
        INSERT INTO dim_employee_mysql (employee_id, name, department, salary, effective_date)
        VALUES (p_employee_id, p_name, p_department, p_salary, p_effective_date);
    END IF;
END$$

DELIMITER ;

-- ทดสอบ
CALL upsert_employee_scd2(101, 'Alice Smith', 'Engineering', 75000, '2023-01-01');
CALL upsert_employee_scd2(101, 'Alice Smith', 'Engineering', 85000, '2024-01-01');
```

---

## 100.7 SCD Type 3 - Limited History

### ตัวอย่างที่ 14: SCD Type 3

```sql
-- SCD Type 3: เก็บ previous value เพิ่มเติม (ไม่ใช่ full history)
CREATE TABLE dim_employee_type3 (
    employee_id         INT PRIMARY KEY,
    name                VARCHAR(200),
    current_department  VARCHAR(100),
    previous_department VARCHAR(100),  -- เก็บแค่ 1 version ก่อนหน้า
    dept_change_date    DATE,
    current_salary      DECIMAL(10,2),
    previous_salary     DECIMAL(10,2),
    salary_change_date  DATE
);

INSERT INTO dim_employee_type3 
    (employee_id, name, current_department, current_salary)
VALUES 
    (101, 'Alice Smith', 'Engineering', 75000),
    (102, 'Bob Johnson', 'Marketing', 65000);

-- Update ด้วย SCD Type 3 (เก็บ previous)
UPDATE dim_employee_type3
SET 
    previous_department = current_department,
    current_department = 'Senior Engineering',
    dept_change_date = '2024-01-01',
    previous_salary = current_salary,
    current_salary = 85000,
    salary_change_date = '2024-01-01'
WHERE employee_id = 101;

SELECT 
    employee_id, name,
    current_department, previous_department,
    current_salary, previous_salary,
    dept_change_date
FROM dim_employee_type3;
```

---

## 100.8 Temporal Joins และ Point-in-Time Queries

### ตัวอย่างที่ 15: Point-in-Time Join

```sql
-- สร้าง fact table และ dimension
CREATE TABLE fact_orders (
    order_id        SERIAL PRIMARY KEY,
    customer_id     INT,
    product_id      INT,
    quantity        INT,
    order_date      DATE,
    amount          DECIMAL(12,2)
);

INSERT INTO fact_orders VALUES
(1, 201, 1, 3, '2023-06-15', 135000),
(2, 201, 1, 2, '2024-02-20', 86000),
(3, 202, 2, 5, '2023-09-10', 125000);

-- Join กับ SCD Type 2 dimension ณ เวลาที่ order เกิดขึ้น
SELECT 
    f.order_id,
    f.order_date,
    f.amount,
    e.name AS employee_name,
    e.department AS dept_at_time_of_order,
    e.salary AS salary_at_time_of_order
FROM fact_orders f
JOIN dim_employee e
    ON e.employee_id = 101  -- ตัวอย่าง: lookup เฉพาะ employee
    AND f.order_date BETWEEN e.effective_date AND e.expiry_date;
```

### ตัวอย่างที่ 16: Period Overlap Detection

```sql
-- ตรวจสอบว่า periods ทับซ้อนกัน
-- Period A: [a_start, a_end)
-- Period B: [b_start, b_end)
-- Overlap ถ้า: a_start < b_end AND b_start < a_end

CREATE TABLE project_assignments (
    assignment_id   SERIAL PRIMARY KEY,
    employee_id     INT,
    project_id      INT,
    start_date      DATE,
    end_date        DATE
);

INSERT INTO project_assignments (employee_id, project_id, start_date, end_date) VALUES
(101, 1, '2024-01-01', '2024-03-31'),
(101, 2, '2024-02-15', '2024-05-31'),  -- ทับซ้อนกับ project 1
(101, 3, '2024-06-01', '2024-09-30'),
(102, 1, '2024-01-01', '2024-06-30');

-- หา overlapping assignments
SELECT 
    a.employee_id,
    a.project_id AS project_a,
    b.project_id AS project_b,
    GREATEST(a.start_date, b.start_date) AS overlap_start,
    LEAST(a.end_date, b.end_date) AS overlap_end,
    LEAST(a.end_date, b.end_date) - GREATEST(a.start_date, b.start_date) + 1 AS overlap_days
FROM project_assignments a
JOIN project_assignments b
    ON a.employee_id = b.employee_id
    AND a.assignment_id < b.assignment_id  -- avoid duplicates
    AND a.start_date <= b.end_date          -- period overlap condition
    AND b.start_date <= a.end_date;
```

---

## 100.9 Temporal Data Patterns ขั้นสูง

### ตัวอย่างที่ 17: Effective Date Management

```sql
-- ราคาที่มีช่วงเวลา - เพิ่ม/แก้ไขอย่างถูกต้อง
CREATE OR REPLACE FUNCTION set_product_price(
    p_product_id    INT,
    p_price         DECIMAL,
    p_valid_from    DATE,
    p_valid_to      DATE DEFAULT '9999-12-31'
)
RETURNS VOID AS $$
BEGIN
    -- Terminate existing period ที่ทับซ้อน
    UPDATE price_history
    SET valid_to = p_valid_from - INTERVAL '1 day'
    WHERE product_id = p_product_id
      AND valid_to >= p_valid_from
      AND valid_from < p_valid_from;
    
    -- Delete periods ที่อยู่ภายใน new period ทั้งหมด
    DELETE FROM price_history
    WHERE product_id = p_product_id
      AND valid_from >= p_valid_from
      AND valid_to <= p_valid_to;
    
    -- Insert new period
    INSERT INTO price_history (product_id, product_name, price, valid_from, valid_to)
    SELECT p_product_id, product_name, p_price, p_valid_from, p_valid_to
    FROM price_history
    WHERE product_id = p_product_id
    LIMIT 1;
END;
$$ LANGUAGE plpgsql;
```

### ตัวอย่างที่ 18: Reconstruct State at Any Point

```sql
-- ดึงข้อมูล "snapshot" ของ data warehouse ณ วันที่ใดก็ได้
CREATE OR REPLACE FUNCTION get_employee_snapshot(p_as_of DATE)
RETURNS TABLE (
    employee_id     INT,
    name            VARCHAR,
    department      VARCHAR,
    salary          DECIMAL,
    effective_date  DATE
) AS $$
BEGIN
    RETURN QUERY
    SELECT 
        e.employee_id,
        e.name,
        e.department,
        e.salary,
        e.effective_date
    FROM dim_employee e
    WHERE p_as_of BETWEEN e.effective_date AND e.expiry_date;
END;
$$ LANGUAGE plpgsql;

-- ดูข้อมูล ณ วันที่ต่างๆ
SELECT * FROM get_employee_snapshot('2023-06-01');
SELECT * FROM get_employee_snapshot('2024-01-15');
```

### ตัวอย่างที่ 19: Temporal Aggregation

```sql
-- คำนวณ average salary ที่ effective ณ แต่ละเดือน
WITH months AS (
    SELECT generate_series(
        '2023-01-01'::DATE,
        '2024-12-01'::DATE,
        '1 month'
    )::DATE AS month_start
),
monthly_snapshot AS (
    SELECT 
        m.month_start,
        e.department,
        e.salary
    FROM months m
    JOIN dim_employee e
        ON m.month_start BETWEEN e.effective_date AND e.expiry_date
)
SELECT 
    month_start,
    department,
    COUNT(*) AS headcount,
    AVG(salary) AS avg_salary,
    SUM(salary) AS total_salary
FROM monthly_snapshot
GROUP BY month_start, department
ORDER BY month_start, department;
```

### ตัวอย่างที่ 20: Change Detection

```sql
-- ตรวจหา records ที่เปลี่ยนแปลงระหว่างสอง snapshots
WITH snapshot_t1 AS (
    SELECT * FROM dim_employee
    WHERE '2023-06-01' BETWEEN effective_date AND expiry_date
),
snapshot_t2 AS (
    SELECT * FROM dim_employee
    WHERE '2024-06-01' BETWEEN effective_date AND expiry_date
)
SELECT 
    COALESCE(t2.employee_id, t1.employee_id) AS employee_id,
    t1.name AS name_before,
    t2.name AS name_after,
    t1.department AS dept_before,
    t2.department AS dept_after,
    t1.salary AS salary_before,
    t2.salary AS salary_after,
    CASE
        WHEN t1.employee_id IS NULL THEN 'NEW'
        WHEN t2.employee_id IS NULL THEN 'DELETED'
        WHEN t1.salary != t2.salary THEN 'SALARY_CHANGE'
        WHEN t1.department != t2.department THEN 'DEPT_CHANGE'
        ELSE 'NO_CHANGE'
    END AS change_type
FROM snapshot_t1 t1
FULL OUTER JOIN snapshot_t2 t2 ON t1.employee_id = t2.employee_id
WHERE t1.employee_id IS NULL 
   OR t2.employee_id IS NULL
   OR t1.salary != t2.salary
   OR t1.department != t2.department
ORDER BY employee_id;
```

### ตัวอย่างที่ 21: Temporal Slowly Changing Dimension Report

```sql
-- รายงานเวลา employee อยู่ในแต่ละ department
SELECT 
    employee_id,
    name,
    department,
    effective_date AS from_date,
    CASE 
        WHEN expiry_date = '9999-12-31' THEN CURRENT_DATE
        ELSE expiry_date
    END AS to_date,
    CASE 
        WHEN expiry_date = '9999-12-31' THEN CURRENT_DATE
        ELSE expiry_date
    END - effective_date AS days_in_dept,
    is_current
FROM dim_employee
ORDER BY employee_id, effective_date;
```

---

## 100.10 Temporal Tables: Best Practices

### ตัวอย่างที่ 22: SCD Type 6 (Hybrid)

```sql
-- SCD Type 6 = Type 1 + Type 2 + Type 3
-- เก็บ current value, previous value, AND full history rows
CREATE TABLE dim_customer_type6 (
    surrogate_key       SERIAL PRIMARY KEY,
    customer_id         INT NOT NULL,
    -- Type 1 fields (always current in all rows)
    current_email       VARCHAR(200),    -- always the latest email
    current_city        VARCHAR(100),    -- always the latest city
    -- Type 2 fields (values at time of row creation)
    historical_segment  VARCHAR(50),
    historical_score    DECIMAL(5,2),
    -- Type 3 fields (previous value)
    previous_segment    VARCHAR(50),
    -- SCD metadata
    effective_date      DATE,
    expiry_date         DATE DEFAULT '9999-12-31',
    is_current          BOOLEAN DEFAULT TRUE,
    row_version         INT DEFAULT 1
);
```

### ตัวอย่างที่ 23: Temporal Data Validation

```sql
-- ตรวจสอบ data quality: periods ต้องไม่ทับซ้อน
SELECT 
    a.employee_id,
    'OVERLAP' AS issue,
    a.effective_date AS period_a_start,
    a.expiry_date AS period_a_end,
    b.effective_date AS period_b_start,
    b.expiry_date AS period_b_end
FROM dim_employee a
JOIN dim_employee b
    ON a.employee_id = b.employee_id
    AND a.surrogate_key < b.surrogate_key
    AND a.effective_date <= b.expiry_date
    AND b.effective_date <= a.expiry_date

UNION ALL

-- ตรวจสอบ gap: ไม่ควรมีช่องว่าง
SELECT 
    a.employee_id,
    'GAP',
    a.expiry_date + 1,
    b.effective_date - 1,
    NULL,
    NULL
FROM dim_employee a
JOIN dim_employee b
    ON a.employee_id = b.employee_id
    AND a.expiry_date + 1 < b.effective_date
    AND NOT EXISTS (
        SELECT 1 FROM dim_employee c
        WHERE c.employee_id = a.employee_id
          AND c.effective_date BETWEEN a.expiry_date + 1 AND b.effective_date - 1
    );
```

### ตัวอย่างที่ 24: Time-Travel Report

```sql
-- Report: track salary changes ทุก employee
WITH salary_history AS (
    SELECT 
        employee_id,
        name,
        salary,
        effective_date AS change_date,
        LAG(salary) OVER (PARTITION BY employee_id ORDER BY effective_date) AS prev_salary
    FROM dim_employee
    WHERE salary IS NOT NULL
)
SELECT 
    employee_id,
    name,
    change_date,
    salary AS new_salary,
    prev_salary AS old_salary,
    salary - COALESCE(prev_salary, salary) AS increase_amount,
    CASE 
        WHEN prev_salary IS NULL THEN NULL
        ELSE ROUND(100.0 * (salary - prev_salary) / prev_salary, 2)
    END AS increase_pct
FROM salary_history
ORDER BY employee_id, change_date;
```

### ตัวอย่างที่ 25: End-to-End ETL with SCD Type 2

```sql
-- Complete ETL pattern: source → staging → dimension
-- Step 1: Staging table
CREATE TEMP TABLE staging_employees AS
SELECT * FROM (VALUES
    (101, 'Alice Smith', 'Senior Engineering', 88000),
    (102, 'Bob Johnson', 'Marketing', 65000),
    (104, 'Diana Lee', 'Finance', 72000)  -- new employee
) AS t(employee_id, name, department, salary);

-- Step 2: Identify changes
WITH changes AS (
    SELECT 
        s.employee_id,
        s.name,
        s.department,
        s.salary,
        d.surrogate_key,
        CASE
            WHEN d.employee_id IS NULL THEN 'INSERT'
            WHEN d.name != s.name OR d.department != s.department OR d.salary != s.salary 
                THEN 'UPDATE'
            ELSE 'NO_CHANGE'
        END AS change_type
    FROM staging_employees s
    LEFT JOIN dim_employee d 
        ON s.employee_id = d.employee_id AND d.is_current = TRUE
)
-- Step 3: Expire changed records
UPDATE dim_employee d
SET expiry_date = CURRENT_DATE - 1,
    is_current = FALSE
FROM changes c
WHERE c.surrogate_key = d.surrogate_key
  AND c.change_type = 'UPDATE';

-- Step 4: Insert new/changed records
INSERT INTO dim_employee (employee_id, name, department, salary, effective_date)
SELECT 
    s.employee_id, s.name, s.department, s.salary, CURRENT_DATE
FROM staging_employees s
JOIN (
    SELECT 
        s2.employee_id,
        CASE
            WHEN d2.employee_id IS NULL THEN 'INSERT'
            WHEN d2.name != s2.name OR d2.department != s2.department OR d2.salary != s2.salary 
                THEN 'UPDATE'
            ELSE 'NO_CHANGE'
        END AS change_type
    FROM staging_employees s2
    LEFT JOIN dim_employee d2 
        ON s2.employee_id = d2.employee_id AND d2.is_current = TRUE
) changes ON s.employee_id = changes.employee_id
WHERE changes.change_type IN ('INSERT', 'UPDATE');
```

---

## แบบฝึกหัดท้ายบท

### แบบฝึกหัดที่ 1
สร้าง SCD Type 2 dimension table สำหรับ products พร้อม function สำหรับ upsert

**คำตอบ:**
```sql
CREATE TABLE dim_product_scd2 (
    surrogate_key   SERIAL PRIMARY KEY,
    product_id      INT NOT NULL,
    name            VARCHAR(200),
    category        VARCHAR(100),
    price           DECIMAL(10,2),
    effective_date  DATE NOT NULL DEFAULT CURRENT_DATE,
    expiry_date     DATE NOT NULL DEFAULT '9999-12-31',
    is_current      BOOLEAN DEFAULT TRUE
);

CREATE OR REPLACE FUNCTION upsert_product_scd2(
    p_product_id    INT,
    p_name          VARCHAR,
    p_category      VARCHAR,
    p_price         DECIMAL
)
RETURNS VOID AS $$
BEGIN
    IF EXISTS (
        SELECT 1 FROM dim_product_scd2
        WHERE product_id = p_product_id
          AND is_current = TRUE
          AND (name != p_name OR category != p_category OR price != p_price)
    ) THEN
        UPDATE dim_product_scd2
        SET expiry_date = CURRENT_DATE - 1, is_current = FALSE
        WHERE product_id = p_product_id AND is_current = TRUE;
        
        INSERT INTO dim_product_scd2 (product_id, name, category, price)
        VALUES (p_product_id, p_name, p_category, p_price);
    ELSIF NOT EXISTS (SELECT 1 FROM dim_product_scd2 WHERE product_id = p_product_id) THEN
        INSERT INTO dim_product_scd2 (product_id, name, category, price)
        VALUES (p_product_id, p_name, p_category, p_price);
    END IF;
END;
$$ LANGUAGE plpgsql;

SELECT upsert_product_scd2(1, 'Laptop Pro', 'Electronics', 45000);
SELECT upsert_product_scd2(1, 'Laptop Pro', 'Electronics', 42000);  -- price change
```

### แบบฝึกหัดที่ 2
Query ข้อมูล SCD Type 2 ณ วันที่ที่กำหนด

**คำตอบ:**
```sql
CREATE OR REPLACE FUNCTION get_product_snapshot(p_as_of DATE DEFAULT CURRENT_DATE)
RETURNS TABLE (
    product_id  INT,
    name        VARCHAR,
    category    VARCHAR,
    price       DECIMAL,
    valid_from  DATE
) AS $$
BEGIN
    RETURN QUERY
    SELECT 
        p.product_id,
        p.name,
        p.category,
        p.price,
        p.effective_date
    FROM dim_product_scd2 p
    WHERE p_as_of BETWEEN p.effective_date AND p.expiry_date;
END;
$$ LANGUAGE plpgsql;

-- ดูข้อมูลปัจจุบัน
SELECT * FROM get_product_snapshot();

-- ดูข้อมูลในอดีต
SELECT * FROM get_product_snapshot('2024-01-15');
```

### แบบฝึกหัดที่ 3
สร้าง audit trigger สำหรับตาราง orders

**คำตอบ:**
```sql
CREATE TABLE orders (
    order_id    SERIAL PRIMARY KEY,
    customer_id INT,
    amount      DECIMAL(12,2),
    status      VARCHAR(20) DEFAULT 'pending'
);

CREATE TABLE orders_audit (
    audit_id    SERIAL PRIMARY KEY,
    order_id    INT,
    operation   VARCHAR(10),
    old_data    JSONB,
    new_data    JSONB,
    changed_by  TEXT DEFAULT CURRENT_USER,
    changed_at  TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE OR REPLACE FUNCTION audit_orders()
RETURNS TRIGGER AS $$
BEGIN
    IF TG_OP = 'INSERT' THEN
        INSERT INTO orders_audit (order_id, operation, new_data)
        VALUES (NEW.order_id, 'INSERT', row_to_json(NEW)::JSONB);
        RETURN NEW;
    ELSIF TG_OP = 'UPDATE' THEN
        INSERT INTO orders_audit (order_id, operation, old_data, new_data)
        VALUES (NEW.order_id, 'UPDATE', row_to_json(OLD)::JSONB, row_to_json(NEW)::JSONB);
        RETURN NEW;
    ELSIF TG_OP = 'DELETE' THEN
        INSERT INTO orders_audit (order_id, operation, old_data)
        VALUES (OLD.order_id, 'DELETE', row_to_json(OLD)::JSONB);
        RETURN OLD;
    END IF;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER orders_audit_trigger
AFTER INSERT OR UPDATE OR DELETE ON orders
FOR EACH ROW EXECUTE FUNCTION audit_orders();
```

### แบบฝึกหัดที่ 4
ตรวจสอบ overlapping periods ใน project assignments

**คำตอบ:**
```sql
SELECT 
    a.employee_id,
    a.project_id AS project1,
    b.project_id AS project2,
    a.start_date AS p1_start, a.end_date AS p1_end,
    b.start_date AS p2_start, b.end_date AS p2_end,
    GREATEST(a.start_date, b.start_date) AS overlap_start,
    LEAST(a.end_date, b.end_date) AS overlap_end
FROM project_assignments a
JOIN project_assignments b
    ON a.employee_id = b.employee_id
    AND a.assignment_id < b.assignment_id
    AND a.start_date <= b.end_date
    AND b.start_date <= a.end_date
ORDER BY a.employee_id, overlap_start;
```

### แบบฝึกหัดที่ 5
สร้าง price history table และ query ราคาสินค้าในอดีต

**คำตอบ:**
```sql
-- query ราคาย้อนหลัง
SELECT 
    p.product_id,
    ph.price,
    ph.valid_from,
    ph.valid_to,
    ph.valid_to - ph.valid_from AS days_valid
FROM (SELECT DISTINCT product_id FROM price_history) p
JOIN price_history ph ON ph.product_id = p.product_id
WHERE p.product_id = 1
ORDER BY ph.valid_from;

-- ราคา ณ วันที่ใดวันหนึ่ง
SELECT product_id, price
FROM price_history
WHERE product_id = 1
  AND '2024-05-01' BETWEEN valid_from AND valid_to - INTERVAL '1 day';
```

### แบบฝึกหัดที่ 6
สร้าง function เพื่อ query records ณ เวลาใดก็ได้ (time-travel query)

**คำตอบ:**
```sql
CREATE OR REPLACE FUNCTION time_travel_employees(p_as_of TIMESTAMPTZ)
RETURNS TABLE (
    employee_id INT,
    name VARCHAR,
    department VARCHAR,
    salary DECIMAL,
    change_date DATE
) AS $$
BEGIN
    RETURN QUERY
    SELECT 
        e.employee_id,
        e.name,
        e.department,
        e.salary,
        e.effective_date
    FROM dim_employee e
    WHERE p_as_of::DATE BETWEEN e.effective_date AND e.expiry_date
    ORDER BY e.employee_id;
END;
$$ LANGUAGE plpgsql;

SELECT * FROM time_travel_employees('2023-07-01');
SELECT * FROM time_travel_employees(NOW());
```

### แบบฝึกหัดที่ 7
คำนวณ average salary ต่อ department สำหรับแต่ละปีโดยใช้ SCD Type 2 data

**คำตอบ:**
```sql
WITH year_series AS (
    SELECT generate_series(2022, 2024) AS yr
),
annual_snapshot AS (
    SELECT 
        y.yr,
        e.department,
        e.salary
    FROM year_series y
    JOIN dim_employee e
        ON make_date(y.yr, 12, 31) BETWEEN e.effective_date AND e.expiry_date
)
SELECT 
    yr AS year,
    department,
    COUNT(*) AS headcount,
    ROUND(AVG(salary), 2) AS avg_salary,
    SUM(salary) AS total_salary_cost
FROM annual_snapshot
GROUP BY yr, department
ORDER BY yr, department;
```

### แบบฝึกหัดที่ 8
สร้าง report แสดง employee tenure ใน แต่ละ department

**คำตอบ:**
```sql
SELECT 
    employee_id,
    name,
    department,
    effective_date AS start_date,
    CASE 
        WHEN expiry_date = '9999-12-31' THEN CURRENT_DATE
        ELSE expiry_date
    END AS end_date,
    CASE 
        WHEN expiry_date = '9999-12-31' THEN CURRENT_DATE - effective_date
        ELSE expiry_date - effective_date
    END AS tenure_days,
    CASE 
        WHEN expiry_date = '9999-12-31' THEN CURRENT_DATE - effective_date
        ELSE expiry_date - effective_date
    END / 365.0 AS tenure_years,
    is_current
FROM dim_employee
ORDER BY employee_id, effective_date;
```

### แบบฝึกหัดที่ 9
สร้าง Change Data Capture (CDC) query ตรวจสอบความแตกต่างระหว่าง 2 snapshots

**คำตอบ:**
```sql
CREATE OR REPLACE FUNCTION detect_changes(
    p_from_date DATE,
    p_to_date DATE
)
RETURNS TABLE (
    employee_id INT,
    change_type TEXT,
    field_changed TEXT,
    old_value TEXT,
    new_value TEXT,
    changed_on DATE
) AS $$
BEGIN
    RETURN QUERY
    WITH snapshot_before AS (
        SELECT * FROM dim_employee
        WHERE p_from_date BETWEEN effective_date AND expiry_date
    ),
    snapshot_after AS (
        SELECT * FROM dim_employee
        WHERE p_to_date BETWEEN effective_date AND expiry_date
    )
    SELECT 
        COALESCE(b.employee_id, a.employee_id)::INT,
        CASE
            WHEN b.employee_id IS NULL THEN 'NEW_EMPLOYEE'
            WHEN a.employee_id IS NULL THEN 'TERMINATED'
            WHEN b.salary != a.salary THEN 'SALARY_CHANGE'
            WHEN b.department != a.department THEN 'DEPT_CHANGE'
        END::TEXT,
        CASE 
            WHEN b.salary != a.salary THEN 'salary'
            WHEN b.department != a.department THEN 'department'
            ELSE 'N/A'
        END::TEXT,
        COALESCE(b.salary::TEXT, b.department)::TEXT,
        COALESCE(a.salary::TEXT, a.department)::TEXT,
        p_to_date
    FROM snapshot_before b
    FULL OUTER JOIN snapshot_after a ON b.employee_id = a.employee_id
    WHERE b.employee_id IS NULL
       OR a.employee_id IS NULL
       OR b.salary != a.salary
       OR b.department != a.department;
END;
$$ LANGUAGE plpgsql;

SELECT * FROM detect_changes('2023-01-01', '2024-01-01');
```

### แบบฝึกหัดที่ 10
สร้าง Complete Data Lineage view ที่แสดง full history ของ employee

**คำตอบ:**
```sql
CREATE OR REPLACE VIEW employee_full_history AS
SELECT 
    employee_id,
    name,
    department,
    salary,
    effective_date AS valid_from,
    CASE WHEN expiry_date = '9999-12-31' THEN NULL ELSE expiry_date END AS valid_to,
    is_current,
    -- เปรียบเทียบกับ version ก่อนหน้า
    LAG(salary) OVER (PARTITION BY employee_id ORDER BY effective_date) AS prev_salary,
    LAG(department) OVER (PARTITION BY employee_id ORDER BY effective_date) AS prev_dept,
    salary - LAG(salary) OVER (PARTITION BY employee_id ORDER BY effective_date) AS salary_delta,
    CASE 
        WHEN LAG(department) OVER (PARTITION BY employee_id ORDER BY effective_date) IS NULL 
            THEN 'Initial Record'
        WHEN LAG(department) OVER (PARTITION BY employee_id ORDER BY effective_date) != department 
            THEN 'Department Transfer'
        WHEN LAG(salary) OVER (PARTITION BY employee_id ORDER BY effective_date) != salary 
            THEN 'Salary Adjustment'
        ELSE 'Other Change'
    END AS change_reason,
    ROW_NUMBER() OVER (PARTITION BY employee_id ORDER BY effective_date) AS version_number,
    COUNT(*) OVER (PARTITION BY employee_id) AS total_versions
FROM dim_employee
ORDER BY employee_id, effective_date;

SELECT * FROM employee_full_history;
```

---

## สรุปบทที่ 100 และสรุปทั้งหมด

### Temporal Data Types สรุป

| ประเภท | คำอธิบาย | ใช้เมื่อ |
|--------|-----------|---------|
| SCD Type 1 | Overwrite | ไม่ต้องการ history |
| SCD Type 2 | Full history rows | ต้องการ time-travel |
| SCD Type 3 | Previous + current | แค่ 1 version ย้อนหลัง |
| SCD Type 6 | Hybrid 1+2+3 | ต้องการทุกอย่าง |
| System-Temporal | DB manages history | SQL Server, MariaDB |
| Application-Temporal | Business valid time | Valid-time tracking |
| Bitemporal | Both system + valid | Corrections + history |

### เส้นทางการเรียนรู้ SQL

```
ส่วนที่ 91-100 ครอบคลุม:
├── CTE และ Recursive CTE (91-92)
├── Window Functions ทั้งหมด (93-95)
├── Advanced Patterns: Gap & Island, Cohort (96)
├── JSON ใน SQL (97)
├── Full-Text Search (98)
├── OLAP: ROLLUP, CUBE, PIVOT, Statistics (99)
└── Temporal Tables และ SCD (100)
```

### Best Practices Temporal Data
1. ใช้ **DATE** สำหรับ application-time, **TIMESTAMP** สำหรับ system-time
2. ใช้ `'9999-12-31'` แทน `NULL` สำหรับ open-ended periods (ง่ายต่อ query)
3. เพิ่ม **is_current** flag เพื่อ performance ของ current-state queries
4. สร้าง **Index** บน (natural_key, is_current) และ (natural_key, effective_date)
5. ใช้ **BETWEEN** สำหรับ date range queries (inclusive)
6. หมั่น **validate** ว่าไม่มี overlapping periods

---

## ยินดีด้วย! คุณเรียนจบ SQL Course ครบ 100 บทแล้ว

จาก SELECT พื้นฐานจนถึง Temporal Tables คุณมีทักษะ SQL ระดับ Expert แล้ว!
