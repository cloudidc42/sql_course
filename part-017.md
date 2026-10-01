# Part 17: Table Constraints Deep Dive

## บทนำ

Constraints (ข้อจำกัด) คือกฎที่กำหนดขึ้นเพื่อรักษาความถูกต้องและความสมบูรณ์ของข้อมูล (Data Integrity) เป็นการป้องกันข้อมูลผิดพลาดตั้งแต่ database layer

ประเภทของ Constraints:
1. **NOT NULL** - ต้องมีค่า
2. **UNIQUE** - ค่าต้องไม่ซ้ำ
3. **PRIMARY KEY** - ระบุตัวตนของแต่ละ row
4. **FOREIGN KEY** - ความสัมพันธ์ระหว่างตาราง
5. **CHECK** - กำหนดเงื่อนไขของค่า
6. **DEFAULT** - ค่าเริ่มต้น

---

## 1. NOT NULL Constraint

```sql
-- ตัวอย่าง 1: NOT NULL พื้นฐาน
CREATE TABLE employees_nn (
    employee_id INT NOT NULL,           -- ต้องมีค่า
    first_name  VARCHAR(50) NOT NULL,   -- ชื่อต้องมีค่า
    last_name   VARCHAR(50) NOT NULL,
    email       VARCHAR(100) NOT NULL,
    phone       VARCHAR(20),            -- อนุญาต NULL
    birth_date  DATE                    -- อนุญาต NULL
);

-- ตัวอย่าง 2: ลองใส่ NULL ใน NOT NULL column
INSERT INTO employees_nn (employee_id, first_name, last_name, email)
VALUES (1, 'John', 'Doe', 'john@example.com');  -- OK

INSERT INTO employees_nn (employee_id, first_name, last_name, email)
VALUES (2, NULL, 'Doe', 'jane@example.com');  -- ERROR! first_name ต้องมีค่า

-- ตัวอย่าง 3: NOT NULL กับ DEFAULT
CREATE TABLE products_nn (
    product_id      INT NOT NULL,
    product_name    VARCHAR(200) NOT NULL,
    price           DECIMAL(10,2) NOT NULL,
    stock           INT NOT NULL DEFAULT 0,    -- NOT NULL แต่มี DEFAULT
    is_active       BOOLEAN NOT NULL DEFAULT TRUE
);

-- ตัวอย่าง 4: NULL ไม่เท่ากับ NULL
SELECT NULL = NULL;     -- NULL (ไม่ใช่ TRUE!)
SELECT NULL IS NULL;    -- TRUE
SELECT NULL IS NOT NULL;-- FALSE
SELECT 1 = NULL;        -- NULL

-- ดังนั้น WHERE condition:
-- WHERE phone = NULL    -- ผิด! ไม่มีผล
-- WHERE phone IS NULL   -- ถูก!
-- WHERE phone IS NOT NULL -- ถูก!
```

---

## 2. UNIQUE Constraint

```sql
-- ตัวอย่าง 5: UNIQUE column
CREATE TABLE users_uq (
    user_id     INT PRIMARY KEY AUTO_INCREMENT,
    username    VARCHAR(50) NOT NULL UNIQUE,      -- inline unique
    email       VARCHAR(254) NOT NULL,
    phone       VARCHAR(20),
    national_id VARCHAR(13),
    
    -- Table-level unique (named)
    CONSTRAINT uq_email     UNIQUE (email),
    CONSTRAINT uq_national_id UNIQUE (national_id)
);

-- ตัวอย่าง 6: Composite UNIQUE
CREATE TABLE product_variants (
    variant_id  INT PRIMARY KEY AUTO_INCREMENT,
    product_id  INT NOT NULL,
    color       VARCHAR(50),
    size        VARCHAR(20),
    sku         VARCHAR(50) UNIQUE,
    
    -- สินค้า 1 ตัว มีแต่ละ color+size ได้ครั้งเดียว
    CONSTRAINT uq_product_color_size UNIQUE (product_id, color, size)
);

-- ตัวอย่าง 7: UNIQUE กับ NULL
-- NULL ไม่เท่ากับ NULL ดังนั้น UNIQUE column อนุญาต NULL หลายตัว
INSERT INTO users_uq (username, email, national_id) 
VALUES ('user1', 'user1@email.com', NULL);  -- OK

INSERT INTO users_uq (username, email, national_id) 
VALUES ('user2', 'user2@email.com', NULL);  -- OK! NULL ซ้ำได้

INSERT INTO users_uq (username, email, national_id) 
VALUES ('user3', 'user3@email.com', '1234567890123');  -- OK

INSERT INTO users_uq (username, email, national_id) 
VALUES ('user4', 'user4@email.com', '1234567890123');  -- ERROR! ค่าซ้ำ

-- ตัวอย่าง 8: ลบ UNIQUE constraint
ALTER TABLE users_uq DROP INDEX uq_email;

-- เพิ่มกลับ
ALTER TABLE users_uq ADD CONSTRAINT uq_email UNIQUE (email);
```

---

## 3. PRIMARY KEY Constraint

```sql
-- ตัวอย่าง 9: PRIMARY KEY พื้นฐาน
CREATE TABLE pk_examples (
    -- Inline PK
    id INT PRIMARY KEY,
    
    -- หรือ AUTO_INCREMENT
    -- id INT AUTO_INCREMENT PRIMARY KEY,
    
    name VARCHAR(100)
);

-- ตัวอย่าง 10: Table-level PRIMARY KEY
CREATE TABLE pk_table_level (
    order_id    INT NOT NULL,
    product_id  INT NOT NULL,
    quantity    INT NOT NULL,
    
    -- Composite Primary Key
    PRIMARY KEY (order_id, product_id)
);

-- ตัวอย่าง 11: PK properties
-- 1. ต้องไม่ NULL (implicit NOT NULL)
-- 2. ต้องไม่ซ้ำ (implicit UNIQUE)
-- 3. ตารางมีได้แค่ 1 PK
-- 4. สร้าง Clustered Index (InnoDB)

-- ตัวอย่าง 12: Natural Key vs Surrogate Key
-- Natural Key: ใช้ข้อมูลจริงเป็น PK
CREATE TABLE countries_natural (
    country_code    CHAR(2) PRIMARY KEY,  -- TH, US, JP
    country_name    VARCHAR(100)
);

-- Surrogate Key: สร้าง ID ขึ้นมาเอง
CREATE TABLE countries_surrogate (
    country_id      INT PRIMARY KEY AUTO_INCREMENT,
    country_code    CHAR(2) UNIQUE NOT NULL,
    country_name    VARCHAR(100)
);
-- Surrogate Key ดีกว่าสำหรับ FK relationships

-- ตัวอย่าง 13: UUID เป็น PK
CREATE TABLE events_uuid (
    event_id    CHAR(36) PRIMARY KEY DEFAULT (UUID()),  -- MySQL 8.0+
    event_name  VARCHAR(200),
    event_date  DATETIME
);
```

---

## 4. FOREIGN KEY Constraint

```sql
-- ตัวอย่าง 14: FOREIGN KEY พื้นฐาน
CREATE TABLE orders_fk (
    order_id    INT PRIMARY KEY AUTO_INCREMENT,
    customer_id INT NOT NULL,
    employee_id INT,
    
    FOREIGN KEY (customer_id) 
        REFERENCES customers(customer_id),
    
    FOREIGN KEY (employee_id) 
        REFERENCES employees(employee_id)
);

-- ตัวอย่าง 15: Named FOREIGN KEY
CREATE TABLE order_items_fk (
    item_id     INT PRIMARY KEY AUTO_INCREMENT,
    order_id    INT NOT NULL,
    product_id  INT NOT NULL,
    quantity    INT NOT NULL DEFAULT 1,
    
    CONSTRAINT fk_items_to_orders 
        FOREIGN KEY (order_id) 
        REFERENCES orders(order_id)
        ON DELETE CASCADE
        ON UPDATE CASCADE,
    
    CONSTRAINT fk_items_to_products 
        FOREIGN KEY (product_id) 
        REFERENCES products(product_id)
        ON DELETE RESTRICT
        ON UPDATE CASCADE
);
```

### ON DELETE / ON UPDATE Actions

```sql
-- ตัวอย่าง 16: ON DELETE CASCADE
-- ลบ parent -> ลบ children อัตโนมัติ
CREATE TABLE cascade_demo (
    id          INT PRIMARY KEY AUTO_INCREMENT,
    parent_id   INT,
    name        VARCHAR(100),
    
    FOREIGN KEY (parent_id) 
        REFERENCES cascade_demo(id)
        ON DELETE CASCADE
);

INSERT INTO cascade_demo (name) VALUES ('Parent 1');  -- id=1
INSERT INTO cascade_demo (parent_id, name) VALUES (1, 'Child A');  -- parent=1
INSERT INTO cascade_demo (parent_id, name) VALUES (1, 'Child B');  -- parent=1

DELETE FROM cascade_demo WHERE id = 1;
-- Child A และ Child B ถูกลบด้วย

-- ตัวอย่าง 17: ON DELETE SET NULL
-- ลบ parent -> children FK กลายเป็น NULL
CREATE TABLE set_null_demo (
    id          INT PRIMARY KEY AUTO_INCREMENT,
    category_id INT,  -- อนุญาต NULL
    name        VARCHAR(100),
    
    FOREIGN KEY (category_id) 
        REFERENCES categories(category_id)
        ON DELETE SET NULL  -- category ถูกลบ -> category_id = NULL
);

-- ตัวอย่าง 18: ON DELETE RESTRICT / NO ACTION
-- ไม่อนุญาตลบ parent ถ้ายังมี children
CREATE TABLE restrict_demo (
    id          INT PRIMARY KEY AUTO_INCREMENT,
    parent_id   INT NOT NULL,
    name        VARCHAR(100),
    
    FOREIGN KEY (parent_id) 
        REFERENCES parent_table(id)
        ON DELETE RESTRICT  -- ERROR ถ้าพยายามลบ parent ที่มี children
);

-- ตัวอย่าง 19: ON DELETE SET DEFAULT
-- ลบ parent -> children FK = DEFAULT value
-- (PostgreSQL รองรับ, MySQL ไม่รองรับ)
/*
CREATE TABLE set_default_demo (
    id          SERIAL PRIMARY KEY,
    category_id INT DEFAULT 1,  -- default category
    name        VARCHAR(100),
    
    FOREIGN KEY (category_id) 
        REFERENCES categories(id)
        ON DELETE SET DEFAULT
);
*/

-- ตัวอย่าง 20: ตาราง employees กับ self-referencing FK
CREATE TABLE employees_hierarchy (
    employee_id INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    first_name  VARCHAR(100) NOT NULL,
    last_name   VARCHAR(100) NOT NULL,
    manager_id  INT UNSIGNED NULL,  -- NULL = top-level (CEO)
    
    FOREIGN KEY (manager_id) 
        REFERENCES employees_hierarchy(employee_id)
        ON DELETE SET NULL
);

INSERT INTO employees_hierarchy (first_name, last_name, manager_id)
VALUES 
    ('สมชาย', 'CEO', NULL),        -- id=1, top level
    ('วิชัย', 'VP', 1),            -- reports to CEO
    ('นงนุช', 'Manager', 2),       -- reports to VP
    ('กาญจนา', 'Staff', 3);        -- reports to Manager

-- ดู hierarchy
SELECT 
    e.employee_id,
    e.first_name,
    m.first_name AS manager_name
FROM employees_hierarchy e
LEFT JOIN employees_hierarchy m ON e.manager_id = m.employee_id;
```

### ON UPDATE Actions

```sql
-- ตัวอย่าง 21: ON UPDATE CASCADE
-- เปลี่ยน PK ของ parent -> FK ของ children เปลี่ยนตาม
CREATE TABLE on_update_demo (
    child_id    INT PRIMARY KEY,
    parent_code CHAR(10),  -- FK
    name        VARCHAR(100),
    
    FOREIGN KEY (parent_code) 
        REFERENCES parent_codes(code)
        ON UPDATE CASCADE  -- parent code เปลี่ยน -> child FK เปลี่ยนตาม
);
```

---

## 5. CHECK Constraint

```sql
-- ตัวอย่าง 22: CHECK constraint พื้นฐาน
CREATE TABLE employees_check (
    employee_id INT PRIMARY KEY,
    first_name  VARCHAR(100) NOT NULL,
    age         TINYINT CHECK (age >= 18 AND age <= 65),
    salary      DECIMAL(10,2) CHECK (salary >= 0),
    gender      CHAR(1) CHECK (gender IN ('M', 'F', 'O'))
);

-- ตัวอย่าง 23: Named CHECK constraints
CREATE TABLE products_check (
    product_id  INT PRIMARY KEY AUTO_INCREMENT,
    name        VARCHAR(200) NOT NULL,
    price       DECIMAL(10,2) NOT NULL,
    cost        DECIMAL(10,2),
    stock       INT DEFAULT 0,
    rating      DECIMAL(3,2),
    
    CONSTRAINT chk_price_positive  CHECK (price > 0),
    CONSTRAINT chk_cost_non_neg    CHECK (cost IS NULL OR cost >= 0),
    CONSTRAINT chk_price_gt_cost   CHECK (cost IS NULL OR price >= cost),
    CONSTRAINT chk_stock_non_neg   CHECK (stock >= 0),
    CONSTRAINT chk_rating_range    CHECK (rating IS NULL OR (rating >= 1 AND rating <= 5))
);

-- ตัวอย่าง 24: CHECK ด้วย pattern matching
CREATE TABLE contacts (
    contact_id  INT PRIMARY KEY AUTO_INCREMENT,
    email       VARCHAR(254) NOT NULL,
    phone       VARCHAR(20),
    postal_code CHAR(5),
    
    -- MySQL 8.0+ รองรับ CHECK
    CONSTRAINT chk_email    CHECK (email REGEXP '^[^@]+@[^@]+\\.[^@]+$'),
    CONSTRAINT chk_postal   CHECK (postal_code REGEXP '^[0-9]{5}$')
);

-- ตัวอย่าง 25: CHECK กับ dates
CREATE TABLE events_check (
    event_id    INT PRIMARY KEY AUTO_INCREMENT,
    title       VARCHAR(200) NOT NULL,
    start_date  DATE NOT NULL,
    end_date    DATE NOT NULL,
    discount    DECIMAL(5,2) DEFAULT 0,
    
    CONSTRAINT chk_end_after_start CHECK (end_date >= start_date),
    CONSTRAINT chk_discount_range  CHECK (discount >= 0 AND discount <= 100)
);

-- ตัวอย่าง 26: CHECK violations
INSERT INTO products_check (name, price, cost)
VALUES ('Product A', 100.00, 150.00);  -- ERROR: price < cost

INSERT INTO events_check (title, start_date, end_date)
VALUES ('Conference', '2024-06-15', '2024-06-10');  -- ERROR: end < start

-- ตัวอย่าง 27: ข้อจำกัดของ CHECK
-- MySQL: CHECK constraint ตั้งแต่ 8.0.16
-- MySQL เก่า: CHECK ถูก parse แต่ไม่ถูก enforce!

-- ทางเลือกใน MySQL เก่า: ใช้ ENUM, Trigger
CREATE TABLE status_enum (
    id      INT PRIMARY KEY,
    status  ENUM('pending', 'active', 'cancelled', 'completed')  -- แทน CHECK
);
```

---

## 6. DEFAULT Constraint

```sql
-- ตัวอย่าง 28: DEFAULT values ต่างๆ
CREATE TABLE defaults_example (
    id          INT PRIMARY KEY AUTO_INCREMENT,
    
    -- String defaults
    status      VARCHAR(20) DEFAULT 'active',
    country     CHAR(2)     DEFAULT 'TH',
    
    -- Numeric defaults
    stock       INT         DEFAULT 0,
    discount    DECIMAL(5,2) DEFAULT 0.00,
    
    -- Boolean defaults
    is_active   BOOLEAN     DEFAULT TRUE,
    is_deleted  BOOLEAN     DEFAULT FALSE,
    
    -- Date/Time defaults
    created_at  TIMESTAMP   DEFAULT CURRENT_TIMESTAMP,
    updated_at  TIMESTAMP   DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    
    -- NULL default (implicit)
    notes       TEXT        DEFAULT NULL,
    
    -- MySQL 8.0+ expression default
    code        VARCHAR(20) DEFAULT (UPPER(UUID()))
);

-- ตัวอย่าง 29: ใช้ DEFAULT keyword ใน INSERT
INSERT INTO defaults_example (id, status, stock)
VALUES (1, DEFAULT, DEFAULT);  -- DEFAULT = ค่า default ที่กำหนด

-- หรือไม่ระบุ column
INSERT INTO defaults_example (id) VALUES (2);
-- ค่าอื่นๆ จะได้ DEFAULT

-- ตัวอย่าง 30: Expression DEFAULT (MySQL 8.0+)
CREATE TABLE products_default (
    product_id  INT PRIMARY KEY AUTO_INCREMENT,
    sku         VARCHAR(50) NOT NULL,
    price       DECIMAL(10,2),
    price_usd   DECIMAL(10,2) DEFAULT (price / 35),  -- MySQL 8.0+
    created_at  DATETIME DEFAULT (NOW())
);
```

---

## 7. Multiple Constraints per Column

```sql
-- ตัวอย่าง 31: หลาย constraints บน column เดียว
CREATE TABLE multi_constraint (
    user_id     INT UNSIGNED NOT NULL AUTO_INCREMENT,
    email       VARCHAR(254) NOT NULL UNIQUE,
    username    VARCHAR(50) NOT NULL UNIQUE,
    age         TINYINT UNSIGNED NOT NULL CHECK (age >= 18),
    score       DECIMAL(5,2) NOT NULL DEFAULT 0.00 CHECK (score BETWEEN 0 AND 100),
    
    PRIMARY KEY (user_id)
);

-- ตัวอย่าง 32: Comprehensive employee table
CREATE TABLE employees_full (
    employee_id     INT UNSIGNED    NOT NULL AUTO_INCREMENT,
    employee_code   VARCHAR(20)     NOT NULL,
    first_name      VARCHAR(100)    NOT NULL,
    last_name       VARCHAR(100)    NOT NULL,
    email           VARCHAR(254)    NOT NULL,
    phone           VARCHAR(20),
    hire_date       DATE            NOT NULL DEFAULT (CURRENT_DATE),
    birth_date      DATE,
    gender          CHAR(1),
    salary          DECIMAL(10,2)   NOT NULL DEFAULT 25000.00,
    department_id   INT UNSIGNED,
    manager_id      INT UNSIGNED,
    is_active       BOOLEAN         NOT NULL DEFAULT TRUE,
    created_at      TIMESTAMP       NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at      TIMESTAMP       NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    
    -- Constraints
    PRIMARY KEY (employee_id),
    CONSTRAINT uq_emp_code  UNIQUE (employee_code),
    CONSTRAINT uq_emp_email UNIQUE (email),
    CONSTRAINT chk_gender   CHECK (gender IN ('M', 'F', 'O') OR gender IS NULL),
    CONSTRAINT chk_salary   CHECK (salary > 0),
    CONSTRAINT chk_hire_after_birth CHECK (birth_date IS NULL OR hire_date > birth_date),
    
    -- Foreign Keys
    CONSTRAINT fk_emp_dept      FOREIGN KEY (department_id) REFERENCES departments(department_id) ON DELETE SET NULL,
    CONSTRAINT fk_emp_manager   FOREIGN KEY (manager_id) REFERENCES employees_full(employee_id) ON DELETE SET NULL,
    
    -- Indexes
    INDEX idx_dept (department_id),
    INDEX idx_manager (manager_id),
    INDEX idx_active (is_active)
);
```

---

## 8. Constraint Violations และ Error Handling

```sql
-- ตัวอย่าง 33: จัดการ constraint violations
-- NOT NULL violation
INSERT INTO employees (first_name, last_name, email, hire_date)
VALUES (NULL, 'Test', 'test@test.com', NOW());
-- ERROR 1048 (23000): Column 'first_name' cannot be null

-- UNIQUE violation
INSERT INTO employees (first_name, last_name, email, hire_date)
VALUES ('John', 'Doe', 'somchai@company.com', NOW());
-- ERROR 1062 (23000): Duplicate entry 'somchai@company.com' for key 'uq_emp_email'

-- FK violation
INSERT INTO employees (first_name, last_name, email, hire_date, department_id)
VALUES ('John', 'Doe', 'john@test.com', NOW(), 999);
-- ERROR 1452 (23000): Cannot add or update a child row: FK constraint fails

-- CHECK violation (MySQL 8.0.16+)
INSERT INTO products_check (name, price, cost) VALUES ('Test', -100, 50);
-- ERROR 3819 (HY000): Check constraint 'chk_price_positive' is violated

-- ตัวอย่าง 34: จัดการ error ใน application
-- Pattern: ตรวจสอบก่อน insert
SET @email = 'somchai@company.com';

SELECT COUNT(*) INTO @exists FROM employees WHERE email = @email;
IF @exists = 0 THEN
    INSERT INTO employees (email, ...) VALUES (@email, ...);
ELSE
    SELECT 'Email already exists' AS error;
END IF;

-- Pattern: ใช้ INSERT IGNORE หรือ ON DUPLICATE KEY UPDATE
INSERT IGNORE INTO employees (email, first_name, last_name, hire_date)
VALUES ('test@test.com', 'Test', 'User', NOW());

-- Pattern: ตรวจสอบ ROW_COUNT
INSERT IGNORE INTO employees (email, first_name, last_name, hire_date)
VALUES ('existing@test.com', 'Test', 'User', NOW());
SELECT ROW_COUNT();  -- 0 = ignored, 1 = inserted
```

---

## 9. Deferred Constraints (PostgreSQL)

```sql
-- ตัวอย่าง 35: PostgreSQL Deferred Constraints
-- Constraints ที่ checked ตอน COMMIT แทนที่จะเป็นตอน statement
/*
CREATE TABLE pg_employees (
    employee_id SERIAL PRIMARY KEY,
    manager_id  INT REFERENCES pg_employees(employee_id) 
                DEFERRABLE INITIALLY DEFERRED
);

-- ใส่ข้อมูลพร้อมกัน (circular reference ชั่วคราว)
BEGIN;
INSERT INTO pg_employees (employee_id, manager_id) VALUES (1, 2);  -- manager ยังไม่มี
INSERT INTO pg_employees (employee_id, manager_id) VALUES (2, 1);  -- OK
COMMIT;  -- ตอนนี้ FK ถูก check -> ทั้งคู่มีอยู่แล้ว = OK

-- โดยไม่มี DEFERRABLE, statement แรกจะ error ทันที
*/
```

---

## 10. ดู Constraints ใน Database

```sql
-- ตัวอย่าง 36: ดู constraints ทั้งหมดของตาราง

-- MySQL
SELECT 
    CONSTRAINT_NAME,
    CONSTRAINT_TYPE,
    TABLE_NAME
FROM information_schema.TABLE_CONSTRAINTS
WHERE TABLE_SCHEMA = DATABASE()
  AND TABLE_NAME = 'employees'
ORDER BY CONSTRAINT_TYPE;

-- ดู CHECK constraints (MySQL 8.0+)
SELECT 
    CONSTRAINT_NAME,
    CHECK_CLAUSE
FROM information_schema.CHECK_CONSTRAINTS
WHERE CONSTRAINT_SCHEMA = DATABASE();

-- ดู FK constraints
SELECT 
    kcu.CONSTRAINT_NAME,
    kcu.COLUMN_NAME,
    kcu.REFERENCED_TABLE_NAME,
    kcu.REFERENCED_COLUMN_NAME,
    rc.DELETE_RULE,
    rc.UPDATE_RULE
FROM information_schema.KEY_COLUMN_USAGE kcu
JOIN information_schema.REFERENTIAL_CONSTRAINTS rc
    ON kcu.CONSTRAINT_NAME = rc.CONSTRAINT_NAME
WHERE kcu.TABLE_SCHEMA = DATABASE()
  AND kcu.TABLE_NAME = 'order_items';

-- PostgreSQL
/*
SELECT 
    conname AS constraint_name,
    contype AS constraint_type,
    pg_get_constraintdef(oid) AS constraint_definition
FROM pg_constraint
WHERE conrelid = 'employees'::regclass;
*/
```

---

## แบบฝึกหัด 10 ข้อ

### ข้อ 1
สร้างตาราง `bank_transactions` พร้อม constraints ครบถ้วน

**เฉลย:**
```sql
CREATE TABLE bank_transactions (
    txn_id          BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    account_id      INT UNSIGNED NOT NULL,
    txn_type        VARCHAR(20) NOT NULL,
    amount          DECIMAL(15,2) NOT NULL,
    balance_after   DECIMAL(15,2) NOT NULL,
    description     VARCHAR(500),
    txn_date        TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    reference_no    VARCHAR(50) UNIQUE,
    
    CONSTRAINT chk_txn_type CHECK (txn_type IN ('DEPOSIT', 'WITHDRAWAL', 'TRANSFER', 'FEE')),
    CONSTRAINT chk_amount_positive CHECK (amount > 0),
    CONSTRAINT chk_balance_non_neg CHECK (balance_after >= 0),
    
    INDEX idx_account (account_id),
    INDEX idx_date (txn_date)
);
```

### ข้อ 2
แสดง FK violation scenarios พร้อมวิธีจัดการ

**เฉลย:**
```sql
-- Scenario 1: Insert child ก่อน parent
INSERT INTO orders (customer_id, total_amount) 
VALUES (9999, 1000);  -- customer_id=9999 ไม่มี

-- วิธีจัดการ: INSERT parent ก่อน หรือตรวจสอบ
SELECT COUNT(*) FROM customers WHERE customer_id = 9999;
-- ถ้า 0: สร้าง customer ก่อน

-- Scenario 2: DELETE parent ที่มี children
DELETE FROM customers WHERE customer_id = 1;  -- มี orders

-- วิธีจัดการ:
-- Option 1: ON DELETE CASCADE (ลบ orders ด้วย)
-- Option 2: ON DELETE SET NULL (orders.customer_id = NULL)
-- Option 3: ลบ children ก่อน แล้วลบ parent

-- Scenario 3: UPDATE parent PK
UPDATE customers SET customer_id = 999 WHERE customer_id = 1;
-- วิธีจัดการ: ON UPDATE CASCADE
```

### ข้อ 3
สร้าง UNIQUE constraint แบบ Composite สำหรับ scheduling system

**เฉลย:**
```sql
CREATE TABLE room_bookings (
    booking_id  INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    room_id     INT UNSIGNED NOT NULL,
    date        DATE NOT NULL,
    start_time  TIME NOT NULL,
    end_time    TIME NOT NULL,
    user_id     INT UNSIGNED NOT NULL,
    purpose     VARCHAR(200),
    
    -- ห้อง 1 ห้อง ในวันเดียวกัน เวลาเดียวกัน ไม่ซ้ำ
    CONSTRAINT uq_room_timeslot UNIQUE (room_id, date, start_time),
    CONSTRAINT chk_end_after_start CHECK (end_time > start_time)
);
```

### ข้อ 4
Implement ON DELETE CASCADE, SET NULL, RESTRICT ในระบบ blog

**เฉลย:**
```sql
CREATE TABLE blog_users (
    user_id INT PRIMARY KEY AUTO_INCREMENT,
    username VARCHAR(50) UNIQUE NOT NULL
);

CREATE TABLE blog_posts (
    post_id INT PRIMARY KEY AUTO_INCREMENT,
    author_id INT,
    title VARCHAR(300) NOT NULL,
    
    -- ลบ user -> SET NULL (post ยังอยู่ แต่ไม่มีผู้เขียน)
    FOREIGN KEY (author_id) REFERENCES blog_users(user_id) ON DELETE SET NULL
);

CREATE TABLE blog_comments (
    comment_id INT PRIMARY KEY AUTO_INCREMENT,
    post_id INT NOT NULL,
    user_id INT,
    content TEXT NOT NULL,
    
    -- ลบ post -> CASCADE (ลบ comments ด้วย)
    FOREIGN KEY (post_id) REFERENCES blog_posts(post_id) ON DELETE CASCADE,
    -- ลบ user -> SET NULL (comment ยังอยู่ แต่ anonymous)
    FOREIGN KEY (user_id) REFERENCES blog_users(user_id) ON DELETE SET NULL
);

CREATE TABLE blog_categories (
    category_id INT PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(100) UNIQUE NOT NULL
);

CREATE TABLE post_categories (
    post_id INT NOT NULL,
    category_id INT NOT NULL,
    PRIMARY KEY (post_id, category_id),
    
    -- ลบ post -> CASCADE
    FOREIGN KEY (post_id) REFERENCES blog_posts(post_id) ON DELETE CASCADE,
    -- ลบ category -> RESTRICT (ต้อง reassign posts ก่อน)
    FOREIGN KEY (category_id) REFERENCES blog_categories(category_id) ON DELETE RESTRICT
);
```

### ข้อ 5
เขียน CHECK constraints สำหรับ salary grading system

**เฉลย:**
```sql
CREATE TABLE salary_grades (
    grade_id    TINYINT UNSIGNED PRIMARY KEY,
    grade_name  VARCHAR(20) NOT NULL UNIQUE,
    min_salary  DECIMAL(10,2) NOT NULL,
    max_salary  DECIMAL(10,2) NOT NULL,
    
    CONSTRAINT chk_salary_range CHECK (min_salary < max_salary),
    CONSTRAINT chk_salary_positive CHECK (min_salary > 0)
);

INSERT INTO salary_grades VALUES
    (1, 'Grade 1', 15000, 25000),
    (2, 'Grade 2', 25000, 40000),
    (3, 'Grade 3', 40000, 60000),
    (4, 'Grade 4', 60000, 100000),
    (5, 'Grade 5', 100000, 200000);

CREATE TABLE graded_employees (
    emp_id      INT PRIMARY KEY AUTO_INCREMENT,
    name        VARCHAR(200) NOT NULL,
    grade_id    TINYINT UNSIGNED NOT NULL,
    salary      DECIMAL(10,2) NOT NULL,
    
    FOREIGN KEY (grade_id) REFERENCES salary_grades(grade_id),
    
    -- salary ต้องอยู่ใน range ของ grade
    CONSTRAINT chk_salary_in_grade 
        CHECK (salary >= (SELECT min_salary FROM salary_grades sg WHERE sg.grade_id = graded_employees.grade_id)
            AND salary <= (SELECT max_salary FROM salary_grades sg WHERE sg.grade_id = graded_employees.grade_id))
    -- หมายเหตุ: MySQL ไม่รองรับ subquery ใน CHECK, ใช้ Trigger แทน
);
```

### ข้อ 6
แสดง cascade behavior สำหรับ category hierarchy

**เฉลย:**
```sql
CREATE TABLE category_tree (
    cat_id      INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    cat_name    VARCHAR(100) NOT NULL,
    parent_id   INT UNSIGNED NULL,
    
    FOREIGN KEY (parent_id) 
        REFERENCES category_tree(cat_id)
        ON DELETE CASCADE  -- ลบ parent -> ลบ subcategories ทั้งหมด
);

INSERT INTO category_tree VALUES
    (1, 'Electronics', NULL),
    (2, 'Phones', 1),
    (3, 'Laptops', 1),
    (4, 'iPhones', 2),
    (5, 'Samsung', 2);

-- ลบ Electronics -> ลบ Phones, Laptops, iPhones, Samsung ด้วย
DELETE FROM category_tree WHERE cat_id = 1;
SELECT * FROM category_tree;  -- ว่างเปล่า
```

### ข้อ 7
สร้าง constraints สำหรับ date range validation

**เฉลย:**
```sql
CREATE TABLE promotions (
    promo_id        INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    promo_name      VARCHAR(200) NOT NULL,
    discount_type   VARCHAR(20) NOT NULL,
    discount_value  DECIMAL(10,2) NOT NULL,
    min_order_amt   DECIMAL(10,2) DEFAULT 0,
    max_uses        INT UNSIGNED,
    used_count      INT UNSIGNED DEFAULT 0,
    start_date      DATE NOT NULL,
    end_date        DATE NOT NULL,
    is_active       BOOLEAN DEFAULT TRUE,
    
    CONSTRAINT chk_discount_type CHECK (discount_type IN ('PERCENT', 'FIXED')),
    CONSTRAINT chk_percent_range CHECK (
        discount_type != 'PERCENT' OR (discount_value > 0 AND discount_value <= 100)
    ),
    CONSTRAINT chk_fixed_positive CHECK (
        discount_type != 'FIXED' OR discount_value > 0
    ),
    CONSTRAINT chk_date_range CHECK (end_date >= start_date),
    CONSTRAINT chk_used_lt_max CHECK (max_uses IS NULL OR used_count <= max_uses),
    CONSTRAINT chk_min_order CHECK (min_order_amt >= 0)
);
```

### ข้อ 8
ดึงข้อมูล FK constraints ทั้งหมดในฐานข้อมูลและแสดงเป็น report

**เฉลย:**
```sql
SELECT 
    kcu.TABLE_NAME AS child_table,
    kcu.COLUMN_NAME AS child_column,
    kcu.CONSTRAINT_NAME AS fk_name,
    kcu.REFERENCED_TABLE_NAME AS parent_table,
    kcu.REFERENCED_COLUMN_NAME AS parent_column,
    rc.DELETE_RULE AS on_delete,
    rc.UPDATE_RULE AS on_update
FROM information_schema.KEY_COLUMN_USAGE kcu
JOIN information_schema.REFERENTIAL_CONSTRAINTS rc
    ON kcu.CONSTRAINT_NAME = rc.CONSTRAINT_NAME
    AND kcu.CONSTRAINT_SCHEMA = rc.CONSTRAINT_SCHEMA
WHERE kcu.TABLE_SCHEMA = DATABASE()
  AND kcu.REFERENCED_TABLE_NAME IS NOT NULL
ORDER BY kcu.TABLE_NAME, kcu.CONSTRAINT_NAME;
```

### ข้อ 9
สร้าง trigger เพื่อ enforce constraint ที่ MySQL CHECK ไม่รองรับ (เวอร์ชันเก่า)

**เฉลย:**
```sql
-- Enforce salary ต้องอยู่ใน grade range
DELIMITER //
CREATE TRIGGER check_salary_grade
BEFORE INSERT ON graded_employees
FOR EACH ROW
BEGIN
    DECLARE v_min_salary DECIMAL(10,2);
    DECLARE v_max_salary DECIMAL(10,2);
    
    SELECT min_salary, max_salary 
    INTO v_min_salary, v_max_salary
    FROM salary_grades
    WHERE grade_id = NEW.grade_id;
    
    IF NEW.salary < v_min_salary OR NEW.salary > v_max_salary THEN
        SIGNAL SQLSTATE '45000' 
        SET MESSAGE_TEXT = 'Salary must be within grade range';
    END IF;
END //

CREATE TRIGGER check_salary_grade_update
BEFORE UPDATE ON graded_employees
FOR EACH ROW
BEGIN
    DECLARE v_min_salary DECIMAL(10,2);
    DECLARE v_max_salary DECIMAL(10,2);
    
    SELECT min_salary, max_salary 
    INTO v_min_salary, v_max_salary
    FROM salary_grades
    WHERE grade_id = NEW.grade_id;
    
    IF NEW.salary < v_min_salary OR NEW.salary > v_max_salary THEN
        SIGNAL SQLSTATE '45000' 
        SET MESSAGE_TEXT = 'Salary must be within grade range';
    END IF;
END //
DELIMITER ;
```

### ข้อ 10
สร้างระบบ constraint ที่สมบูรณ์สำหรับ vehicle registration

**เฉลย:**
```sql
CREATE TABLE vehicle_registrations (
    reg_id          INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    plate_number    VARCHAR(10) NOT NULL UNIQUE,
    owner_id        INT UNSIGNED NOT NULL,
    make            VARCHAR(50) NOT NULL,
    model           VARCHAR(100) NOT NULL,
    year            SMALLINT UNSIGNED NOT NULL,
    color           VARCHAR(50) NOT NULL,
    engine_cc       SMALLINT UNSIGNED,
    fuel_type       VARCHAR(20) NOT NULL DEFAULT 'GASOLINE',
    reg_date        DATE NOT NULL DEFAULT (CURRENT_DATE),
    expiry_date     DATE NOT NULL,
    insurance_no    VARCHAR(30) UNIQUE,
    insurance_expiry DATE,
    status          VARCHAR(20) DEFAULT 'ACTIVE',
    
    CONSTRAINT chk_year_range 
        CHECK (year BETWEEN 1900 AND YEAR(CURRENT_DATE) + 1),
    CONSTRAINT chk_fuel_type 
        CHECK (fuel_type IN ('GASOLINE', 'DIESEL', 'ELECTRIC', 'HYBRID', 'LPG', 'NGV')),
    CONSTRAINT chk_expiry_after_reg 
        CHECK (expiry_date > reg_date),
    CONSTRAINT chk_insurance_expiry 
        CHECK (insurance_expiry IS NULL OR insurance_expiry >= reg_date),
    CONSTRAINT chk_status 
        CHECK (status IN ('ACTIVE', 'EXPIRED', 'CANCELLED', 'SUSPENDED')),
    
    INDEX idx_owner (owner_id),
    INDEX idx_status (status),
    INDEX idx_expiry (expiry_date)
);
```

---

## สรุป

ในบทนี้เราเรียนรู้:
1. **NOT NULL**: ป้องกัน NULL values, NULL semantics
2. **UNIQUE**: ค่าไม่ซ้ำ, composite unique, NULL กับ UNIQUE
3. **PRIMARY KEY**: ระบุตัวตน, natural vs surrogate key
4. **FOREIGN KEY**: ความสัมพันธ์ระหว่างตาราง, ON DELETE/UPDATE actions
5. **CHECK**: กำหนดเงื่อนไข, pattern matching, date validation
6. **DEFAULT**: ค่าเริ่มต้น, expression defaults
7. **Constraint violations**: error codes, handling strategies
8. **Deferred constraints**: PostgreSQL DEFERRABLE

> **กฎทอง**: Constraints ควรอยู่ที่ database layer ไม่ใช่แค่ application layer เพราะหลาย applications อาจ access database เดียวกัน
