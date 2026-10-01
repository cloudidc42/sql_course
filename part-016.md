# Part 16: Modifying Tables - ALTER TABLE

## บทนำ

`ALTER TABLE` ใช้แก้ไขโครงสร้างตารางที่มีอยู่แล้ว เช่น เพิ่ม/ลบ columns, เปลี่ยน data type, เพิ่ม/ลบ constraints, เปลี่ยนชื่อ column หรือตาราง การทำ ALTER TABLE บน production database ที่มีข้อมูลจำนวนมากต้องใช้ความระมัดระวังสูง

---

## 1. ADD COLUMN

```sql
-- ตัวอย่าง 1: เพิ่ม column พื้นฐาน
ALTER TABLE employees
ADD COLUMN middle_name VARCHAR(50);

-- ตัวอย่าง 2: เพิ่มพร้อม constraints
ALTER TABLE employees
ADD COLUMN phone_work VARCHAR(20) DEFAULT NULL,
ADD COLUMN phone_home VARCHAR(20) DEFAULT NULL,
ADD COLUMN is_manager BOOLEAN DEFAULT FALSE;

-- ตัวอย่าง 3: เพิ่ม column ที่ตำแหน่งที่ต้องการ (MySQL)
ALTER TABLE employees
ADD COLUMN birth_date DATE AFTER last_name;

ALTER TABLE employees
ADD COLUMN employee_code VARCHAR(20) FIRST;

-- ตัวอย่าง 4: เพิ่ม column ที่มี DEFAULT ไม่ NULL
ALTER TABLE products
ADD COLUMN discount_percent DECIMAL(5,2) NOT NULL DEFAULT 0.00;

-- ตัวอย่าง 5: เพิ่ม column สำหรับ soft delete
ALTER TABLE customers
ADD COLUMN deleted_at TIMESTAMP NULL DEFAULT NULL,
ADD COLUMN deleted_by INT UNSIGNED NULL;

-- ตัวอย่าง 6: เพิ่ม column พร้อม AFTER ในหลาย DB
-- MySQL:
ALTER TABLE orders ADD COLUMN shipped_at TIMESTAMP NULL AFTER status;
ALTER TABLE orders ADD COLUMN delivered_at TIMESTAMP NULL AFTER shipped_at;

-- PostgreSQL: ไม่มี AFTER (ต่อท้ายเสมอ)
-- ALTER TABLE orders ADD COLUMN shipped_at TIMESTAMP;

-- SQL Server: ต่อท้ายเสมอ
-- ALTER TABLE orders ADD shipped_at DATETIME NULL;
```

---

## 2. DROP COLUMN

```sql
-- ตัวอย่าง 7: ลบ column
ALTER TABLE employees
DROP COLUMN middle_name;

-- ตัวอย่าง 8: ลบหลาย columns พร้อมกัน (MySQL)
ALTER TABLE employees
DROP COLUMN phone_home,
DROP COLUMN is_manager;

-- ตัวอย่าง 9: ลบ column IF EXISTS (MySQL 8.0+)
ALTER TABLE products
DROP COLUMN IF EXISTS old_price;

-- ตัวอย่าง 10: ลบ column ที่มี index (ต้อง drop index ก่อน)
-- ใน MySQL, dropping a column จะ auto-drop indexes ที่ใช้ column นั้น
-- แต่ถ้า index มีหลาย columns ต้อง drop index ก่อน

-- ดู indexes บน column
SHOW INDEX FROM products WHERE Column_name = 'sku';

-- ลบ index ก่อน ถ้าจำเป็น
ALTER TABLE products DROP INDEX idx_sku;

-- แล้วค่อยลบ column
ALTER TABLE products DROP COLUMN sku;

-- ⚠️ ไม่สามารถลบ column ที่เป็นส่วนหนึ่งของ FOREIGN KEY
-- ต้อง drop FK constraint ก่อน
ALTER TABLE employees DROP FOREIGN KEY fk_emp_dept;
ALTER TABLE employees DROP COLUMN department_id;
```

---

## 3. MODIFY / ALTER COLUMN (เปลี่ยน Data Type)

```sql
-- ตัวอย่าง 11: MySQL MODIFY COLUMN
ALTER TABLE employees
MODIFY COLUMN phone VARCHAR(30);  -- ขยายจาก 20 เป็น 30

ALTER TABLE products
MODIFY COLUMN price DECIMAL(12,2);  -- ขยาย precision

ALTER TABLE customers
MODIFY COLUMN birth_date DATE NOT NULL;  -- เพิ่ม NOT NULL

-- ตัวอย่าง 12: MySQL CHANGE COLUMN (เปลี่ยนชื่อด้วย)
ALTER TABLE employees
CHANGE COLUMN phone phone_number VARCHAR(30);

-- ตัวอย่าง 13: PostgreSQL ALTER COLUMN
/*
ALTER TABLE employees
ALTER COLUMN phone TYPE VARCHAR(30);

ALTER TABLE employees
ALTER COLUMN birth_date SET NOT NULL;

ALTER TABLE employees
ALTER COLUMN salary SET DEFAULT 30000;

ALTER TABLE employees
ALTER COLUMN salary DROP DEFAULT;

ALTER TABLE employees
ALTER COLUMN is_active SET DEFAULT TRUE;
*/

-- ตัวอย่าง 14: ระวัง data loss เมื่อเปลี่ยน type
-- ปลอดภัย: ขยายขนาด
ALTER TABLE employees MODIFY COLUMN email VARCHAR(500);  -- 100 -> 500 OK

-- อาจมี data loss: ลดขนาด
ALTER TABLE products MODIFY COLUMN description VARCHAR(500);  -- TEXT -> VARCHAR(500) อาจตัด

-- อาจ error: เปลี่ยน type ไม่ compatible
-- ALTER TABLE products MODIFY COLUMN product_name INT;  -- ERROR: ข้อความแปลงเป็น INT ไม่ได้

-- ตรวจสอบก่อนเปลี่ยน type
SELECT MAX(LENGTH(description)) FROM products;  -- ดู max length ก่อน
```

---

## 4. RENAME COLUMN

```sql
-- ตัวอย่าง 15: MySQL 8.0+ RENAME COLUMN
ALTER TABLE employees
RENAME COLUMN email TO email_address;

-- ตัวอย่าง 16: MySQL เก่ากว่า 8.0 ใช้ CHANGE
ALTER TABLE employees
CHANGE COLUMN email email_address VARCHAR(100);

-- ตัวอย่าง 17: PostgreSQL
/*
ALTER TABLE employees
RENAME COLUMN email TO email_address;
*/

-- ตัวอย่าง 18: เปลี่ยนกลับ
ALTER TABLE employees
RENAME COLUMN email_address TO email;
```

---

## 5. RENAME TABLE

```sql
-- ตัวอย่าง 19: RENAME TABLE
RENAME TABLE employees TO staff;
RENAME TABLE staff TO employees;  -- เปลี่ยนกลับ

-- เปลี่ยนหลายตารางพร้อมกัน (MySQL)
RENAME TABLE 
    old_orders TO orders_v1,
    old_customers TO customers_v1;

-- ตัวอย่าง 20: ALTER TABLE ... RENAME
ALTER TABLE employees RENAME TO staff;
ALTER TABLE staff RENAME TO employees;

-- PostgreSQL
-- ALTER TABLE employees RENAME TO staff;
```

---

## 6. ADD CONSTRAINT

```sql
-- ตัวอย่าง 21: เพิ่ม PRIMARY KEY
ALTER TABLE legacy_table
ADD PRIMARY KEY (id);

-- ตัวอย่าง 22: เพิ่ม UNIQUE constraint
ALTER TABLE employees
ADD CONSTRAINT uq_emp_email UNIQUE (email);

-- ตัวอย่าง 23: เพิ่ม FOREIGN KEY
ALTER TABLE employees
ADD CONSTRAINT fk_emp_dept 
FOREIGN KEY (department_id) 
REFERENCES departments(department_id)
ON DELETE SET NULL
ON UPDATE CASCADE;

ALTER TABLE order_items
ADD CONSTRAINT fk_items_order 
FOREIGN KEY (order_id) REFERENCES orders(order_id) ON DELETE CASCADE;

ALTER TABLE order_items
ADD CONSTRAINT fk_items_product 
FOREIGN KEY (product_id) REFERENCES products(product_id);

-- ตัวอย่าง 24: เพิ่ม CHECK constraint
ALTER TABLE employees
ADD CONSTRAINT chk_salary_positive CHECK (salary > 0);

ALTER TABLE products
ADD CONSTRAINT chk_price_positive CHECK (price > 0),
ADD CONSTRAINT chk_stock_non_negative CHECK (stock_quantity >= 0);

-- ตัวอย่าง 25: เพิ่ม Index
ALTER TABLE employees
ADD INDEX idx_department (department_id);

ALTER TABLE orders
ADD INDEX idx_customer_date (customer_id, order_date);

-- ตัวอย่าง 26: เพิ่ม FULLTEXT index
ALTER TABLE products
ADD FULLTEXT INDEX ft_search (product_name, description);
```

---

## 7. DROP CONSTRAINT

```sql
-- ตัวอย่าง 27: Drop PRIMARY KEY
ALTER TABLE old_table
DROP PRIMARY KEY;

-- ตัวอย่าง 28: Drop FOREIGN KEY
-- หา constraint names ก่อน
SELECT CONSTRAINT_NAME
FROM information_schema.TABLE_CONSTRAINTS
WHERE TABLE_SCHEMA = DATABASE()
  AND TABLE_NAME = 'employees'
  AND CONSTRAINT_TYPE = 'FOREIGN KEY';

ALTER TABLE employees
DROP FOREIGN KEY fk_emp_dept;

-- ตัวอย่าง 29: Drop UNIQUE constraint
ALTER TABLE employees
DROP INDEX uq_emp_email;  -- MySQL: UNIQUE constraint ถูก implement เป็น index

-- PostgreSQL
-- ALTER TABLE employees DROP CONSTRAINT uq_emp_email;

-- ตัวอย่าง 30: Drop CHECK constraint
-- MySQL 8.0+
ALTER TABLE employees
DROP CONSTRAINT chk_salary_positive;

-- ตัวอย่าง 31: Drop Index
ALTER TABLE employees
DROP INDEX idx_department;

-- หรือ
DROP INDEX idx_department ON employees;
```

---

## 8. SET DEFAULT / DROP DEFAULT

```sql
-- ตัวอย่าง 32: SET DEFAULT
-- MySQL
ALTER TABLE employees
MODIFY COLUMN is_active BOOLEAN DEFAULT TRUE;

-- PostgreSQL
-- ALTER TABLE employees ALTER COLUMN is_active SET DEFAULT TRUE;

-- ตัวอย่าง 33: DROP DEFAULT
-- MySQL
ALTER TABLE employees
MODIFY COLUMN is_active BOOLEAN;  -- ลบ DEFAULT

-- PostgreSQL
-- ALTER TABLE employees ALTER COLUMN is_active DROP DEFAULT;

-- ตัวอย่าง 34: เปลี่ยน DEFAULT value
ALTER TABLE orders
MODIFY COLUMN status VARCHAR(20) DEFAULT 'pending';

ALTER TABLE products
MODIFY COLUMN stock_quantity INT DEFAULT 0;
```

---

## 9. NOT NULL Constraints

```sql
-- ตัวอย่าง 35: เพิ่ม NOT NULL
-- ก่อนอื่น ต้องแน่ใจว่าไม่มีข้อมูล NULL
UPDATE employees SET phone = 'N/A' WHERE phone IS NULL;

-- แล้วค่อยเพิ่ม NOT NULL
ALTER TABLE employees
MODIFY COLUMN phone VARCHAR(20) NOT NULL DEFAULT 'N/A';

-- PostgreSQL
/*
-- ตรวจสอบ NULL ก่อน
SELECT COUNT(*) FROM employees WHERE phone IS NULL;

-- อัปเดต NULL values
UPDATE employees SET phone = 'N/A' WHERE phone IS NULL;

-- เพิ่ม NOT NULL
ALTER TABLE employees 
ALTER COLUMN phone SET NOT NULL;
*/

-- ตัวอย่าง 36: ลบ NOT NULL (อนุญาต NULL)
-- MySQL
ALTER TABLE employees
MODIFY COLUMN phone VARCHAR(20) NULL;

-- PostgreSQL
-- ALTER TABLE employees ALTER COLUMN phone DROP NOT NULL;
```

---

## 10. Migration-Safe ALTER Practices

```sql
-- ตัวอย่าง 37: Online DDL (MySQL InnoDB)
-- MySQL 5.6+ รองรับ Online DDL: แก้โครงสร้างโดยไม่ lock table นาน

-- ตรวจสอบว่าเป็น Online DDL
ALTER TABLE orders
ADD COLUMN tracking_number VARCHAR(100),
ALGORITHM=INPLACE,  -- ทำ in-place ไม่ rebuild table
LOCK=NONE;          -- ไม่ lock table ระหว่าง operation

-- ถ้า operation ไม่รองรับ INPLACE จะ fallback ไป COPY
ALTER TABLE orders
ADD INDEX idx_tracking (tracking_number),
ALGORITHM=INPLACE, LOCK=NONE;

-- ตัวอย่าง 38: ลำดับขั้นตอน Migration ที่ปลอดภัย

-- Step 1: Backup ก่อน
-- mysqldump mydb employees > employees_backup.sql

-- Step 2: Test ใน staging environment

-- Step 3: Plan rollback
-- ถ้า migration ล้มเหลว จะทำอะไร?

-- Step 4: Run migration
ALTER TABLE employees
ADD COLUMN linkedin_url VARCHAR(500);

-- Step 5: Verify
DESCRIBE employees;
SELECT COUNT(*) FROM employees WHERE linkedin_url IS NOT NULL;

-- ตัวอย่าง 39: gh-ost (GitHub's online schema change tool)
-- สำหรับ large tables ใน production
-- gh-ost --user="db_user" --password="..." --host="..."
--   --database="mydb" --table="employees"
--   --alter="ADD COLUMN linkedin_url VARCHAR(500)"
--   --execute

-- ตัวอย่าง 40: pt-online-schema-change (Percona)
-- pt-online-schema-change --alter "ADD COLUMN linkedin_url VARCHAR(500)"
--   D=mydb,t=employees --execute
```

---

## 11. ตัวอย่าง Migration Scripts จริง

```sql
-- ตัวอย่าง 41: Migration v1.1 - เพิ่มฟีเจอร์ Loyalty Points

-- ไฟล์: migration_001_add_loyalty_points.sql
-- Date: 2024-01-15
-- Author: DBA Team
-- Description: Add loyalty points system to customers

-- UP migration
ALTER TABLE customers
ADD COLUMN loyalty_points INT UNSIGNED DEFAULT 0 AFTER country,
ADD COLUMN loyalty_tier VARCHAR(20) DEFAULT 'Bronze' AFTER loyalty_points,
ADD COLUMN last_point_update TIMESTAMP NULL AFTER loyalty_tier;

-- Create indexes
ALTER TABLE customers
ADD INDEX idx_loyalty_tier (loyalty_tier);

-- Initialize points based on historical orders
UPDATE customers c
SET c.loyalty_points = COALESCE((
    SELECT FLOOR(SUM(o.total_amount) / 100)
    FROM orders o
    WHERE o.customer_id = c.customer_id
      AND o.status = 'completed'
), 0);

-- Update tiers
UPDATE customers
SET loyalty_tier = CASE
    WHEN loyalty_points >= 10000 THEN 'Platinum'
    WHEN loyalty_points >= 5000  THEN 'Gold'
    WHEN loyalty_points >= 1000  THEN 'Silver'
    ELSE 'Bronze'
END;

-- ตัวอย่าง 42: Migration v1.2 - เพิ่ม Product Categories hierarchy

ALTER TABLE categories
ADD COLUMN level TINYINT UNSIGNED DEFAULT 0 AFTER parent_id,
ADD COLUMN path VARCHAR(500) AFTER level,  -- เช่น '1/3/7'
ADD COLUMN sort_order SMALLINT DEFAULT 0 AFTER path;

-- สร้าง path สำหรับ top-level categories
UPDATE categories
SET path = CAST(category_id AS CHAR),
    level = 0
WHERE parent_id IS NULL;

-- ตัวอย่าง 43: Migration v2.0 - Schema refactoring ที่ใหญ่

-- เพิ่ม address table แยก
CREATE TABLE IF NOT EXISTS addresses (
    address_id INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    customer_id INT UNSIGNED NOT NULL,
    address_type VARCHAR(20) DEFAULT 'shipping',
    address_line1 VARCHAR(300) NOT NULL,
    city VARCHAR(100) NOT NULL,
    province VARCHAR(100),
    postal_code CHAR(5),
    is_default BOOLEAN DEFAULT FALSE,
    FOREIGN KEY (customer_id) REFERENCES customers(customer_id)
);

-- Copy address data จาก customers
INSERT INTO addresses (customer_id, address_type, address_line1, city, is_default)
SELECT customer_id, 'shipping', address, city, TRUE
FROM customers
WHERE address IS NOT NULL;

-- (ไม่ลบ column เดิมทันที: ให้ applications ใหม่ใช้ addresses table ก่อน)
-- หลังจาก deploy ใหม่ทั้งหมดแล้ว ค่อย drop column เดิม
-- ALTER TABLE customers DROP COLUMN address;
```

---

## 12. ดู Table Structure

```sql
-- ตัวอย่าง 44: ดูโครงสร้างตาราง

-- MySQL
DESCRIBE employees;
DESC employees;
SHOW COLUMNS FROM employees;
SHOW FULL COLUMNS FROM employees;  -- รวม comment ด้วย
SHOW CREATE TABLE employees;       -- ดู CREATE TABLE statement

-- PostgreSQL
-- \d employees  -- psql command
-- SELECT column_name, data_type, is_nullable, column_default
-- FROM information_schema.columns
-- WHERE table_name = 'employees'
-- ORDER BY ordinal_position;

-- ตัวอย่าง 45: ดู indexes
SHOW INDEX FROM employees;
SHOW INDEX FROM employees WHERE Non_unique = 0;  -- unique indexes เท่านั้น

-- ตัวอย่าง 46: ดู constraints
SELECT 
    tc.CONSTRAINT_NAME,
    tc.CONSTRAINT_TYPE,
    kcu.COLUMN_NAME,
    kcu.REFERENCED_TABLE_NAME,
    kcu.REFERENCED_COLUMN_NAME
FROM information_schema.TABLE_CONSTRAINTS tc
JOIN information_schema.KEY_COLUMN_USAGE kcu 
    ON tc.CONSTRAINT_NAME = kcu.CONSTRAINT_NAME
WHERE tc.TABLE_SCHEMA = DATABASE()
  AND tc.TABLE_NAME = 'employees';
```

---

## แบบฝึกหัด 10 ข้อ

### ข้อ 1
เพิ่ม columns social_media ลงใน customers table

**เฉลย:**
```sql
ALTER TABLE customers
ADD COLUMN line_id VARCHAR(100) AFTER phone,
ADD COLUMN facebook_url VARCHAR(500) AFTER line_id,
ADD COLUMN instagram VARCHAR(100) AFTER facebook_url;
```

### ข้อ 2
เปลี่ยน product price เป็น DECIMAL(12,2) และเพิ่ม CHECK constraint

**เฉลย:**
```sql
ALTER TABLE products
MODIFY COLUMN price DECIMAL(12,2) NOT NULL,
MODIFY COLUMN cost DECIMAL(12,2);

ALTER TABLE products
ADD CONSTRAINT chk_price_cost CHECK (cost IS NULL OR price >= cost);
```

### ข้อ 3
เพิ่ม audit columns (created_by, updated_by) ลงทุกตาราง

**เฉลย:**
```sql
ALTER TABLE employees
ADD COLUMN created_by INT UNSIGNED NULL,
ADD COLUMN updated_by INT UNSIGNED NULL;

ALTER TABLE products
ADD COLUMN created_by INT UNSIGNED NULL,
ADD COLUMN updated_by INT UNSIGNED NULL;

ALTER TABLE orders
ADD COLUMN created_by INT UNSIGNED NULL,
ADD COLUMN updated_by INT UNSIGNED NULL;

ALTER TABLE customers
ADD COLUMN created_by INT UNSIGNED NULL,
ADD COLUMN updated_by INT UNSIGNED NULL;
```

### ข้อ 4
RENAME columns ใน employees table ให้ตรงตาม naming convention

**เฉลย:**
```sql
-- MySQL 8.0+
ALTER TABLE employees
RENAME COLUMN first_name TO firstname,
RENAME COLUMN last_name TO lastname;

-- เปลี่ยนกลับ
ALTER TABLE employees
RENAME COLUMN firstname TO first_name,
RENAME COLUMN lastname TO last_name;
```

### ข้อ 5
เพิ่ม FOREIGN KEY constraints ที่หายไปในตาราง order_items

**เฉลย:**
```sql
-- ดู FK ที่มีอยู่ก่อน
SELECT CONSTRAINT_NAME FROM information_schema.TABLE_CONSTRAINTS
WHERE TABLE_NAME = 'order_items' AND CONSTRAINT_TYPE = 'FOREIGN KEY';

-- เพิ่ม FK ที่ขาด
ALTER TABLE order_items
ADD CONSTRAINT fk_items_order 
    FOREIGN KEY (order_id) REFERENCES orders(order_id) ON DELETE CASCADE,
ADD CONSTRAINT fk_items_product 
    FOREIGN KEY (product_id) REFERENCES products(product_id) ON DELETE RESTRICT;
```

### ข้อ 6
เปลี่ยน VARCHAR(20) ของ phone เป็น VARCHAR(30) อย่างปลอดภัย

**เฉลย:**
```sql
-- ตรวจสอบข้อมูล
SELECT MAX(LENGTH(phone)) AS max_phone_length FROM employees;
SELECT MAX(LENGTH(phone)) AS max_phone_length FROM customers;

-- เปลี่ยน data type
ALTER TABLE employees 
MODIFY COLUMN phone VARCHAR(30);

ALTER TABLE customers
MODIFY COLUMN phone VARCHAR(30);

-- ยืนยัน
DESCRIBE employees;
DESCRIBE customers;
```

### ข้อ 7
ลบ column ที่ไม่ใช้แล้วจากตาราง orders

**เฉลย:**
```sql
-- ตรวจสอบว่า column มีข้อมูลไหม
SELECT 
    COUNT(*) AS total_rows,
    COUNT(notes) AS has_notes,
    COUNT(shipping_fee) AS has_shipping_fee
FROM orders;

-- ดู dependencies
SELECT CONSTRAINT_NAME FROM information_schema.TABLE_CONSTRAINTS
WHERE TABLE_NAME = 'orders';

-- ลบ columns ที่ไม่ใช้
-- (สมมติ shipping_fee และ notes ไม่ใช้แล้ว)
-- ALTER TABLE orders 
-- DROP COLUMN shipping_fee,
-- DROP COLUMN notes;

-- ในทางปฏิบัติ: rename ก่อนเพื่อทดสอบ
ALTER TABLE orders
RENAME COLUMN notes TO _deprecated_notes;
-- รัน application สักพัก ถ้าไม่มีปัญหา ค่อย drop
```

### ข้อ 8
สร้าง migration script เพิ่มระบบ product rating

**เฉลย:**
```sql
-- migration: add_product_ratings.sql

-- 1. เพิ่ม columns ใน products
ALTER TABLE products
ADD COLUMN avg_rating DECIMAL(3,2) DEFAULT 0.00,
ADD COLUMN review_count INT UNSIGNED DEFAULT 0;

-- 2. สร้างตาราง reviews
CREATE TABLE IF NOT EXISTS product_reviews (
    review_id   INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    product_id  INT UNSIGNED NOT NULL,
    customer_id INT UNSIGNED NOT NULL,
    rating      TINYINT UNSIGNED NOT NULL,
    title       VARCHAR(200),
    content     TEXT,
    is_verified BOOLEAN DEFAULT FALSE,
    created_at  TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    
    UNIQUE KEY uq_customer_product (customer_id, product_id),
    CONSTRAINT chk_rating CHECK (rating BETWEEN 1 AND 5),
    
    FOREIGN KEY (product_id) REFERENCES products(product_id) ON DELETE CASCADE,
    FOREIGN KEY (customer_id) REFERENCES customers(customer_id)
);

-- 3. สร้าง indexes
ALTER TABLE product_reviews
ADD INDEX idx_product (product_id),
ADD INDEX idx_customer (customer_id),
ADD INDEX idx_rating (rating);
```

### ข้อ 9
Drop และ re-create Foreign Key ที่มี ON DELETE action ผิด

**เฉลย:**
```sql
-- ดู FK เดิม
SELECT CONSTRAINT_NAME, DELETE_RULE
FROM information_schema.REFERENTIAL_CONSTRAINTS
WHERE TABLE_NAME = 'order_items' AND CONSTRAINT_SCHEMA = DATABASE();

-- Drop FK เดิม
ALTER TABLE order_items 
DROP FOREIGN KEY fk_items_order;

-- เพิ่ม FK ใหม่พร้อม ON DELETE CASCADE
ALTER TABLE order_items
ADD CONSTRAINT fk_items_order 
FOREIGN KEY (order_id) REFERENCES orders(order_id)
ON DELETE CASCADE
ON UPDATE CASCADE;

-- ยืนยัน
SELECT CONSTRAINT_NAME, DELETE_RULE, UPDATE_RULE
FROM information_schema.REFERENTIAL_CONSTRAINTS
WHERE CONSTRAINT_SCHEMA = DATABASE()
  AND TABLE_NAME = 'order_items';
```

### ข้อ 10
สร้าง rollback script สำหรับ migration

**เฉลย:**
```sql
-- migration_up.sql: เพิ่ม feature ใหม่
ALTER TABLE customers
ADD COLUMN rewards_balance DECIMAL(10,2) DEFAULT 0.00,
ADD COLUMN rewards_tier VARCHAR(20) DEFAULT 'Bronze';

ALTER TABLE customers
ADD CONSTRAINT chk_rewards_balance CHECK (rewards_balance >= 0),
ADD INDEX idx_rewards_tier (rewards_tier);

-- migration_down.sql: rollback ถ้ามีปัญหา
ALTER TABLE customers
DROP CONSTRAINT chk_rewards_balance,
DROP INDEX idx_rewards_tier,
DROP COLUMN rewards_balance,
DROP COLUMN rewards_tier;

-- ทดสอบ rollback ใน staging
-- Source migration_up.sql
-- ตรวจสอบ...
-- ถ้ามีปัญหา: Source migration_down.sql
```

---

## สรุป

ในบทนี้เราเรียนรู้:
1. **ADD COLUMN**: เพิ่ม columns พร้อม constraints และ AFTER/FIRST positioning
2. **DROP COLUMN**: ลบ columns พร้อมจัดการ indexes และ FK
3. **MODIFY/CHANGE COLUMN**: เปลี่ยน data type อย่างปลอดภัย
4. **RENAME COLUMN/TABLE**: เปลี่ยนชื่อ
5. **ADD/DROP CONSTRAINT**: เพิ่ม/ลบ PK, FK, UNIQUE, CHECK
6. **SET/DROP DEFAULT**: จัดการ default values
7. **NOT NULL**: เพิ่ม/ลบ NOT NULL constraints
8. **Migration practices**: Online DDL, migration scripts, rollback
9. **ดู table structure**: DESCRIBE, SHOW CREATE TABLE, information_schema

> **กฎทอง Migration**: เสมอมี UP script และ DOWN script (rollback) ทดสอบใน staging ก่อน backup ก่อน deploy production
