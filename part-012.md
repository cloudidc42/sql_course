# Part 12: Creating Tables - CREATE TABLE

## บทนำ

`CREATE TABLE` เป็นคำสั่งที่ใช้สร้างตารางใหม่ในฐานข้อมูล เป็นพื้นฐานของการออกแบบโครงสร้างข้อมูล (Schema Design) การสร้างตารางที่ดีตั้งแต่ต้นจะช่วยให้ระบบทำงานได้อย่างมีประสิทธิภาพและรองรับการขยายตัวในอนาคต

---

## 1. Syntax พื้นฐาน

```sql
CREATE TABLE table_name (
    column1_name  datatype  [constraints],
    column2_name  datatype  [constraints],
    ...
    [table_level_constraints]
);
```

### ตัวอย่าง 1: ตารางอย่างง่าย

```sql
-- ตารางง่ายที่สุด
CREATE TABLE simple_table (
    id      INT,
    name    VARCHAR(100)
);

-- ตารางพื้นฐานระดับ production
CREATE TABLE departments (
    department_id   INT             PRIMARY KEY,
    department_name VARCHAR(100)    NOT NULL,
    location        VARCHAR(100),
    budget          DECIMAL(15,2),
    created_at      TIMESTAMP       DEFAULT CURRENT_TIMESTAMP
);
```

---

## 2. Column Definitions - การกำหนด Column

### ตัวอย่าง 2: Column พร้อม constraints ครบถ้วน

```sql
CREATE TABLE employees (
    -- Primary Key แบบ inline
    employee_id     INT             PRIMARY KEY AUTO_INCREMENT,
    
    -- NOT NULL constraint
    first_name      VARCHAR(50)     NOT NULL,
    last_name       VARCHAR(50)     NOT NULL,
    
    -- UNIQUE constraint
    email           VARCHAR(100)    NOT NULL UNIQUE,
    
    -- DEFAULT value
    hire_date       DATE            NOT NULL DEFAULT (CURRENT_DATE),
    
    -- NULL allowed (ค่าเริ่มต้น)
    phone           VARCHAR(20),
    
    -- DEFAULT numeric
    salary          DECIMAL(10,2)   DEFAULT 0.00,
    
    -- CHECK constraint (inline)
    age             TINYINT         CHECK (age >= 18 AND age <= 65),
    
    -- DEFAULT boolean
    is_active       BOOLEAN         DEFAULT TRUE,
    
    -- FOREIGN KEY inline
    department_id   INT             REFERENCES departments(department_id)
);
```

### ตัวอย่าง 3: ตาราง products สมบูรณ์

```sql
CREATE TABLE products (
    product_id      INT             PRIMARY KEY AUTO_INCREMENT,
    product_name    VARCHAR(200)    NOT NULL,
    category        VARCHAR(100),
    price           DECIMAL(10,2)   NOT NULL    CHECK (price >= 0),
    cost            DECIMAL(10,2)               CHECK (cost >= 0),
    stock_quantity  INT             NOT NULL    DEFAULT 0   CHECK (stock_quantity >= 0),
    weight_kg       DECIMAL(8,3),
    description     TEXT,
    is_available    BOOLEAN         NOT NULL    DEFAULT TRUE,
    created_at      TIMESTAMP       DEFAULT CURRENT_TIMESTAMP,
    updated_at      TIMESTAMP       DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);
```

---

## 3. Inline vs Table-Level Constraints

```sql
-- ตัวอย่าง 4: Inline constraints (เขียนหลัง column)
CREATE TABLE inline_example (
    id          INT         PRIMARY KEY,
    email       VARCHAR(100) NOT NULL UNIQUE,
    age         INT         CHECK (age > 0)
);

-- ตัวอย่าง 5: Table-level constraints (เขียนแยก)
CREATE TABLE table_level_example (
    id          INT         NOT NULL,
    first_name  VARCHAR(50) NOT NULL,
    last_name   VARCHAR(50) NOT NULL,
    email       VARCHAR(100) NOT NULL,
    department_id INT,
    
    -- Table-level constraints พร้อมชื่อ (Named constraints)
    CONSTRAINT pk_table_level  PRIMARY KEY (id),
    CONSTRAINT uq_email        UNIQUE (email),
    CONSTRAINT uq_fullname     UNIQUE (first_name, last_name),  -- composite unique
    CONSTRAINT fk_department   FOREIGN KEY (department_id) 
                               REFERENCES departments(department_id)
                               ON DELETE SET NULL
                               ON UPDATE CASCADE,
    CONSTRAINT chk_email_format CHECK (email LIKE '%@%.%')
);

-- ตัวอย่าง 6: Composite Primary Key
CREATE TABLE order_items (
    order_id    INT     NOT NULL,
    product_id  INT     NOT NULL,
    quantity    INT     NOT NULL    DEFAULT 1,
    unit_price  DECIMAL(10,2) NOT NULL,
    
    -- Composite PK ต้องเป็น table-level
    PRIMARY KEY (order_id, product_id),
    FOREIGN KEY (order_id) REFERENCES orders(order_id) ON DELETE CASCADE,
    FOREIGN KEY (product_id) REFERENCES products(product_id)
);
```

---

## 4. CREATE TABLE IF NOT EXISTS

```sql
-- ตัวอย่าง 7: ป้องกัน error ถ้าตารางมีอยู่แล้ว
CREATE TABLE IF NOT EXISTS customers (
    customer_id     INT             PRIMARY KEY AUTO_INCREMENT,
    first_name      VARCHAR(50)     NOT NULL,
    last_name       VARCHAR(50)     NOT NULL,
    email           VARCHAR(100)    UNIQUE NOT NULL,
    phone           VARCHAR(20),
    address         TEXT,
    city            VARCHAR(100),
    country         VARCHAR(100)    DEFAULT 'Thailand',
    birth_date      DATE,
    created_at      TIMESTAMP       DEFAULT CURRENT_TIMESTAMP
);

-- ตัวอย่าง 8: ใช้ในสคริปต์ที่รันซ้ำได้
-- script: setup_database.sql
CREATE TABLE IF NOT EXISTS settings (
    setting_key     VARCHAR(100)    PRIMARY KEY,
    setting_value   TEXT,
    description     VARCHAR(500),
    updated_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);

-- ใส่ค่าเริ่มต้น
INSERT IGNORE INTO settings (setting_key, setting_value, description) VALUES
('app_version', '1.0.0', 'เวอร์ชันแอปพลิเคชัน'),
('maintenance_mode', 'false', 'โหมดปิดปรับปรุง'),
('max_upload_size', '5MB', 'ขนาดไฟล์อัปโหลดสูงสุด');
```

---

## 5. CREATE TABLE AS (CTAS) - สร้างตารางจาก SELECT

```sql
-- ตัวอย่าง 9: สร้างตารางจาก query ง่ายๆ
CREATE TABLE employees_backup AS
SELECT * FROM employees;

-- ตัวอย่าง 10: สร้างตารางโดยเลือก columns บางตัว
CREATE TABLE employee_summary AS
SELECT 
    employee_id,
    CONCAT(first_name, ' ', last_name) AS full_name,
    email,
    hire_date,
    salary
FROM employees
WHERE is_active = TRUE;

-- ตัวอย่าง 11: CTAS พร้อม JOIN
CREATE TABLE order_report AS
SELECT 
    o.order_id,
    c.first_name,
    c.last_name,
    c.email,
    o.order_date,
    o.total_amount,
    o.status
FROM orders o
JOIN customers c ON o.customer_id = c.customer_id
WHERE o.order_date >= '2024-01-01';

-- ตัวอย่าง 12: CTAS พร้อม Aggregation
CREATE TABLE monthly_sales_summary AS
SELECT 
    YEAR(order_date) AS year,
    MONTH(order_date) AS month,
    COUNT(*) AS total_orders,
    SUM(total_amount) AS total_revenue,
    AVG(total_amount) AS avg_order_value,
    MIN(total_amount) AS min_order,
    MAX(total_amount) AS max_order
FROM orders
WHERE status = 'completed'
GROUP BY YEAR(order_date), MONTH(order_date);

-- หมายเหตุ: CTAS ไม่ copy constraints เช่น PK, FK, INDEX
-- ต้องเพิ่มเองหลัง CTAS
ALTER TABLE employees_backup ADD PRIMARY KEY (employee_id);

-- PostgreSQL syntax
-- CREATE TABLE employees_backup AS SELECT * FROM employees;
-- CREATE TABLE employees_backup AS TABLE employees;  -- copy ทั้งตาราง

-- SQL Server syntax
-- SELECT * INTO employees_backup FROM employees;
```

---

## 6. Temporary Tables

```sql
-- ตัวอย่าง 13: สร้าง Temporary Table
-- ตารางชั่วคราว ถูกลบอัตโนมัติเมื่อ session สิ้นสุด
CREATE TEMPORARY TABLE temp_high_earners (
    employee_id     INT,
    full_name       VARCHAR(100),
    salary          DECIMAL(10,2),
    department_name VARCHAR(100)
);

-- เติมข้อมูล
INSERT INTO temp_high_earners
SELECT 
    e.employee_id,
    CONCAT(e.first_name, ' ', e.last_name),
    e.salary,
    d.department_name
FROM employees e
JOIN departments d ON e.department_id = d.department_id
WHERE e.salary > 50000;

-- ใช้งาน
SELECT * FROM temp_high_earners ORDER BY salary DESC;

-- ตัวอย่าง 14: Temp table สำหรับการคำนวณซับซ้อน
CREATE TEMPORARY TABLE temp_order_stats AS
SELECT 
    customer_id,
    COUNT(*) AS order_count,
    SUM(total_amount) AS total_spent,
    AVG(total_amount) AS avg_order,
    MAX(order_date) AS last_order_date
FROM orders
GROUP BY customer_id;

-- ใช้ต่อ
SELECT 
    c.first_name,
    c.last_name,
    c.email,
    t.order_count,
    t.total_spent,
    CASE 
        WHEN t.total_spent >= 100000 THEN 'VIP'
        WHEN t.total_spent >= 50000 THEN 'Gold'
        WHEN t.total_spent >= 10000 THEN 'Silver'
        ELSE 'Bronze'
    END AS customer_tier
FROM customers c
JOIN temp_order_stats t ON c.customer_id = t.customer_id
ORDER BY t.total_spent DESC;

-- ตัวอย่าง 15: Temp table ใน stored procedure
DELIMITER //
CREATE PROCEDURE calculate_bonuses()
BEGIN
    -- สร้าง temp table
    CREATE TEMPORARY TABLE IF NOT EXISTS temp_bonuses AS
    SELECT 
        e.employee_id,
        e.salary,
        SUM(oi.quantity * oi.unit_price) AS total_sales
    FROM employees e
    LEFT JOIN orders o ON o.status = 'completed'
    LEFT JOIN order_items oi ON o.order_id = oi.order_id
    GROUP BY e.employee_id, e.salary;
    
    -- คำนวณและอัปเดต
    UPDATE employees e
    JOIN temp_bonuses tb ON e.employee_id = tb.employee_id
    SET e.salary = e.salary * 1.05
    WHERE tb.total_sales > 100000;
    
    -- ลบ temp table
    DROP TEMPORARY TABLE IF EXISTS temp_bonuses;
END //
DELIMITER ;

-- ตัวอย่าง 16: PostgreSQL temporary table
/*
CREATE TEMP TABLE pg_temp_data AS
SELECT * FROM orders WHERE order_date >= NOW() - INTERVAL '30 days';

-- ON COMMIT: พฤติกรรมเมื่อ COMMIT
CREATE TEMP TABLE pg_temp_session (
    id      SERIAL PRIMARY KEY,
    data    TEXT
) ON COMMIT PRESERVE ROWS;  -- ค่าเริ่มต้น: ข้อมูลยังอยู่หลัง COMMIT
-- ON COMMIT DELETE ROWS: ลบข้อมูลเมื่อ COMMIT แต่ตารางยังอยู่
-- ON COMMIT DROP: ลบทั้งตารางเมื่อ COMMIT
*/
```

---

## 7. Table และ Column Comments

```sql
-- ตัวอย่าง 17: Comments ใน MySQL
CREATE TABLE products_with_comments (
    product_id      INT AUTO_INCREMENT COMMENT 'รหัสสินค้า (auto generated)',
    product_name    VARCHAR(200) NOT NULL COMMENT 'ชื่อสินค้า ต้องไม่ว่าง',
    category_id     INT COMMENT 'รหัสหมวดหมู่ อ้างอิงตาราง categories',
    price           DECIMAL(10,2) NOT NULL COMMENT 'ราคาขาย (บาท)',
    sku             VARCHAR(50) UNIQUE COMMENT 'Stock Keeping Unit - รหัสสินค้า',
    PRIMARY KEY (product_id)
) COMMENT = 'ตารางสินค้าในระบบ e-commerce';

-- ดู comments
SHOW FULL COLUMNS FROM products_with_comments;
SHOW CREATE TABLE products_with_comments;

-- ตัวอย่าง 18: PostgreSQL COMMENT syntax
/*
CREATE TABLE pg_products (
    product_id  SERIAL PRIMARY KEY,
    name        VARCHAR(200) NOT NULL,
    price       DECIMAL(10,2)
);

-- เพิ่ม comment แยก
COMMENT ON TABLE pg_products IS 'ตารางสินค้าหลัก';
COMMENT ON COLUMN pg_products.product_id IS 'รหัสสินค้า auto-generated';
COMMENT ON COLUMN pg_products.price IS 'ราคาขาย ต้องไม่ติดลบ';

-- ดู comments
SELECT 
    c.column_name,
    pg_catalog.col_description(c.table_name::regclass, c.ordinal_position) AS comment
FROM information_schema.columns c
WHERE c.table_name = 'pg_products';
*/
```

---

## 8. Table Naming Conventions

```sql
-- ตัวอย่าง 19: ตัวอย่าง naming conventions ที่ดี
-- snake_case (แนะนำ)
CREATE TABLE user_profiles (
    user_profile_id     INT PRIMARY KEY,
    first_name          VARCHAR(50),
    last_name           VARCHAR(50),
    profile_picture_url VARCHAR(2048)
);

-- ตัวอย่าง 20: ชื่อที่ชัดเจน ไม่กำกวม
-- ไม่ดี:
CREATE TABLE tbl_emp (
    emp_id  INT,
    emp_nm  VARCHAR(50),
    dept    INT
);

-- ดี:
CREATE TABLE employees (
    employee_id     INT,
    employee_name   VARCHAR(100),
    department_id   INT
);

-- ตัวอย่าง 21: การตั้งชื่อ Foreign Key
CREATE TABLE orders (
    order_id        INT PRIMARY KEY AUTO_INCREMENT,
    customer_id     INT NOT NULL,   -- ชื่อ = referenced_table_id
    employee_id     INT,            -- พนักงานที่ดูแล
    shipping_address_id INT,        -- FK ไปยัง addresses

    FOREIGN KEY (customer_id) REFERENCES customers(customer_id),
    FOREIGN KEY (employee_id) REFERENCES employees(employee_id),
    FOREIGN KEY (shipping_address_id) REFERENCES addresses(address_id)
);
```

---

## 9. ตัวอย่าง Schema สมบูรณ์ - E-Commerce System

```sql
-- ตัวอย่าง 22: Schema E-Commerce สมบูรณ์

-- 1. Categories
CREATE TABLE IF NOT EXISTS categories (
    category_id     INT UNSIGNED    AUTO_INCREMENT PRIMARY KEY,
    category_name   VARCHAR(100)    NOT NULL,
    parent_id       INT UNSIGNED,   -- self-referencing: หมวดหมู่ย่อย
    description     TEXT,
    is_active       BOOLEAN         DEFAULT TRUE,
    sort_order      SMALLINT        DEFAULT 0,
    created_at      TIMESTAMP       DEFAULT CURRENT_TIMESTAMP,
    
    FOREIGN KEY (parent_id) REFERENCES categories(category_id)
);

-- 2. Brands
CREATE TABLE IF NOT EXISTS brands (
    brand_id        INT UNSIGNED    AUTO_INCREMENT PRIMARY KEY,
    brand_name      VARCHAR(100)    NOT NULL UNIQUE,
    website         VARCHAR(2048),
    country_code    CHAR(2),
    is_active       BOOLEAN         DEFAULT TRUE
);

-- 3. Products
CREATE TABLE IF NOT EXISTS products (
    product_id      INT UNSIGNED    AUTO_INCREMENT PRIMARY KEY,
    sku             VARCHAR(50)     NOT NULL UNIQUE,
    product_name    VARCHAR(300)    NOT NULL,
    category_id     INT UNSIGNED,
    brand_id        INT UNSIGNED,
    price           DECIMAL(10,2)   NOT NULL,
    sale_price      DECIMAL(10,2),
    cost            DECIMAL(10,2),
    stock           INT UNSIGNED    DEFAULT 0,
    weight_g        INT UNSIGNED,
    description     TEXT,
    is_active       BOOLEAN         DEFAULT TRUE,
    created_at      TIMESTAMP       DEFAULT CURRENT_TIMESTAMP,
    updated_at      TIMESTAMP       DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    
    CONSTRAINT chk_price_positive  CHECK (price > 0),
    CONSTRAINT chk_cost_positive   CHECK (cost IS NULL OR cost >= 0),
    CONSTRAINT chk_sale_price      CHECK (sale_price IS NULL OR sale_price <= price),
    
    FOREIGN KEY (category_id) REFERENCES categories(category_id),
    FOREIGN KEY (brand_id) REFERENCES brands(brand_id),
    
    INDEX idx_category (category_id),
    INDEX idx_brand (brand_id),
    INDEX idx_price (price),
    INDEX idx_active (is_active)
);

-- 4. Customers
CREATE TABLE IF NOT EXISTS customers (
    customer_id     INT UNSIGNED    AUTO_INCREMENT PRIMARY KEY,
    email           VARCHAR(254)    NOT NULL UNIQUE,
    password_hash   VARCHAR(128)    NOT NULL,
    first_name      VARCHAR(100)    NOT NULL,
    last_name       VARCHAR(100)    NOT NULL,
    phone           VARCHAR(20),
    birth_date      DATE,
    gender          CHAR(1),
    is_verified     BOOLEAN         DEFAULT FALSE,
    is_active       BOOLEAN         DEFAULT TRUE,
    created_at      TIMESTAMP       DEFAULT CURRENT_TIMESTAMP,
    last_login      TIMESTAMP,
    
    CONSTRAINT chk_gender CHECK (gender IN ('M', 'F', 'O') OR gender IS NULL)
);

-- 5. Addresses
CREATE TABLE IF NOT EXISTS addresses (
    address_id      INT UNSIGNED    AUTO_INCREMENT PRIMARY KEY,
    customer_id     INT UNSIGNED    NOT NULL,
    label           VARCHAR(50)     DEFAULT 'Home',
    recipient_name  VARCHAR(200)    NOT NULL,
    phone           VARCHAR(20),
    address_line1   VARCHAR(300)    NOT NULL,
    address_line2   VARCHAR(300),
    city            VARCHAR(100)    NOT NULL,
    province        VARCHAR(100)    NOT NULL,
    postal_code     CHAR(5)         NOT NULL,
    country         CHAR(2)         DEFAULT 'TH',
    is_default      BOOLEAN         DEFAULT FALSE,
    
    FOREIGN KEY (customer_id) REFERENCES customers(customer_id) ON DELETE CASCADE,
    INDEX idx_customer (customer_id)
);

-- 6. Orders
CREATE TABLE IF NOT EXISTS orders (
    order_id        INT UNSIGNED    AUTO_INCREMENT PRIMARY KEY,
    order_number    VARCHAR(20)     NOT NULL UNIQUE,   -- readable order number
    customer_id     INT UNSIGNED    NOT NULL,
    address_id      INT UNSIGNED,
    status          VARCHAR(20)     NOT NULL    DEFAULT 'pending',
    subtotal        DECIMAL(12,2)   NOT NULL,
    discount        DECIMAL(10,2)   DEFAULT 0.00,
    shipping_fee    DECIMAL(8,2)    DEFAULT 0.00,
    tax             DECIMAL(8,2)    DEFAULT 0.00,
    total           DECIMAL(12,2)   NOT NULL,
    notes           TEXT,
    paid_at         TIMESTAMP       NULL,
    shipped_at      TIMESTAMP       NULL,
    delivered_at    TIMESTAMP       NULL,
    created_at      TIMESTAMP       DEFAULT CURRENT_TIMESTAMP,
    updated_at      TIMESTAMP       DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    
    CONSTRAINT chk_status CHECK (status IN ('pending','confirmed','processing',
                                            'shipped','delivered','cancelled','refunded')),
    CONSTRAINT chk_total  CHECK (total >= 0),
    
    FOREIGN KEY (customer_id) REFERENCES customers(customer_id),
    FOREIGN KEY (address_id) REFERENCES addresses(address_id),
    
    INDEX idx_customer (customer_id),
    INDEX idx_status (status),
    INDEX idx_created (created_at)
);

-- 7. Order Items
CREATE TABLE IF NOT EXISTS order_items (
    item_id         INT UNSIGNED    AUTO_INCREMENT PRIMARY KEY,
    order_id        INT UNSIGNED    NOT NULL,
    product_id      INT UNSIGNED    NOT NULL,
    product_name    VARCHAR(300)    NOT NULL,   -- snapshot ชื่อ ณ เวลาสั่ง
    product_sku     VARCHAR(50),               -- snapshot SKU
    quantity        INT UNSIGNED    NOT NULL,
    unit_price      DECIMAL(10,2)   NOT NULL,  -- snapshot ราคา ณ เวลาสั่ง
    discount        DECIMAL(8,2)    DEFAULT 0.00,
    subtotal        DECIMAL(12,2)   NOT NULL,
    
    FOREIGN KEY (order_id) REFERENCES orders(order_id) ON DELETE CASCADE,
    FOREIGN KEY (product_id) REFERENCES products(product_id),
    
    INDEX idx_order (order_id),
    INDEX idx_product (product_id)
);
```

---

## 10. ตัวอย่าง Schema - HR System

```sql
-- ตัวอย่าง 23: HR System Schema

CREATE TABLE IF NOT EXISTS job_positions (
    position_id     INT UNSIGNED    AUTO_INCREMENT PRIMARY KEY,
    position_code   VARCHAR(20)     NOT NULL UNIQUE,
    position_name   VARCHAR(100)    NOT NULL,
    department_id   INT UNSIGNED,
    min_salary      DECIMAL(10,2),
    max_salary      DECIMAL(10,2),
    is_active       BOOLEAN         DEFAULT TRUE,
    
    FOREIGN KEY (department_id) REFERENCES departments(department_id)
);

CREATE TABLE IF NOT EXISTS departments (
    department_id   INT UNSIGNED    AUTO_INCREMENT PRIMARY KEY,
    dept_code       VARCHAR(10)     NOT NULL UNIQUE,
    dept_name       VARCHAR(100)    NOT NULL,
    parent_dept_id  INT UNSIGNED,   -- self-referencing
    manager_id      INT UNSIGNED,   -- จะ reference employees
    location        VARCHAR(100),
    cost_center     VARCHAR(20),
    budget          DECIMAL(15,2),
    is_active       BOOLEAN         DEFAULT TRUE,
    created_at      TIMESTAMP       DEFAULT CURRENT_TIMESTAMP,
    
    FOREIGN KEY (parent_dept_id) REFERENCES departments(department_id)
);

CREATE TABLE IF NOT EXISTS employees (
    employee_id         INT UNSIGNED    AUTO_INCREMENT PRIMARY KEY,
    employee_code       VARCHAR(20)     NOT NULL UNIQUE,
    
    -- Personal info
    prefix              VARCHAR(10),
    first_name          VARCHAR(100)    NOT NULL,
    last_name           VARCHAR(100)    NOT NULL,
    first_name_en       VARCHAR(100),
    last_name_en        VARCHAR(100),
    birth_date          DATE,
    gender              CHAR(1),
    national_id         CHAR(13)        UNIQUE,
    
    -- Contact
    work_email          VARCHAR(254)    UNIQUE NOT NULL,
    personal_email      VARCHAR(254),
    work_phone          VARCHAR(20),
    personal_phone      VARCHAR(20),
    
    -- Employment
    department_id       INT UNSIGNED    NOT NULL,
    position_id         INT UNSIGNED    NOT NULL,
    manager_id          INT UNSIGNED,
    hire_date           DATE            NOT NULL,
    employment_type     VARCHAR(20)     DEFAULT 'FULL_TIME',
    
    -- Compensation
    base_salary         DECIMAL(10,2)   NOT NULL,
    allowance           DECIMAL(10,2)   DEFAULT 0,
    
    -- Status
    is_active           BOOLEAN         DEFAULT TRUE,
    termination_date    DATE,
    
    -- Timestamps
    created_at          TIMESTAMP       DEFAULT CURRENT_TIMESTAMP,
    updated_at          TIMESTAMP       DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    
    CONSTRAINT chk_gender           CHECK (gender IN ('M', 'F')),
    CONSTRAINT chk_employment_type  CHECK (employment_type IN ('FULL_TIME', 'PART_TIME', 'CONTRACT', 'INTERN')),
    CONSTRAINT chk_salary_positive  CHECK (base_salary > 0),
    
    FOREIGN KEY (department_id) REFERENCES departments(department_id),
    FOREIGN KEY (position_id)   REFERENCES job_positions(position_id),
    FOREIGN KEY (manager_id)    REFERENCES employees(employee_id)
);

-- เพิ่ม FK ที่ circular reference
ALTER TABLE departments 
ADD CONSTRAINT fk_dept_manager 
FOREIGN KEY (manager_id) REFERENCES employees(employee_id);
```

---

## 11. ตัวอย่าง Schema - Inventory System

```sql
-- ตัวอย่าง 24: Warehouse/Inventory

CREATE TABLE IF NOT EXISTS warehouses (
    warehouse_id    INT UNSIGNED    AUTO_INCREMENT PRIMARY KEY,
    warehouse_code  VARCHAR(10)     NOT NULL UNIQUE,
    warehouse_name  VARCHAR(100)    NOT NULL,
    address         TEXT,
    city            VARCHAR(100),
    country         CHAR(2)         DEFAULT 'TH',
    capacity        INT UNSIGNED,   -- จำนวนหน่วยที่เก็บได้
    is_active       BOOLEAN         DEFAULT TRUE
);

CREATE TABLE IF NOT EXISTS inventory (
    inventory_id    INT UNSIGNED    AUTO_INCREMENT PRIMARY KEY,
    product_id      INT UNSIGNED    NOT NULL,
    warehouse_id    INT UNSIGNED    NOT NULL,
    quantity        INT             NOT NULL    DEFAULT 0,
    reserved        INT             NOT NULL    DEFAULT 0,  -- จองแล้วยังไม่จัดส่ง
    min_quantity    INT             DEFAULT 0,  -- แจ้งเตือนเมื่อต่ำกว่า
    max_quantity    INT,                        -- ปริมาณสูงสุดที่เก็บ
    location_code   VARCHAR(20),               -- ตำแหน่งในคลัง เช่น A-01-02
    last_updated    TIMESTAMP       DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    
    UNIQUE KEY uq_product_warehouse (product_id, warehouse_id),
    
    FOREIGN KEY (product_id) REFERENCES products(product_id),
    FOREIGN KEY (warehouse_id) REFERENCES warehouses(warehouse_id)
);

CREATE TABLE IF NOT EXISTS stock_movements (
    movement_id     BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    product_id      INT UNSIGNED    NOT NULL,
    warehouse_id    INT UNSIGNED    NOT NULL,
    movement_type   VARCHAR(20)     NOT NULL,  -- IN, OUT, TRANSFER, ADJUST
    quantity        INT             NOT NULL,
    reference_type  VARCHAR(50),               -- ORDER, PURCHASE, ADJUSTMENT
    reference_id    INT UNSIGNED,              -- ID ของ document อ้างอิง
    unit_cost       DECIMAL(10,2),
    notes           TEXT,
    created_by      INT UNSIGNED,
    created_at      TIMESTAMP       DEFAULT CURRENT_TIMESTAMP,
    
    CONSTRAINT chk_movement_type CHECK (movement_type IN ('IN','OUT','TRANSFER','ADJUST')),
    
    FOREIGN KEY (product_id) REFERENCES products(product_id),
    FOREIGN KEY (warehouse_id) REFERENCES warehouses(warehouse_id),
    
    INDEX idx_product_date (product_id, created_at),
    INDEX idx_movement_type (movement_type)
);
```

---

## 12. ตัวอย่าง Schema - Blog/CMS System

```sql
-- ตัวอย่าง 25: Blog/CMS

CREATE TABLE IF NOT EXISTS users (
    user_id         INT UNSIGNED    AUTO_INCREMENT PRIMARY KEY,
    username        VARCHAR(50)     NOT NULL UNIQUE,
    email           VARCHAR(254)    NOT NULL UNIQUE,
    password_hash   VARCHAR(128)    NOT NULL,
    display_name    VARCHAR(100),
    bio             VARCHAR(500),
    avatar_url      VARCHAR(2048),
    role            VARCHAR(20)     DEFAULT 'reader',
    is_active       BOOLEAN         DEFAULT TRUE,
    email_verified  BOOLEAN         DEFAULT FALSE,
    created_at      TIMESTAMP       DEFAULT CURRENT_TIMESTAMP,
    
    CONSTRAINT chk_role CHECK (role IN ('admin', 'editor', 'author', 'reader'))
);

CREATE TABLE IF NOT EXISTS posts (
    post_id         INT UNSIGNED    AUTO_INCREMENT PRIMARY KEY,
    title           VARCHAR(300)    NOT NULL,
    slug            VARCHAR(300)    NOT NULL UNIQUE,
    content         LONGTEXT,
    excerpt         VARCHAR(1000),
    author_id       INT UNSIGNED    NOT NULL,
    status          VARCHAR(20)     DEFAULT 'draft',
    featured_image  VARCHAR(2048),
    view_count      INT UNSIGNED    DEFAULT 0,
    comment_count   INT UNSIGNED    DEFAULT 0,
    published_at    TIMESTAMP       NULL,
    created_at      TIMESTAMP       DEFAULT CURRENT_TIMESTAMP,
    updated_at      TIMESTAMP       DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    
    CONSTRAINT chk_post_status CHECK (status IN ('draft','published','archived')),
    
    FOREIGN KEY (author_id) REFERENCES users(user_id),
    
    FULLTEXT INDEX ft_search (title, content),
    INDEX idx_status (status),
    INDEX idx_published (published_at)
);

CREATE TABLE IF NOT EXISTS tags (
    tag_id          INT UNSIGNED    AUTO_INCREMENT PRIMARY KEY,
    tag_name        VARCHAR(50)     NOT NULL UNIQUE,
    tag_slug        VARCHAR(50)     NOT NULL UNIQUE,
    description     VARCHAR(500)
);

-- Many-to-Many: posts และ tags
CREATE TABLE IF NOT EXISTS post_tags (
    post_id         INT UNSIGNED    NOT NULL,
    tag_id          INT UNSIGNED    NOT NULL,
    
    PRIMARY KEY (post_id, tag_id),
    FOREIGN KEY (post_id) REFERENCES posts(post_id) ON DELETE CASCADE,
    FOREIGN KEY (tag_id)  REFERENCES tags(tag_id)   ON DELETE CASCADE
);

CREATE TABLE IF NOT EXISTS comments (
    comment_id      INT UNSIGNED    AUTO_INCREMENT PRIMARY KEY,
    post_id         INT UNSIGNED    NOT NULL,
    user_id         INT UNSIGNED,
    parent_id       INT UNSIGNED,   -- nested comments
    author_name     VARCHAR(100),   -- สำหรับ guest comments
    author_email    VARCHAR(254),
    content         TEXT            NOT NULL,
    is_approved     BOOLEAN         DEFAULT FALSE,
    created_at      TIMESTAMP       DEFAULT CURRENT_TIMESTAMP,
    
    FOREIGN KEY (post_id)   REFERENCES posts(post_id) ON DELETE CASCADE,
    FOREIGN KEY (user_id)   REFERENCES users(user_id) ON DELETE SET NULL,
    FOREIGN KEY (parent_id) REFERENCES comments(comment_id) ON DELETE CASCADE,
    
    INDEX idx_post (post_id),
    INDEX idx_approved (is_approved)
);
```

---

## 13. Advanced Table Features

```sql
-- ตัวอย่าง 26: Table Partitioning (MySQL)
CREATE TABLE sales_partitioned (
    sale_id         INT UNSIGNED    AUTO_INCREMENT,
    sale_date       DATE            NOT NULL,
    customer_id     INT UNSIGNED,
    amount          DECIMAL(10,2),
    PRIMARY KEY (sale_id, sale_date)  -- partition key ต้องอยู่ใน PK
)
PARTITION BY RANGE (YEAR(sale_date)) (
    PARTITION p2022 VALUES LESS THAN (2023),
    PARTITION p2023 VALUES LESS THAN (2024),
    PARTITION p2024 VALUES LESS THAN (2025),
    PARTITION p_future VALUES LESS THAN MAXVALUE
);

-- ตัวอย่าง 27: Character Set และ Collation
CREATE TABLE multilingual_content (
    id              INT PRIMARY KEY AUTO_INCREMENT,
    thai_text       VARCHAR(500) CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci,
    english_text    VARCHAR(500) CHARACTER SET utf8mb4 COLLATE utf8mb4_0900_ai_ci,
    emoji_text      VARCHAR(100) CHARACTER SET utf8mb4 COLLATE utf8mb4_bin
) CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;

-- ตัวอย่าง 28: Generated Columns (MySQL 5.7+)
CREATE TABLE products_with_calculated (
    product_id      INT PRIMARY KEY AUTO_INCREMENT,
    product_name    VARCHAR(200),
    price           DECIMAL(10,2),
    tax_rate        DECIMAL(5,2) DEFAULT 7.00,
    
    -- VIRTUAL: คำนวณทุกครั้งที่อ่าน (ไม่เก็บ)
    price_with_tax  DECIMAL(12,2) GENERATED ALWAYS AS (price * (1 + tax_rate/100)) VIRTUAL,
    
    -- STORED: คำนวณและเก็บ (เสียพื้นที่ แต่ index ได้)
    price_cents     BIGINT GENERATED ALWAYS AS (ROUND(price * 100)) STORED,
    
    INDEX idx_price_cents (price_cents)
);

-- ดูผล
INSERT INTO products_with_calculated (product_name, price) VALUES ('Test', 100.00);
SELECT product_name, price, price_with_tax, price_cents FROM products_with_calculated;
```

---

## 14. CREATE TABLE Patterns สำหรับ Multi-tenant

```sql
-- ตัวอย่าง 29: Schema-per-tenant (PostgreSQL)
/*
-- สร้าง schema สำหรับแต่ละ tenant
CREATE SCHEMA tenant_abc;
CREATE SCHEMA tenant_xyz;

-- สร้างตารางเหมือนกันในแต่ละ schema
CREATE TABLE tenant_abc.orders (LIKE public.orders INCLUDING ALL);
CREATE TABLE tenant_xyz.orders (LIKE public.orders INCLUDING ALL);
*/

-- ตัวอย่าง 30: Shared schema with tenant_id
CREATE TABLE multi_tenant_orders (
    order_id        INT UNSIGNED    AUTO_INCREMENT,
    tenant_id       INT UNSIGNED    NOT NULL,
    customer_id     INT UNSIGNED    NOT NULL,
    total           DECIMAL(10,2),
    created_at      TIMESTAMP       DEFAULT CURRENT_TIMESTAMP,
    
    -- Composite PK ที่รวม tenant_id
    PRIMARY KEY (tenant_id, order_id),
    
    -- Index ทุก query ต้องมี tenant_id
    INDEX idx_tenant_customer (tenant_id, customer_id),
    INDEX idx_tenant_date (tenant_id, created_at)
);

-- Row-level security ใน PostgreSQL
/*
CREATE TABLE rls_orders (
    order_id    SERIAL PRIMARY KEY,
    tenant_id   INT NOT NULL,
    data        TEXT
);

-- เปิด RLS
ALTER TABLE rls_orders ENABLE ROW LEVEL SECURITY;

-- Policy: ดูได้เฉพาะ tenant ของตัวเอง
CREATE POLICY tenant_isolation ON rls_orders
    USING (tenant_id = current_setting('app.tenant_id')::INT);
*/
```

---

## แบบฝึกหัด 10 ข้อ

### ข้อ 1
สร้างตาราง `books` สำหรับระบบห้องสมุด ต้องมี: book_id, isbn, title, author, publisher, publish_year, category, total_copies, available_copies, price

**เฉลย:**
```sql
CREATE TABLE books (
    book_id         INT UNSIGNED    AUTO_INCREMENT PRIMARY KEY,
    isbn            CHAR(13)        UNIQUE NOT NULL,
    title           VARCHAR(500)    NOT NULL,
    author          VARCHAR(300)    NOT NULL,
    publisher       VARCHAR(200),
    publish_year    SMALLINT        CHECK (publish_year BETWEEN 1000 AND 2099),
    category        VARCHAR(100),
    total_copies    SMALLINT UNSIGNED NOT NULL DEFAULT 1,
    available_copies SMALLINT UNSIGNED NOT NULL DEFAULT 1,
    price           DECIMAL(8,2),
    created_at      TIMESTAMP       DEFAULT CURRENT_TIMESTAMP,
    
    CONSTRAINT chk_available CHECK (available_copies <= total_copies)
);
```

### ข้อ 2
สร้างตาราง `loan_records` สำหรับการยืม-คืนหนังสือ

**เฉลย:**
```sql
CREATE TABLE loan_records (
    loan_id         INT UNSIGNED    AUTO_INCREMENT PRIMARY KEY,
    book_id         INT UNSIGNED    NOT NULL,
    member_id       INT UNSIGNED    NOT NULL,
    loan_date       DATE            NOT NULL DEFAULT (CURRENT_DATE),
    due_date        DATE            NOT NULL,
    return_date     DATE,
    fine_amount     DECIMAL(8,2)    DEFAULT 0.00,
    status          VARCHAR(20)     DEFAULT 'BORROWED',
    notes           TEXT,
    
    CONSTRAINT chk_due_after_loan CHECK (due_date > loan_date),
    CONSTRAINT chk_loan_status CHECK (status IN ('BORROWED', 'RETURNED', 'OVERDUE', 'LOST')),
    
    FOREIGN KEY (book_id) REFERENCES books(book_id),
    
    INDEX idx_book (book_id),
    INDEX idx_member (member_id),
    INDEX idx_due_date (due_date)
);
```

### ข้อ 3
ใช้ CREATE TABLE AS สร้าง summary ของ orders ที่สถานะ 'completed' โดยจัดกลุ่มตาม month

**เฉลย:**
```sql
CREATE TABLE monthly_order_summary AS
SELECT 
    DATE_FORMAT(order_date, '%Y-%m') AS month,
    COUNT(*) AS total_orders,
    SUM(total_amount) AS revenue,
    AVG(total_amount) AS avg_order,
    COUNT(DISTINCT customer_id) AS unique_customers
FROM orders
WHERE status = 'completed'
GROUP BY DATE_FORMAT(order_date, '%Y-%m')
ORDER BY month;

-- เพิ่ม Primary Key หลัง CTAS
ALTER TABLE monthly_order_summary ADD PRIMARY KEY (month);
```

### ข้อ 4
สร้าง Temporary table เพื่อหา top 5 customers ที่ใช้จ่ายสูงสุดในเดือนนี้

**เฉลย:**
```sql
CREATE TEMPORARY TABLE temp_top_customers AS
SELECT 
    c.customer_id,
    CONCAT(c.first_name, ' ', c.last_name) AS customer_name,
    c.email,
    COUNT(o.order_id) AS orders_count,
    SUM(o.total_amount) AS total_spent
FROM customers c
JOIN orders o ON c.customer_id = o.customer_id
WHERE 
    o.status = 'completed'
    AND YEAR(o.order_date) = YEAR(CURRENT_DATE)
    AND MONTH(o.order_date) = MONTH(CURRENT_DATE)
GROUP BY c.customer_id, c.first_name, c.last_name, c.email
ORDER BY total_spent DESC
LIMIT 5;

SELECT * FROM temp_top_customers;
```

### ข้อ 5
สร้างตาราง `vehicle_fleet` สำหรับบริษัทขนส่ง

**เฉลย:**
```sql
CREATE TABLE vehicle_fleet (
    vehicle_id      INT UNSIGNED    AUTO_INCREMENT PRIMARY KEY,
    plate_number    VARCHAR(20)     NOT NULL UNIQUE,
    vehicle_type    VARCHAR(50)     NOT NULL,
    brand           VARCHAR(50)     NOT NULL,
    model           VARCHAR(100)    NOT NULL,
    year            SMALLINT        NOT NULL,
    capacity_kg     INT UNSIGNED,
    fuel_type       VARCHAR(20),
    current_mileage INT UNSIGNED    DEFAULT 0,
    last_service_date DATE,
    next_service_km INT UNSIGNED,
    insurance_expiry DATE,
    is_available    BOOLEAN         DEFAULT TRUE,
    current_driver_id INT UNSIGNED,
    created_at      TIMESTAMP       DEFAULT CURRENT_TIMESTAMP,
    
    CONSTRAINT chk_year CHECK (year BETWEEN 1990 AND 2030),
    CONSTRAINT chk_fuel CHECK (fuel_type IN ('GASOLINE','DIESEL','ELECTRIC','HYBRID') OR fuel_type IS NULL)
);
```

### ข้อ 6
อธิบายความแตกต่างระหว่าง CREATE TABLE, CREATE TABLE IF NOT EXISTS, และ CREATE TABLE AS พร้อมตัวอย่าง

**เฉลย:**
```sql
-- 1. CREATE TABLE: สร้างใหม่, error ถ้ามีอยู่แล้ว
CREATE TABLE t1 (id INT);

-- 2. CREATE TABLE IF NOT EXISTS: สร้างถ้ายังไม่มี, ไม่ error ถ้ามีแล้ว
CREATE TABLE IF NOT EXISTS t1 (id INT);  -- ไม่ทำอะไรถ้า t1 มีอยู่แล้ว

-- 3. CREATE TABLE AS: สร้างจาก query result
CREATE TABLE t1_backup AS SELECT * FROM t1;
-- ข้อสังเกต: ไม่ copy PRIMARY KEY, FOREIGN KEY, INDEX
-- ต้องเพิ่มเอง: ALTER TABLE t1_backup ADD PRIMARY KEY (id);
```

### ข้อ 7
สร้างตาราง `meetings` ระบุ start_time, end_time, และ computed column duration_minutes

**เฉลย:**
```sql
CREATE TABLE meetings (
    meeting_id      INT UNSIGNED    AUTO_INCREMENT PRIMARY KEY,
    title           VARCHAR(200)    NOT NULL,
    organizer_id    INT UNSIGNED    NOT NULL,
    room_id         INT UNSIGNED,
    start_time      DATETIME        NOT NULL,
    end_time        DATETIME        NOT NULL,
    
    -- Generated column (MySQL 5.7+)
    duration_minutes SMALLINT GENERATED ALWAYS AS 
        (TIMESTAMPDIFF(MINUTE, start_time, end_time)) VIRTUAL,
    
    description     TEXT,
    is_online       BOOLEAN         DEFAULT FALSE,
    meeting_url     VARCHAR(2048),
    created_at      TIMESTAMP       DEFAULT CURRENT_TIMESTAMP,
    
    CONSTRAINT chk_end_after_start CHECK (end_time > start_time)
);
```

### ข้อ 8
สร้าง schema สำหรับระบบโรงพยาบาล: patients, doctors, appointments tables

**เฉลย:**
```sql
CREATE TABLE IF NOT EXISTS patients (
    patient_id      INT UNSIGNED    AUTO_INCREMENT PRIMARY KEY,
    hn              VARCHAR(20)     UNIQUE NOT NULL,    -- HN = Hospital Number
    first_name      VARCHAR(100)    NOT NULL,
    last_name       VARCHAR(100)    NOT NULL,
    birth_date      DATE            NOT NULL,
    gender          CHAR(1)         NOT NULL,
    blood_type      CHAR(3),
    national_id     CHAR(13)        UNIQUE,
    phone           VARCHAR(20)     NOT NULL,
    emergency_contact_name  VARCHAR(200),
    emergency_contact_phone VARCHAR(20),
    allergies       TEXT,
    created_at      TIMESTAMP       DEFAULT CURRENT_TIMESTAMP,
    
    CONSTRAINT chk_gender CHECK (gender IN ('M', 'F')),
    CONSTRAINT chk_blood  CHECK (blood_type IN ('A+','A-','B+','B-','O+','O-','AB+','AB-') OR blood_type IS NULL)
);

CREATE TABLE IF NOT EXISTS doctors (
    doctor_id       INT UNSIGNED    AUTO_INCREMENT PRIMARY KEY,
    license_number  VARCHAR(20)     UNIQUE NOT NULL,
    first_name      VARCHAR(100)    NOT NULL,
    last_name       VARCHAR(100)    NOT NULL,
    specialization  VARCHAR(100),
    department      VARCHAR(100),
    phone           VARCHAR(20),
    email           VARCHAR(254)    UNIQUE,
    consultation_fee DECIMAL(8,2),
    is_available    BOOLEAN         DEFAULT TRUE
);

CREATE TABLE IF NOT EXISTS appointments (
    appointment_id  INT UNSIGNED    AUTO_INCREMENT PRIMARY KEY,
    patient_id      INT UNSIGNED    NOT NULL,
    doctor_id       INT UNSIGNED    NOT NULL,
    appointment_date DATE            NOT NULL,
    appointment_time TIME            NOT NULL,
    duration_minutes SMALLINT       DEFAULT 15,
    reason          VARCHAR(500),
    status          VARCHAR(20)     DEFAULT 'SCHEDULED',
    notes           TEXT,
    created_at      TIMESTAMP       DEFAULT CURRENT_TIMESTAMP,
    
    CONSTRAINT chk_appt_status CHECK (status IN ('SCHEDULED','COMPLETED','CANCELLED','NO_SHOW')),
    
    FOREIGN KEY (patient_id) REFERENCES patients(patient_id),
    FOREIGN KEY (doctor_id) REFERENCES doctors(doctor_id),
    
    INDEX idx_patient (patient_id),
    INDEX idx_doctor (doctor_id),
    INDEX idx_date (appointment_date)
);
```

### ข้อ 9
สร้างตาราง `api_logs` สำหรับ logging API requests ที่ต้องรองรับข้อมูลจำนวนมาก

**เฉลย:**
```sql
CREATE TABLE api_logs (
    log_id          BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    request_id      CHAR(36)        NOT NULL,           -- UUID
    api_key         VARCHAR(64),
    user_id         INT UNSIGNED,
    method          CHAR(7)         NOT NULL,            -- GET, POST, PUT, DELETE
    endpoint        VARCHAR(500)    NOT NULL,
    status_code     SMALLINT UNSIGNED NOT NULL,
    response_time_ms INT UNSIGNED,                      -- milliseconds
    request_size    INT UNSIGNED,                       -- bytes
    response_size   INT UNSIGNED,                       -- bytes
    ip_address      VARCHAR(45),                        -- IPv6 max = 45 chars
    user_agent      VARCHAR(500),
    error_message   TEXT,
    created_at      TIMESTAMP(6) DEFAULT CURRENT_TIMESTAMP(6),  -- microsecond precision
    
    INDEX idx_api_key (api_key),
    INDEX idx_user_id (user_id),
    INDEX idx_endpoint (endpoint(100)),
    INDEX idx_status (status_code),
    INDEX idx_created (created_at)
) 
-- MySQL: แบ่ง partition ตามเดือน
PARTITION BY RANGE (UNIX_TIMESTAMP(created_at)) (
    PARTITION p202401 VALUES LESS THAN (UNIX_TIMESTAMP('2024-02-01')),
    PARTITION p202402 VALUES LESS THAN (UNIX_TIMESTAMP('2024-03-01')),
    PARTITION p_future VALUES LESS THAN MAXVALUE
);
```

### ข้อ 10
สร้าง schema สำหรับระบบโรงเรียน: students, courses, enrollments tables พร้อม composite unique constraint

**เฉลย:**
```sql
CREATE TABLE IF NOT EXISTS students (
    student_id      INT UNSIGNED    AUTO_INCREMENT PRIMARY KEY,
    student_code    VARCHAR(20)     NOT NULL UNIQUE,
    first_name      VARCHAR(100)    NOT NULL,
    last_name       VARCHAR(100)    NOT NULL,
    email           VARCHAR(254)    UNIQUE NOT NULL,
    phone           VARCHAR(20),
    grade_year      TINYINT UNSIGNED NOT NULL,
    major           VARCHAR(100),
    gpa             DECIMAL(3,2)    DEFAULT 0.00,
    is_active       BOOLEAN         DEFAULT TRUE,
    enrolled_date   DATE            NOT NULL,
    
    CONSTRAINT chk_grade CHECK (grade_year BETWEEN 1 AND 6),
    CONSTRAINT chk_gpa CHECK (gpa BETWEEN 0 AND 4)
);

CREATE TABLE IF NOT EXISTS courses (
    course_id       INT UNSIGNED    AUTO_INCREMENT PRIMARY KEY,
    course_code     VARCHAR(20)     NOT NULL UNIQUE,
    course_name     VARCHAR(200)    NOT NULL,
    credits         TINYINT UNSIGNED NOT NULL,
    max_students    SMALLINT UNSIGNED DEFAULT 30,
    current_students SMALLINT UNSIGNED DEFAULT 0,
    instructor_id   INT UNSIGNED,
    semester        CHAR(1)         NOT NULL,
    year            SMALLINT        NOT NULL,
    
    CONSTRAINT chk_semester CHECK (semester IN ('1','2','3')),
    CONSTRAINT chk_credits  CHECK (credits BETWEEN 1 AND 6)
);

CREATE TABLE IF NOT EXISTS enrollments (
    enrollment_id   INT UNSIGNED    AUTO_INCREMENT PRIMARY KEY,
    student_id      INT UNSIGNED    NOT NULL,
    course_id       INT UNSIGNED    NOT NULL,
    enrolled_at     TIMESTAMP       DEFAULT CURRENT_TIMESTAMP,
    grade           CHAR(2),
    grade_points    DECIMAL(3,2),
    status          VARCHAR(20)     DEFAULT 'ACTIVE',
    
    -- นักเรียน 1 คน ลงทะเบียน 1 วิชาได้ครั้งเดียว
    UNIQUE KEY uq_student_course (student_id, course_id),
    
    CONSTRAINT chk_grade_val CHECK (grade IN ('A','B+','B','C+','C','D+','D','F','W') OR grade IS NULL),
    CONSTRAINT chk_enrollment_status CHECK (status IN ('ACTIVE','DROPPED','COMPLETED')),
    
    FOREIGN KEY (student_id) REFERENCES students(student_id),
    FOREIGN KEY (course_id)  REFERENCES courses(course_id)
);
```

---

## สรุป

ในบทนี้เราเรียนรู้:
1. **Syntax พื้นฐาน** ของ CREATE TABLE และการกำหนด columns
2. **Inline vs Table-level constraints** และเมื่อใดใช้อะไร
3. **CREATE TABLE IF NOT EXISTS** - ป้องกัน error ในสคริปต์
4. **CREATE TABLE AS (CTAS)** - สร้างตารางจาก query
5. **Temporary Tables** - ตารางชั่วคราวสำหรับการคำนวณ
6. **Comments** - เพิ่ม documentation ใน schema
7. **Naming conventions** - ตั้งชื่อให้ชัดเจน
8. **Schema ตัวอย่าง** - E-Commerce, HR, Inventory, Blog

> **หลักการ**: ออกแบบตารางโดยคิดถึง queries ที่จะใช้บ่อยที่สุด สร้าง index ล่วงหน้า และเพิ่ม constraints เพื่อรักษาความถูกต้องของข้อมูล
