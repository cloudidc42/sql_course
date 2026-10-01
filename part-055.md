# Part 055: Third Normal Form (3NF)
# Normal Form ระดับสาม

---

## บทนำ: ทบทวน 1NF และ 2NF

```
Normalization ที่เรียนมา:
1NF: ทุก Column มีค่า Atomic, ไม่มี Repeating Groups
2NF: ไม่มี Partial Dependencies (Non-key → Part of PK)

3NF เพิ่มเติม:
- ไม่มี Transitive Dependencies
- Non-key Attributes ต้อง depend โดยตรงบน Primary Key
- ไม่ใช่ depend ผ่าน Non-key Attribute อื่น
```

---

## Transitive Dependency คืออะไร?

**Transitive Dependency** เกิดขึ้นเมื่อ:
```
A → B → C
นั่นคือ A → C แบบ Transitive (ผ่าน B)

ตัวอย่าง:
emp_id → dept_id → dept_name
(รู้ emp_id → รู้ dept_id → รู้ dept_name)
→ emp_id → dept_name เป็น Transitive Dependency
```

```
ตัวอย่างในชีวิตจริง:

ตาราง EMPLOYEES(emp_id, name, dept_id, dept_name, dept_location)

FD Chain:
emp_id → dept_id            ✓ Direct
dept_id → dept_name         ✓ Direct
dept_id → dept_location     ✓ Direct
emp_id → dept_name          ← Transitive (ผ่าน dept_id)!
emp_id → dept_location      ← Transitive (ผ่าน dept_id)!

ปัญหา:
- เปลี่ยนชื่อแผนก ต้องแก้หลาย rows (Update Anomaly)
- ลบพนักงานคนสุดท้ายของแผนก → ข้อมูลแผนกหาย (Delete Anomaly)
- เพิ่มแผนกใหม่ที่ยังไม่มีพนักงานทำไม่ได้ (Insert Anomaly)
```

---

## 3NF คืออะไร?

ตารางอยู่ใน **3NF** เมื่อ:
1. อยู่ใน 2NF
2. ไม่มี Non-key Attribute ที่ **Transitively Dependent** บน Primary Key

หรือพูดง่ายๆ:
**ทุก Non-key Column ต้อง depend บน Primary Key โดยตรง ไม่ใช่ผ่าน Column อื่น**

### คำนิยามทางการ (Codd's Definition)
```
ตาราง R อยู่ใน 3NF ก็ต่อเมื่อ:
สำหรับทุก FD: X → Y ใน R
อย่างน้อยหนึ่งในนี้ต้องเป็นจริง:
1. X เป็น Superkey ของ R
2. Y เป็น Prime Attribute (ส่วนหนึ่งของ Candidate Key)
```

---

## Anomalies จาก Transitive Dependencies

### ตัวอย่าง: EMPLOYEES ที่ละเมิด 3NF

```sql
CREATE TABLE employees_bad (
    emp_id      INTEGER PRIMARY KEY,
    name        VARCHAR(100),
    dept_id     INTEGER,
    dept_name   VARCHAR(100),   -- Transitive: emp_id → dept_id → dept_name
    dept_manager VARCHAR(100)   -- Transitive: emp_id → dept_id → dept_manager
);

-- ข้อมูล:
INSERT INTO employees_bad VALUES
(1, 'Alice', 10, 'Engineering', 'Bob'),
(2, 'Charlie', 10, 'Engineering', 'Bob'),
(3, 'David', 20, 'Marketing', 'Eve'),
(4, 'Frank', 10, 'Engineering', 'Bob');
```

```
Insert Anomaly:
ต้องการเพิ่มแผนก 'Finance' แต่ยังไม่มีพนักงาน → ทำไม่ได้!
เพราะ dept_id ไม่มี NOT NULL แต่ emp_id เป็น PK ต้องมีพนักงาน

Update Anomaly:
เปลี่ยน dept_manager ของ Engineering จาก Bob เป็น Alice:
→ ต้องแก้ 3 rows (emp_id: 1, 2, 4)
→ ถ้าแก้ไม่ครบ → Inconsistency!

Delete Anomaly:
ลบ emp_id=3 (David, แผนก Marketing):
→ ถ้า David เป็นพนักงานคนเดียวในแผนก Marketing
→ ข้อมูล Marketing (dept_id=20, manager=Eve) หายไปด้วย!
```

---

## ตัวอย่างที่ 1: EMPLOYEES → 3NF

```sql
-- ❌ BEFORE (ละเมิด 3NF)
CREATE TABLE employees_bad (
    emp_id       INTEGER PRIMARY KEY,
    name         VARCHAR(100),
    dept_id      INTEGER,
    dept_name    VARCHAR(100),    -- Transitive!
    dept_manager VARCHAR(100),    -- Transitive!
    zip_code     VARCHAR(10),
    city         VARCHAR(100),    -- Transitive! zip_code → city
    state        VARCHAR(50)      -- Transitive! zip_code → state
);
```

```
FD Analysis:
emp_id → dept_id    (Direct)
dept_id → dept_name  (Transitive: emp_id → dept_id → dept_name)
dept_id → dept_manager (Transitive)
emp_id → zip_code   (Direct)
zip_code → city      (Transitive: emp_id → zip_code → city)
zip_code → state     (Transitive)
```

```sql
-- ✅ AFTER (3NF)
CREATE TABLE departments (
    dept_id   INTEGER PRIMARY KEY,
    name      VARCHAR(100) NOT NULL,
    manager   VARCHAR(100)
);

CREATE TABLE zip_codes (
    zip_code  VARCHAR(10) PRIMARY KEY,
    city      VARCHAR(100),
    state     VARCHAR(50)
);

CREATE TABLE employees (
    emp_id    INTEGER PRIMARY KEY,
    name      VARCHAR(100) NOT NULL,
    dept_id   INTEGER REFERENCES departments,
    zip_code  VARCHAR(10) REFERENCES zip_codes
);
```

---

## ตัวอย่างที่ 2: ORDERS กับ CUSTOMER INFO

```sql
-- ❌ BEFORE (ละเมิด 3NF)
CREATE TABLE orders_bad (
    order_id      INTEGER PRIMARY KEY,
    customer_id   INTEGER,
    customer_name VARCHAR(100),   -- Transitive: order_id → customer_id → customer_name
    customer_email VARCHAR(255),  -- Transitive
    customer_city VARCHAR(100),   -- Transitive
    order_date    DATE,
    total_amount  DECIMAL(10,2)
);
```

```sql
-- ✅ AFTER (3NF)
CREATE TABLE customers (
    customer_id INTEGER PRIMARY KEY,
    name        VARCHAR(100) NOT NULL,
    email       VARCHAR(255) UNIQUE,
    city        VARCHAR(100)
);

CREATE TABLE orders (
    order_id    INTEGER PRIMARY KEY,
    customer_id INTEGER NOT NULL REFERENCES customers,
    order_date  DATE NOT NULL,
    total_amount DECIMAL(10,2)
);
```

---

## ตัวอย่างที่ 3: PRODUCTS กับ CATEGORY INFO

```sql
-- ❌ BEFORE (ละเมิด 3NF)
CREATE TABLE products_bad (
    product_id    INTEGER PRIMARY KEY,
    name          VARCHAR(200),
    price         DECIMAL(10,2),
    category_id   INTEGER,
    category_name VARCHAR(100),    -- Transitive: product_id → category_id → category_name
    category_desc TEXT,            -- Transitive
    supplier_id   INTEGER,
    supplier_name VARCHAR(200),    -- Transitive: product_id → supplier_id → supplier_name
    supplier_country VARCHAR(50)   -- Transitive
);
```

```sql
-- ✅ AFTER (3NF)
CREATE TABLE categories (
    category_id   INTEGER PRIMARY KEY,
    name          VARCHAR(100) NOT NULL,
    description   TEXT
);

CREATE TABLE suppliers (
    supplier_id   INTEGER PRIMARY KEY,
    name          VARCHAR(200) NOT NULL,
    country       VARCHAR(50)
);

CREATE TABLE products (
    product_id   INTEGER PRIMARY KEY,
    name         VARCHAR(200) NOT NULL,
    price        DECIMAL(10,2) NOT NULL,
    category_id  INTEGER REFERENCES categories,
    supplier_id  INTEGER REFERENCES suppliers
);
```

---

## ตัวอย่างที่ 4: INVOICE กับ TAX INFO

```sql
-- ❌ BEFORE (ละเมิด 3NF)
CREATE TABLE invoices_bad (
    invoice_id    INTEGER PRIMARY KEY,
    customer_id   INTEGER,
    subtotal      DECIMAL(10,2),
    tax_code      VARCHAR(10),
    tax_rate      DECIMAL(5,4),    -- Transitive: invoice_id → tax_code → tax_rate
    tax_name      VARCHAR(50),     -- Transitive
    tax_amount    DECIMAL(10,2),   -- Derived (subtotal × tax_rate)
    total         DECIMAL(10,2)    -- Derived
);
```

```sql
-- ✅ AFTER (3NF)
CREATE TABLE tax_codes (
    tax_code   VARCHAR(10) PRIMARY KEY,
    name       VARCHAR(50) NOT NULL,
    rate       DECIMAL(5,4) NOT NULL
);

CREATE TABLE invoices (
    invoice_id  INTEGER PRIMARY KEY,
    customer_id INTEGER NOT NULL,
    subtotal    DECIMAL(10,2) NOT NULL,
    tax_code    VARCHAR(10) REFERENCES tax_codes,
    -- tax_amount และ total คำนวณจาก subtotal และ tax_rate
    -- ไม่เก็บ derived values
    invoice_date DATE NOT NULL
);

-- View สำหรับแสดง total
CREATE VIEW invoice_summary AS
SELECT
    i.invoice_id,
    i.customer_id,
    i.subtotal,
    tc.rate AS tax_rate,
    i.subtotal * tc.rate AS tax_amount,
    i.subtotal * (1 + tc.rate) AS total
FROM invoices i
LEFT JOIN tax_codes tc ON i.tax_code = tc.tax_code;
```

---

## ตัวอย่างที่ 5: EMPLOYEES กับ JOB GRADES

```sql
-- ❌ BEFORE (ละเมิด 3NF)
CREATE TABLE employee_salary_bad (
    emp_id         INTEGER PRIMARY KEY,
    name           VARCHAR(100),
    job_grade      VARCHAR(5),
    min_salary     DECIMAL(10,2),   -- Transitive: emp_id → job_grade → min_salary
    max_salary     DECIMAL(10,2),   -- Transitive
    grade_title    VARCHAR(50),     -- Transitive: emp_id → job_grade → grade_title
    actual_salary  DECIMAL(10,2)
);
```

```sql
-- ✅ AFTER (3NF)
CREATE TABLE job_grades (
    grade      VARCHAR(5) PRIMARY KEY,
    title      VARCHAR(50) NOT NULL,
    min_salary DECIMAL(10,2) NOT NULL,
    max_salary DECIMAL(10,2) NOT NULL,
    CONSTRAINT chk_salary CHECK (max_salary >= min_salary)
);

CREATE TABLE employees (
    emp_id         INTEGER PRIMARY KEY,
    name           VARCHAR(100) NOT NULL,
    job_grade      VARCHAR(5) REFERENCES job_grades,
    actual_salary  DECIMAL(10,2) NOT NULL
);
```

---

## ตัวอย่างที่ 6: SHIPPING กับ CARRIER INFO

```sql
-- ❌ BEFORE
CREATE TABLE shipments_bad (
    shipment_id   INTEGER PRIMARY KEY,
    order_id      INTEGER,
    carrier_code  VARCHAR(10),
    carrier_name  VARCHAR(100),     -- Transitive
    carrier_phone VARCHAR(20),      -- Transitive
    tracking_no   VARCHAR(50),
    shipped_at    TIMESTAMP,
    estimated_delivery DATE,
    status        VARCHAR(20)
);
```

```sql
-- ✅ AFTER (3NF)
CREATE TABLE carriers (
    carrier_code  VARCHAR(10) PRIMARY KEY,
    name          VARCHAR(100) NOT NULL,
    phone         VARCHAR(20),
    tracking_url  VARCHAR(300)
);

CREATE TABLE shipments (
    shipment_id  INTEGER PRIMARY KEY,
    order_id     INTEGER NOT NULL,
    carrier_code VARCHAR(10) REFERENCES carriers,
    tracking_no  VARCHAR(50),
    shipped_at   TIMESTAMP,
    estimated_delivery DATE,
    status       VARCHAR(20) DEFAULT 'in_transit'
);
```

---

## ตัวอย่างที่ 7: STUDENTS กับ SCHOLARSHIP

```sql
-- ❌ BEFORE
CREATE TABLE student_scholarships_bad (
    student_id      INTEGER PRIMARY KEY,
    student_name    VARCHAR(100),
    scholarship_id  INTEGER,
    scholarship_name VARCHAR(200),   -- Transitive
    scholarship_amount DECIMAL(10,2), -- Transitive
    sponsor_name    VARCHAR(200),    -- Transitive: scholarship_id → sponsor
    academic_year   VARCHAR(10)
);
```

```sql
-- ✅ AFTER (3NF)
CREATE TABLE scholarships (
    scholarship_id  INTEGER PRIMARY KEY,
    name            VARCHAR(200) NOT NULL,
    amount          DECIMAL(10,2) NOT NULL,
    sponsor_name    VARCHAR(200),
    criteria        TEXT
);

CREATE TABLE students (
    student_id    INTEGER PRIMARY KEY,
    name          VARCHAR(100) NOT NULL,
    scholarship_id INTEGER REFERENCES scholarships,
    academic_year VARCHAR(10)
);
```

---

## ตัวอย่างที่ 8: SALES กับ REGION INFO

```sql
-- ❌ BEFORE
CREATE TABLE sales_bad (
    sale_id       INTEGER PRIMARY KEY,
    salesperson_id INTEGER,
    region_code   VARCHAR(10),
    region_name   VARCHAR(100),   -- Transitive: sale_id → region_code → region_name
    country       VARCHAR(50),    -- Transitive
    timezone      VARCHAR(50),    -- Transitive
    amount        DECIMAL(10,2),
    sale_date     DATE
);
```

```sql
-- ✅ AFTER (3NF)
CREATE TABLE regions (
    region_code VARCHAR(10) PRIMARY KEY,
    name        VARCHAR(100) NOT NULL,
    country     VARCHAR(50),
    timezone    VARCHAR(50)
);

CREATE TABLE salespeople (
    sp_id       INTEGER PRIMARY KEY,
    name        VARCHAR(100) NOT NULL,
    region_code VARCHAR(10) REFERENCES regions
);

CREATE TABLE sales (
    sale_id       INTEGER PRIMARY KEY,
    salesperson_id INTEGER NOT NULL REFERENCES salespeople,
    amount        DECIMAL(10,2) NOT NULL,
    sale_date     DATE NOT NULL
);
```

---

## ตัวอย่างที่ 9: COURSES กับ DEPARTMENT INFO

```sql
-- ❌ BEFORE
CREATE TABLE courses_bad (
    course_id   INTEGER PRIMARY KEY,
    code        VARCHAR(10) UNIQUE,
    title       VARCHAR(200),
    credits     INTEGER,
    dept_id     INTEGER,
    dept_name   VARCHAR(100),      -- Transitive
    dept_head   VARCHAR(100),      -- Transitive
    building    VARCHAR(50),       -- Transitive: dept_id → building
    room_prefix VARCHAR(10)        -- Transitive
);
```

```sql
-- ✅ AFTER (3NF)
CREATE TABLE departments (
    dept_id     INTEGER PRIMARY KEY,
    name        VARCHAR(100) NOT NULL,
    head_name   VARCHAR(100),
    building    VARCHAR(50),
    room_prefix VARCHAR(10)
);

CREATE TABLE courses (
    course_id INTEGER PRIMARY KEY,
    dept_id   INTEGER REFERENCES departments,
    code      VARCHAR(10) UNIQUE NOT NULL,
    title     VARCHAR(200) NOT NULL,
    credits   INTEGER NOT NULL
);
```

---

## ตัวอย่างที่ 10: MEDICAL RECORDS

```sql
-- ❌ BEFORE
CREATE TABLE medical_records_bad (
    record_id     INTEGER PRIMARY KEY,
    patient_id    INTEGER,
    diagnosis_code VARCHAR(10),
    diagnosis_name TEXT,          -- Transitive: record_id → diagnosis_code → diagnosis_name
    diagnosis_category VARCHAR(50), -- Transitive
    treatment_code VARCHAR(10),
    treatment_name TEXT,          -- Transitive: treatment_code → treatment_name
    treatment_cost DECIMAL(10,2), -- Transitive
    record_date   DATE
);
```

```sql
-- ✅ AFTER (3NF)
CREATE TABLE diagnoses (
    diagnosis_code VARCHAR(10) PRIMARY KEY,
    name           TEXT NOT NULL,
    category       VARCHAR(50),
    description    TEXT
);

CREATE TABLE treatments (
    treatment_code VARCHAR(10) PRIMARY KEY,
    name           TEXT NOT NULL,
    standard_cost  DECIMAL(10,2)
);

CREATE TABLE medical_records (
    record_id      INTEGER PRIMARY KEY,
    patient_id     INTEGER NOT NULL,
    diagnosis_code VARCHAR(10) REFERENCES diagnoses,
    treatment_code VARCHAR(10) REFERENCES treatments,
    actual_cost    DECIMAL(10,2),
    record_date    DATE NOT NULL,
    notes          TEXT
);
```

---

## Lossless-Join Decomposition ใน 3NF

การ Decompose ต้องเป็น Lossless (ไม่สูญเสียข้อมูล)

```sql
-- ตัวอย่าง: R(A, B, C) ที่ A → B → C
-- PK: A
-- FDs: A → B, B → C

-- Decomposition:
-- R1(A, B) ← PK: A
-- R2(B, C) ← PK: B

-- ตรวจสอบ Lossless:
-- Join R1 ⋈ R2 ON R1.B = R2.B
-- ได้ผลเหมือน R เดิม ✓

-- เพราะ B เป็น PK ของ R2
-- และเป็น FK ใน R1
-- → Lossless Join guaranteed

-- ตัวอย่าง SQL:
CREATE TABLE r1 (
    a INTEGER PRIMARY KEY,
    b INTEGER NOT NULL,
    FOREIGN KEY (b) REFERENCES r2(b)
);

CREATE TABLE r2 (
    b INTEGER PRIMARY KEY,
    c TEXT
);

-- JOIN เพื่อ reconstruct R:
SELECT r1.a, r1.b, r2.c
FROM r1 JOIN r2 ON r1.b = r2.b;
```

---

## Dependency Preservation

การ Normalize ต้องรักษา FDs ทั้งหมดไว้

```
ตัวอย่าง:
ตาราง R(A, B, C)
FDs: A → B, B → C

Decomposition:
R1(A, B) และ R2(B, C)

FD A → B: อยู่ใน R1 ✓ (preserved)
FD B → C: อยู่ใน R2 ✓ (preserved)
FD A → C: implied โดย A → B และ B → C ✓

→ Dependency Preservation: OK
```

```
ตัวอย่างที่ Dependency ไม่ได้รับการ Preserve:
R(A, B, C)
FDs: A → B, B → C, A → C

Decomposition ที่ผิด:
R1(A, C) และ R2(B, C)

FD A → B: ไม่อยู่ใน R1 หรือ R2! ← ไม่ Preserved!
→ ต้องทดสอบ A → B ด้วยการ JOIN R1 ⋈ R2 (ซับซ้อนกว่า)
```

---

## Armstrong's Axioms (หลักการของ Armstrong)

Armstrong's Axioms ใช้สำหรับ derive FDs ใหม่จาก FDs ที่รู้อยู่แล้ว

### 3 Axioms พื้นฐาน

```
1. Reflexivity (การสะท้อน):
   ถ้า Y ⊆ X แล้ว X → Y
   ตัวอย่าง: {A, B} → A (หรือ B หรือ {A,B})

2. Augmentation (การขยาย):
   ถ้า X → Y แล้ว XZ → YZ
   ตัวอย่าง: emp_id → dept_id
   → (emp_id, name) → (dept_id, name)

3. Transitivity (การถ่ายทอด):
   ถ้า X → Y และ Y → Z แล้ว X → Z
   ตัวอย่าง: emp_id → dept_id, dept_id → dept_name
   → emp_id → dept_name
```

### FDs ที่ Derived ได้

```
4. Union:
   ถ้า X → Y และ X → Z แล้ว X → YZ
   ตัวอย่าง: emp_id → name, emp_id → email
   → emp_id → {name, email}

5. Decomposition:
   ถ้า X → YZ แล้ว X → Y และ X → Z
   ตัวอย่าง: emp_id → {name, email}
   → emp_id → name และ emp_id → email

6. Pseudotransitivity:
   ถ้า X → Y และ WY → Z แล้ว WX → Z
```

### ตัวอย่างการใช้ Armstrong's Axioms

```sql
-- กำหนด FDs:
-- emp_id → dept_id
-- dept_id → {dept_name, dept_location}
-- emp_id → {emp_name, salary}

-- Derive FDs ใหม่:
-- โดย Transitivity:
--   emp_id → dept_id และ dept_id → dept_name
--   → emp_id → dept_name ✓

-- โดย Union:
--   emp_id → dept_id, dept_id → dept_name, dept_id → dept_location
--   → emp_id → {dept_name, dept_location} ✓

-- โดย Augmentation:
--   emp_id → dept_id
--   → (emp_id, emp_name) → (dept_id, emp_name) ✓

-- ใช้ Armstrong's Axioms เพื่อระบุ 3NF Violations:
-- emp_id → dept_name (Transitive) → ละเมิด 3NF!
-- ต้องแยก DEPARTMENTS table
```

---

## Common Patterns ที่ละเมิด 3NF

### Pattern 1: Location Hierarchy

```sql
-- ❌ ปัญหา: zip → city → state → country
CREATE TABLE addresses_bad (
    address_id INTEGER PRIMARY KEY,
    street     VARCHAR(200),
    zip_code   VARCHAR(10),
    city       VARCHAR(100),   -- Transitive
    state      VARCHAR(50),    -- Transitive
    country    VARCHAR(50)     -- Transitive
);

-- ✅ แก้ไข
CREATE TABLE locations (
    zip_code VARCHAR(10) PRIMARY KEY,
    city     VARCHAR(100) NOT NULL,
    state    VARCHAR(50) NOT NULL,
    country  VARCHAR(50) NOT NULL DEFAULT 'TH'
);

CREATE TABLE addresses (
    address_id INTEGER PRIMARY KEY,
    street     VARCHAR(200),
    zip_code   VARCHAR(10) REFERENCES locations
);
```

### Pattern 2: Code → Description Lookup

```sql
-- ❌ ปัญหา: status_code → status_description
CREATE TABLE orders_bad (
    order_id     INTEGER PRIMARY KEY,
    status_code  VARCHAR(20),
    status_desc  TEXT  -- Transitive: status_code → status_desc
);

-- ✅ แก้ไข
CREATE TABLE order_statuses (
    code        VARCHAR(20) PRIMARY KEY,
    description TEXT NOT NULL
);

CREATE TABLE orders (
    order_id   INTEGER PRIMARY KEY,
    status_code VARCHAR(20) REFERENCES order_statuses
);
```

### Pattern 3: Manager Hierarchy

```sql
-- ❌ ปัญหา: emp_id → manager_id → manager_name
CREATE TABLE employees_bad (
    emp_id       INTEGER PRIMARY KEY,
    name         VARCHAR(100),
    manager_id   INTEGER,
    manager_name VARCHAR(100)  -- Transitive!
);

-- ✅ แก้ไข (Self-referencing)
CREATE TABLE employees (
    emp_id     INTEGER PRIMARY KEY,
    name       VARCHAR(100) NOT NULL,
    manager_id INTEGER REFERENCES employees
);

-- manager_name ดูได้จาก JOIN
SELECT e.name, m.name AS manager_name
FROM employees e
LEFT JOIN employees m ON e.manager_id = m.emp_id;
```

### Pattern 4: Category Hierarchy

```sql
-- ❌ ปัญหา: product → subcategory → category
CREATE TABLE products_bad (
    product_id    INTEGER PRIMARY KEY,
    name          VARCHAR(200),
    subcat_id     INTEGER,
    subcat_name   VARCHAR(100),   -- Transitive
    cat_id        INTEGER,        -- Transitive
    cat_name      VARCHAR(100)    -- Transitive
);

-- ✅ แก้ไข
CREATE TABLE categories (
    cat_id    INTEGER PRIMARY KEY,
    name      VARCHAR(100) NOT NULL,
    parent_id INTEGER REFERENCES categories  -- Self-referencing
);

CREATE TABLE products (
    product_id INTEGER PRIMARY KEY,
    name       VARCHAR(200) NOT NULL,
    cat_id     INTEGER REFERENCES categories
);
```

---

## Complete Normalization Walkthrough

### โจทย์: แปลงตาราง SALES_DATA ให้อยู่ใน 3NF

```sql
-- Input: ตารางข้อมูลการขายที่ยังไม่ Normalized
CREATE TABLE sales_unnormalized (
    sale_id        INTEGER,
    sale_date      DATE,
    
    -- Customer Info
    customer_id    INTEGER,
    customer_name  VARCHAR(100),
    customer_email VARCHAR(255),
    customer_city  VARCHAR(100),
    customer_state VARCHAR(50),
    
    -- Salesperson Info
    sp_id          INTEGER,
    sp_name        VARCHAR(100),
    sp_region_code VARCHAR(10),
    sp_region_name VARCHAR(100),
    
    -- Product Info (Repeating Group!)
    product1_id    INTEGER,
    product1_name  VARCHAR(200),
    product1_cat   VARCHAR(100),
    product1_qty   INTEGER,
    product1_price DECIMAL(10,2),
    
    product2_id    INTEGER,
    product2_name  VARCHAR(200),
    product2_cat   VARCHAR(100),
    product2_qty   INTEGER,
    product2_price DECIMAL(10,2)
);
```

#### ขั้นตอนที่ 1: แปลงเป็น 1NF (กำจัด Repeating Groups)

```sql
-- หลัง 1NF: แยก Products ออกมา
-- ตาราง sales_1nf(sale_id, sale_date, customer_id, customer_name,
--                  customer_email, customer_city, customer_state,
--                  sp_id, sp_name, sp_region_code, sp_region_name,
--                  product_id, product_name, product_cat,
--                  quantity, unit_price)
-- PK: (sale_id, product_id)
```

#### ขั้นตอนที่ 2: แปลงเป็น 2NF (กำจัด Partial Dependencies)

```
FD Analysis (PK = sale_id, product_id):
sale_id → sale_date, customer_id, customer_name, ...  ← Partial!
sale_id → sp_id, sp_name, sp_region_code, ...          ← Partial!
product_id → product_name, product_cat                  ← Partial!
(sale_id, product_id) → quantity, unit_price            ← Full ✓
```

```sql
-- หลัง 2NF:
-- SALES(sale_id PK, sale_date, customer_id, sp_id)
-- CUSTOMERS(customer_id PK, name, email, city, state)     ← ยังละเมิด 3NF?
-- SALESPERSONS(sp_id PK, name, region_code, region_name)  ← ยังละเมิด 3NF?
-- PRODUCTS(product_id PK, name, category)
-- SALE_ITEMS(sale_id FK, product_id FK, quantity, unit_price, PK(sale_id,product_id))
```

#### ขั้นตอนที่ 3: แปลงเป็น 3NF (กำจัด Transitive Dependencies)

```
FD Analysis สำหรับ CUSTOMERS:
customer_id → city, state                  ← Direct OK
แต่ city → state (บางครั้ง)?              ← Transitive?
ในทางปฏิบัติ: เมืองหนึ่งอาจอยู่ได้หลาย state
→ city → state ไม่จำเป็นต้องเป็น FD
→ เก็บทั้ง city และ state ใน customers ได้ ถ้าไม่มี FD นี้

FD Analysis สำหรับ SALESPERSONS:
sp_id → region_code, region_name
region_code → region_name                  ← Transitive! sp_id → region_code → region_name
```

```sql
-- ✅ Final 3NF Schema

CREATE TABLE regions (
    region_code  VARCHAR(10) PRIMARY KEY,
    region_name  VARCHAR(100) NOT NULL
);

CREATE TABLE customers (
    customer_id  INTEGER PRIMARY KEY,
    name         VARCHAR(100) NOT NULL,
    email        VARCHAR(255) UNIQUE,
    city         VARCHAR(100),
    state        VARCHAR(50)
);

CREATE TABLE salespersons (
    sp_id        INTEGER PRIMARY KEY,
    name         VARCHAR(100) NOT NULL,
    region_code  VARCHAR(10) REFERENCES regions
);

CREATE TABLE products (
    product_id   INTEGER PRIMARY KEY,
    name         VARCHAR(200) NOT NULL,
    category     VARCHAR(100)
);

CREATE TABLE sales (
    sale_id      INTEGER PRIMARY KEY,
    customer_id  INTEGER NOT NULL REFERENCES customers,
    sp_id        INTEGER REFERENCES salespersons,
    sale_date    DATE NOT NULL
);

CREATE TABLE sale_items (
    sale_id      INTEGER REFERENCES sales,
    product_id   INTEGER REFERENCES products,
    quantity     INTEGER NOT NULL,
    unit_price   DECIMAL(10,2) NOT NULL,
    PRIMARY KEY (sale_id, product_id)
);
```

---

## 3NF vs BCNF (Preview)

3NF บางครั้งยังยอมให้มี Anomalies เล็กน้อย เมื่อ FD มาจาก Non-key → Key

```sql
-- ตัวอย่าง: ตาราง TEACHING
-- TEACHING(student, subject, teacher)
-- FDs:
--   (student, subject) → teacher   ← teacher depends on (student, subject)
--   teacher → subject              ← subject depends on teacher!

-- Candidate Keys: (student, subject) และ (student, teacher)
-- PK: (student, subject)

-- 3NF Check:
-- teacher → subject: teacher ไม่ใช่ Superkey แต่ subject เป็น Prime Attribute
-- → ไม่ละเมิด 3NF!

-- แต่ยังมี Anomaly:
-- ถ้า teacher เปลี่ยน subject ที่สอน ต้องแก้หลาย rows

-- BCNF ต้องการ: ทุก FD: X → Y ต้อง X เป็น Superkey
-- teacher → subject: teacher ไม่ใช่ Superkey → ละเมิด BCNF!

-- (จะเรียนรายละเอียดใน Part 056)
```

---

## สรุป: 1NF vs 2NF vs 3NF

```
Normal Form | กฎที่เพิ่ม              | ปัญหาที่แก้
-----------|------------------------|---------------------------
1NF        | Atomic values,         | Multiple values in column,
           | No repeating groups    | Repeating group columns
           |                        |
2NF        | No Partial Dependencies| Redundancy from
           | (Composite PK only)    | Partial Dependencies on PK
           |                        |
3NF        | No Transitive          | Redundancy from
           | Dependencies           | Non-key → Non-key chains

ความสัมพันธ์: 3NF ⊃ 2NF ⊃ 1NF
(ถ้าอยู่ใน 3NF ก็อยู่ใน 2NF และ 1NF ด้วย)
```

---

## แบบฝึกหัด (10 ข้อ)

### ข้อ 1
ระบุ Transitive Dependencies ในตารางนี้:
```
STUDENTS(student_id PK, name, major_id, major_name, advisor_id, advisor_name, college_name)
FDs: student_id → major_id, major_id → major_name, major_id → college_name,
     student_id → advisor_id, advisor_id → advisor_name
```

**เฉลย**:
```
Transitive Dependencies:
1. student_id → major_id → major_name  (Transitive!)
2. student_id → major_id → college_name (Transitive!)
3. student_id → advisor_id → advisor_name (Transitive!)

Direct Dependencies:
student_id → major_id (Direct ✓)
student_id → advisor_id (Direct ✓)
student_id → name (Direct ✓)

แก้ไข:
MAJORS(major_id PK, name, college_name)
ADVISORS(advisor_id PK, name)
STUDENTS(student_id PK, name, major_id FK, advisor_id FK)
```

### ข้อ 2
แปลงตารางนี้ให้อยู่ใน 3NF:
```sql
CREATE TABLE orders (
    order_id     INTEGER PRIMARY KEY,
    customer_id  INTEGER,
    customer_name VARCHAR(100),
    customer_tier VARCHAR(20),  -- 'gold', 'silver', 'bronze'
    discount_pct  DECIMAL(3,2), -- based on tier: gold=0.20, silver=0.10, bronze=0.05
    total_before  DECIMAL(10,2),
    total_after   DECIMAL(10,2)
);
```

**เฉลย**:
```sql
-- FDs: order_id → customer_id → customer_name, customer_tier
--      customer_tier → discount_pct (Transitive!)
--      total_after derived from total_before and discount_pct

CREATE TABLE customer_tiers (
    tier         VARCHAR(20) PRIMARY KEY,
    discount_pct DECIMAL(3,2) NOT NULL
);

CREATE TABLE customers (
    customer_id   INTEGER PRIMARY KEY,
    name          VARCHAR(100) NOT NULL,
    tier          VARCHAR(20) REFERENCES customer_tiers
);

CREATE TABLE orders (
    order_id     INTEGER PRIMARY KEY,
    customer_id  INTEGER NOT NULL REFERENCES customers,
    total_before DECIMAL(10,2) NOT NULL
    -- discount_pct ดูได้จาก customers JOIN customer_tiers
    -- total_after คำนวณได้ ไม่เก็บ
);

INSERT INTO customer_tiers VALUES
('gold', 0.20), ('silver', 0.10), ('bronze', 0.05);
```

### ข้อ 3
อธิบาย Armstrong's Axioms ทั้ง 3 ข้อพร้อมตัวอย่าง

**เฉลย**:
```
1. Reflexivity: ถ้า Y ⊆ X แล้ว X → Y
   ตัวอย่าง: {employee_id, name} → employee_id
   (สมเหตุสมผล: ถ้ารู้ทั้ง emp_id และ name ก็รู้ emp_id)

2. Augmentation: ถ้า X → Y แล้ว XZ → YZ
   ตัวอย่าง: emp_id → dept_id
   ดังนั้น: (emp_id, hire_date) → (dept_id, hire_date)
   (เพิ่ม hire_date ทั้งสองฝั่งได้)

3. Transitivity: ถ้า X → Y และ Y → Z แล้ว X → Z
   ตัวอย่าง: emp_id → dept_id และ dept_id → dept_name
   ดังนั้น: emp_id → dept_name
   (นี่คือ Transitive Dependency ที่ 3NF ต้องการกำจัด)
```

### ข้อ 4
เขียน SQL Queries เพื่อตรวจหา 3NF Violations

**เฉลย**:
```sql
-- ตรวจสอบว่า dept_id → dept_name เป็น Consistent
-- (ถ้าไม่ Consistent = ข้อมูลขัดแย้ง จาก Transitive Dependency)

SELECT dept_id, COUNT(DISTINCT dept_name) AS distinct_dept_names
FROM employees_bad
GROUP BY dept_id
HAVING COUNT(DISTINCT dept_name) > 1;
-- ถ้ามีผลลัพธ์ = Inconsistency! ควรแยกตาราง departments

-- ตรวจสอบ Redundancy (ข้อมูลซ้ำ)
SELECT
    dept_id,
    dept_name,
    COUNT(*) AS employee_count
FROM employees_bad
GROUP BY dept_id, dept_name
ORDER BY employee_count DESC;
-- dept_name ซ้ำหลายครั้ง = Redundancy จาก Transitive Dependency
```

### ข้อ 5
อะไรคือความแตกต่างระหว่าง Partial Dependency และ Transitive Dependency?

**เฉลย**:
```
Partial Dependency:
- เกิดเมื่อมี Composite PK
- Non-key Attribute depends on PART ของ PK
- ละเมิด 2NF
ตัวอย่าง: (order_id, product_id) เป็น PK
          product_name depends on product_id เพียงอย่างเดียว

Transitive Dependency:
- เกิดได้แม้ PK เป็น Simple Key
- Non-key A → Non-key B → Non-key C (chain)
- ละเมิด 3NF
ตัวอย่าง: emp_id → dept_id → dept_name
          (dept_id เป็น Non-key ที่เชื่อม emp_id ไปหา dept_name)

หลักการ:
2NF: กำจัด Partial (Non-key depends on PART of key)
3NF: กำจัด Transitive (Non-key depends through Non-key)
```

### ข้อ 6
แปลง E-Commerce Schema นี้ให้อยู่ใน 3NF อย่างครบถ้วน:
```
ORDERS(order_id, customer_id, customer_name, city_code, city_name, 
       country_code, country_name, payment_method_id, payment_method_name)
```

**เฉลย**:
```sql
-- FD Analysis:
-- order_id → customer_id, payment_method_id, city_code
-- customer_id → customer_name, city_code (Transitive: order_id → cust_id → city_code)
-- city_code → city_name, country_code (Transitive!)
-- country_code → country_name (Transitive!)
-- payment_method_id → payment_method_name (Transitive!)

CREATE TABLE countries (
    country_code CHAR(2) PRIMARY KEY,
    name         VARCHAR(100) NOT NULL
);

CREATE TABLE cities (
    city_code    VARCHAR(10) PRIMARY KEY,
    name         VARCHAR(100) NOT NULL,
    country_code CHAR(2) NOT NULL REFERENCES countries
);

CREATE TABLE payment_methods (
    method_id    INTEGER PRIMARY KEY,
    name         VARCHAR(50) NOT NULL
);

CREATE TABLE customers (
    customer_id  INTEGER PRIMARY KEY,
    name         VARCHAR(100) NOT NULL,
    city_code    VARCHAR(10) REFERENCES cities
);

CREATE TABLE orders (
    order_id     INTEGER PRIMARY KEY,
    customer_id  INTEGER NOT NULL REFERENCES customers,
    payment_id   INTEGER REFERENCES payment_methods,
    order_date   DATE NOT NULL
);
```

### ข้อ 7
ออกแบบ Schema ระดับ 3NF สำหรับ "ระบบบัญชีเงินเดือน" ที่มี:
พนักงาน, แผนก, ตำแหน่ง (ตำแหน่ง → ระดับเงินเดือน), การจ่ายเงินเดือน

**เฉลย**:
```sql
CREATE TABLE salary_grades (
    grade_code  VARCHAR(5) PRIMARY KEY,
    title       VARCHAR(100) NOT NULL,
    min_salary  DECIMAL(10,2) NOT NULL,
    max_salary  DECIMAL(10,2) NOT NULL
);

CREATE TABLE positions (
    position_id   INTEGER PRIMARY KEY,
    title         VARCHAR(100) NOT NULL,
    grade_code    VARCHAR(5) REFERENCES salary_grades,
    description   TEXT
);

CREATE TABLE departments (
    dept_id       INTEGER PRIMARY KEY,
    name          VARCHAR(100) NOT NULL,
    cost_center   VARCHAR(20),
    manager_id    INTEGER  -- FK to employees (set after)
);

CREATE TABLE employees (
    emp_id        INTEGER PRIMARY KEY,
    dept_id       INTEGER REFERENCES departments,
    position_id   INTEGER REFERENCES positions,
    name          VARCHAR(100) NOT NULL,
    hire_date     DATE NOT NULL,
    current_salary DECIMAL(10,2) NOT NULL
);

CREATE TABLE payroll_runs (
    run_id        SERIAL PRIMARY KEY,
    pay_date      DATE NOT NULL,
    period_start  DATE NOT NULL,
    period_end    DATE NOT NULL,
    status        VARCHAR(20) DEFAULT 'pending'
);

CREATE TABLE payroll_items (
    item_id       SERIAL PRIMARY KEY,
    run_id        INTEGER NOT NULL REFERENCES payroll_runs,
    emp_id        INTEGER NOT NULL REFERENCES employees,
    gross_pay     DECIMAL(10,2) NOT NULL,
    tax_deduction DECIMAL(10,2) NOT NULL,
    net_pay       DECIMAL(10,2) NOT NULL,
    UNIQUE (run_id, emp_id)
);
```

### ข้อ 8
อธิบายความแตกต่างระหว่าง Lossless-Join Decomposition และ Dependency Preservation

**เฉลย**:
```
Lossless-Join Decomposition:
- เมื่อ JOIN ตารางที่แยกออกมากลับมา ได้ผลเหมือนตาราง Original
- ไม่มีข้อมูลสูญหาย ไม่มี Spurious Tuples เพิ่ม
- ทำได้โดยทำให้ Join Attribute เป็น PK ของอย่างน้อยหนึ่งตาราง
- สำคัญมาก: ถ้าไม่ Lossless การ Decompose ไม่ถูกต้อง!

Dependency Preservation:
- FD ทุกตัวจาก Original ยังสามารถตรวจสอบได้ใน Decomposed Tables
- โดยไม่ต้อง JOIN ตาราง (ตรวจสอบใน Single Table)
- บางครั้งยอมแลก Dependency Preservation เพื่อได้ BCNF
- ถ้า FD ไม่ได้รับการ Preserve ต้องใช้ Trigger หรือ View เพื่อ enforce

ตัวอย่าง Lossless แต่ไม่ Dependency Preserved:
R(A, B, C) FDs: A → B, B → A, B → C, A → C
Decompose เป็น R1(A, C) และ R2(B, C)
- Lossless: JOIN R1 ⋈ R2 ON C ได้ R เดิม? ไม่แน่! อาจมี spurious tuples
```

### ข้อ 9
แปลง Schema ของระบบ Delivery Service ให้อยู่ใน 3NF:
```
DELIVERIES(delivery_id, order_id, driver_id, driver_name, driver_license,
           vehicle_id, vehicle_plate, vehicle_type,
           zone_code, zone_name, zone_rate_per_km)
```

**เฉลย**:
```sql
-- FD Analysis:
-- delivery_id → order_id, driver_id, vehicle_id, zone_code
-- driver_id → driver_name, driver_license (Transitive through driver_id)
-- vehicle_id → vehicle_plate, vehicle_type (Transitive through vehicle_id)
-- zone_code → zone_name, zone_rate_per_km (Transitive through zone_code)

CREATE TABLE drivers (
    driver_id     INTEGER PRIMARY KEY,
    name          VARCHAR(100) NOT NULL,
    license_no    VARCHAR(20) UNIQUE NOT NULL,
    is_active     BOOLEAN DEFAULT TRUE
);

CREATE TABLE vehicles (
    vehicle_id    INTEGER PRIMARY KEY,
    plate_number  VARCHAR(15) UNIQUE NOT NULL,
    vehicle_type  VARCHAR(30),  -- 'motorcycle', 'van', 'truck'
    capacity_kg   DECIMAL(6,2)
);

CREATE TABLE delivery_zones (
    zone_code     VARCHAR(10) PRIMARY KEY,
    name          VARCHAR(100) NOT NULL,
    rate_per_km   DECIMAL(6,2) NOT NULL
);

CREATE TABLE deliveries (
    delivery_id   INTEGER PRIMARY KEY,
    order_id      INTEGER NOT NULL,
    driver_id     INTEGER NOT NULL REFERENCES drivers,
    vehicle_id    INTEGER REFERENCES vehicles,
    zone_code     VARCHAR(10) REFERENCES delivery_zones,
    distance_km   DECIMAL(6,2),
    pickup_time   TIMESTAMP,
    delivery_time TIMESTAMP,
    status        VARCHAR(20) DEFAULT 'assigned'
);
```

### ข้อ 10
เขียน Full SQL Script ที่แสดงขั้นตอน Normalize จาก Unnormalized Table → 1NF → 2NF → 3NF

**เฉลย**:
```sql
-- Unnormalized: ระบบจัดการหลักสูตร
-- COURSE_DATA(course_id, course_title, dept_id, dept_name, 
--             instructor_id, instructor_name, instructor_email,
--             student1_id, student1_name, student1_grade,
--             student2_id, student2_name, student2_grade)

-- Step 1: แปลงเป็น 1NF (กำจัด Repeating Groups)
-- ENROLLMENTS_1NF(course_id, course_title, dept_id, dept_name,
--                  instructor_id, instructor_name, instructor_email,
--                  student_id, student_name, grade)
-- PK: (course_id, student_id)

-- Step 2: แปลงเป็น 2NF (กำจัด Partial Dependencies)
-- course_id → course_title, dept_id, dept_name, instructor_id, ...  ← Partial
-- student_id → student_name  ← Partial
-- (course_id, student_id) → grade  ← Full

-- หลัง 2NF:
-- COURSES_2NF(course_id PK, course_title, dept_id, dept_name,
--              instructor_id, instructor_name, instructor_email)
-- STUDENTS(student_id PK, student_name)
-- ENROLLMENTS(course_id FK, student_id FK, grade)

-- Step 3: แปลงเป็น 3NF (กำจัด Transitive Dependencies)
-- ใน COURSES_2NF:
-- course_id → dept_id → dept_name  ← Transitive!
-- course_id → instructor_id → instructor_name, email ← Transitive!

-- Final 3NF:
CREATE TABLE departments (
    dept_id   INTEGER PRIMARY KEY,
    name      VARCHAR(100) NOT NULL
);

CREATE TABLE instructors (
    instructor_id INTEGER PRIMARY KEY,
    name          VARCHAR(100) NOT NULL,
    email         VARCHAR(255) UNIQUE
);

CREATE TABLE courses (
    course_id     INTEGER PRIMARY KEY,
    title         VARCHAR(200) NOT NULL,
    dept_id       INTEGER REFERENCES departments,
    instructor_id INTEGER REFERENCES instructors
);

CREATE TABLE students (
    student_id INTEGER PRIMARY KEY,
    name       VARCHAR(100) NOT NULL
);

CREATE TABLE enrollments (
    course_id  INTEGER REFERENCES courses,
    student_id INTEGER REFERENCES students,
    grade      VARCHAR(3),
    PRIMARY KEY (course_id, student_id)
);

-- ตรวจสอบ: ทุกตารางอยู่ใน 3NF ✓
-- - ไม่มี Repeating Groups (1NF ✓)
-- - ไม่มี Partial Dependencies (2NF ✓)
-- - ไม่มี Transitive Dependencies (3NF ✓)
```

---

*จบ Part 055: Third Normal Form (3NF)*

**ในส่วนต่อไป (Part 056)**: เราจะเรียนรู้เรื่อง BCNF, 4NF, และ 5NF ซึ่งเป็น Normal Forms ระดับสูงกว่าที่ใช้กับกรณีพิเศษ
