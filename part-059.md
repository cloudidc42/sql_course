# Part 059: Advanced Schema Topics - Inheritance
# การสืบทอด Schema และ Polymorphism ใน SQL

---

## บทนำ: Inheritance ใน Database

เมื่อเรามี Entity Types ที่มีความสัมพันธ์แบบ "is-a" เช่น:
- iPhone **is a** Product
- Manager **is an** Employee
- Car **is a** Vehicle

มีสามวิธีหลักในการออกแบบ:

```
1. Table-per-Hierarchy (Single Table Inheritance)
   → ทุก Type ในตารางเดียว
   
2. Table-per-Type (Class Table Inheritance)
   → ตาราง Base + ตาราง Subtype แยกกัน
   
3. Table-per-Concrete-Class (Concrete Table Inheritance)
   → แต่ละ Concrete Type มีตารางของตัวเอง
```

---

## Pattern 1: Table-per-Hierarchy (Single Table Inheritance)

### แนวคิด

```
             [products]
         /       |        \
    [phone]  [laptop]  [clothing]
    
→ ทุกอย่างอยู่ในตาราง products ตารางเดียว
  มี column 'type' เพื่อบอกว่าเป็น type ไหน
```

### Product Catalog Example

```sql
CREATE TABLE products (
    product_id    SERIAL PRIMARY KEY,
    product_type  VARCHAR(20) NOT NULL
                  CHECK (product_type IN ('phone','laptop','tablet','clothing','book')),
    -- Common attributes (ทุก type มี)
    name          VARCHAR(200) NOT NULL,
    sku           VARCHAR(50) UNIQUE NOT NULL,
    price         DECIMAL(10,2) NOT NULL,
    stock_qty     INTEGER DEFAULT 0,
    is_active     BOOLEAN DEFAULT TRUE,
    created_at    TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    
    -- Phone-specific attributes
    phone_brand        VARCHAR(50),
    phone_model        VARCHAR(100),
    phone_storage_gb   INTEGER,
    phone_ram_gb       INTEGER,
    phone_color        VARCHAR(30),
    phone_screen_inch  DECIMAL(4,2),
    phone_battery_mah  INTEGER,
    phone_5g_capable   BOOLEAN,
    
    -- Laptop-specific attributes
    laptop_brand        VARCHAR(50),
    laptop_processor    VARCHAR(100),
    laptop_storage_gb   INTEGER,
    laptop_ram_gb       INTEGER,
    laptop_screen_inch  DECIMAL(4,2),
    laptop_os           VARCHAR(50),
    laptop_weight_kg    DECIMAL(5,3),
    
    -- Clothing-specific attributes
    clothing_brand    VARCHAR(50),
    clothing_size     VARCHAR(10),
    clothing_color    VARCHAR(30),
    clothing_material VARCHAR(100),
    clothing_gender   VARCHAR(20),
    
    -- Book-specific attributes
    book_isbn         VARCHAR(20),
    book_author       VARCHAR(200),
    book_publisher    VARCHAR(100),
    book_pages        INTEGER,
    book_language     VARCHAR(20)
);

-- Partial Indexes สำหรับแต่ละ Type
CREATE INDEX idx_products_phones ON products(phone_brand, phone_storage_gb)
    WHERE product_type = 'phone';

CREATE INDEX idx_products_laptops ON products(laptop_brand, laptop_ram_gb)
    WHERE product_type = 'laptop';

-- Insert data
INSERT INTO products (product_type, name, sku, price, phone_brand, phone_storage_gb, phone_ram_gb)
VALUES ('phone', 'iPhone 15 Pro', 'IPHONE15PRO-128', 42900.00, 'Apple', 128, 8);

INSERT INTO products (product_type, name, sku, price, laptop_brand, laptop_processor, laptop_ram_gb)
VALUES ('laptop', 'MacBook Air M3', 'MBA-M3-16', 52900.00, 'Apple', 'M3', 16);

INSERT INTO products (product_type, name, sku, price, book_isbn, book_author, book_pages)
VALUES ('book', 'Database Design Principles', 'BOOK-DB-001', 890.00, '978-0-123456-78-9', 'John Smith', 450);

-- Query เฉพาะ type
SELECT
    product_id, name, sku, price,
    phone_brand AS brand,
    phone_storage_gb AS storage_gb,
    phone_5g_capable AS is_5g
FROM products
WHERE product_type = 'phone'
AND phone_storage_gb >= 256
ORDER BY price;
```

### ข้อดี/ข้อเสีย ของ Single Table Inheritance

```
✅ ข้อดี:
- Simple: ตารางเดียว
- Fast reads: ไม่ต้อง JOIN
- ง่ายต่อ Ad-hoc queries ข้าม types
- Schema changes ง่าย

❌ ข้อเสีย:
- Nullable columns เยอะมาก (Sparse table)
- ไม่สามารถ NOT NULL constraint สำหรับ type-specific columns ได้
- ตาราง "โต" มาก
- ยากต่อการ Validate (phone ต้องมี phone_brand แต่ DB ไม่รู้)
- เมื่อเพิ่ม type ใหม่ ต้องเพิ่ม columns (ALTER TABLE)

เหมาะกับ:
- Types ที่มี attributes ซ้อนทับกันมาก
- Types ไม่หลากหลายมาก
- Read-heavy workloads
- Prototype/Simple applications
```

---

## Pattern 2: Table-per-Type (Class Table Inheritance)

### แนวคิด

```
             [products]           ← Base Table (shared attributes)
         /       |        \
    [phones]  [laptops]  [clothing]  ← Subtype Tables (type-specific)
    
แต่ละ Subtype Table มี FK → products.product_id
```

### Implementation

```sql
-- Base Table
CREATE TABLE products (
    product_id    SERIAL PRIMARY KEY,
    product_type  VARCHAR(20) NOT NULL
                  CHECK (product_type IN ('phone','laptop','tablet','clothing','book')),
    name          VARCHAR(200) NOT NULL,
    sku           VARCHAR(50) UNIQUE NOT NULL,
    price         DECIMAL(10,2) NOT NULL,
    stock_qty     INTEGER DEFAULT 0,
    is_active     BOOLEAN DEFAULT TRUE,
    created_at    TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Phone Subtype Table
CREATE TABLE phones (
    product_id    INTEGER PRIMARY KEY REFERENCES products ON DELETE CASCADE,
    brand         VARCHAR(50) NOT NULL,
    model         VARCHAR(100) NOT NULL,
    storage_gb    INTEGER NOT NULL,
    ram_gb        INTEGER NOT NULL,
    color         VARCHAR(30),
    screen_inch   DECIMAL(4,2),
    battery_mah   INTEGER,
    is_5g         BOOLEAN DEFAULT FALSE,
    os            VARCHAR(20) DEFAULT 'Android'
);

-- Laptop Subtype Table
CREATE TABLE laptops (
    product_id    INTEGER PRIMARY KEY REFERENCES products ON DELETE CASCADE,
    brand         VARCHAR(50) NOT NULL,
    processor     VARCHAR(100) NOT NULL,
    storage_gb    INTEGER NOT NULL,
    ram_gb        INTEGER NOT NULL,
    screen_inch   DECIMAL(4,2),
    os            VARCHAR(50),
    weight_kg     DECIMAL(5,3),
    has_touchscreen BOOLEAN DEFAULT FALSE
);

-- Clothing Subtype Table
CREATE TABLE clothing (
    product_id    INTEGER PRIMARY KEY REFERENCES products ON DELETE CASCADE,
    brand         VARCHAR(50),
    size          VARCHAR(10) NOT NULL,
    color         VARCHAR(30),
    material      VARCHAR(100),
    gender        VARCHAR(20) CHECK (gender IN ('men','women','unisex','kids'))
);

-- Book Subtype Table
CREATE TABLE books (
    product_id    INTEGER PRIMARY KEY REFERENCES products ON DELETE CASCADE,
    isbn          VARCHAR(20) UNIQUE,
    author        VARCHAR(200) NOT NULL,
    publisher     VARCHAR(100),
    pages         INTEGER,
    language      VARCHAR(20) DEFAULT 'Thai'
);

-- Insert Phone
BEGIN;
    INSERT INTO products (product_type, name, sku, price, stock_qty)
    VALUES ('phone', 'Samsung Galaxy S25', 'SAMS25-128', 32900.00, 50)
    RETURNING product_id INTO v_product_id;

    INSERT INTO phones (product_id, brand, model, storage_gb, ram_gb, is_5g)
    VALUES (v_product_id, 'Samsung', 'Galaxy S25', 128, 12, TRUE);
COMMIT;

-- View สำหรับ Phones
CREATE VIEW phone_catalog AS
SELECT
    p.product_id,
    p.name,
    p.sku,
    p.price,
    p.stock_qty,
    ph.brand,
    ph.model,
    ph.storage_gb,
    ph.ram_gb,
    ph.is_5g,
    ph.screen_inch
FROM products p
JOIN phones ph ON p.product_id = ph.product_id
WHERE p.is_active = TRUE;

-- Query ง่ายด้วย View
SELECT * FROM phone_catalog WHERE brand = 'Samsung' AND storage_gb >= 256;

-- Query ทุก type (Base Table)
SELECT p.name, p.price, p.product_type
FROM products p
WHERE p.price < 10000 AND p.is_active = TRUE
ORDER BY p.price;
```

### ข้อดี/ข้อเสีย ของ Class Table Inheritance

```
✅ ข้อดี:
- Clean Schema: ไม่มี Nullable columns
- Type Safety: NOT NULL ทำได้
- Efficient storage
- ง่ายต่อการเพิ่ม type ใหม่ (เพิ่มตาราง ไม่ต้องแก้ตารางเดิม)

❌ ข้อเสีย:
- JOIN ทุกครั้งที่ Query Type-specific data
- Query ข้าม Types ซับซ้อนขึ้น
- Insert ต้อง Insert 2 ตาราง (ต้อง Transaction)
- Delete Cascade ต้องระวัง

เหมาะกับ:
- Types มี Attributes ต่างกันมาก
- ต้องการ Database-level Constraints
- Medium-large applications
- Subtypes มีความหมายทาง Business ชัดเจน
```

---

## Pattern 3: Table-per-Concrete-Class

### แนวคิด

```
ไม่มี Base Table
แต่ละ Concrete Type มีตาราง Full ของตัวเอง

[phones]       ← เก็บทุก phone attributes + common attributes
[laptops]      ← เก็บทุก laptop attributes + common attributes  
[clothing]     ← เก็บทุก clothing attributes + common attributes
```

### Implementation

```sql
-- แต่ละตาราง Duplicate common columns
CREATE TABLE phones (
    product_id    SERIAL PRIMARY KEY,
    -- Common
    name          VARCHAR(200) NOT NULL,
    sku           VARCHAR(50) UNIQUE NOT NULL,
    price         DECIMAL(10,2) NOT NULL,
    stock_qty     INTEGER DEFAULT 0,
    is_active     BOOLEAN DEFAULT TRUE,
    created_at    TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    -- Phone-specific (NOT NULL เพราะรู้ว่าต้องมี)
    brand         VARCHAR(50) NOT NULL,
    model         VARCHAR(100) NOT NULL,
    storage_gb    INTEGER NOT NULL,
    ram_gb        INTEGER NOT NULL,
    is_5g         BOOLEAN DEFAULT FALSE
);

CREATE TABLE laptops (
    product_id    SERIAL PRIMARY KEY,
    -- Common (Duplicated!)
    name          VARCHAR(200) NOT NULL,
    sku           VARCHAR(50) UNIQUE NOT NULL,
    price         DECIMAL(10,2) NOT NULL,
    stock_qty     INTEGER DEFAULT 0,
    is_active     BOOLEAN DEFAULT TRUE,
    created_at    TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    -- Laptop-specific
    brand         VARCHAR(50) NOT NULL,
    processor     VARCHAR(100) NOT NULL,
    storage_gb    INTEGER NOT NULL,
    ram_gb        INTEGER NOT NULL
);

CREATE TABLE books (
    product_id    SERIAL PRIMARY KEY,
    -- Common (Duplicated!)
    name          VARCHAR(200) NOT NULL,
    sku           VARCHAR(50) UNIQUE NOT NULL,
    price         DECIMAL(10,2) NOT NULL,
    stock_qty     INTEGER DEFAULT 0,
    is_active     BOOLEAN DEFAULT TRUE,
    created_at    TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    -- Book-specific
    isbn          VARCHAR(20) UNIQUE NOT NULL,
    author        VARCHAR(200) NOT NULL,
    publisher     VARCHAR(100),
    pages         INTEGER
);

-- Query ข้าม Types ต้องใช้ UNION
SELECT product_id, name, price, 'phone' AS type FROM phones WHERE is_active = TRUE
UNION ALL
SELECT product_id, name, price, 'laptop' AS type FROM laptops WHERE is_active = TRUE
UNION ALL
SELECT product_id, name, price, 'book' AS type FROM books WHERE is_active = TRUE
ORDER BY name;
```

### ข้อดี/ข้อเสีย ของ Concrete Table Inheritance

```
✅ ข้อดี:
- ง่ายที่สุดในแต่ละ Type ตัวเอง
- Query ต่อ Type เร็วมาก (ไม่ต้อง JOIN)
- Type-specific Indexes ง่าย
- แต่ละตาราง Independent

❌ ข้อเสีย:
- Duplicate columns ทุกตาราง (ยากต่อ Schema Changes)
- ไม่มี Common Primary Key
- Query ข้าม Types ต้อง UNION (ช้า)
- เพิ่ม common column ต้อง ALTER ทุกตาราง
- ยากต่อ Polymorphic References (FK ไปที่ตารางไหน?)

เหมาะกับ:
- Types มีความแตกต่างมาก ไม่ค่อย Query ข้าม Types
- Read-heavy, Single-type queries
- Simple Applications
```

---

## Employee Types Example

### Class Table Inheritance สำหรับ HR System

```sql
-- Base Table: employees
CREATE TABLE employees (
    employee_id   SERIAL PRIMARY KEY,
    emp_type      VARCHAR(20) NOT NULL
                  CHECK (emp_type IN ('full_time','part_time','contractor','intern')),
    -- Common info
    first_name    VARCHAR(100) NOT NULL,
    last_name     VARCHAR(100) NOT NULL,
    email         VARCHAR(255) UNIQUE NOT NULL,
    phone         VARCHAR(20),
    hire_date     DATE NOT NULL,
    department_id INTEGER REFERENCES departments,
    manager_id    INTEGER REFERENCES employees,
    is_active     BOOLEAN DEFAULT TRUE
);

-- Full-time Employee
CREATE TABLE full_time_employees (
    employee_id     INTEGER PRIMARY KEY REFERENCES employees ON DELETE CASCADE,
    base_salary     DECIMAL(12,2) NOT NULL,
    bonus_eligible  BOOLEAN DEFAULT TRUE,
    annual_leave_days INTEGER DEFAULT 15,
    health_insurance BOOLEAN DEFAULT TRUE,
    pension_scheme   BOOLEAN DEFAULT TRUE,
    probation_end_date DATE
);

-- Part-time Employee
CREATE TABLE part_time_employees (
    employee_id     INTEGER PRIMARY KEY REFERENCES employees ON DELETE CASCADE,
    hourly_rate     DECIMAL(8,2) NOT NULL,
    max_hours_week  SMALLINT DEFAULT 20,
    schedule        JSONB,  -- {"mon": "09:00-13:00", "wed": "09:00-13:00"}
    overtime_eligible BOOLEAN DEFAULT FALSE
);

-- Contractor
CREATE TABLE contractors (
    employee_id     INTEGER PRIMARY KEY REFERENCES employees ON DELETE CASCADE,
    company_name    VARCHAR(200),
    contract_start  DATE NOT NULL,
    contract_end    DATE NOT NULL,
    daily_rate      DECIMAL(10,2) NOT NULL,
    payment_terms   VARCHAR(50) DEFAULT 'monthly',
    tax_id          VARCHAR(20),
    CONSTRAINT valid_contract CHECK (contract_end > contract_start)
);

-- Intern
CREATE TABLE interns (
    employee_id     INTEGER PRIMARY KEY REFERENCES employees ON DELETE CASCADE,
    university      VARCHAR(200) NOT NULL,
    program         VARCHAR(200),
    graduation_year SMALLINT,
    stipend_monthly DECIMAL(8,2),
    mentor_id       INTEGER REFERENCES employees
);

-- Views per type
CREATE VIEW full_time_staff AS
SELECT
    e.employee_id, e.first_name, e.last_name, e.email,
    e.hire_date, e.department_id, e.is_active,
    ft.base_salary, ft.bonus_eligible, ft.annual_leave_days,
    ft.health_insurance, ft.pension_scheme
FROM employees e
JOIN full_time_employees ft ON e.employee_id = ft.employee_id;

CREATE VIEW all_employees_summary AS
SELECT
    e.employee_id,
    e.first_name || ' ' || e.last_name AS full_name,
    e.emp_type,
    e.email,
    d.name AS department,
    CASE e.emp_type
        WHEN 'full_time'  THEN ft.base_salary || '/month'
        WHEN 'part_time'  THEN pt.hourly_rate || '/hour'
        WHEN 'contractor' THEN c.daily_rate || '/day'
        WHEN 'intern'     THEN COALESCE(i.stipend_monthly::TEXT, 'unpaid') || '/month'
    END AS compensation
FROM employees e
LEFT JOIN departments d ON e.department_id = d.department_id
LEFT JOIN full_time_employees ft ON e.employee_id = ft.employee_id
LEFT JOIN part_time_employees pt ON e.employee_id = pt.employee_id
LEFT JOIN contractors c ON e.employee_id = c.employee_id
LEFT JOIN interns i ON e.employee_id = i.employee_id
WHERE e.is_active = TRUE;
```

---

## PostgreSQL Native Table Inheritance

### PostgreSQL INHERITS Keyword

```sql
-- PostgreSQL มี Built-in Table Inheritance!
CREATE TABLE vehicles (
    vehicle_id    SERIAL,
    make          VARCHAR(50) NOT NULL,
    model         VARCHAR(100) NOT NULL,
    year          SMALLINT NOT NULL,
    vin           VARCHAR(17) UNIQUE,
    color         VARCHAR(30),
    price         DECIMAL(10,2)
);

-- Child tables inherit all columns from vehicles
CREATE TABLE cars (
    num_doors     SMALLINT DEFAULT 4,
    body_type     VARCHAR(20)  -- 'sedan', 'suv', 'hatchback'
) INHERITS (vehicles);

CREATE TABLE motorcycles (
    engine_cc     INTEGER,
    has_sidecar   BOOLEAN DEFAULT FALSE
) INHERITS (vehicles);

CREATE TABLE trucks (
    payload_tons  DECIMAL(6,2),
    num_axles     SMALLINT DEFAULT 2
) INHERITS (vehicles);

-- Insert
INSERT INTO cars (make, model, year, color, price, num_doors, body_type)
VALUES ('Toyota', 'Camry', 2024, 'silver', 1200000.00, 4, 'sedan');

INSERT INTO motorcycles (make, model, year, engine_cc)
VALUES ('Honda', 'CBR600RR', 2024, 600);

-- Query ทุก vehicles (รวม child tables!)
SELECT * FROM ONLY vehicles;  -- เฉพาะ vehicles table
SELECT * FROM vehicles;       -- รวม child tables ด้วย!

-- ข้อจำกัดของ PostgreSQL INHERITS:
-- ❌ FK ที่ชี้มาที่ Parent Table ไม่ Cascade ถึง Child
-- ❌ UNIQUE บน Parent ไม่ครอบคลุม Child
-- ✅ CHECK Constraints ถ่ายทอดมาได้
-- ✅ Column definitions ถ่ายทอด
```

---

## เปรียบเทียบ 3 Patterns

```
┌─────────────────────────┬──────────────────┬──────────────────┬──────────────────┐
│ Feature                 │ Single Table     │ Class Table      │ Concrete Table   │
├─────────────────────────┼──────────────────┼──────────────────┼──────────────────┤
│ Tables needed           │ 1                │ 1 + N subtypes   │ N subtypes       │
│ Nullable columns        │ Many             │ None             │ None             │
│ Type constraints (DB)   │ No               │ Yes              │ Yes              │
│ Cross-type queries      │ Easy             │ Moderate         │ Hard (UNION)     │
│ Single-type queries     │ Fast             │ Fast (JOIN)      │ Fastest          │
│ Add new type            │ ALTER TABLE      │ CREATE TABLE     │ CREATE TABLE     │
│ Add common column       │ 1 ALTER          │ 1 ALTER          │ N ALTERs         │
│ Polymorphic FK          │ Easy             │ Easy             │ Impossible       │
│ Storage efficiency      │ Poor (sparse)    │ Good             │ Good             │
└─────────────────────────┴──────────────────┴──────────────────┴──────────────────┘
```

---

## Polymorphism in SQL

### Using Views for Polymorphism

```sql
-- Polymorphic "interface" ด้วย Views
CREATE VIEW all_sellable_items AS
    SELECT product_id, name, price, 'product' AS item_type FROM products WHERE is_active = TRUE
    UNION ALL
    SELECT subscription_id, name, monthly_price, 'subscription' FROM subscriptions WHERE is_active = TRUE
    UNION ALL
    SELECT service_id, name, hourly_rate, 'service' FROM services WHERE is_active = TRUE;

-- Order Line Items รองรับทุก item types
CREATE TABLE order_items (
    order_item_id SERIAL PRIMARY KEY,
    order_id      INTEGER REFERENCES orders,
    item_type     VARCHAR(20) NOT NULL CHECK (item_type IN ('product','subscription','service')),
    item_id       INTEGER NOT NULL,
    quantity      INTEGER NOT NULL DEFAULT 1,
    unit_price    DECIMAL(10,2) NOT NULL
);
```

### Discriminated Union Pattern

```sql
-- Payment Methods: Credit Card, Bank Transfer, Wallet
CREATE TABLE payment_methods (
    method_id   SERIAL PRIMARY KEY,
    user_id     INTEGER REFERENCES users,
    method_type VARCHAR(20) NOT NULL CHECK (method_type IN ('credit_card','bank_transfer','wallet')),
    is_default  BOOLEAN DEFAULT FALSE,
    created_at  TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE credit_cards (
    method_id       INTEGER PRIMARY KEY REFERENCES payment_methods ON DELETE CASCADE,
    card_holder     VARCHAR(100) NOT NULL,
    card_last4      CHAR(4) NOT NULL,
    card_type       VARCHAR(20),  -- 'visa', 'mastercard'
    expiry_month    SMALLINT NOT NULL CHECK (expiry_month BETWEEN 1 AND 12),
    expiry_year     SMALLINT NOT NULL,
    token           VARCHAR(200)  -- payment gateway token (never store full card)
);

CREATE TABLE bank_transfers (
    method_id       INTEGER PRIMARY KEY REFERENCES payment_methods ON DELETE CASCADE,
    bank_name       VARCHAR(100) NOT NULL,
    account_number  VARCHAR(20) NOT NULL,  -- Should be encrypted
    account_name    VARCHAR(100) NOT NULL,
    routing_number  VARCHAR(20)
);

CREATE TABLE digital_wallets (
    method_id       INTEGER PRIMARY KEY REFERENCES payment_methods ON DELETE CASCADE,
    wallet_type     VARCHAR(30),  -- 'promptpay', 'truemoney', 'linepay'
    wallet_id       VARCHAR(100) NOT NULL,  -- phone number or email
    display_name    VARCHAR(100)
);

-- View: User's Payment Methods
CREATE VIEW user_payment_methods AS
SELECT
    pm.method_id,
    pm.user_id,
    pm.method_type,
    pm.is_default,
    CASE pm.method_type
        WHEN 'credit_card' THEN cc.card_type || ' ending ' || cc.card_last4
        WHEN 'bank_transfer' THEN bt.bank_name || ' ' || bt.account_number
        WHEN 'wallet' THEN dw.wallet_type || ' - ' || dw.wallet_id
    END AS display_name
FROM payment_methods pm
LEFT JOIN credit_cards cc ON pm.method_id = cc.method_id
LEFT JOIN bank_transfers bt ON pm.method_id = bt.method_id
LEFT JOIN digital_wallets dw ON pm.method_id = dw.method_id;
```

---

## แบบฝึกหัด (10 ข้อ)

### ข้อ 1
ออกแบบ Single Table Inheritance สำหรับ Notification System ที่มี 3 types: Email, SMS, Push Notification

**เฉลย**:
```sql
CREATE TABLE notifications (
    notification_id  SERIAL PRIMARY KEY,
    notif_type       VARCHAR(10) NOT NULL CHECK (notif_type IN ('email','sms','push')),
    user_id          INTEGER NOT NULL REFERENCES users,
    subject          VARCHAR(300),
    body             TEXT NOT NULL,
    status           VARCHAR(20) DEFAULT 'pending',
    created_at       TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    sent_at          TIMESTAMP,
    -- Email-specific
    email_from       VARCHAR(255),
    email_to         VARCHAR(255),
    email_cc         TEXT,
    -- SMS-specific
    sms_from_number  VARCHAR(20),
    sms_to_number    VARCHAR(20),
    sms_provider     VARCHAR(20),
    -- Push-specific
    push_device_token TEXT,
    push_platform    VARCHAR(10),  -- 'ios', 'android'
    push_badge_count INTEGER
);

CREATE INDEX idx_notifs_user_type ON notifications(user_id, notif_type, status);
CREATE INDEX idx_notifs_pending ON notifications(status, created_at)
    WHERE status = 'pending';
```

### ข้อ 2
ออกแบบ Class Table Inheritance สำหรับ Document System (Word, Excel, PDF, Image)

**เฉลย**:
```sql
CREATE TABLE documents (
    doc_id       SERIAL PRIMARY KEY,
    doc_type     VARCHAR(10) NOT NULL CHECK (doc_type IN ('word','excel','pdf','image')),
    title        VARCHAR(300) NOT NULL,
    file_path    TEXT NOT NULL,
    file_size_kb INTEGER,
    owner_id     INTEGER REFERENCES users,
    created_at   TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at   TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE word_documents (
    doc_id       INTEGER PRIMARY KEY REFERENCES documents ON DELETE CASCADE,
    page_count   INTEGER,
    word_count   INTEGER,
    has_tracked_changes BOOLEAN DEFAULT FALSE,
    template     VARCHAR(100)
);

CREATE TABLE excel_sheets (
    doc_id       INTEGER PRIMARY KEY REFERENCES documents ON DELETE CASCADE,
    sheet_count  SMALLINT DEFAULT 1,
    row_count    INTEGER,
    has_macros   BOOLEAN DEFAULT FALSE,
    has_charts   BOOLEAN DEFAULT FALSE
);

CREATE TABLE pdf_files (
    doc_id       INTEGER PRIMARY KEY REFERENCES documents ON DELETE CASCADE,
    page_count   INTEGER,
    is_searchable BOOLEAN DEFAULT TRUE,
    is_password_protected BOOLEAN DEFAULT FALSE
);

CREATE TABLE images (
    doc_id       INTEGER PRIMARY KEY REFERENCES documents ON DELETE CASCADE,
    width_px     INTEGER,
    height_px    INTEGER,
    color_mode   VARCHAR(10),  -- 'RGB', 'CMYK', 'grayscale'
    format       VARCHAR(10)   -- 'jpg', 'png', 'webp'
);
```

### ข้อ 3
ออกแบบ Concrete Table Inheritance สำหรับ Sensor Data (Temperature, Humidity, Pressure)

**เฉลย**:
```sql
-- Common columns duplicated per table
CREATE TABLE temperature_readings (
    reading_id   BIGSERIAL PRIMARY KEY,
    sensor_id    VARCHAR(50) NOT NULL,
    location_id  INTEGER,
    recorded_at  TIMESTAMP NOT NULL,
    -- Type-specific
    celsius      DECIMAL(6,2) NOT NULL,
    fahrenheit   DECIMAL(6,2) GENERATED ALWAYS AS (celsius * 9.0/5.0 + 32) STORED
) PARTITION BY RANGE (recorded_at);

CREATE TABLE humidity_readings (
    reading_id   BIGSERIAL PRIMARY KEY,
    sensor_id    VARCHAR(50) NOT NULL,
    location_id  INTEGER,
    recorded_at  TIMESTAMP NOT NULL,
    -- Type-specific
    percent      DECIMAL(5,2) NOT NULL CHECK (percent BETWEEN 0 AND 100)
) PARTITION BY RANGE (recorded_at);

CREATE TABLE pressure_readings (
    reading_id   BIGSERIAL PRIMARY KEY,
    sensor_id    VARCHAR(50) NOT NULL,
    location_id  INTEGER,
    recorded_at  TIMESTAMP NOT NULL,
    -- Type-specific
    hpa          DECIMAL(8,2) NOT NULL,  -- hectopascals
    altitude_m   DECIMAL(8,2)
) PARTITION BY RANGE (recorded_at);
```

### ข้อ 4
เปรียบเทียบ 3 Inheritance Patterns สำหรับ Banking System ที่มี Account Types: Savings, Checking, Investment

**เฉลย**:
```
Banking System Analysis:

1. Single Table Inheritance:
   - ✅ Easy cross-account queries: "แสดง accounts ทั้งหมดของ user"
   - ❌ Savings ต้องการ interest_rate (NOT NULL) แต่ DB บังคับไม่ได้
   - ❌ Investment ต้องการ portfolio_data ที่ Nullable ใน Checking/Savings

2. Class Table Inheritance (แนะนำ):
   - ✅ Type-safe: savings.min_balance ไม่ต้อง Nullable
   - ✅ Query cross-account ยังทำได้ผ่าน accounts base table
   - ✅ เพิ่ม account type ใหม่ทำได้โดยไม่กระทบตารางเดิม

3. Concrete Table:
   - ❌ ยากมากสำหรับ "แสดงทุก accounts ของ user"
   - ❌ Transaction ข้าม account types ซับซ้อน

สรุป: ใช้ Class Table Inheritance

CREATE TABLE accounts (
    account_id    SERIAL PRIMARY KEY,
    account_type  VARCHAR(20) NOT NULL CHECK (account_type IN ('savings','checking','investment')),
    user_id       INTEGER REFERENCES users,
    account_no    VARCHAR(20) UNIQUE NOT NULL,
    balance       DECIMAL(15,2) DEFAULT 0,
    opened_date   DATE DEFAULT CURRENT_DATE,
    is_active     BOOLEAN DEFAULT TRUE
);

CREATE TABLE savings_accounts (
    account_id    INTEGER PRIMARY KEY REFERENCES accounts,
    interest_rate DECIMAL(5,4) NOT NULL,  -- 0.0150 = 1.5%
    min_balance   DECIMAL(10,2) DEFAULT 0,
    compounding   VARCHAR(20) DEFAULT 'monthly'
);

CREATE TABLE checking_accounts (
    account_id    INTEGER PRIMARY KEY REFERENCES accounts,
    overdraft_limit DECIMAL(10,2) DEFAULT 0,
    monthly_fee     DECIMAL(8,2) DEFAULT 0,
    free_atm_count  INTEGER DEFAULT 3
);

CREATE TABLE investment_accounts (
    account_id    INTEGER PRIMARY KEY REFERENCES accounts,
    risk_profile  VARCHAR(20) CHECK (risk_profile IN ('conservative','balanced','aggressive')),
    portfolio     JSONB DEFAULT '{}',
    fee_percent   DECIMAL(5,4)
);
```

### ข้อ 5
Implement Polymorphic Address สำหรับ Customers และ Suppliers ที่ share Address logic

**เฉลย**:
```sql
CREATE TABLE addresses (
    address_id     SERIAL PRIMARY KEY,
    addressable_type VARCHAR(20) NOT NULL CHECK (addressable_type IN ('customer','supplier')),
    addressable_id   INTEGER NOT NULL,
    type           VARCHAR(20) DEFAULT 'main',  -- 'main', 'shipping', 'billing'
    street         VARCHAR(200) NOT NULL,
    city           VARCHAR(100) NOT NULL,
    province       VARCHAR(100),
    postal_code    VARCHAR(10),
    country_code   CHAR(2) DEFAULT 'TH',
    is_default     BOOLEAN DEFAULT FALSE,
    UNIQUE (addressable_type, addressable_id, type)
);

CREATE INDEX idx_addresses_entity ON addresses(addressable_type, addressable_id);

-- ดู addresses ของ Customer
SELECT * FROM addresses WHERE addressable_type = 'customer' AND addressable_id = 1001;

-- ดู default shipping address
SELECT a.*
FROM addresses a
WHERE addressable_type = 'customer'
AND addressable_id = 1001
AND type = 'shipping'
AND is_default = TRUE;
```

### ข้อ 6
ออกแบบ Class Table Inheritance สำหรับ Vehicle Fleet Management

**เฉลย**:
```sql
CREATE TABLE vehicles (
    vehicle_id   SERIAL PRIMARY KEY,
    vehicle_type VARCHAR(15) NOT NULL CHECK (vehicle_type IN ('car','truck','motorcycle','bus')),
    plate_no     VARCHAR(20) UNIQUE NOT NULL,
    make         VARCHAR(50) NOT NULL,
    model        VARCHAR(100) NOT NULL,
    year         SMALLINT NOT NULL,
    color        VARCHAR(30),
    fuel_type    VARCHAR(20) DEFAULT 'gasoline',
    mileage      INTEGER DEFAULT 0,
    last_service DATE,
    is_available BOOLEAN DEFAULT TRUE
);

CREATE TABLE fleet_cars (
    vehicle_id   INTEGER PRIMARY KEY REFERENCES vehicles ON DELETE CASCADE,
    num_seats    SMALLINT DEFAULT 5,
    body_type    VARCHAR(20),
    ac_present   BOOLEAN DEFAULT TRUE
);

CREATE TABLE fleet_trucks (
    vehicle_id    INTEGER PRIMARY KEY REFERENCES vehicles ON DELETE CASCADE,
    payload_tons  DECIMAL(6,2) NOT NULL,
    num_axles     SMALLINT DEFAULT 2,
    refrigerated  BOOLEAN DEFAULT FALSE
);

CREATE TABLE fleet_buses (
    vehicle_id    INTEGER PRIMARY KEY REFERENCES vehicles ON DELETE CASCADE,
    seating_cap   SMALLINT NOT NULL,
    route_id      INTEGER,
    has_wifi      BOOLEAN DEFAULT FALSE
);

-- Query available vehicles
SELECT v.plate_no, v.make, v.model, v.vehicle_type,
       CASE v.vehicle_type
           WHEN 'car' THEN c.num_seats || ' seats'
           WHEN 'truck' THEN t.payload_tons || ' tons'
           WHEN 'bus' THEN b.seating_cap || ' passengers'
       END AS capacity
FROM vehicles v
LEFT JOIN fleet_cars c ON v.vehicle_id = c.vehicle_id
LEFT JOIN fleet_trucks t ON v.vehicle_id = t.vehicle_id
LEFT JOIN fleet_buses b ON v.vehicle_id = b.vehicle_id
WHERE v.is_available = TRUE;
```

### ข้อ 7
สร้าง Single Table Inheritance สำหรับ Task Management (Todo, Meeting, Reminder)

**เฉลย**:
```sql
CREATE TABLE tasks (
    task_id       SERIAL PRIMARY KEY,
    task_type     VARCHAR(10) NOT NULL CHECK (task_type IN ('todo','meeting','reminder')),
    user_id       INTEGER NOT NULL REFERENCES users,
    title         VARCHAR(300) NOT NULL,
    description   TEXT,
    status        VARCHAR(20) DEFAULT 'pending',
    priority      VARCHAR(10) DEFAULT 'medium',
    due_date      DATE,
    created_at    TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    -- Meeting-specific
    meeting_location  VARCHAR(200),
    meeting_url       TEXT,
    meeting_start     TIMESTAMP,
    meeting_end       TIMESTAMP,
    meeting_attendees INTEGER[],  -- Array of user_ids
    -- Reminder-specific
    reminder_at       TIMESTAMP,
    reminder_repeat   VARCHAR(20),  -- 'daily', 'weekly', 'none'
    -- Todo-specific
    todo_checklist    JSONB,  -- [{"item": "Buy milk", "done": false}]
    todo_estimate_min INTEGER
);

CREATE INDEX idx_tasks_user ON tasks(user_id, task_type, status);
CREATE INDEX idx_tasks_meeting ON tasks(meeting_start) WHERE task_type = 'meeting';
CREATE INDEX idx_tasks_reminder ON tasks(reminder_at) WHERE task_type = 'reminder' AND status = 'pending';
```

### ข้อ 8
อธิบายเมื่อใดควรใช้ PostgreSQL INHERITS vs Class Table Inheritance

**เฉลย**:
```
PostgreSQL INHERITS:
- ใช้เมื่อ: Table Partitioning เป็น main goal
- ใช้เมื่อ: Time-series data แบ่งตาม date ranges
- ข้อจำกัด: UNIQUE constraints ไม่ cross-table, FK pointing to parent ไม่ cascade to children
- เหมาะกับ: PostgreSQL-specific applications

Class Table Inheritance:
- ใช้เมื่อ: ต้องการ Full Relational Integrity
- ใช้เมื่อ: Application ต้องทำงานได้กับ Multiple DBs
- ใช้เมื่อ: Types ต้องการ Type-specific Constraints (NOT NULL, CHECK, FK)
- เหมาะกับ: Production Business Applications

ตัวอย่าง PostgreSQL INHERITS ที่เหมาะ:
CREATE TABLE orders_2024 () INHERITS (orders);
CREATE TABLE orders_2025 () INHERITS (orders);
-- ใช้สำหรับ Partitioning ตามปี
```

### ข้อ 9
ออกแบบ Class Table Inheritance สำหรับ Content Management (Article, Video, Podcast, Gallery)

**เฉลย**:
```sql
CREATE TABLE content (
    content_id   SERIAL PRIMARY KEY,
    content_type VARCHAR(10) NOT NULL CHECK (content_type IN ('article','video','podcast','gallery')),
    author_id    INTEGER NOT NULL REFERENCES users,
    title        VARCHAR(300) NOT NULL,
    slug         VARCHAR(300) UNIQUE NOT NULL,
    status       VARCHAR(20) DEFAULT 'draft',
    published_at TIMESTAMP,
    created_at   TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    views        INTEGER DEFAULT 0,
    likes        INTEGER DEFAULT 0
);

CREATE TABLE articles (
    content_id   INTEGER PRIMARY KEY REFERENCES content ON DELETE CASCADE,
    body         TEXT NOT NULL,
    word_count   INTEGER,
    reading_min  INTEGER,
    cover_image  TEXT
);

CREATE TABLE videos (
    content_id    INTEGER PRIMARY KEY REFERENCES content ON DELETE CASCADE,
    video_url     TEXT NOT NULL,
    duration_sec  INTEGER NOT NULL,
    thumbnail_url TEXT,
    resolution    VARCHAR(10)  -- '1080p', '4k'
);

CREATE TABLE podcasts (
    content_id    INTEGER PRIMARY KEY REFERENCES content ON DELETE CASCADE,
    audio_url     TEXT NOT NULL,
    duration_sec  INTEGER NOT NULL,
    episode_no    INTEGER,
    season_no     INTEGER,
    transcript    TEXT
);

CREATE TABLE galleries (
    content_id    INTEGER PRIMARY KEY REFERENCES content ON DELETE CASCADE,
    image_count   INTEGER NOT NULL DEFAULT 0,
    cover_image   TEXT
);

CREATE TABLE gallery_images (
    image_id     SERIAL PRIMARY KEY,
    content_id   INTEGER NOT NULL REFERENCES galleries(content_id) ON DELETE CASCADE,
    image_url    TEXT NOT NULL,
    caption      VARCHAR(300),
    sort_order   SMALLINT DEFAULT 0
);
```

### ข้อ 10
Implement Full Class Table Inheritance สำหรับ Insurance Policy System

**เฉลย**:
```sql
CREATE TABLE insurance_policies (
    policy_id    SERIAL PRIMARY KEY,
    policy_type  VARCHAR(20) NOT NULL CHECK (policy_type IN ('life','health','auto','property')),
    customer_id  INTEGER NOT NULL REFERENCES customers,
    policy_no    VARCHAR(30) UNIQUE NOT NULL,
    start_date   DATE NOT NULL,
    end_date     DATE NOT NULL,
    premium      DECIMAL(12,2) NOT NULL,
    status       VARCHAR(20) DEFAULT 'active',
    CONSTRAINT valid_period CHECK (end_date > start_date)
);

CREATE TABLE life_policies (
    policy_id       INTEGER PRIMARY KEY REFERENCES insurance_policies ON DELETE CASCADE,
    insured_name    VARCHAR(100) NOT NULL,
    coverage_amount DECIMAL(14,2) NOT NULL,
    beneficiary     VARCHAR(100) NOT NULL,
    death_benefit   DECIMAL(14,2),
    has_riders      BOOLEAN DEFAULT FALSE
);

CREATE TABLE health_policies (
    policy_id       INTEGER PRIMARY KEY REFERENCES insurance_policies ON DELETE CASCADE,
    deductible      DECIMAL(10,2) DEFAULT 0,
    max_coverage    DECIMAL(12,2) NOT NULL,
    covers_dental   BOOLEAN DEFAULT FALSE,
    covers_vision   BOOLEAN DEFAULT FALSE,
    network_type    VARCHAR(20) DEFAULT 'HMO'
);

CREATE TABLE auto_policies (
    policy_id       INTEGER PRIMARY KEY REFERENCES insurance_policies ON DELETE CASCADE,
    vehicle_id      INTEGER REFERENCES vehicles,
    plate_no        VARCHAR(20) NOT NULL,
    coverage_type   VARCHAR(20),  -- 'liability', 'comprehensive', 'collision'
    roadside_assist BOOLEAN DEFAULT FALSE
);

CREATE TABLE property_policies (
    policy_id       INTEGER PRIMARY KEY REFERENCES insurance_policies ON DELETE CASCADE,
    property_address VARCHAR(300) NOT NULL,
    property_value  DECIMAL(14,2) NOT NULL,
    coverage_type   VARCHAR(20),  -- 'fire', 'flood', 'all-risk'
    replacement_value BOOLEAN DEFAULT TRUE
);

-- Customer's active policies
SELECT
    ip.policy_no,
    ip.policy_type,
    ip.start_date,
    ip.end_date,
    ip.premium,
    CASE ip.policy_type
        WHEN 'life'     THEN 'Coverage: ' || lp.coverage_amount
        WHEN 'health'   THEN 'Max: ' || hp.max_coverage
        WHEN 'auto'     THEN 'Plate: ' || ap.plate_no
        WHEN 'property' THEN 'Property: ' || pp.property_address
    END AS details
FROM insurance_policies ip
LEFT JOIN life_policies lp ON ip.policy_id = lp.policy_id
LEFT JOIN health_policies hp ON ip.policy_id = hp.policy_id
LEFT JOIN auto_policies ap ON ip.policy_id = ap.policy_id
LEFT JOIN property_policies pp ON ip.policy_id = pp.policy_id
WHERE ip.customer_id = 1001 AND ip.status = 'active';
```

---

*จบ Part 059: Advanced Schema Topics - Inheritance*

**ในส่วนต่อไป (Part 060)**: เราจะนำความรู้ทั้งหมดมาประยุกต์ใช้ในโปรเจกต์จริง 4 ระบบ: E-Commerce, Hospital Management, Social Media Platform และ School Management System
