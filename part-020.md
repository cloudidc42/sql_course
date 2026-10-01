# Part 20: Data Import and Export

## บทนำ

การ Import/Export ข้อมูลเป็นทักษะสำคัญสำหรับ:
- ย้ายข้อมูลระหว่างระบบ (Data Migration)
- สำรองข้อมูล (Backup/Restore)
- แชร์ข้อมูลกับ tools อื่น (Reporting, Analytics)
- Load ข้อมูลจำนวนมาก (Bulk Loading)

---

## 1. PostgreSQL COPY Command

```sql
-- ตัวอย่าง 1: COPY - Export ไป CSV
/*
-- Export ทั้งตาราง
COPY employees TO '/tmp/employees.csv' WITH (FORMAT CSV, HEADER TRUE);

-- Export พร้อม delimiter เฉพาะ
COPY employees TO '/tmp/employees.tsv' WITH (FORMAT CSV, HEADER TRUE, DELIMITER '\t');

-- Export จาก query
COPY (
    SELECT e.employee_id, e.first_name, e.last_name, d.department_name
    FROM employees e
    JOIN departments d ON e.department_id = d.department_id
    WHERE e.is_active = TRUE
) TO '/tmp/active_employees.csv' WITH (FORMAT CSV, HEADER TRUE);

-- Export ไป pipe-delimited
COPY products TO '/tmp/products.psv' WITH (FORMAT CSV, HEADER TRUE, DELIMITER '|');

-- ตัวอย่าง 2: COPY - Import จาก CSV
COPY employees (first_name, last_name, email, hire_date, salary)
FROM '/tmp/new_employees.csv' 
WITH (FORMAT CSV, HEADER TRUE, NULL 'NULL');

-- Import พร้อม error handling
COPY employees 
FROM '/tmp/employees.csv' 
WITH (FORMAT CSV, HEADER TRUE, NULL '', QUOTE '"');

-- ตัวอย่าง 3: \copy ใน psql (client-side)
-- แตกต่างจาก COPY ตรงที่ทำจาก client ไม่ใช่ server
-- ไม่ต้องการ superuser privileges
-- \copy employees TO '/home/user/employees.csv' CSV HEADER
-- \copy employees FROM '/home/user/new_employees.csv' CSV HEADER

-- ตัวอย่าง 4: COPY กับ JSON
COPY (SELECT row_to_json(e) FROM employees e) 
TO '/tmp/employees.json';
*/
```

---

## 2. MySQL LOAD DATA INFILE

```sql
-- ตัวอย่าง 5: LOAD DATA INFILE พื้นฐาน
LOAD DATA INFILE '/tmp/employees.csv'
INTO TABLE employees
FIELDS TERMINATED BY ','
ENCLOSED BY '"'
LINES TERMINATED BY '\n'
IGNORE 1 ROWS;  -- ข้าม header row

-- ตัวอย่าง 6: LOAD DATA กับ column mapping
LOAD DATA INFILE '/tmp/products.csv'
INTO TABLE products
FIELDS TERMINATED BY ','
ENCLOSED BY '"'
LINES TERMINATED BY '\n'
IGNORE 1 ROWS
(product_name, category, @price_str, stock_quantity)
SET price = CAST(@price_str AS DECIMAL(10,2)),
    created_at = NOW();

-- ตัวอย่าง 7: LOAD DATA LOCAL INFILE (จาก client)
-- ต้องเปิด local_infile ก่อน
SET GLOBAL local_infile = 1;

LOAD DATA LOCAL INFILE '/home/user/customers.csv'
INTO TABLE customers
FIELDS TERMINATED BY ','
ENCLOSED BY '"'
LINES TERMINATED BY '\r\n'
IGNORE 1 ROWS
(first_name, last_name, email, phone, city);

-- ตัวอย่าง 8: LOAD DATA พร้อม error handling
LOAD DATA INFILE '/tmp/data.csv'
INTO TABLE staging_table
FIELDS TERMINATED BY ','
IGNORE 1 ROWS;

-- ดู warnings
SHOW WARNINGS;
SHOW ERRORS;

-- ตัวอย่าง 9: LOAD DATA กับ Tab-delimited
LOAD DATA INFILE '/tmp/data.tsv'
INTO TABLE products
FIELDS TERMINATED BY '\t'
LINES TERMINATED BY '\n'
IGNORE 1 ROWS;
```

---

## 3. Exporting ด้วย SELECT INTO OUTFILE (MySQL)

```sql
-- ตัวอย่าง 10: SELECT INTO OUTFILE
SELECT * 
INTO OUTFILE '/tmp/employees_export.csv'
FIELDS TERMINATED BY ','
ENCLOSED BY '"'
LINES TERMINATED BY '\n'
FROM employees;

-- ตัวอย่าง 11: Export พร้อม Header
-- MySQL ไม่รองรับ HEADER โดยตรง ต้องใช้ UNION
(SELECT 'employee_id', 'first_name', 'last_name', 'email', 'salary', 'department')
UNION ALL
(SELECT 
    CAST(e.employee_id AS CHAR),
    e.first_name,
    e.last_name,
    e.email,
    CAST(e.salary AS CHAR),
    d.department_name
FROM employees e
LEFT JOIN departments d ON e.department_id = d.department_id)
INTO OUTFILE '/tmp/employees_with_header.csv'
FIELDS TERMINATED BY ','
ENCLOSED BY '"'
LINES TERMINATED BY '\n';

-- ตัวอย่าง 12: Export ข้อมูลที่ filter แล้ว
SELECT 
    order_id,
    customer_id,
    DATE_FORMAT(order_date, '%Y-%m-%d') AS order_date,
    status,
    total_amount
INTO OUTFILE '/tmp/orders_2024.csv'
FIELDS TERMINATED BY ','
ENCLOSED BY '"'
LINES TERMINATED BY '\n'
FROM orders
WHERE YEAR(order_date) = 2024
ORDER BY order_date;
```

---

## 4. mysqldump - MySQL Backup

```sql
-- ตัวอย่าง 13: mysqldump พื้นฐาน (command line)
-- Dump ทั้งฐานข้อมูล
-- mysqldump -u root -p ecommerce > ecommerce_backup.sql

-- Dump เฉพาะตาราง
-- mysqldump -u root -p ecommerce employees orders > tables_backup.sql

-- Dump โครงสร้างอย่างเดียว
-- mysqldump -u root -p --no-data ecommerce > structure_only.sql

-- Dump ข้อมูลอย่างเดียว (ไม่มี CREATE TABLE)
-- mysqldump -u root -p --no-create-info ecommerce > data_only.sql

-- ตัวอย่าง 14: mysqldump options ที่ใช้บ่อย
/*
mysqldump \
  -u root \
  -p \
  --host=localhost \
  --port=3306 \
  --single-transaction \        # ใช้ transaction (ไม่ lock tables)
  --routines \                   # รวม stored procedures
  --triggers \                   # รวม triggers
  --events \                     # รวม events
  --hex-blob \                   # binary data เป็น hex
  --compress \                   # compress output
  ecommerce \                    # ชื่อ database
  > /backup/ecommerce_$(date +%Y%m%d).sql
*/

-- ตัวอย่าง 15: Restore จาก dump
-- mysql -u root -p ecommerce < ecommerce_backup.sql
-- หรือ
-- mysql -u root -p -e "SOURCE /path/to/backup.sql" ecommerce

-- ตัวอย่าง 16: Compressed backup
-- mysqldump -u root -p ecommerce | gzip > ecommerce_backup.sql.gz
-- Restore: gunzip < ecommerce_backup.sql.gz | mysql -u root -p ecommerce
```

---

## 5. pg_dump - PostgreSQL Backup

```sql
-- ตัวอย่าง 17: pg_dump พื้นฐาน
/*
-- Plain SQL format
pg_dump -U postgres -d ecommerce -F p -f ecommerce_backup.sql

-- Custom format (สามารถ restore เฉพาะบางตารางได้)
pg_dump -U postgres -d ecommerce -F c -f ecommerce_backup.dump

-- Directory format (parallel restore)
pg_dump -U postgres -d ecommerce -F d -f /backup/ecommerce/

-- Dump เฉพาะตาราง
pg_dump -U postgres -d ecommerce -t employees -t orders -f selected_tables.sql

-- Dump schema เท่านั้น
pg_dump -U postgres -d ecommerce --schema-only -f schema_only.sql

-- Dump ข้อมูลเท่านั้น
pg_dump -U postgres -d ecommerce --data-only -f data_only.sql

-- ตัวอย่าง 18: pg_restore
-- Restore custom format
pg_restore -U postgres -d ecommerce_new -F c ecommerce_backup.dump

-- Restore เฉพาะตาราง
pg_restore -U postgres -d ecommerce -t employees ecommerce_backup.dump

-- Restore parallel
pg_restore -U postgres -d ecommerce -F d -j 4 /backup/ecommerce/

-- ตัวอย่าง 19: pg_dumpall - Dump ทุก databases
pg_dumpall -U postgres > all_databases.sql
*/
```

---

## 6. SQLite Import/Export

```sql
-- ตัวอย่าง 20: SQLite .import command
/*
-- ใน sqlite3 command line
sqlite3 mydb.db

-- Import CSV
.mode csv
.import /path/to/data.csv employees

-- Import TSV
.mode tabs
.import /path/to/data.tsv products

-- ตั้งค่า CSV และข้าม header
.headers on
.mode csv
.separator ","

-- Import ด้วยการข้าม header
CREATE TABLE temp_import (col1 TEXT, col2 TEXT, col3 TEXT);
.import --skip 1 /path/to/data.csv temp_import

-- ตัวอย่าง 21: SQLite export
-- Export ไป CSV
.headers on
.mode csv
.output /path/to/output.csv
SELECT * FROM employees;
.output stdout  -- กลับมาแสดงบน screen

-- Dump database
.dump  -- แสดง SQL สำหรับ recreate database
.output /path/to/backup.sql
.dump
.output stdout

-- Export เป็น SQL
sqlite3 mydb.db .dump > backup.sql

-- Restore
sqlite3 newdb.db < backup.sql
*/
```

---

## 7. Importing CSV ด้วย SQL

```sql
-- ตัวอย่าง 22: Import CSV แบบ manual (ผ่าน staging table)
-- Step 1: สร้าง staging table
CREATE TEMPORARY TABLE staging_employees (
    raw_id          VARCHAR(20),
    raw_first_name  VARCHAR(200),
    raw_last_name   VARCHAR(200),
    raw_email       VARCHAR(300),
    raw_hire_date   VARCHAR(50),
    raw_salary      VARCHAR(50),
    raw_department  VARCHAR(200)
);

-- Step 2: Load ข้อมูล raw
LOAD DATA INFILE '/tmp/employees_import.csv'
INTO TABLE staging_employees
FIELDS TERMINATED BY ','
ENCLOSED BY '"'
LINES TERMINATED BY '\n'
IGNORE 1 ROWS;

-- Step 3: ตรวจสอบข้อมูล
SELECT 
    COUNT(*) AS total_rows,
    SUM(CASE WHEN raw_email LIKE '%@%.%' THEN 0 ELSE 1 END) AS invalid_emails,
    SUM(CASE WHEN raw_salary REGEXP '^[0-9]+\\.?[0-9]*$' THEN 0 ELSE 1 END) AS invalid_salary,
    SUM(CASE WHEN STR_TO_DATE(raw_hire_date, '%Y-%m-%d') IS NULL THEN 1 ELSE 0 END) AS invalid_dates
FROM staging_employees;

-- Step 4: Insert ข้อมูลที่ valid
INSERT INTO employees (first_name, last_name, email, hire_date, salary, department_id)
SELECT 
    TRIM(raw_first_name),
    TRIM(raw_last_name),
    LOWER(TRIM(raw_email)),
    STR_TO_DATE(raw_hire_date, '%Y-%m-%d'),
    CAST(raw_salary AS DECIMAL(10,2)),
    d.department_id
FROM staging_employees s
LEFT JOIN departments d ON d.department_name = TRIM(s.raw_department)
WHERE raw_email LIKE '%@%.%'
  AND raw_salary REGEXP '^[0-9]+\\.?[0-9]*$'
  AND STR_TO_DATE(raw_hire_date, '%Y-%m-%d') IS NOT NULL;

-- Step 5: ดูผลลัพธ์
SELECT ROW_COUNT() AS rows_imported;
```

---

## 8. JSON Import/Export

```sql
-- ตัวอย่าง 23: Export เป็น JSON (MySQL)
-- Export เป็น JSON array
SELECT JSON_ARRAYAGG(
    JSON_OBJECT(
        'employee_id', employee_id,
        'name', CONCAT(first_name, ' ', last_name),
        'email', email,
        'salary', salary,
        'department_id', department_id
    )
) AS employees_json
FROM employees
WHERE is_active = TRUE;

-- ตัวอย่าง 24: Export nested JSON
SELECT JSON_ARRAYAGG(
    JSON_OBJECT(
        'department_id', d.department_id,
        'department_name', d.department_name,
        'employee_count', COUNT(e.employee_id),
        'total_salary', SUM(e.salary),
        'employees', (
            SELECT JSON_ARRAYAGG(
                JSON_OBJECT('id', e2.employee_id, 'name', CONCAT(e2.first_name, ' ', e2.last_name))
            )
            FROM employees e2
            WHERE e2.department_id = d.department_id
        )
    )
) AS departments_json
FROM departments d
LEFT JOIN employees e ON d.department_id = e.department_id
GROUP BY d.department_id, d.department_name;

-- ตัวอย่าง 25: Import จาก JSON (MySQL 8.0+)
CREATE TABLE json_import_staging (
    json_data JSON
);

-- Insert JSON string
INSERT INTO json_import_staging VALUES
('{"id": 1, "name": "Product A", "price": 99.99}'),
('{"id": 2, "name": "Product B", "price": 149.99}');

-- แปลง JSON ไป rows
INSERT INTO products (product_id, product_name, price)
SELECT 
    json_data->>'$.id',
    json_data->>'$.name',
    json_data->>'$.price'
FROM json_import_staging;

-- ตัวอย่าง 26: Import JSON array
SET @json_data = '[
    {"name": "iPhone 15", "price": 42900, "stock": 50},
    {"name": "Samsung S24", "price": 35900, "stock": 30},
    {"name": "MacBook Air", "price": 49900, "stock": 20}
]';

INSERT INTO products (product_name, price, stock_quantity)
SELECT 
    JSON_UNQUOTE(JSON_EXTRACT(item.value, '$.name')),
    JSON_EXTRACT(item.value, '$.price'),
    JSON_EXTRACT(item.value, '$.stock')
FROM JSON_TABLE(@json_data, '$[*]' COLUMNS (
    value JSON PATH '$'
)) AS item;
```

---

## 9. Data Migration ระหว่าง Databases

```sql
-- ตัวอย่าง 27: Migration script template
-- File: migrate_v1_to_v2.sql

-- 1. สร้าง tables ใหม่ใน v2 format
CREATE TABLE v2_customers (
    customer_id     INT UNSIGNED PRIMARY KEY,
    email           VARCHAR(254) NOT NULL,
    full_name       VARCHAR(200),  -- v2 รวม first+last
    phone           VARCHAR(20),
    city            VARCHAR(100),
    tier            VARCHAR(20) DEFAULT 'Bronze',
    created_at      TIMESTAMP
);

-- 2. Migrate ข้อมูลจาก v1 -> v2
INSERT INTO v2_customers (customer_id, email, full_name, phone, city, created_at)
SELECT 
    customer_id,
    email,
    CONCAT(first_name, ' ', last_name) AS full_name,  -- รวม name
    phone,
    city,
    created_at
FROM customers;  -- v1 table

-- 3. ตรวจสอบ
SELECT COUNT(*) AS v1_count FROM customers;
SELECT COUNT(*) AS v2_count FROM v2_customers;
-- ต้องเท่ากัน

-- 4. Verify data
SELECT 
    c.customer_id,
    CONCAT(c.first_name, ' ', c.last_name) AS v1_name,
    v2.full_name AS v2_name
FROM customers c
JOIN v2_customers v2 ON c.customer_id = v2.customer_id
WHERE CONCAT(c.first_name, ' ', c.last_name) != v2.full_name;
-- ควรไม่มีผล

-- ตัวอย่าง 28: ETL (Extract, Transform, Load) Pipeline
-- Extract
CREATE TEMPORARY TABLE etl_extract AS
SELECT 
    o.order_id,
    o.order_date,
    o.total_amount,
    o.status,
    c.email AS customer_email,
    c.city AS customer_city,
    COUNT(oi.item_id) AS item_count,
    SUM(oi.quantity) AS total_units
FROM orders o
JOIN customers c ON o.customer_id = c.customer_id
LEFT JOIN order_items oi ON o.order_id = oi.order_id
WHERE o.order_date >= '2024-01-01'
GROUP BY o.order_id, o.order_date, o.total_amount, o.status, c.email, c.city;

-- Transform
ALTER TABLE etl_extract
ADD COLUMN order_month VARCHAR(7),
ADD COLUMN is_large_order BOOLEAN;

UPDATE etl_extract
SET 
    order_month = DATE_FORMAT(order_date, '%Y-%m'),
    is_large_order = (total_amount > 10000);

-- Load
INSERT INTO data_warehouse.fact_orders
    (order_id, order_date, order_month, customer_email, customer_city,
     total_amount, status, item_count, total_units, is_large_order)
SELECT * FROM etl_extract;
```

---

## 10. Practical Import Scripts

```sql
-- ตัวอย่าง 29: Script Import ข้อมูลสินค้าจาก Excel (CSV)
-- สมมติไฟล์ products.csv มีโครงสร้าง:
-- SKU,Product Name,Category,Price,Cost,Stock,Description

CREATE TEMPORARY TABLE product_import (
    sku             VARCHAR(50),
    product_name    VARCHAR(300),
    category        VARCHAR(100),
    price_str       VARCHAR(20),
    cost_str        VARCHAR(20),
    stock_str       VARCHAR(20),
    description     TEXT
);

LOAD DATA LOCAL INFILE '/tmp/products.csv'
INTO TABLE product_import
FIELDS TERMINATED BY ','
ENCLOSED BY '"'
LINES TERMINATED BY '\n'
IGNORE 1 ROWS;

-- ตรวจสอบและ clean ข้อมูล
SELECT 
    COUNT(*) AS total,
    SUM(CASE WHEN sku = '' OR sku IS NULL THEN 1 ELSE 0 END) AS missing_sku,
    SUM(CASE WHEN price_str NOT REGEXP '^[0-9]+\\.?[0-9]*$' THEN 1 ELSE 0 END) AS invalid_price,
    SUM(CASE WHEN CAST(price_str AS DECIMAL(10,2)) <= 0 THEN 1 ELSE 0 END) AS zero_price
FROM product_import;

-- Import สินค้าใหม่ (ไม่มี SKU ซ้ำ)
INSERT INTO products (sku, product_name, category, price, cost, stock_quantity, description)
SELECT 
    TRIM(sku),
    TRIM(product_name),
    TRIM(category),
    CAST(price_str AS DECIMAL(10,2)),
    NULLIF(CAST(NULLIF(TRIM(cost_str), '') AS DECIMAL(10,2)), 0),
    CAST(COALESCE(NULLIF(TRIM(stock_str), ''), '0') AS UNSIGNED),
    NULLIF(TRIM(description), '')
FROM product_import
WHERE TRIM(sku) != ''
  AND price_str REGEXP '^[0-9]+\\.?[0-9]*$'
  AND CAST(price_str AS DECIMAL(10,2)) > 0
ON DUPLICATE KEY UPDATE
    product_name = VALUES(product_name),
    price = VALUES(price),
    stock_quantity = VALUES(stock_quantity);

SELECT ROW_COUNT() AS rows_affected;

-- ตัวอย่าง 30: Import customers จาก legacy system
-- legacy_customers.csv:
-- id, fname, lname, email_addr, mobile, register_date, home_city

CREATE TEMPORARY TABLE legacy_customer_import (
    legacy_id       VARCHAR(20),
    fname           VARCHAR(100),
    lname           VARCHAR(100),
    email_addr      VARCHAR(300),
    mobile          VARCHAR(30),
    register_date   VARCHAR(30),
    home_city       VARCHAR(100)
);

LOAD DATA LOCAL INFILE '/tmp/legacy_customers.csv'
INTO TABLE legacy_customer_import
FIELDS TERMINATED BY ','
ENCLOSED BY '"'
LINES TERMINATED BY '\n'
IGNORE 1 ROWS;

-- Import พร้อม transformation
INSERT INTO customers (first_name, last_name, email, phone, city, created_at)
SELECT 
    TRIM(fname),
    TRIM(lname),
    LOWER(TRIM(email_addr)),
    REGEXP_REPLACE(TRIM(mobile), '[^0-9+]', ''),  -- เอาเฉพาะตัวเลขและ +
    TRIM(home_city),
    COALESCE(
        STR_TO_DATE(register_date, '%d/%m/%Y'),
        STR_TO_DATE(register_date, '%Y-%m-%d'),
        NOW()
    )
FROM legacy_customer_import
WHERE TRIM(email_addr) LIKE '%@%.%'
  AND TRIM(fname) != ''
  AND TRIM(lname) != ''
ON DUPLICATE KEY UPDATE
    phone = VALUES(phone),
    city = VALUES(city);
```

---

## 11. Scheduled Export Jobs

```sql
-- ตัวอย่าง 31: Automated daily export
DELIMITER //
CREATE PROCEDURE export_daily_report()
BEGIN
    -- สร้าง report table
    CREATE TEMPORARY TABLE daily_report AS
    SELECT 
        DATE(o.order_date) AS report_date,
        d.department_name,
        COUNT(o.order_id) AS order_count,
        SUM(o.total_amount) AS revenue,
        AVG(o.total_amount) AS avg_order,
        COUNT(DISTINCT o.customer_id) AS unique_customers
    FROM orders o
    JOIN order_items oi ON o.order_id = oi.order_id
    JOIN products p ON oi.product_id = p.product_id
    JOIN categories cat ON p.category = cat.category_name
    LEFT JOIN departments d ON cat.category_id = d.department_id
    WHERE DATE(o.order_date) = CURRENT_DATE - INTERVAL 1 DAY
      AND o.status = 'completed'
    GROUP BY DATE(o.order_date), d.department_name;
    
    -- Insert ลง summary table
    INSERT INTO reports.daily_summary
    SELECT *, NOW() AS generated_at FROM daily_report;
    
    SELECT CONCAT('Daily report generated for ', CURRENT_DATE - INTERVAL 1 DAY) AS status;
END //
DELIMITER ;

-- Schedule ด้วย Event
CREATE EVENT daily_report_export
ON SCHEDULE EVERY 1 DAY
STARTS '2024-01-01 01:00:00'
DO CALL export_daily_report();

-- ตัวอย่าง 32: Export ไปยัง external table (MySQL Federated ไม่นิยม)
-- ใช้ Stored Procedure แทน
DELIMITER //
CREATE PROCEDURE nightly_archive()
BEGIN
    -- Archive old orders
    INSERT INTO archive_db.orders_archive
    SELECT *, NOW() AS archived_at
    FROM orders
    WHERE order_date < DATE_SUB(NOW(), INTERVAL 2 YEAR)
      AND status IN ('completed', 'cancelled');
    
    -- Delete archived orders
    DELETE FROM orders
    WHERE order_date < DATE_SUB(NOW(), INTERVAL 2 YEAR)
      AND status IN ('completed', 'cancelled');
    
    SELECT CONCAT(ROW_COUNT(), ' orders archived') AS result;
END //
DELIMITER ;
```

---

## 12. CSV Export Patterns

```sql
-- ตัวอย่าง 33: สร้าง CSV Export สำหรับ reports
-- Export order report
SELECT 
    CONCAT('"Order ID"', ',',
           '"Customer"', ',',
           '"Order Date"', ',',
           '"Status"', ',',
           '"Items"', ',',
           '"Total"')
UNION ALL
SELECT 
    CONCAT(
        '"', o.order_id, '"', ',',
        '"', CONCAT(c.first_name, ' ', c.last_name), '"', ',',
        '"', DATE_FORMAT(o.order_date, '%d/%m/%Y'), '"', ',',
        '"', o.status, '"', ',',
        '"', COUNT(oi.item_id), '"', ',',
        '"', FORMAT(o.total_amount, 2), '"'
    )
FROM orders o
JOIN customers c ON o.customer_id = c.customer_id
LEFT JOIN order_items oi ON o.order_id = oi.order_id
WHERE YEAR(o.order_date) = 2024
GROUP BY o.order_id, o.order_date, o.status, o.total_amount, 
         c.first_name, c.last_name
ORDER BY o.order_date;

-- ตัวอย่าง 34: Export พร้อม Thai characters (UTF-8 BOM)
-- BOM (Byte Order Mark) ช่วยให้ Excel เปิดไฟล์ภาษาไทยได้ถูกต้อง
SELECT CONCAT(CHAR(0xEF, 0xBB, 0xBF),  -- UTF-8 BOM
    'รหัสพนักงาน,ชื่อ,นามสกุล,แผนก,เงินเดือน')
UNION ALL
SELECT CONCAT(
    employee_id, ',',
    first_name, ',',
    last_name, ',',
    COALESCE(d.department_name, ''), ',',
    salary
)
FROM employees e
LEFT JOIN departments d ON e.department_id = d.department_id
INTO OUTFILE '/tmp/employees_thai.csv'
CHARACTER SET utf8mb4
LINES TERMINATED BY '\n';
```

---

## 13. bcp - SQL Server Import/Export

```sql
-- ตัวอย่าง 35: SQL Server bcp utility (command line)
/*
-- Export ตาราง
bcp mydb.dbo.employees out "C:\data\employees.csv" -c -t"," -r"\n" -S server -U user -P password

-- Import จาก CSV
bcp mydb.dbo.employees in "C:\data\employees.csv" -c -t"," -r"\n" -S server -U user -P password

-- Export ด้วย query
bcp "SELECT * FROM mydb.dbo.employees WHERE is_active = 1" queryout "C:\data\active_emp.csv" -c -S server -U user -P password

-- BULK INSERT (T-SQL)
BULK INSERT employees
FROM 'C:\data\employees.csv'
WITH (
    FIELDTERMINATOR = ',',
    ROWTERMINATOR = '\n',
    FIRSTROW = 2  -- ข้าม header
);
*/
```

---

## แบบฝึกหัด 10 ข้อ

### ข้อ 1
เขียน script import products จาก CSV พร้อม validation

**เฉลย:**
```sql
-- Create staging table
CREATE TEMPORARY TABLE IF NOT EXISTS stg_products (
    raw_sku         VARCHAR(100),
    raw_name        VARCHAR(500),
    raw_category    VARCHAR(200),
    raw_price       VARCHAR(50),
    raw_stock       VARCHAR(50)
);

-- Load data (adjust path)
LOAD DATA LOCAL INFILE '/tmp/products.csv'
INTO TABLE stg_products
FIELDS TERMINATED BY ','
ENCLOSED BY '"'
LINES TERMINATED BY '\n'
IGNORE 1 ROWS;

-- Validation report
SELECT 
    COUNT(*) AS total_rows,
    SUM(CASE WHEN TRIM(raw_sku) = '' THEN 1 ELSE 0 END) AS blank_sku,
    SUM(CASE WHEN TRIM(raw_name) = '' THEN 1 ELSE 0 END) AS blank_name,
    SUM(CASE WHEN raw_price NOT REGEXP '^[0-9]+(\\.[0-9]+)?$' THEN 1 ELSE 0 END) AS invalid_price,
    SUM(CASE WHEN CAST(NULLIF(raw_price, '') AS DECIMAL(10,2)) <= 0 THEN 1 ELSE 0 END) AS zero_price
FROM stg_products;

-- Import valid rows
INSERT INTO products (sku, product_name, category, price, stock_quantity)
SELECT 
    TRIM(raw_sku),
    TRIM(raw_name),
    TRIM(raw_category),
    CAST(raw_price AS DECIMAL(10,2)),
    CAST(COALESCE(NULLIF(TRIM(raw_stock), ''), '0') AS UNSIGNED)
FROM stg_products
WHERE TRIM(raw_sku) != ''
  AND TRIM(raw_name) != ''
  AND raw_price REGEXP '^[0-9]+(\\.[0-9]+)?$'
  AND CAST(raw_price AS DECIMAL(10,2)) > 0
ON DUPLICATE KEY UPDATE
    product_name = VALUES(product_name),
    price = VALUES(price),
    stock_quantity = VALUES(stock_quantity);

SELECT ROW_COUNT() AS imported;
```

### ข้อ 2
Export orders เป็น CSV พร้อม summary rows

**เฉลย:**
```sql
-- Export orders ปี 2024 พร้อม grand total
(SELECT 
    'Order ID' AS col1,
    'Customer' AS col2,
    'Date' AS col3,
    'Status' AS col4,
    'Amount' AS col5)
UNION ALL
(SELECT 
    CAST(o.order_id AS CHAR),
    CONCAT(c.first_name, ' ', c.last_name),
    DATE_FORMAT(o.order_date, '%Y-%m-%d'),
    o.status,
    CAST(o.total_amount AS CHAR)
FROM orders o
JOIN customers c ON o.customer_id = c.customer_id
WHERE YEAR(o.order_date) = 2024
ORDER BY o.order_date)
UNION ALL
(SELECT 
    '',
    'TOTAL',
    '',
    '',
    CAST(SUM(total_amount) AS CHAR)
FROM orders
WHERE YEAR(order_date) = 2024)
INTO OUTFILE '/tmp/orders_2024.csv'
FIELDS TERMINATED BY ','
ENCLOSED BY '"'
LINES TERMINATED BY '\n';
```

### ข้อ 3
สร้าง stored procedure สำหรับ bulk import customers

**เฉลย:**
```sql
DELIMITER //
CREATE PROCEDURE import_customers_bulk(IN p_file_path VARCHAR(500))
BEGIN
    DECLARE v_imported INT DEFAULT 0;
    DECLARE v_skipped INT DEFAULT 0;
    
    -- Load จาก file
    CREATE TEMPORARY TABLE IF NOT EXISTS tmp_cust_import (
        email VARCHAR(300),
        first_name VARCHAR(200),
        last_name VARCHAR(200),
        phone VARCHAR(30),
        city VARCHAR(200)
    );
    
    SET @sql = CONCAT(
        'LOAD DATA LOCAL INFILE "', p_file_path, '" ',
        'INTO TABLE tmp_cust_import ',
        'FIELDS TERMINATED BY "," ENCLOSED BY "\"" ',
        'LINES TERMINATED BY "\n" IGNORE 1 ROWS'
    );
    
    PREPARE stmt FROM @sql;
    EXECUTE stmt;
    DEALLOCATE PREPARE stmt;
    
    -- Import valid rows
    INSERT IGNORE INTO customers (email, first_name, last_name, phone, city)
    SELECT 
        LOWER(TRIM(email)),
        TRIM(first_name),
        TRIM(last_name),
        REGEXP_REPLACE(TRIM(phone), '[^0-9+]', ''),
        TRIM(city)
    FROM tmp_cust_import
    WHERE email LIKE '%@%.%'
      AND TRIM(first_name) != ''
      AND TRIM(last_name) != '';
    
    SET v_imported = ROW_COUNT();
    SET v_skipped = (SELECT COUNT(*) FROM tmp_cust_import) - v_imported;
    
    DROP TEMPORARY TABLE tmp_cust_import;
    
    SELECT v_imported AS imported, v_skipped AS skipped;
END //
DELIMITER ;
```

### ข้อ 4
Export employee report เป็น JSON format

**เฉลย:**
```sql
SELECT JSON_PRETTY(JSON_ARRAYAGG(
    JSON_OBJECT(
        'employee_id', e.employee_id,
        'name', CONCAT(e.first_name, ' ', e.last_name),
        'email', e.email,
        'hire_date', DATE_FORMAT(e.hire_date, '%Y-%m-%d'),
        'salary', e.salary,
        'department', JSON_OBJECT(
            'id', d.department_id,
            'name', d.department_name
        ),
        'years_employed', TIMESTAMPDIFF(YEAR, e.hire_date, CURRENT_DATE)
    )
)) AS json_export
FROM employees e
LEFT JOIN departments d ON e.department_id = d.department_id
WHERE e.is_active = TRUE;
```

### ข้อ 5
สร้าง data migration script จาก old schema ไป new schema

**เฉลย:**
```sql
-- Migration: เพิ่ม full_address column จาก address fields แยก
-- Old schema: address_line1, address_line2, city, province, postal_code
-- New schema: full_address (combined)

-- 1. เพิ่ม column ใหม่
ALTER TABLE customers 
ADD COLUMN full_address VARCHAR(500) GENERATED ALWAYS AS (
    CONCAT_WS(', ',
        NULLIF(TRIM(address), ''),
        NULLIF(TRIM(city), ''),
        NULLIF(TRIM(CONCAT('Thailand ', postal_code)), 'Thailand ')
    )
) VIRTUAL;

-- 2. สร้างตารางใหม่ที่ใช้ full_address
CREATE TABLE customers_v2 AS
SELECT 
    customer_id,
    first_name,
    last_name,
    email,
    phone,
    city,
    country,
    birth_date,
    CONCAT_WS(', ',
        NULLIF(TRIM(address), ''),
        NULLIF(TRIM(city), ''),
        country
    ) AS full_address,
    created_at
FROM customers;

-- 3. ตรวจสอบ
SELECT COUNT(*) AS v1, (SELECT COUNT(*) FROM customers_v2) AS v2;
```

### ข้อ 6
Import JSON array ของ products ลงในตาราง

**เฉลย:**
```sql
SET @products_json = '[
    {"sku": "PHONE001", "name": "iPhone 15 Pro", "price": 42900, "stock": 50, "category": "Electronics"},
    {"sku": "PHONE002", "name": "Samsung S24", "price": 35900, "stock": 30, "category": "Electronics"},
    {"sku": "SHOE001", "name": "Nike Air Max", "price": 4500, "stock": 100, "category": "Shoes"}
]';

INSERT INTO products (sku, product_name, price, stock_quantity, category)
SELECT 
    jt.sku,
    jt.name,
    jt.price,
    jt.stock,
    jt.category
FROM JSON_TABLE(@products_json, '$[*]' COLUMNS (
    sku      VARCHAR(50)  PATH '$.sku',
    name     VARCHAR(200) PATH '$.name',
    price    DECIMAL(10,2) PATH '$.price',
    stock    INT          PATH '$.stock',
    category VARCHAR(100) PATH '$.category'
)) AS jt
ON DUPLICATE KEY UPDATE
    product_name = VALUES(product_name),
    price = VALUES(price),
    stock_quantity = VALUES(stock_quantity);
```

### ข้อ 7
สร้าง scheduled nightly export สำหรับ analytics

**เฉลย:**
```sql
-- Analytics database
CREATE DATABASE IF NOT EXISTS analytics_db CHARACTER SET utf8mb4;

USE analytics_db;

CREATE TABLE IF NOT EXISTS daily_kpis (
    kpi_date        DATE PRIMARY KEY,
    total_orders    INT DEFAULT 0,
    total_revenue   DECIMAL(15,2) DEFAULT 0,
    avg_order_value DECIMAL(10,2) DEFAULT 0,
    new_customers   INT DEFAULT 0,
    returning_customers INT DEFAULT 0,
    top_product_id  INT,
    top_category    VARCHAR(100),
    generated_at    TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Procedure สร้าง KPIs
DELIMITER //
CREATE PROCEDURE generate_daily_kpis(IN p_date DATE)
BEGIN
    INSERT INTO analytics_db.daily_kpis
    (kpi_date, total_orders, total_revenue, avg_order_value, new_customers, generated_at)
    SELECT 
        p_date,
        COUNT(DISTINCT o.order_id),
        COALESCE(SUM(o.total_amount), 0),
        COALESCE(AVG(o.total_amount), 0),
        (SELECT COUNT(*) FROM ecommerce.customers WHERE DATE(created_at) = p_date),
        NOW()
    FROM ecommerce.orders o
    WHERE DATE(o.order_date) = p_date
      AND o.status = 'completed'
    ON DUPLICATE KEY UPDATE
        total_orders = VALUES(total_orders),
        total_revenue = VALUES(total_revenue),
        avg_order_value = VALUES(avg_order_value),
        generated_at = NOW();
END //
DELIMITER ;

-- Event: รันทุกคืน
CREATE EVENT nightly_kpi_export
ON SCHEDULE EVERY 1 DAY
STARTS '2024-01-01 02:00:00'
DO CALL generate_daily_kpis(CURRENT_DATE - INTERVAL 1 DAY);
```

### ข้อ 8
สร้าง full backup script ด้วย SQL commands

**เฉลย:**
```sql
-- Backup script (ใช้ mysqldump ใน production)
-- แต่สำหรับ SQL level:

DELIMITER //
CREATE PROCEDURE create_full_backup(IN p_backup_suffix VARCHAR(20))
BEGIN
    -- สร้าง backup schema
    SET @backup_db = CONCAT('backup_', DATABASE(), '_', p_backup_suffix);
    SET @sql = CONCAT('CREATE DATABASE IF NOT EXISTS ', @backup_db, 
                      ' CHARACTER SET utf8mb4');
    PREPARE stmt FROM @sql;
    EXECUTE stmt;
    DEALLOCATE PREPARE stmt;
    
    -- Copy tables
    SET @sql = CONCAT('CREATE TABLE ', @backup_db, '.employees AS SELECT * FROM employees');
    PREPARE stmt FROM @sql;
    EXECUTE stmt;
    DEALLOCATE PREPARE stmt;
    
    SET @sql = CONCAT('CREATE TABLE ', @backup_db, '.products AS SELECT * FROM products');
    PREPARE stmt FROM @sql;
    EXECUTE stmt;
    DEALLOCATE PREPARE stmt;
    
    SET @sql = CONCAT('CREATE TABLE ', @backup_db, '.customers AS SELECT * FROM customers');
    PREPARE stmt FROM @sql;
    EXECUTE stmt;
    DEALLOCATE PREPARE stmt;
    
    SET @sql = CONCAT('CREATE TABLE ', @backup_db, '.orders AS SELECT * FROM orders');
    PREPARE stmt FROM @sql;
    EXECUTE stmt;
    DEALLOCATE PREPARE stmt;
    
    SELECT CONCAT('Backup created: ', @backup_db) AS status;
END //
DELIMITER ;

-- สร้าง backup
CALL create_full_backup('20240115');
```

### ข้อ 9
Import orders จาก Excel CSV ที่มีข้อมูลหลายหน้า (multiple sheets)

**เฉลย:**
```sql
-- สมมติ export Excel เป็น CSV หลายไฟล์
-- sheet1 = orders_jan.csv, sheet2 = orders_feb.csv

-- สร้าง staging table
CREATE TEMPORARY TABLE orders_staging (
    order_ref       VARCHAR(50),
    customer_email  VARCHAR(300),
    order_date_str  VARCHAR(30),
    product_sku     VARCHAR(50),
    quantity_str    VARCHAR(20),
    price_str       VARCHAR(30),
    status          VARCHAR(20)
);

-- Load January
LOAD DATA LOCAL INFILE '/tmp/orders_jan.csv'
INTO TABLE orders_staging
FIELDS TERMINATED BY ','
ENCLOSED BY '"'
LINES TERMINATED BY '\n'
IGNORE 1 ROWS;

-- Load February (append)
LOAD DATA LOCAL INFILE '/tmp/orders_feb.csv'
INTO TABLE orders_staging
FIELDS TERMINATED BY ','
ENCLOSED BY '"'
LINES TERMINATED BY '\n'
IGNORE 1 ROWS;

-- Process staging data
INSERT INTO orders (customer_id, order_date, status, total_amount)
SELECT 
    c.customer_id,
    STR_TO_DATE(s.order_date_str, '%d/%m/%Y'),
    LOWER(TRIM(s.status)),
    SUM(CAST(s.quantity_str AS UNSIGNED) * CAST(s.price_str AS DECIMAL(10,2)))
FROM orders_staging s
JOIN customers c ON c.email = LOWER(TRIM(s.customer_email))
WHERE c.customer_id IS NOT NULL
  AND STR_TO_DATE(s.order_date_str, '%d/%m/%Y') IS NOT NULL
GROUP BY s.order_ref, c.customer_id, STR_TO_DATE(s.order_date_str, '%d/%m/%Y'), s.status;
```

### ข้อ 10
สร้าง complete ETL pipeline สำหรับ data warehouse

**เฉลย:**
```sql
-- ETL Pipeline สำหรับ Sales Data Warehouse
DELIMITER //
CREATE PROCEDURE run_etl_pipeline(IN p_start_date DATE, IN p_end_date DATE)
BEGIN
    DECLARE v_records INT;
    
    -- Step 1: Extract
    DROP TEMPORARY TABLE IF EXISTS etl_orders;
    CREATE TEMPORARY TABLE etl_orders AS
    SELECT 
        o.order_id,
        o.order_date,
        o.status,
        o.total_amount,
        c.customer_id,
        c.email AS customer_email,
        c.city AS customer_city,
        c.country AS customer_country,
        CONCAT(c.first_name, ' ', c.last_name) AS customer_name
    FROM orders o
    JOIN customers c ON o.customer_id = c.customer_id
    WHERE o.order_date BETWEEN p_start_date AND p_end_date
      AND o.status = 'completed';
    
    -- Step 2: Transform
    ALTER TABLE etl_orders
    ADD COLUMN year_month CHAR(7),
    ADD COLUMN quarter TINYINT,
    ADD COLUMN day_of_week TINYINT;
    
    UPDATE etl_orders SET
        year_month = DATE_FORMAT(order_date, '%Y-%m'),
        quarter = QUARTER(order_date),
        day_of_week = DAYOFWEEK(order_date);
    
    -- Step 3: Load ไปยัง DW
    INSERT INTO datawarehouse.fact_sales
    (order_id, order_date, year_month, quarter, day_of_week,
     customer_id, customer_email, customer_city, customer_country,
     total_amount, load_date)
    SELECT 
        order_id, order_date, year_month, quarter, day_of_week,
        customer_id, customer_email, customer_city, customer_country,
        total_amount, NOW()
    FROM etl_orders
    ON DUPLICATE KEY UPDATE
        load_date = NOW();
    
    SET v_records = ROW_COUNT();
    
    -- Log
    INSERT INTO etl_run_log (pipeline_name, start_date, end_date, records_processed, run_at)
    VALUES ('sales_etl', p_start_date, p_end_date, v_records, NOW());
    
    SELECT v_records AS records_processed, 
           p_start_date AS from_date, 
           p_end_date AS to_date;
END //
DELIMITER ;

-- รัน ETL
CALL run_etl_pipeline('2024-01-01', '2024-01-31');
```

---

## สรุป

ในบทนี้เราเรียนรู้:
1. **PostgreSQL COPY**: เร็วที่สุดสำหรับ bulk import/export ใน PostgreSQL
2. **MySQL LOAD DATA INFILE**: import CSV, TSV ด้วย field/line terminators
3. **SELECT INTO OUTFILE**: export ข้อมูลเป็น CSV
4. **mysqldump**: backup ทั้ง database หรือเฉพาะตาราง
5. **pg_dump/pg_restore**: PostgreSQL backup tools
6. **SQLite .import**: import CSV ใน SQLite
7. **JSON Import/Export**: JSON_ARRAYAGG, JSON_TABLE
8. **Data Migration**: staging table pattern, ETL pipeline
9. **Scheduled Export**: Event Scheduler, automated reports

> **Best Practices สำหรับ Import/Export**:
> 1. ใช้ staging table ก่อน import ข้อมูลจริงเสมอ
> 2. Validate ข้อมูลก่อน insert
> 3. ใช้ Transaction สำหรับ import ขนาดใหญ่
> 4. Log ทุก import/export operation
> 5. Test กับข้อมูลเล็กก่อน ค่อย run จริง
> 6. มี rollback plan เสมอ
