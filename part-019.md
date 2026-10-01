# Part 19: Database และ Schema Management

## บทนำ

การบริหาร Database และ Schema เป็นงานของ DBA (Database Administrator) และ Developer ที่ต้องทำงานกับฐานข้อมูลในระดับ server บทนี้ครอบคลุม:
- สร้าง/ลบ Database
- Schema management
- ดูรายการ Tables และ Objects
- System catalogs
- Multi-tenant design

---

## 1. CREATE DATABASE

```sql
-- ตัวอย่าง 1: CREATE DATABASE พื้นฐาน
CREATE DATABASE myapp;

-- ตัวอย่าง 2: สร้างพร้อม character set
CREATE DATABASE ecommerce
CHARACTER SET utf8mb4
COLLATE utf8mb4_unicode_ci;

-- ตัวอย่าง 3: IF NOT EXISTS
CREATE DATABASE IF NOT EXISTS myapp;

-- ตัวอย่าง 4: ตัวอย่าง database ต่างๆ
-- สำหรับ Production
CREATE DATABASE prod_ecommerce
    CHARACTER SET utf8mb4
    COLLATE utf8mb4_unicode_ci;

-- สำหรับ Development
CREATE DATABASE dev_ecommerce
    CHARACTER SET utf8mb4
    COLLATE utf8mb4_unicode_ci;

-- สำหรับ Testing
CREATE DATABASE test_ecommerce
    CHARACTER SET utf8mb4
    COLLATE utf8mb4_unicode_ci;

-- PostgreSQL
/*
CREATE DATABASE myapp
    WITH 
    OWNER = postgres
    ENCODING = 'UTF8'
    LC_COLLATE = 'en_US.UTF-8'
    LC_CTYPE = 'en_US.UTF-8'
    TEMPLATE = template0
    CONNECTION LIMIT = 100;
*/
```

---

## 2. DROP DATABASE

```sql
-- ตัวอย่าง 5: DROP DATABASE
DROP DATABASE old_database;

-- ตัวอย่าง 6: IF EXISTS
DROP DATABASE IF EXISTS temp_db;

-- ⚠️ ระวัง: DROP DATABASE ลบทุกอย่างในฐานข้อมูลถาวร!
-- ตรวจสอบก่อน
SHOW DATABASES;
SELECT DATABASE();  -- ดูว่ากำลังใช้ database อะไรอยู่

-- MySQL: ไม่สามารถ drop database ที่กำลังใช้อยู่
-- ต้อง USE อื่นก่อน
USE other_database;
DROP DATABASE temp_db;
```

---

## 3. USE - เลือก Database

```sql
-- ตัวอย่าง 7: USE command (MySQL)
USE myapp;
USE ecommerce;

-- ดู database ปัจจุบัน
SELECT DATABASE();  -- MySQL
-- SELECT current_database();  -- PostgreSQL

-- ตัวอย่าง 8: Fully qualified names (ไม่ต้อง USE)
SELECT * FROM ecommerce.products;
SELECT * FROM ecommerce.customers;

-- ตัวอย่าง 9: Cross-database queries (MySQL)
SELECT 
    e.employee_id,
    e.first_name,
    o.order_id
FROM company_db.employees e
JOIN orders_db.orders o ON e.customer_id = o.customer_id;
```

---

## 4. SHOW DATABASES / SHOW TABLES

```sql
-- ตัวอย่าง 10: SHOW DATABASES
SHOW DATABASES;
SHOW SCHEMAS;  -- MySQL: คำสั่งเดียวกัน

-- กรองด้วย LIKE หรือ WHERE
SHOW DATABASES LIKE 'prod%';
SHOW DATABASES LIKE '%test%';

-- ตัวอย่าง 11: SHOW TABLES
USE ecommerce;
SHOW TABLES;
SHOW FULL TABLES;  -- รวม view type

-- กรอง
SHOW TABLES LIKE '%order%';
SHOW TABLES WHERE Tables_in_ecommerce LIKE '%product%';

-- ตัวอย่าง 12: ดูข้อมูลตาราง
SHOW TABLE STATUS;
SHOW TABLE STATUS LIKE 'employees';
SHOW TABLE STATUS WHERE Engine = 'InnoDB';

-- ตัวอย่าง 13: ดู columns
DESCRIBE employees;
DESC products;
SHOW COLUMNS FROM orders;
SHOW FULL COLUMNS FROM customers;  -- รวม collation, comment

-- ตัวอย่าง 14: ดู CREATE statement
SHOW CREATE TABLE employees;
SHOW CREATE TABLE orders;
SHOW CREATE DATABASE ecommerce;
```

---

## 5. Information Schema

```sql
-- ตัวอย่าง 15: information_schema เบื้องต้น
-- information_schema เป็น virtual database ที่เก็บ metadata

-- ดู tables ทั้งหมดในฐานข้อมูล
SELECT 
    TABLE_NAME,
    TABLE_TYPE,
    TABLE_ROWS,
    ENGINE,
    CREATE_TIME
FROM information_schema.TABLES
WHERE TABLE_SCHEMA = DATABASE()
ORDER BY TABLE_NAME;

-- ตัวอย่าง 16: ดู columns ของตาราง
SELECT 
    COLUMN_NAME,
    ORDINAL_POSITION,
    DATA_TYPE,
    CHARACTER_MAXIMUM_LENGTH,
    NUMERIC_PRECISION,
    NUMERIC_SCALE,
    IS_NULLABLE,
    COLUMN_DEFAULT,
    COLUMN_KEY,
    EXTRA,
    COLUMN_COMMENT
FROM information_schema.COLUMNS
WHERE TABLE_SCHEMA = DATABASE()
  AND TABLE_NAME = 'employees'
ORDER BY ORDINAL_POSITION;

-- ตัวอย่าง 17: ดูขนาดตาราง
SELECT 
    TABLE_NAME,
    TABLE_ROWS AS approx_rows,
    ROUND(DATA_LENGTH / 1024 / 1024, 2) AS data_mb,
    ROUND(INDEX_LENGTH / 1024 / 1024, 2) AS index_mb,
    ROUND((DATA_LENGTH + INDEX_LENGTH) / 1024 / 1024, 2) AS total_mb
FROM information_schema.TABLES
WHERE TABLE_SCHEMA = DATABASE()
ORDER BY (DATA_LENGTH + INDEX_LENGTH) DESC;

-- ตัวอย่าง 18: ดูขนาดทุกฐานข้อมูล
SELECT 
    TABLE_SCHEMA AS database_name,
    COUNT(*) AS table_count,
    ROUND(SUM(DATA_LENGTH + INDEX_LENGTH) / 1024 / 1024, 2) AS total_mb
FROM information_schema.TABLES
GROUP BY TABLE_SCHEMA
ORDER BY total_mb DESC;
```

---

## 6. CREATE SCHEMA (Namespacing)

```sql
-- ตัวอย่าง 19: PostgreSQL Schema
/*
-- Schema = namespace ภายใน database
-- Default schema = public

-- สร้าง schemas สำหรับ modules ต่างๆ
CREATE SCHEMA hr;           -- Human Resources
CREATE SCHEMA finance;      -- Finance  
CREATE SCHEMA inventory;    -- Inventory
CREATE SCHEMA reports;      -- Reports / Analytics

-- สร้างตารางใน schema
CREATE TABLE hr.employees (
    employee_id SERIAL PRIMARY KEY,
    name        VARCHAR(200)
);

CREATE TABLE finance.accounts (
    account_id  SERIAL PRIMARY KEY,
    balance     DECIMAL(15,2)
);

-- Query ด้วย schema prefix
SELECT * FROM hr.employees;
SELECT * FROM finance.accounts;

-- กำหนด search_path (default schemas)
SET search_path TO hr, public;
SELECT * FROM employees;  -- ค้นหาใน hr schema ก่อน ถ้าไม่พบค่อยไป public

-- ดู schemas
SELECT schema_name FROM information_schema.schemata;
-- \dn ใน psql
*/

-- MySQL: ไม่มี schema แยก, CREATE SCHEMA = CREATE DATABASE
CREATE SCHEMA IF NOT EXISTS myapp;  -- เหมือน CREATE DATABASE myapp

-- ตัวอย่าง 20: MySQL database organization แทน schema
-- ใช้ prefix ในชื่อตารางแทน
-- hr_employees แทน hr.employees
-- finance_accounts แทน finance.accounts
```

---

## 7. System Catalogs

```sql
-- ตัวอย่าง 21: MySQL System Tables
-- ดู engines ที่รองรับ
SHOW ENGINES;
SHOW ENGINE INNODB STATUS;

-- ดู variables
SHOW VARIABLES;
SHOW VARIABLES LIKE 'max_connections';
SHOW VARIABLES LIKE '%timeout%';
SHOW VARIABLES LIKE 'character_set%';
SHOW VARIABLES LIKE 'collation%';

-- ดู status
SHOW STATUS;
SHOW STATUS LIKE 'Threads%';
SHOW STATUS LIKE 'Com_select';
SHOW STATUS LIKE 'Innodb_buffer_pool%';

-- ตัวอย่าง 22: ดู processes กำลังรัน
SHOW PROCESSLIST;
SHOW FULL PROCESSLIST;
-- ดู queries ที่รัน/รอ

-- ฆ่า process ที่รัน query นาน
-- KILL 12345;  -- ฆ่า connection id 12345

-- ตัวอย่าง 23: information_schema queries สำคัญ

-- ดู indexes ทั้งหมด
SELECT 
    TABLE_NAME,
    INDEX_NAME,
    SEQ_IN_INDEX,
    COLUMN_NAME,
    NON_UNIQUE,
    INDEX_TYPE
FROM information_schema.STATISTICS
WHERE TABLE_SCHEMA = DATABASE()
ORDER BY TABLE_NAME, INDEX_NAME, SEQ_IN_INDEX;

-- ดู views
SELECT TABLE_NAME, VIEW_DEFINITION
FROM information_schema.VIEWS
WHERE TABLE_SCHEMA = DATABASE();

-- ดู stored procedures
SELECT ROUTINE_NAME, ROUTINE_TYPE, CREATED, LAST_ALTERED
FROM information_schema.ROUTINES
WHERE ROUTINE_SCHEMA = DATABASE();

-- ดู triggers
SELECT TRIGGER_NAME, EVENT_MANIPULATION, EVENT_OBJECT_TABLE, ACTION_TIMING
FROM information_schema.TRIGGERS
WHERE TRIGGER_SCHEMA = DATABASE();

-- ตัวอย่าง 24: PostgreSQL System Catalogs (pg_catalog)
/*
-- ดู tables
SELECT tablename, tableowner 
FROM pg_catalog.pg_tables
WHERE schemaname = 'public';

-- ดู columns
SELECT 
    a.attname AS column_name,
    t.typname AS data_type,
    a.atttypmod,
    a.attnotnull AS not_null
FROM pg_catalog.pg_attribute a
JOIN pg_catalog.pg_type t ON a.atttypid = t.oid
JOIN pg_catalog.pg_class c ON a.attrelid = c.oid
WHERE c.relname = 'employees'
  AND a.attnum > 0;

-- ดู sequences
SELECT sequencename, last_value, start_value, increment_by
FROM pg_sequences
WHERE schemaname = 'public';

-- ดู table sizes
SELECT 
    relname AS table_name,
    pg_size_pretty(pg_total_relation_size(relid)) AS total_size,
    pg_size_pretty(pg_relation_size(relid)) AS table_size,
    pg_size_pretty(pg_indexes_size(relid)) AS indexes_size
FROM pg_catalog.pg_statio_user_tables
ORDER BY pg_total_relation_size(relid) DESC;
*/
```

---

## 8. Schema Design Patterns

### 8.1 Multi-Tenant Database Design

```sql
-- ตัวอย่าง 25: Multi-tenant - Shared Database, Shared Schema
-- วิธีที่ 1: ทุก tenant ใช้ตารางเดียวกัน มี tenant_id column

CREATE TABLE mt_tenants (
    tenant_id   INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    tenant_code VARCHAR(50) UNIQUE NOT NULL,
    tenant_name VARCHAR(200) NOT NULL,
    plan        VARCHAR(20) DEFAULT 'basic',
    is_active   BOOLEAN DEFAULT TRUE,
    created_at  TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE mt_products (
    product_id  INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    tenant_id   INT UNSIGNED NOT NULL,  -- ← ระบุ tenant
    name        VARCHAR(200) NOT NULL,
    price       DECIMAL(10,2),
    is_active   BOOLEAN DEFAULT TRUE,
    
    -- ทุก query ต้องมี tenant_id
    FOREIGN KEY (tenant_id) REFERENCES mt_tenants(tenant_id),
    INDEX idx_tenant (tenant_id),
    
    -- Composite unique: ชื่อสินค้า unique per tenant
    UNIQUE KEY uq_tenant_name (tenant_id, name)
);

CREATE TABLE mt_orders (
    order_id    INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    tenant_id   INT UNSIGNED NOT NULL,
    customer_id INT UNSIGNED NOT NULL,
    total       DECIMAL(10,2),
    created_at  TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    
    FOREIGN KEY (tenant_id) REFERENCES mt_tenants(tenant_id),
    INDEX idx_tenant_date (tenant_id, created_at)
);

-- ตัวอย่าง 26: Application-level tenant isolation
-- ทุก query ต้องใส่ tenant_id

-- ดู products ของ tenant A
SELECT * FROM mt_products WHERE tenant_id = 1 AND is_active = TRUE;

-- ดู orders ของ tenant A ในเดือนนี้
SELECT 
    o.order_id,
    o.created_at,
    o.total
FROM mt_orders o
WHERE 
    o.tenant_id = 1
    AND YEAR(o.created_at) = YEAR(NOW())
    AND MONTH(o.created_at) = MONTH(NOW());

-- ตัวอย่าง 27: Stored procedure ที่ enforce tenant isolation
DELIMITER //
CREATE PROCEDURE get_tenant_products(IN p_tenant_id INT)
BEGIN
    SELECT product_id, name, price
    FROM mt_products
    WHERE tenant_id = p_tenant_id
      AND is_active = TRUE
    ORDER BY name;
END //
DELIMITER ;

-- ตัวอย่าง 28: Schema-per-tenant (PostgreSQL)
/*
-- แต่ละ tenant มี schema ของตัวเอง

-- สร้าง tenant schema
CREATE SCHEMA tenant_acme;
CREATE SCHEMA tenant_corp;

-- Function สร้าง tables สำหรับ tenant ใหม่
CREATE OR REPLACE FUNCTION create_tenant_schema(p_tenant_name TEXT)
RETURNS VOID AS $$
BEGIN
    EXECUTE 'CREATE SCHEMA IF NOT EXISTS ' || quote_ident(p_tenant_name);
    
    EXECUTE '
    CREATE TABLE IF NOT EXISTS ' || quote_ident(p_tenant_name) || '.products (
        product_id  SERIAL PRIMARY KEY,
        name        VARCHAR(200) NOT NULL,
        price       DECIMAL(10,2)
    )';
    
    EXECUTE '
    CREATE TABLE IF NOT EXISTS ' || quote_ident(p_tenant_name) || '.orders (
        order_id    SERIAL PRIMARY KEY,
        customer_id INT,
        total       DECIMAL(10,2)
    )';
END;
$$ LANGUAGE plpgsql;

SELECT create_tenant_schema('tenant_new');
*/
```

---

## 9. Database Naming Conventions

```sql
-- ตัวอย่าง 29: Database Naming

-- Production databases
-- prod_ecommerce, prod_hrms, prod_inventory
-- หรือ ecommerce_prod, hrms_prod

-- Development/Test
-- dev_ecommerce, staging_ecommerce, test_ecommerce
-- หรือ ecommerce_dev, ecommerce_staging

-- ตัวอย่าง 30: ดูข้อมูล database configuration
SELECT 
    SCHEMA_NAME,
    DEFAULT_CHARACTER_SET_NAME,
    DEFAULT_COLLATION_NAME
FROM information_schema.SCHEMATA
WHERE SCHEMA_NAME = DATABASE();

-- เปลี่ยน character set ของ database
ALTER DATABASE ecommerce 
    CHARACTER SET utf8mb4 
    COLLATE utf8mb4_unicode_ci;

-- เปลี่ยน character set ของทุกตารางในฐานข้อมูล (MySQL)
-- ต้องทำทีละตาราง:
ALTER TABLE employees CONVERT TO CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
ALTER TABLE products  CONVERT TO CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
-- ... ทุกตาราง
```

---

## 10. Useful Admin Queries

```sql
-- ตัวอย่าง 31: Query สำคัญสำหรับ DBA

-- ดู tables ที่ไม่มี Primary Key
SELECT TABLE_NAME
FROM information_schema.TABLES t
WHERE TABLE_SCHEMA = DATABASE()
  AND TABLE_TYPE = 'BASE TABLE'
  AND NOT EXISTS (
    SELECT 1 FROM information_schema.TABLE_CONSTRAINTS tc
    WHERE tc.TABLE_SCHEMA = t.TABLE_SCHEMA
      AND tc.TABLE_NAME = t.TABLE_NAME
      AND tc.CONSTRAINT_TYPE = 'PRIMARY KEY'
  );

-- ตัวอย่าง 32: ดู tables ที่ไม่มี indexes นอกจาก PK
SELECT t.TABLE_NAME
FROM information_schema.TABLES t
WHERE t.TABLE_SCHEMA = DATABASE()
  AND t.TABLE_TYPE = 'BASE TABLE'
  AND (
    SELECT COUNT(*) 
    FROM information_schema.STATISTICS s
    WHERE s.TABLE_SCHEMA = t.TABLE_SCHEMA
      AND s.TABLE_NAME = t.TABLE_NAME
      AND s.INDEX_NAME != 'PRIMARY'
  ) = 0;

-- ตัวอย่าง 33: ดู Foreign Keys ทั้งหมด
SELECT 
    kcu.TABLE_NAME AS child_table,
    kcu.COLUMN_NAME,
    kcu.REFERENCED_TABLE_NAME AS parent_table,
    kcu.REFERENCED_COLUMN_NAME,
    rc.DELETE_RULE,
    rc.UPDATE_RULE
FROM information_schema.KEY_COLUMN_USAGE kcu
JOIN information_schema.REFERENTIAL_CONSTRAINTS rc
    ON kcu.CONSTRAINT_NAME = rc.CONSTRAINT_NAME
    AND kcu.CONSTRAINT_SCHEMA = rc.CONSTRAINT_SCHEMA
WHERE kcu.TABLE_SCHEMA = DATABASE()
ORDER BY kcu.TABLE_NAME;

-- ตัวอย่าง 34: ดู tables ที่ไม่มีข้อมูล
SELECT TABLE_NAME, TABLE_ROWS
FROM information_schema.TABLES
WHERE TABLE_SCHEMA = DATABASE()
  AND TABLE_ROWS = 0
  AND TABLE_TYPE = 'BASE TABLE';

-- ตัวอย่าง 35: Generate DROP statements สำหรับ cleanup
SELECT CONCAT('DROP TABLE IF EXISTS `', TABLE_NAME, '`;') AS drop_statement
FROM information_schema.TABLES
WHERE TABLE_SCHEMA = 'temp_database'
ORDER BY TABLE_NAME;

-- ตัวอย่าง 36: ดู table dependencies (via FK)
WITH RECURSIVE table_deps AS (
    SELECT 
        TABLE_NAME,
        REFERENCED_TABLE_NAME AS depends_on,
        0 AS depth
    FROM information_schema.KEY_COLUMN_USAGE
    WHERE TABLE_SCHEMA = DATABASE()
      AND REFERENCED_TABLE_NAME IS NOT NULL
    
    UNION ALL
    
    SELECT 
        td.TABLE_NAME,
        kcu.REFERENCED_TABLE_NAME,
        td.depth + 1
    FROM table_deps td
    JOIN information_schema.KEY_COLUMN_USAGE kcu
        ON kcu.TABLE_NAME = td.depends_on
        AND kcu.TABLE_SCHEMA = DATABASE()
    WHERE kcu.REFERENCED_TABLE_NAME IS NOT NULL
      AND td.depth < 5
)
SELECT DISTINCT TABLE_NAME, depends_on, depth
FROM table_deps
ORDER BY depth, TABLE_NAME;
```

---

## 11. Database Backup References

```sql
-- ตัวอย่าง 37: ดูข้อมูลสำหรับ Backup planning
SELECT 
    TABLE_SCHEMA AS database_name,
    SUM(TABLE_ROWS) AS total_rows,
    ROUND(SUM(DATA_LENGTH + INDEX_LENGTH) / 1024 / 1024 / 1024, 2) AS total_gb,
    COUNT(*) AS table_count,
    MAX(CREATE_TIME) AS newest_table,
    MAX(UPDATE_TIME) AS last_updated
FROM information_schema.TABLES
WHERE TABLE_TYPE = 'BASE TABLE'
GROUP BY TABLE_SCHEMA
ORDER BY total_gb DESC;

-- ตัวอย่าง 38: ดู slow queries (ต้องเปิด slow_query_log)
-- SHOW VARIABLES LIKE 'slow_query_log';
-- SET GLOBAL slow_query_log = 'ON';
-- SET GLOBAL long_query_time = 2;  -- log queries > 2 seconds

-- ดู slow query log
-- SHOW VARIABLES LIKE 'slow_query_log_file';
```

---

## แบบฝึกหัด 10 ข้อ

### ข้อ 1
สร้าง database สำหรับระบบโรงพยาบาลพร้อม charset ที่รองรับภาษาไทย

**เฉลย:**
```sql
CREATE DATABASE IF NOT EXISTS hospital_system
    CHARACTER SET utf8mb4
    COLLATE utf8mb4_unicode_ci;

USE hospital_system;

-- ยืนยัน
SELECT 
    SCHEMA_NAME,
    DEFAULT_CHARACTER_SET_NAME,
    DEFAULT_COLLATION_NAME
FROM information_schema.SCHEMATA
WHERE SCHEMA_NAME = 'hospital_system';
```

### ข้อ 2
เขียน query ดู schema ทั้งหมดที่มีขนาดใหญ่กว่า 100 MB

**เฉลย:**
```sql
SELECT 
    TABLE_SCHEMA AS database_name,
    COUNT(*) AS table_count,
    ROUND(SUM(DATA_LENGTH + INDEX_LENGTH) / 1024 / 1024, 2) AS total_mb,
    ROUND(SUM(DATA_LENGTH) / 1024 / 1024, 2) AS data_mb,
    ROUND(SUM(INDEX_LENGTH) / 1024 / 1024, 2) AS index_mb
FROM information_schema.TABLES
WHERE TABLE_TYPE = 'BASE TABLE'
  AND TABLE_SCHEMA NOT IN ('mysql', 'information_schema', 'performance_schema', 'sys')
GROUP BY TABLE_SCHEMA
HAVING total_mb > 100
ORDER BY total_mb DESC;
```

### ข้อ 3
ดูรายการ tables ทั้งหมดในฐานข้อมูล พร้อมจำนวน columns และ size

**เฉลย:**
```sql
SELECT 
    t.TABLE_NAME,
    COUNT(c.COLUMN_NAME) AS column_count,
    t.TABLE_ROWS AS approx_rows,
    ROUND(t.DATA_LENGTH / 1024, 2) AS data_kb,
    ROUND(t.INDEX_LENGTH / 1024, 2) AS index_kb,
    t.ENGINE,
    t.CREATE_TIME
FROM information_schema.TABLES t
LEFT JOIN information_schema.COLUMNS c 
    ON t.TABLE_SCHEMA = c.TABLE_SCHEMA 
    AND t.TABLE_NAME = c.TABLE_NAME
WHERE t.TABLE_SCHEMA = DATABASE()
  AND t.TABLE_TYPE = 'BASE TABLE'
GROUP BY t.TABLE_NAME, t.TABLE_ROWS, t.DATA_LENGTH, t.INDEX_LENGTH, t.ENGINE, t.CREATE_TIME
ORDER BY t.TABLE_NAME;
```

### ข้อ 4
สร้าง multi-tenant schema สำหรับ SaaS application

**เฉลย:**
```sql
-- Main database สำหรับ SaaS
CREATE DATABASE saas_platform CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
USE saas_platform;

-- Tenant registry
CREATE TABLE tenants (
    tenant_id       INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    subdomain       VARCHAR(50) UNIQUE NOT NULL,
    company_name    VARCHAR(200) NOT NULL,
    plan            ENUM('free', 'basic', 'pro', 'enterprise') DEFAULT 'free',
    max_users       SMALLINT UNSIGNED DEFAULT 5,
    max_storage_mb  INT UNSIGNED DEFAULT 100,
    is_active       BOOLEAN DEFAULT TRUE,
    created_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Shared tables with tenant_id
CREATE TABLE tenant_users (
    user_id     INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    tenant_id   INT UNSIGNED NOT NULL,
    email       VARCHAR(254) NOT NULL,
    name        VARCHAR(200) NOT NULL,
    role        VARCHAR(20) DEFAULT 'user',
    
    UNIQUE KEY uq_tenant_email (tenant_id, email),
    FOREIGN KEY (tenant_id) REFERENCES tenants(tenant_id) ON DELETE CASCADE
);

CREATE TABLE tenant_data (
    data_id     INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    tenant_id   INT UNSIGNED NOT NULL,
    data_key    VARCHAR(200) NOT NULL,
    data_value  MEDIUMTEXT,
    created_at  TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    
    UNIQUE KEY uq_tenant_key (tenant_id, data_key),
    FOREIGN KEY (tenant_id) REFERENCES tenants(tenant_id) ON DELETE CASCADE
);

-- ใส่ tenant ทดสอบ
INSERT INTO tenants (subdomain, company_name, plan) VALUES 
    ('acme', 'ACME Corp', 'pro'),
    ('startup', 'Startup Inc', 'basic');
```

### ข้อ 5
Query ดู tables ที่ไม่มี PRIMARY KEY

**เฉลย:**
```sql
SELECT 
    t.TABLE_NAME,
    t.TABLE_ROWS,
    t.ENGINE
FROM information_schema.TABLES t
WHERE t.TABLE_SCHEMA = DATABASE()
  AND t.TABLE_TYPE = 'BASE TABLE'
  AND t.TABLE_NAME NOT IN (
    SELECT DISTINCT TABLE_NAME
    FROM information_schema.TABLE_CONSTRAINTS
    WHERE TABLE_SCHEMA = DATABASE()
      AND CONSTRAINT_TYPE = 'PRIMARY KEY'
  )
ORDER BY t.TABLE_NAME;
```

### ข้อ 6
สร้าง reporting schema แยกออกจาก transactional schema

**เฉลย:**
```sql
-- Transactional database
CREATE DATABASE IF NOT EXISTS ecommerce_app CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;

-- Reporting database (read-only replicas จะ point ที่นี่)
CREATE DATABASE IF NOT EXISTS ecommerce_reports CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;

USE ecommerce_reports;

-- Summary tables สำหรับ reports
CREATE TABLE daily_sales (
    sale_date       DATE PRIMARY KEY,
    order_count     INT DEFAULT 0,
    revenue         DECIMAL(15,2) DEFAULT 0,
    avg_order       DECIMAL(10,2) DEFAULT 0,
    new_customers   INT DEFAULT 0,
    updated_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);

CREATE TABLE product_performance (
    product_id      INT UNSIGNED,
    period          DATE,
    units_sold      INT DEFAULT 0,
    revenue         DECIMAL(12,2) DEFAULT 0,
    return_rate     DECIMAL(5,2) DEFAULT 0,
    PRIMARY KEY (product_id, period)
);

-- Populate สำหรับ current month
INSERT INTO daily_sales (sale_date, order_count, revenue, avg_order)
SELECT 
    DATE(o.order_date),
    COUNT(*),
    SUM(o.total_amount),
    AVG(o.total_amount)
FROM ecommerce_app.orders o
WHERE o.status = 'completed'
GROUP BY DATE(o.order_date);
```

### ข้อ 7
เขียน script ตรวจสอบ foreign key constraints ที่อาจมีปัญหา

**เฉลย:**
```sql
-- ตรวจสอบ orphaned records ใน child tables
SELECT 
    'order_items' AS child_table,
    'orders' AS parent_table,
    COUNT(*) AS orphaned_rows
FROM order_items oi
LEFT JOIN orders o ON oi.order_id = o.order_id
WHERE o.order_id IS NULL

UNION ALL

SELECT 
    'order_items',
    'products',
    COUNT(*)
FROM order_items oi
LEFT JOIN products p ON oi.product_id = p.product_id
WHERE p.product_id IS NULL

UNION ALL

SELECT 
    'orders',
    'customers',
    COUNT(*)
FROM orders o
LEFT JOIN customers c ON o.customer_id = c.customer_id
WHERE c.customer_id IS NULL

UNION ALL

SELECT 
    'employees',
    'departments',
    COUNT(*)
FROM employees e
LEFT JOIN departments d ON e.department_id = d.department_id
WHERE e.department_id IS NOT NULL AND d.department_id IS NULL;
```

### ข้อ 8
สร้าง database health check report

**เฉลย:**
```sql
-- Database health check
SELECT 'DATABASE HEALTH REPORT' AS report_section, DATABASE() AS database_name, NOW() AS generated_at;

-- 1. ขนาด database
SELECT 
    'Database Size' AS metric,
    CONCAT(ROUND(SUM(DATA_LENGTH + INDEX_LENGTH) / 1024 / 1024, 2), ' MB') AS value
FROM information_schema.TABLES
WHERE TABLE_SCHEMA = DATABASE();

-- 2. จำนวน tables
SELECT 
    'Total Tables' AS metric,
    COUNT(*) AS value
FROM information_schema.TABLES
WHERE TABLE_SCHEMA = DATABASE() AND TABLE_TYPE = 'BASE TABLE';

-- 3. Tables ไม่มี PK
SELECT 
    'Tables Without PK' AS metric,
    COUNT(*) AS value
FROM information_schema.TABLES t
WHERE t.TABLE_SCHEMA = DATABASE()
  AND t.TABLE_TYPE = 'BASE TABLE'
  AND NOT EXISTS (
    SELECT 1 FROM information_schema.TABLE_CONSTRAINTS tc
    WHERE tc.TABLE_SCHEMA = t.TABLE_SCHEMA
      AND tc.TABLE_NAME = t.TABLE_NAME
      AND tc.CONSTRAINT_TYPE = 'PRIMARY KEY'
  );

-- 4. Table row counts
SELECT TABLE_NAME, TABLE_ROWS AS approx_rows
FROM information_schema.TABLES
WHERE TABLE_SCHEMA = DATABASE()
  AND TABLE_TYPE = 'BASE TABLE'
ORDER BY TABLE_ROWS DESC;
```

### ข้อ 9
Generate CREATE TABLE scripts สำหรับ schema documentation

**เฉลย:**
```sql
-- สร้าง documentation ของ schema
SELECT 
    c.TABLE_NAME,
    c.ORDINAL_POSITION AS pos,
    c.COLUMN_NAME,
    c.COLUMN_TYPE,
    c.IS_NULLABLE,
    c.COLUMN_DEFAULT,
    c.EXTRA,
    COALESCE(c.COLUMN_COMMENT, '') AS comment,
    CASE 
        WHEN tc.CONSTRAINT_TYPE = 'PRIMARY KEY' THEN 'PK'
        WHEN tc.CONSTRAINT_TYPE = 'UNIQUE' THEN 'UQ'
        ELSE ''
    END AS key_type
FROM information_schema.COLUMNS c
LEFT JOIN information_schema.KEY_COLUMN_USAGE kcu
    ON c.TABLE_SCHEMA = kcu.TABLE_SCHEMA
    AND c.TABLE_NAME = kcu.TABLE_NAME
    AND c.COLUMN_NAME = kcu.COLUMN_NAME
LEFT JOIN information_schema.TABLE_CONSTRAINTS tc
    ON kcu.CONSTRAINT_NAME = tc.CONSTRAINT_NAME
    AND kcu.TABLE_SCHEMA = tc.TABLE_SCHEMA
    AND tc.CONSTRAINT_TYPE IN ('PRIMARY KEY', 'UNIQUE')
WHERE c.TABLE_SCHEMA = DATABASE()
ORDER BY c.TABLE_NAME, c.ORDINAL_POSITION;
```

### ข้อ 10
สร้าง script สำหรับ cleanup ฐานข้อมูล development

**เฉลย:**
```sql
-- Script สำหรับ reset development database
-- ⚠️ ใช้เฉพาะ development เท่านั้น!

-- ตรวจสอบว่าอยู่ใน dev
SELECT DATABASE() AS current_db;
-- ต้องเป็น dev_xxx เท่านั้น

-- ปิด FK checks
SET FOREIGN_KEY_CHECKS = 0;

-- Truncate ทุกตาราง
SELECT CONCAT('TRUNCATE TABLE `', TABLE_NAME, '`;') AS truncate_sql
FROM information_schema.TABLES
WHERE TABLE_SCHEMA = DATABASE()
  AND TABLE_TYPE = 'BASE TABLE'
ORDER BY TABLE_NAME;

-- รัน statements เหล่านั้น (ด้วย dynamic SQL หรือ manual)
TRUNCATE TABLE order_items;
TRUNCATE TABLE orders;
TRUNCATE TABLE products;
TRUNCATE TABLE customers;
TRUNCATE TABLE employees;
TRUNCATE TABLE departments;

-- เปิด FK checks กลับ
SET FOREIGN_KEY_CHECKS = 1;

-- Insert seed data
SOURCE /path/to/seed.sql;

SELECT 'Development database reset complete' AS status;
```

---

## สรุป

ในบทนี้เราเรียนรู้:
1. **CREATE/DROP DATABASE**: สร้าง/ลบ database พร้อม charset
2. **USE**: เลือก database ปัจจุบัน
3. **SHOW DATABASES/TABLES**: ดูรายการ objects
4. **information_schema**: metadata ของ database
5. **CREATE SCHEMA**: namespacing (PostgreSQL)
6. **System Catalogs**: SHOW STATUS, PROCESSLIST, pg_catalog
7. **Multi-tenant Design**: shared schema vs schema-per-tenant
8. **Admin Queries**: health check, size monitoring, dependency analysis
9. **Database Naming**: conventions ที่ดี

> **Best Practice**: ใช้ information_schema เพื่อ document และ monitor ฐานข้อมูล สร้าง health check queries รัน regularly เพื่อตรวจสอบ FK orphans, tables ไม่มี PK, และขนาดที่เพิ่มขึ้น
