# Part 062: Creating and Managing Indexes

## การสร้างและจัดการ Indexes

---

## บทนำ

ใน Part 061 เราเข้าใจแล้วว่า Index คืออะไรและทำงานอย่างไร ใน Part 062 นี้ เราจะเรียนรู้การสร้าง จัดการ และดูแล Index อย่างละเอียด พร้อมตัวอย่างมากกว่า 30 รายการ

---

## 1. CREATE INDEX Syntax พื้นฐาน

### PostgreSQL

```sql
-- Syntax พื้นฐาน
CREATE INDEX index_name ON table_name (column_name);

-- ตัวอย่างที่ 1: Index บนคอลัมน์เดียว
CREATE INDEX idx_employees_last_name ON employees(last_name);

-- ตัวอย่างที่ 2: Index บนหลายคอลัมน์ (Composite)
CREATE INDEX idx_employees_dept_salary ON employees(department_id, salary);

-- ตัวอย่างที่ 3: Index พร้อมระบุ schema
CREATE INDEX idx_hr_employees_email ON hr.employees(email);

-- ตัวอย่างที่ 4: Descending index
CREATE INDEX idx_employees_salary_desc ON employees(salary DESC);

-- ตัวอย่างที่ 5: Index บน expression
CREATE INDEX idx_employees_lower_email ON employees(LOWER(email));
```

### MySQL

```sql
-- Syntax พื้นฐาน
CREATE INDEX index_name ON table_name (column_name);

-- ตัวอย่างที่ 6: Index บนคอลัมน์ใน MySQL
CREATE INDEX idx_customers_email ON customers(email);

-- ตัวอย่างที่ 7: Index พร้อม length สำหรับ VARCHAR
CREATE INDEX idx_products_name ON products(product_name(50));
-- สำคัญ: MySQL ต้องระบุ length สำหรับ TEXT/BLOB columns

-- ตัวอย่างที่ 8: สร้าง index ตอน ALTER TABLE
ALTER TABLE employees ADD INDEX idx_hire_date (hire_date);

-- ตัวอย่างที่ 9: สร้าง index ตอน CREATE TABLE
CREATE TABLE orders (
    order_id INT AUTO_INCREMENT PRIMARY KEY,
    customer_id INT,
    order_date DATE,
    total_amount DECIMAL(10,2),
    status VARCHAR(20),
    INDEX idx_customer (customer_id),
    INDEX idx_order_date (order_date),
    INDEX idx_status (status)
);
```

### SQLite

```sql
-- ตัวอย่างที่ 10: Index ใน SQLite
CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_created_at ON users(created_at);
```

---

## 2. CREATE UNIQUE INDEX

Unique Index ไม่เพียงแต่ช่วยเพิ่มความเร็วค้นหา แต่ยังบังคับให้ค่าในคอลัมน์ไม่ซ้ำกัน

```sql
-- ตัวอย่างที่ 11: Unique index เดียว
CREATE UNIQUE INDEX idx_employees_email ON employees(email);

-- ตัวอย่างที่ 12: Unique index บนหลายคอลัมน์
-- (combination ของ first_name + last_name + birth_date ต้องไม่ซ้ำ)
CREATE UNIQUE INDEX idx_employees_name_dob 
ON employees(first_name, last_name, birth_date);

-- ตัวอย่างที่ 13: Unique index ที่ allow NULL (PostgreSQL)
-- NULL ไม่นับว่าซ้ำกัน ดังนั้น NULL หลายค่าได้
CREATE UNIQUE INDEX idx_products_sku ON products(sku);
-- ถ้า sku เป็น NULL ได้ หลาย rows มี NULL ได้

-- ตัวอย่างที่ 14: Unique index ใน MySQL
CREATE UNIQUE INDEX idx_customers_phone ON customers(phone_number);

-- ตัวอย่างที่ 15: เพิ่ม UNIQUE constraint ด้วย ALTER TABLE
ALTER TABLE employees ADD UNIQUE INDEX idx_emp_ssn (ssn);
```

### ความแตกต่างระหว่าง UNIQUE INDEX และ UNIQUE CONSTRAINT

```sql
-- ใน PostgreSQL ทั้งสองให้ผลเหมือนกัน:

-- วิธีที่ 1: UNIQUE CONSTRAINT (สร้าง index โดยอัตโนมัติ)
ALTER TABLE employees ADD CONSTRAINT uq_employees_email UNIQUE (email);

-- วิธีที่ 2: CREATE UNIQUE INDEX
CREATE UNIQUE INDEX idx_employees_email ON employees(email);

-- ความแตกต่าง:
-- CONSTRAINT: สามารถ DEFERRABLE ได้, มีชื่อ constraint
-- INDEX: liขage มากกว่า (เช่น partial unique index)
```

---

## 3. Dropping Indexes

```sql
-- ตัวอย่างที่ 16: DROP INDEX ใน PostgreSQL
DROP INDEX idx_employees_last_name;
DROP INDEX IF EXISTS idx_employees_last_name;  -- ไม่ error ถ้าไม่มี

-- ตัวอย่างที่ 17: DROP INDEX ใน MySQL
DROP INDEX idx_customers_email ON customers;
ALTER TABLE customers DROP INDEX idx_customers_email;

-- ตัวอย่างที่ 18: DROP INDEX ใน SQLite
DROP INDEX idx_users_email;

-- ตัวอย่างที่ 19: DROP INDEX ที่ระบุ schema
DROP INDEX hr.idx_employees_email;  -- PostgreSQL

-- ตัวอย่างที่ 20: ลบ index โดย disable ก่อน (SQL Server)
-- ALTER INDEX idx_name ON table_name DISABLE;
-- เพื่อทดสอบก่อนลบจริง
```

---

## 4. Index Naming Conventions

การตั้งชื่อ Index ที่ดีช่วยให้ดูแลรักษาง่าย

```
รูปแบบที่แนะนำ:

idx_{table}_{columns}[_{type}]

ตัวอย่าง:
- idx_employees_last_name       → B-tree index บน last_name
- idx_employees_dept_salary     → Composite index
- uidx_employees_email          → Unique index
- pidx_orders_date              → Partial index
- gin_products_description      → GIN index
```

```sql
-- ตัวอย่างที่ 21: การตั้งชื่อ indexes ตามมาตรฐาน

-- B-tree ปกติ: idx_table_column
CREATE INDEX idx_employees_dept_id ON employees(department_id);
CREATE INDEX idx_orders_customer_id ON orders(customer_id);
CREATE INDEX idx_products_category_price ON products(category_id, price);

-- Unique: uidx_ หรือ uk_
CREATE UNIQUE INDEX uidx_employees_email ON employees(email);
CREATE UNIQUE INDEX uidx_products_sku ON products(sku);

-- Partial: pidx_ หรือ filtidx_
CREATE INDEX pidx_orders_pending ON orders(created_at) WHERE status = 'pending';

-- Expression: eidx_ 
CREATE INDEX eidx_employees_lower_email ON employees(LOWER(email));
```

---

## 5. Listing Existing Indexes

### PostgreSQL

```sql
-- ตัวอย่างที่ 22: ดู indexes ทั้งหมดใน PostgreSQL
SELECT 
    schemaname,
    tablename,
    indexname,
    indexdef
FROM pg_indexes
WHERE schemaname = 'public'
ORDER BY tablename, indexname;

-- ตัวอย่างที่ 23: ดู indexes พร้อมข้อมูลการใช้งาน
SELECT 
    s.relname AS table_name,
    s.indexrelname AS index_name,
    s.idx_scan AS times_scanned,
    s.idx_tup_read AS tuples_read,
    s.idx_tup_fetch AS tuples_fetched,
    pg_size_pretty(pg_relation_size(s.indexrelid)) AS index_size
FROM pg_stat_user_indexes s
JOIN pg_index i ON s.indexrelid = i.indexrelid
WHERE s.schemaname = 'public'
ORDER BY s.idx_scan DESC;

-- ตัวอย่างที่ 24: ดู index columns รายละเอียด
SELECT 
    t.relname AS table_name,
    i.relname AS index_name,
    ix.indisprimary,
    ix.indisunique,
    ix.indisvalid,
    a.attname AS column_name,
    a.attnum AS column_position,
    ix.indoption[a.attnum - 1] AS column_options  -- 0=ASC, 1=DESC
FROM pg_class t
JOIN pg_index ix ON t.oid = ix.indrelid
JOIN pg_class i ON i.oid = ix.indexrelid
JOIN pg_attribute a ON a.attrelid = t.oid AND a.attnum = ANY(ix.indkey)
WHERE t.relkind = 'r'
AND t.relname NOT LIKE 'pg_%'
ORDER BY t.relname, i.relname;
```

### MySQL

```sql
-- ตัวอย่างที่ 25: ดู indexes ใน MySQL
SHOW INDEX FROM employees;

-- ตัวอย่างที่ 26: ดู indexes พร้อมรายละเอียด
SELECT 
    TABLE_NAME,
    INDEX_NAME,
    COLUMN_NAME,
    SEQ_IN_INDEX,
    NON_UNIQUE,
    INDEX_TYPE,
    CARDINALITY,
    NULLABLE
FROM information_schema.STATISTICS
WHERE TABLE_SCHEMA = DATABASE()
ORDER BY TABLE_NAME, INDEX_NAME, SEQ_IN_INDEX;

-- ตัวอย่างที่ 27: ขนาดของ indexes ใน MySQL
SELECT 
    TABLE_NAME,
    INDEX_NAME,
    ROUND(SUM(stat_value * @@innodb_page_size) / 1024 / 1024, 2) AS size_mb
FROM mysql.innodb_index_stats
WHERE stat_name = 'size'
AND database_name = DATABASE()
GROUP BY TABLE_NAME, INDEX_NAME
ORDER BY size_mb DESC;
```

### SQLite

```sql
-- ตัวอย่างที่ 28: ดู indexes ใน SQLite
SELECT name, tbl_name, sql 
FROM sqlite_master 
WHERE type = 'index'
ORDER BY tbl_name, name;

-- ดู index info
PRAGMA index_list(employees);
PRAGMA index_info(idx_employees_last_name);
```

---

## 6. Rebuilding และ Reorganizing Indexes

เมื่อเวลาผ่านไป indexes อาจเกิด fragmentation ทำให้ประสิทธิภาพลดลง

### PostgreSQL: REINDEX

```sql
-- ตัวอย่างที่ 29: REINDEX index เดียว
REINDEX INDEX idx_employees_last_name;

-- ตัวอย่างที่ 30: REINDEX ทุก indexes ของตาราง
REINDEX TABLE employees;

-- ตัวอย่างที่ 31: REINDEX ทั้ง database
REINDEX DATABASE mydb;

-- ตัวอย่างที่ 32: REINDEX CONCURRENTLY (PostgreSQL 12+)
-- ไม่ lock table ระหว่าง reindex
REINDEX INDEX CONCURRENTLY idx_employees_last_name;
REINDEX TABLE CONCURRENTLY employees;
```

### ตรวจสอบ Index Bloat ใน PostgreSQL

```sql
-- ตัวอย่างที่ 33: ตรวจสอบ index bloat
SELECT 
    schemaname,
    tablename,
    indexname,
    pg_size_pretty(pg_relation_size(indexrelid)) AS index_size,
    idx_scan,
    CASE 
        WHEN idx_scan = 0 THEN 'Never used'
        WHEN idx_scan < 100 THEN 'Rarely used'
        ELSE 'Frequently used'
    END AS usage_status
FROM pg_stat_user_indexes
ORDER BY pg_relation_size(indexrelid) DESC
LIMIT 20;
```

### MySQL: OPTIMIZE TABLE

```sql
-- ตัวอย่างที่ 34: Optimize table ใน MySQL (rebuild indexes)
OPTIMIZE TABLE employees;

-- ตัวอย่างที่ 35: Analyze table statistics
ANALYZE TABLE employees;

-- ตัวอย่างที่ 36: ตรวจสอบ fragmentation ใน MySQL
SELECT 
    TABLE_NAME,
    ENGINE,
    DATA_FREE,
    DATA_LENGTH,
    INDEX_LENGTH,
    ROUND(DATA_FREE / (DATA_LENGTH + INDEX_LENGTH) * 100, 2) AS fragmentation_pct
FROM information_schema.TABLES
WHERE TABLE_SCHEMA = DATABASE()
AND DATA_FREE > 0
ORDER BY DATA_FREE DESC;
```

---

## 7. Index Fill Factor

Fill Factor กำหนดว่าแต่ละ page ของ index จะถูกเติมเต็มเท่าใด (%)

```
Fill Factor = 100%: ใช้พื้นที่น้อยสุด แต่ INSERT/UPDATE ทำให้ page splits บ่อย
Fill Factor = 70%: ใช้พื้นที่มากขึ้น แต่มีพื้นที่ว่างสำหรับการ insert/update

ตัวอย่าง:
Fill Factor 100%:
┌──────────────────────┐
│ FULL │ FULL │ FULL  │ → INSERT ต้องสร้าง page ใหม่ (page split)
└──────────────────────┘

Fill Factor 70%:
┌────────────┬──────────────────────┐
│ DATA (70%) │ FREE SPACE (30%)     │ → INSERT ใช้พื้นที่ว่าง (ไม่มี split)
└────────────┴──────────────────────┘
```

```sql
-- ตัวอย่างที่ 37: สร้าง index พร้อม fill factor (PostgreSQL)
CREATE INDEX idx_orders_created_at ON orders(created_at) 
WITH (fillfactor = 70);

-- ตัวอย่างที่ 38: สร้าง index พร้อม fill factor สำหรับ heavily updated table
CREATE INDEX idx_products_price ON products(price) 
WITH (fillfactor = 80);

-- ตัวอย่างที่ 39: เปลี่ยน fill factor ของ index ที่มีอยู่แล้ว
ALTER INDEX idx_orders_created_at SET (fillfactor = 75);

-- ตัวอย่างที่ 40: ดู fill factor ของ indexes
SELECT 
    indexname,
    reloptions
FROM pg_indexes
JOIN pg_class ON pg_class.relname = indexname
WHERE tablename = 'orders'
AND reloptions IS NOT NULL;
```

### เมื่อใดควรใช้ Fill Factor < 100%?

```
- ตารางที่มีการ UPDATE คอลัมน์ที่ index บ่อย
- ตารางที่มีการ INSERT/DELETE ปริมาณมาก
- ตารางที่เกิด page splits บ่อย (ตรวจสอบจาก monitoring)

แนะนำ:
- Read-heavy table: fillfactor = 100 (default)
- Write-heavy table: fillfactor = 70-90
- Time-series data: fillfactor = 90 (append-only)
```

---

## 8. Concurrent Index Creation (PostgreSQL)

ปกติการสร้าง index จะ lock table ไม่ให้ write ได้ระหว่างสร้าง ซึ่งอาจปัญหาใน production

```sql
-- ตัวอย่างที่ 41: สร้าง index แบบปกติ (LOCK table ระหว่างสร้าง)
CREATE INDEX idx_orders_customer_id ON orders(customer_id);
-- ปัญหา: ตาราง orders ถูก lock ระหว่างสร้าง!
-- ถ้าตารางใหญ่ อาจใช้เวลานาน = downtime!

-- ตัวอย่างที่ 42: CREATE INDEX CONCURRENTLY (ไม่ lock table)
CREATE INDEX CONCURRENTLY idx_orders_customer_id ON orders(customer_id);
-- ข้อดี: ตารางยังใช้งานได้ระหว่างสร้าง index
-- ข้อเสีย: ใช้เวลานานกว่า, ใช้ resources มากกว่า

-- ตัวอย่างที่ 43: REINDEX CONCURRENTLY
REINDEX INDEX CONCURRENTLY idx_orders_customer_id;
```

### ข้อควรระวัง CONCURRENTLY

```sql
-- ปัญหาที่อาจเกิด: index ที่ invalid
-- ถ้า CONCURRENTLY index creation ถูก interrupt จะเกิด index ที่ INVALID

-- ตรวจสอบ invalid indexes
SELECT 
    schemaname,
    tablename,
    indexname,
    indisvalid,
    pg_size_pretty(pg_relation_size(indexrelid)) AS size
FROM pg_indexes
JOIN pg_class ON pg_class.relname = indexname
JOIN pg_index ON pg_index.indexrelid = pg_class.oid
WHERE NOT indisvalid;

-- วิธีแก้: ลบ invalid index แล้วสร้างใหม่
DROP INDEX CONCURRENTLY idx_orders_customer_id;
CREATE INDEX CONCURRENTLY idx_orders_customer_id ON orders(customer_id);
```

---

## 9. Online Index Operations

### PostgreSQL: Partial Index Creation

```sql
-- ตัวอย่างที่ 44: Partial index (เฉพาะบาง rows)
CREATE INDEX idx_orders_active_customers 
ON orders(customer_id)
WHERE status NOT IN ('cancelled', 'refunded');

-- ตัวอย่างที่ 45: สร้าง partial unique index
CREATE UNIQUE INDEX idx_users_active_email
ON users(email)
WHERE deleted_at IS NULL;
-- อนุญาตให้ email ซ้ำได้ถ้า user ถูก soft-delete แล้ว
```

### Index Maintenance Scripts

```sql
-- Script 1: ตรวจสอบ indexes ที่ไม่ถูกใช้และใหญ่
-- ควรรัน monthly เพื่อ cleanup
SELECT 
    schemaname || '.' || tablename AS full_table_name,
    indexname,
    pg_size_pretty(pg_relation_size(indexrelid)) AS index_size,
    idx_scan AS times_used,
    idx_tup_read AS tuples_read
FROM pg_stat_user_indexes
LEFT JOIN pg_index ON pg_stat_user_indexes.indexrelid = pg_index.indexrelid
WHERE idx_scan = 0
AND NOT indisprimary
AND NOT indisunique
AND pg_relation_size(pg_stat_user_indexes.indexrelid) > 1024 * 1024  -- > 1MB
ORDER BY pg_relation_size(pg_stat_user_indexes.indexrelid) DESC;
```

```sql
-- Script 2: ตรวจสอบ duplicate indexes
-- indexes ที่มีคอลัมน์ซ้ำกัน
WITH index_columns AS (
    SELECT 
        i.indexrelid,
        i.indrelid,
        t.relname AS table_name,
        ix.relname AS index_name,
        array_agg(a.attname ORDER BY array_position(i.indkey, a.attnum)) AS columns
    FROM pg_index i
    JOIN pg_class t ON t.oid = i.indrelid
    JOIN pg_class ix ON ix.oid = i.indexrelid
    JOIN pg_attribute a ON a.attrelid = t.oid AND a.attnum = ANY(i.indkey)
    WHERE t.relkind = 'r'
    GROUP BY i.indexrelid, i.indrelid, t.relname, ix.relname
)
SELECT 
    a.table_name,
    a.index_name AS index1,
    b.index_name AS index2,
    a.columns
FROM index_columns a
JOIN index_columns b ON a.indrelid = b.indrelid 
    AND a.indexrelid < b.indexrelid
    AND a.columns = b.columns
ORDER BY a.table_name;
```

```sql
-- Script 3: ขนาด indexes ทั้งหมดแยกตามตาราง
SELECT 
    tablename,
    pg_size_pretty(pg_total_relation_size(quote_ident(tablename))) AS total_size,
    pg_size_pretty(pg_relation_size(quote_ident(tablename))) AS table_size,
    pg_size_pretty(pg_total_relation_size(quote_ident(tablename)) - 
                   pg_relation_size(quote_ident(tablename))) AS index_size,
    ROUND((pg_total_relation_size(quote_ident(tablename)) - 
           pg_relation_size(quote_ident(tablename)))::numeric /
           pg_total_relation_size(quote_ident(tablename)) * 100, 2) AS index_pct
FROM pg_tables
WHERE schemaname = 'public'
ORDER BY pg_total_relation_size(quote_ident(tablename)) DESC
LIMIT 20;
```

---

## 10. ตัวอย่างที่ครอบคลุม: 30+ Index Creation Examples

### E-commerce Database Indexes

```sql
-- ตัวอย่างที่ 46-55: indexes สำหรับระบบ E-commerce

-- Table: customers
CREATE TABLE customers (
    customer_id SERIAL PRIMARY KEY,
    email VARCHAR(200) UNIQUE NOT NULL,
    phone VARCHAR(20),
    first_name VARCHAR(50),
    last_name VARCHAR(50),
    city VARCHAR(100),
    country VARCHAR(50),
    created_at TIMESTAMP DEFAULT NOW(),
    is_active BOOLEAN DEFAULT TRUE
);

-- Indexes สำหรับ customers
CREATE INDEX idx_customers_name ON customers(last_name, first_name);
CREATE INDEX idx_customers_phone ON customers(phone) WHERE phone IS NOT NULL;
CREATE INDEX idx_customers_city_country ON customers(country, city);
CREATE INDEX idx_customers_created_at ON customers(created_at);
CREATE INDEX idx_customers_active ON customers(created_at) WHERE is_active = TRUE;

-- Table: products
CREATE TABLE products (
    product_id SERIAL PRIMARY KEY,
    sku VARCHAR(50) UNIQUE NOT NULL,
    name VARCHAR(200) NOT NULL,
    category_id INT REFERENCES categories(category_id),
    brand_id INT REFERENCES brands(brand_id),
    price DECIMAL(10,2),
    stock_quantity INT DEFAULT 0,
    is_active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP DEFAULT NOW()
);

-- Indexes สำหรับ products
CREATE INDEX idx_products_category ON products(category_id);
CREATE INDEX idx_products_brand ON products(brand_id);
CREATE INDEX idx_products_price ON products(price);
CREATE INDEX idx_products_category_price ON products(category_id, price);
CREATE INDEX idx_products_active_stock ON products(stock_quantity) 
    WHERE is_active = TRUE AND stock_quantity > 0;
CREATE INDEX idx_products_name_search ON products(LOWER(name));

-- Table: orders
CREATE TABLE orders (
    order_id SERIAL PRIMARY KEY,
    customer_id INT REFERENCES customers(customer_id),
    order_date TIMESTAMP DEFAULT NOW(),
    status VARCHAR(20) DEFAULT 'pending',
    total_amount DECIMAL(12,2),
    shipping_address_id INT
);

-- Indexes สำหรับ orders
CREATE INDEX idx_orders_customer ON orders(customer_id);
CREATE INDEX idx_orders_date ON orders(order_date);
CREATE INDEX idx_orders_customer_date ON orders(customer_id, order_date DESC);
CREATE INDEX idx_orders_status ON orders(status) WHERE status IN ('pending', 'processing');
CREATE INDEX idx_orders_date_status ON orders(order_date, status);

-- Table: order_items
CREATE TABLE order_items (
    item_id SERIAL PRIMARY KEY,
    order_id INT REFERENCES orders(order_id),
    product_id INT REFERENCES products(product_id),
    quantity INT,
    unit_price DECIMAL(10,2)
);

-- Indexes สำหรับ order_items
CREATE INDEX idx_order_items_order ON order_items(order_id);
CREATE INDEX idx_order_items_product ON order_items(product_id);
CREATE INDEX idx_order_items_order_product ON order_items(order_id, product_id);
```

### Analytics Database Indexes

```sql
-- ตัวอย่างที่ 56-65: indexes สำหรับ Analytics

-- Table: page_views (time-series data)
CREATE TABLE page_views (
    view_id BIGSERIAL PRIMARY KEY,
    session_id VARCHAR(50),
    user_id INT,
    page_url TEXT,
    viewed_at TIMESTAMP NOT NULL,
    duration_seconds INT,
    country_code CHAR(2)
);

-- Indexes สำหรับ analytics queries
CREATE INDEX idx_page_views_session ON page_views(session_id);
CREATE INDEX idx_page_views_user ON page_views(user_id) WHERE user_id IS NOT NULL;
CREATE INDEX idx_page_views_time ON page_views(viewed_at);
CREATE INDEX idx_page_views_country_time ON page_views(country_code, viewed_at);

-- BRIN index สำหรับ time-series (ประหยัดพื้นที่มาก)
CREATE INDEX idx_page_views_time_brin ON page_views USING BRIN(viewed_at);
```

### HR Database Indexes

```sql
-- ตัวอย่างที่ 66-75: indexes สำหรับ HR System

-- Table: employees (ขยายจากตัวอย่างก่อน)
CREATE TABLE employees (
    emp_id SERIAL PRIMARY KEY,
    emp_number VARCHAR(20) UNIQUE,
    first_name VARCHAR(50),
    last_name VARCHAR(50),
    email VARCHAR(200) UNIQUE,
    phone VARCHAR(20),
    department_id INT,
    manager_id INT REFERENCES employees(emp_id),
    hire_date DATE,
    salary DECIMAL(10,2),
    status VARCHAR(20) DEFAULT 'active',
    birth_date DATE
);

-- Indexes สำหรับ HR queries
CREATE INDEX idx_emp_name ON employees(last_name, first_name);
CREATE INDEX idx_emp_department ON employees(department_id);
CREATE INDEX idx_emp_manager ON employees(manager_id);
CREATE INDEX idx_emp_hire_date ON employees(hire_date);
CREATE INDEX idx_emp_status_dept ON employees(status, department_id) 
    WHERE status = 'active';
CREATE INDEX idx_emp_salary_range ON employees(salary, department_id);

-- สำหรับ case-insensitive search
CREATE INDEX idx_emp_lower_email ON employees(LOWER(email));
CREATE INDEX idx_emp_lower_name ON employees(LOWER(last_name), LOWER(first_name));
```

---

## 11. Index Management Scripts สมบูรณ์

```sql
-- Master Script: Index Health Report
DO $$
DECLARE
    v_table TEXT;
BEGIN
    RAISE NOTICE 'Starting Index Health Report';
    RAISE NOTICE '================================';
    
    -- ตรวจสอบ indexes ที่ไม่ถูกใช้
    RAISE NOTICE 'Unused Indexes (never scanned):';
    FOR v_table IN 
        SELECT indexname || ' on ' || tablename || ' (' || 
               pg_size_pretty(pg_relation_size(indexrelid)) || ')'
        FROM pg_stat_user_indexes
        LEFT JOIN pg_index ON pg_stat_user_indexes.indexrelid = pg_index.indexrelid
        WHERE idx_scan = 0
        AND NOT indisprimary
        AND NOT indisunique
        LOOP
        RAISE NOTICE '  - %', v_table;
    END LOOP;
END $$;
```

```sql
-- สร้าง Script สำหรับ Generate DROP statements ของ unused indexes
SELECT 
    'DROP INDEX CONCURRENTLY IF EXISTS ' || indexname || ';' AS drop_statement,
    tablename,
    pg_size_pretty(pg_relation_size(indexrelid)) AS index_size,
    idx_scan AS times_used
FROM pg_stat_user_indexes
LEFT JOIN pg_index ON pg_stat_user_indexes.indexrelid = pg_index.indexrelid
WHERE idx_scan = 0
AND NOT indisprimary
AND NOT indisunique
AND pg_relation_size(indexrelid) > 10 * 1024 * 1024  -- > 10MB
ORDER BY pg_relation_size(indexrelid) DESC;
```

```sql
-- Script: แสดงสถานะ indexes ทั้งหมดพร้อม health score
SELECT 
    t.relname AS table_name,
    i.relname AS index_name,
    ix.indisprimary AS is_pk,
    ix.indisunique AS is_unique,
    ix.indisvalid AS is_valid,
    s.idx_scan AS times_used,
    pg_size_pretty(pg_relation_size(i.oid)) AS index_size,
    CASE 
        WHEN NOT ix.indisvalid THEN 'INVALID - needs rebuild'
        WHEN s.idx_scan = 0 AND NOT ix.indisprimary AND NOT ix.indisunique 
            THEN 'UNUSED - consider dropping'
        WHEN s.idx_scan < 10 THEN 'RARELY USED'
        WHEN s.idx_scan < 100 THEN 'OCCASIONALLY USED'
        ELSE 'ACTIVELY USED'
    END AS health_status
FROM pg_class t
JOIN pg_index ix ON t.oid = ix.indrelid
JOIN pg_class i ON i.oid = ix.indexrelid
LEFT JOIN pg_stat_user_indexes s ON s.indexrelid = i.oid
WHERE t.relkind = 'r'
AND t.relname NOT LIKE 'pg_%'
ORDER BY t.relname, i.relname;
```

---

## 12. Monitoring Index Usage ใน Production

```sql
-- ตรวจสอบ index cache hit ratio
SELECT 
    sum(idx_blks_hit) AS cache_hits,
    sum(idx_blks_read) AS disk_reads,
    ROUND(sum(idx_blks_hit)::numeric / 
          NULLIF(sum(idx_blks_hit) + sum(idx_blks_read), 0) * 100, 2) 
          AS cache_hit_ratio_pct
FROM pg_statio_user_indexes;
-- ควรได้ > 95%
```

```sql
-- Top indexes โดย scan count (ที่ถูกใช้มากที่สุด)
SELECT 
    schemaname,
    tablename,
    indexname,
    idx_scan,
    idx_tup_read,
    idx_tup_fetch,
    pg_size_pretty(pg_relation_size(indexrelid)) AS size
FROM pg_stat_user_indexes
ORDER BY idx_scan DESC
LIMIT 20;
```

---

## แบบฝึกหัด (10 ข้อ)

**ข้อ 1:** สร้าง index บนตาราง `blog_posts` (post_id, author_id, category_id, published_at, status, title, content) สำหรับ query ต่อไปนี้:
```sql
SELECT * FROM blog_posts WHERE author_id = 5 AND status = 'published' ORDER BY published_at DESC;
```

**เฉลยข้อ 1:**
```sql
-- Composite index ที่ดีที่สุดสำหรับ query นี้
CREATE INDEX idx_blog_posts_author_status_date 
ON blog_posts(author_id, status, published_at DESC);

-- Rationale:
-- 1. author_id (equality) → ต้องเป็น column แรก
-- 2. status (equality) → เป็น column ที่สอง  
-- 3. published_at DESC → sort order ตรงกับ ORDER BY
```

---

**ข้อ 2:** เขียน script เพื่อแสดง index ทั้งหมดที่ขนาดใหญ่กว่า 100MB ใน PostgreSQL

**เฉลยข้อ 2:**
```sql
SELECT 
    schemaname || '.' || tablename AS table,
    indexname,
    pg_size_pretty(pg_relation_size(indexrelid)) AS index_size,
    idx_scan AS times_used
FROM pg_stat_user_indexes
WHERE pg_relation_size(indexrelid) > 100 * 1024 * 1024  -- 100MB
ORDER BY pg_relation_size(indexrelid) DESC;
```

---

**ข้อ 3:** ทำไมต้องใช้ `CREATE INDEX CONCURRENTLY` แทน `CREATE INDEX` ปกติใน production? มีข้อเสียอะไรบ้าง?

**เฉลยข้อ 3:**
ใช้ CONCURRENTLY เพราะ:
- CREATE INDEX ปกติ lock table ไม่ให้ write ได้ระหว่างสร้าง → downtime
- CONCURRENTLY ไม่ lock table → application ยังทำงานได้

ข้อเสีย CONCURRENTLY:
1. ใช้เวลานานกว่า (2x) เพราะต้องสแกน table หลายรอบ
2. ใช้ resources มากกว่า
3. ไม่ทำงานใน transaction block
4. ถ้า interrupt ระหว่างสร้าง จะเกิด INVALID index ต้องล้างเอง

---

**ข้อ 4:** สร้าง UNIQUE index ที่อนุญาตให้ email ซ้ำได้ถ้า account ถูก soft-delete (deleted_at IS NOT NULL)

**เฉลยข้อ 4:**
```sql
-- Partial unique index
CREATE UNIQUE INDEX uidx_users_active_email
ON users(email)
WHERE deleted_at IS NULL;

-- ตรวจสอบ: users ที่ถูก soft-delete สามารถมี email ซ้ำได้
-- แต่ users ที่ active ต้องมี email ไม่ซ้ำ
```

---

**ข้อ 5:** เขียน query เพื่อหา indexes ที่มีขนาดใหญ่กว่าตารางหลัก (index_size > table_size)

**เฉลยข้อ 5:**
```sql
SELECT 
    t.relname AS table_name,
    pg_size_pretty(pg_relation_size(t.oid)) AS table_size,
    pg_size_pretty(SUM(pg_relation_size(i.indexrelid))) AS total_index_size,
    COUNT(*) AS index_count
FROM pg_class t
JOIN pg_index i ON t.oid = i.indrelid
WHERE t.relkind = 'r'
GROUP BY t.relname, t.oid
HAVING SUM(pg_relation_size(i.indexrelid)) > pg_relation_size(t.oid)
ORDER BY SUM(pg_relation_size(i.indexrelid)) DESC;
```

---

**ข้อ 6:** ตาราง `logs` มีข้อมูล 5 ปีย้อนหลัง (100 ล้านแถว) ควรใช้ fill factor เท่าไหร่ สำหรับ index บน `created_at` และทำไม?

**เฉลยข้อ 6:**
ควรใช้ fill factor = 90-100% เพราะ:
- Log table ส่วนใหญ่เป็น append-only (INSERT แต่ไม่ UPDATE/DELETE เก่า)
- created_at ใหม่จะถูก insert ที่ท้าย B-tree เสมอ (monotonically increasing)
- ไม่เกิด page splits ตรงกลาง B-tree
- Fill factor สูงหมายถึงใช้พื้นที่น้อยกว่า

```sql
CREATE INDEX idx_logs_created_at ON logs(created_at) 
WITH (fillfactor = 95);
```

---

**ข้อ 7:** สร้าง indexes สำหรับตาราง `inventory` ที่มี columns: product_id, warehouse_id, quantity, last_updated สำหรับ use cases: (1) ดูสินค้าคงคลังทั้งหมดของ warehouse (2) ดูสินค้าที่ stock ต่ำกว่า 10 (3) ดูการ update ล่าสุด

**เฉลยข้อ 7:**
```sql
-- 1. ดูสินค้าคงคลังของ warehouse
CREATE INDEX idx_inventory_warehouse ON inventory(warehouse_id, product_id);

-- 2. สินค้า stock ต่ำ (partial index)
CREATE INDEX idx_inventory_low_stock ON inventory(product_id, warehouse_id)
WHERE quantity < 10;

-- 3. ดูการ update ล่าสุด
CREATE INDEX idx_inventory_last_updated ON inventory(last_updated DESC);
```

---

**ข้อ 8:** ทำไม MySQL จึงสร้าง index บน foreign key อัตโนมัติ ต่างจาก PostgreSQL? มีผลอะไรกับ performance?

**เฉลยข้อ 8:**
MySQL สร้าง FK index อัตโนมัติเพราะ InnoDB ต้องใช้ index เพื่อ:
- ตรวจสอบ parent record ก่อน DELETE/UPDATE (referential integrity check)
- ถ้าไม่มี index MySQL จะ lock parent table ทั้งหมด

PostgreSQL ไม่บังคับเพราะ:
- ใช้ row-level locking + MVCC แทน
- DBA ควรตัดสินใจเองว่า FK ไหนควร index

ผล performance:
- MySQL: FK indexes อาจ overhead สูงสำหรับ tables ที่ไม่ค่อย query FK
- PostgreSQL: ถ้าลืมสร้าง FK index จะทำให้ CASCADE DELETE ช้ามาก

---

**ข้อ 9:** เขียน query เพื่อแสดง "index health" ของทุก indexes รวมถึง validity, usage frequency, และ size

**เฉลยข้อ 9:**
```sql
SELECT 
    t.relname AS table_name,
    i.relname AS index_name,
    ix.indisprimary AS is_pk,
    ix.indisunique AS is_unique,
    ix.indisvalid AS is_valid,
    COALESCE(s.idx_scan, 0) AS scans,
    pg_size_pretty(pg_relation_size(i.oid)) AS size,
    CASE 
        WHEN NOT ix.indisvalid THEN 'CRITICAL: Invalid'
        WHEN COALESCE(s.idx_scan, 0) = 0 AND NOT ix.indisprimary THEN 'WARNING: Unused'
        WHEN COALESCE(s.idx_scan, 0) < 50 THEN 'INFO: Low usage'
        ELSE 'OK: Active'
    END AS health
FROM pg_class t
JOIN pg_index ix ON t.oid = ix.indrelid
JOIN pg_class i ON i.oid = ix.indexrelid
LEFT JOIN pg_stat_user_indexes s ON s.indexrelid = i.oid
WHERE t.relkind = 'r'
AND t.relname NOT LIKE 'pg_%'
ORDER BY health, pg_relation_size(i.oid) DESC;
```

---

**ข้อ 10:** สร้าง complete index strategy สำหรับตาราง `transactions` ที่มี 500 ล้านแถว columns: trans_id, account_id, trans_type, amount, currency, created_at, status, merchant_id

**เฉลยข้อ 10:**
```sql
-- Strategy สำหรับ 500 ล้านแถว:

-- 1. Primary Key (auto)
-- trans_id SERIAL PRIMARY KEY

-- 2. ค้นหา transactions ของ account (most common)
CREATE INDEX idx_trans_account_date 
ON transactions(account_id, created_at DESC)
WITH (fillfactor = 90);

-- 3. Pending/processing transactions (operational)
CREATE INDEX idx_trans_pending 
ON transactions(account_id, created_at DESC)
WHERE status IN ('pending', 'processing');
-- Small partial index - very fast!

-- 4. Merchant analytics
CREATE INDEX idx_trans_merchant_date
ON transactions(merchant_id, created_at)
WITH (fillfactor = 90);

-- 5. Range queries on amount (reporting)
CREATE INDEX idx_trans_amount_range
ON transactions(currency, amount, created_at)
WHERE status = 'completed';

-- 6. BRIN index สำหรับ time-series reporting
CREATE INDEX idx_trans_created_brin
ON transactions USING BRIN(created_at)
WITH (pages_per_range = 128);
-- ประหยัดพื้นที่มาก เหมาะสำหรับ sequential data

-- Note: Consider PARTITIONING ด้วย (Part 069)
```

---

## สรุป

ใน Part 062 เราได้เรียนรู้:

1. **CREATE INDEX syntax** สำหรับทุก database
2. **CREATE UNIQUE INDEX** และความแตกต่างจาก constraint
3. **DROP INDEX** และ IF EXISTS
4. **Index naming conventions** ที่เป็นมาตรฐาน
5. **Listing indexes** ด้วย system views
6. **REINDEX** และ OPTIMIZE TABLE
7. **Fill Factor** และเมื่อใดควรใช้
8. **CREATE INDEX CONCURRENTLY** สำหรับ production
9. **Index management scripts** สำหรับ monitoring และ cleanup
10. **30+ ตัวอย่าง** จากระบบจริงๆ

ใน Part 063 เราจะเรียนรู้เกี่ยวกับ Index Types ที่หลากหลาย เช่น GIN, GiST, BRIN และ Full-text indexes
