# Part 054: Second Normal Form (2NF)
# Normal Form ระดับสอง

---

## บทนำ: ทบทวน 1NF และก้าวสู่ 2NF

ก่อนจะเข้าใจ 2NF ต้องทำความเข้าใจ Functional Dependency ซึ่งเป็นแนวคิดพื้นฐานของ Normalization ทั้งหมด

```
สรุป 1NF (ทบทวน):
✓ ทุก Column มีค่า Atomic
✓ ไม่มี Repeating Groups
✓ ทุก Row มี Primary Key

2NF เพิ่มเติม:
✓ ต้องเป็น 1NF ก่อน
✓ ไม่มี Partial Dependency
   (Non-key Columns ต้อง depend on ENTIRE Primary Key)
```

---

## Functional Dependency (FD) คืออะไร?

**Functional Dependency** คือความสัมพันธ์ที่ค่าของ Column หนึ่งหรือกลุ่ม Column "กำหนด" ค่าของ Column อื่น

```
สัญลักษณ์: A → B
หมายความว่า: "A Functionally Determines B"
             "ถ้ารู้ค่า A จะรู้ค่า B เสมอ"

ตัวอย่าง:
customer_id → customer_name    (รู้ ID รู้ชื่อ)
customer_id → email            (รู้ ID รู้ email)
product_id  → product_name     (รู้ ID รู้ชื่อสินค้า)
product_id  → unit_price       (รู้ ID รู้ราคา)
zip_code    → city             (รู้ zip รู้เมือง)

สิ่งที่ไม่ใช่ Functional Dependency:
customer_id → order_id   (ลูกค้าหนึ่งคนมีหลาย order!)
```

### ตัวอย่าง FD Diagram

```
ตาราง ORDERS:
┌─────────────────────────────────────────────────────────┐
│  order_id, product_id → quantity, unit_price           │
│  order_id → customer_id, order_date                    │
│  product_id → product_name, category                   │
│  customer_id → customer_name, email                    │
└─────────────────────────────────────────────────────────┘

FD Diagram:
                    order_id ──────────────────→ customer_id
                                                  order_date
                    
                    product_id ────────────────→ product_name
                                                  category
                    
(order_id, product_id) ────────────────────────→ quantity
                                                  unit_price
```

### ประเภทของ Functional Dependencies

```
1. Full Functional Dependency
   {A, B} → C
   C depends on BOTH A and B (ต้องใช้ทั้ง A และ B)

2. Partial Functional Dependency
   {A, B} → C แต่ A → C เพียงพอแล้ว
   (C depends on PART of the key)
   ← นี่คือสิ่งที่ 2NF ต้องการกำจัด

3. Transitive Functional Dependency
   A → B → C
   (C depends on A ผ่าน B)
   ← สิ่งที่ 3NF ต้องการกำจัด
```

---

## Partial Dependency คืออะไร?

**Partial Dependency** เกิดขึ้นเมื่อ:
- ตารางมี **Composite Primary Key**
- Non-key Column **depend on PART** of the Composite Key (ไม่ใช่ทั้งหมด)

```
ตัวอย่าง Partial Dependency:

ตาราง ORDER_DETAILS(order_id, product_id, quantity, product_name, product_price)
PK: (order_id, product_id)  ← Composite Key

FD Analysis:
(order_id, product_id) → quantity       ✓ Full Dependency
product_id             → product_name   ✗ Partial! (ไม่ต้องใช้ order_id)
product_id             → product_price  ✗ Partial! (ไม่ต้องใช้ order_id)

ปัญหา:
- ถ้าเปลี่ยนราคาสินค้า ต้องแก้หลาย rows
- ลบ order แล้ว product info หายไปด้วย (Delete Anomaly)
```

---

## 2NF คืออะไร?

ตารางอยู่ใน **2NF** เมื่อ:
1. อยู่ใน 1NF
2. Non-key Attributes ทุกตัว **Fully Functionally Dependent** บน Primary Key ทั้งหมด (ไม่ใช่แค่ส่วนหนึ่ง)

**หมายเหตุ**: 2NF สำคัญเฉพาะเมื่อ PK เป็น Composite Key

---

## Anomalies ที่เกิดจาก 2NF Violations

### Insert Anomaly

```sql
-- ตาราง ORDER_DETAILS ที่ละเมิด 2NF
CREATE TABLE order_details_bad (
    order_id      INTEGER,
    product_id    INTEGER,
    quantity      INTEGER,
    unit_price    DECIMAL,
    product_name  VARCHAR(200),  -- ← Partial Dependency
    category      VARCHAR(100),  -- ← Partial Dependency
    PRIMARY KEY (order_id, product_id)
);

-- ❌ Insert Anomaly:
-- ต้องการเพิ่มสินค้าใหม่แต่ยังไม่มี order
-- ทำไม่ได้เพราะต้องการ order_id (ซึ่งเป็น part ของ PK)!
INSERT INTO order_details_bad (product_id, product_name, category)
VALUES (999, 'New Product', 'Electronics');
-- Error: order_id cannot be NULL (it's part of PK)
```

### Update Anomaly

```sql
-- ❌ Update Anomaly:
-- เปลี่ยนชื่อสินค้า product_id=5 ต้องแก้ทุก Row ที่มีสินค้านี้!
UPDATE order_details_bad
SET product_name = 'New Product Name'
WHERE product_id = 5;
-- ต้องแก้กี่ Row? เท่ากับจำนวน orders ที่มีสินค้านี้!
-- ถ้าแก้ไม่ครบ → ข้อมูลขัดแย้ง (Inconsistency)
```

### Delete Anomaly

```sql
-- ❌ Delete Anomaly:
-- ลบ order ที่มีสินค้าบางตัวเพียงใน order นั้น
-- ข้อมูลสินค้าหายไปด้วย!
DELETE FROM order_details_bad
WHERE order_id = 1001;
-- ถ้า order_id=1001 เป็น order เดียวที่มี product_id=999
-- ข้อมูล product_id=999 จะหายไปจากระบบ!
```

---

## ตัวอย่างที่ 1: ORDER_DETAILS

```
FD Diagram สำหรับ ORDER_DETAILS_BAD:

order_id ─────────────────────────────→ order_date
order_id ─────────────────────────────→ customer_id
                                          
product_id ────────────────────────────→ product_name  ← Partial!
product_id ────────────────────────────→ unit_price    ← Partial!
product_id ────────────────────────────→ category      ← Partial!
                                          
(order_id, product_id) ────────────────→ quantity      ← Full!
```

```sql
-- ❌ BEFORE (ละเมิด 2NF)
CREATE TABLE order_details_bad (
    order_id      INTEGER,
    product_id    INTEGER,
    quantity      INTEGER,
    unit_price    DECIMAL(10,2),
    order_date    DATE,             -- ← partial (depends on order_id only)
    customer_name VARCHAR(100),     -- ← partial (depends on order_id only)
    product_name  VARCHAR(200),     -- ← partial (depends on product_id only)
    category      VARCHAR(100),     -- ← partial (depends on product_id only)
    PRIMARY KEY (order_id, product_id)
);
```

```sql
-- ✅ AFTER (2NF) - แยกออกเป็น 3 ตาราง

-- ตาราง 1: Orders (depends on order_id)
CREATE TABLE orders (
    order_id      INTEGER PRIMARY KEY,
    customer_id   INTEGER NOT NULL REFERENCES customers,
    order_date    DATE NOT NULL
);

-- ตาราง 2: Products (depends on product_id)
CREATE TABLE products (
    product_id    INTEGER PRIMARY KEY,
    product_name  VARCHAR(200) NOT NULL,
    category      VARCHAR(100),
    unit_price    DECIMAL(10,2) NOT NULL
);

-- ตาราง 3: Order Items (depends on FULL PK)
CREATE TABLE order_items (
    order_id    INTEGER REFERENCES orders,
    product_id  INTEGER REFERENCES products,
    quantity    INTEGER NOT NULL,
    unit_price  DECIMAL(10,2) NOT NULL,  -- snapshot of price at order time
    PRIMARY KEY (order_id, product_id)
);
```

---

## ตัวอย่างที่ 2: STUDENT_COURSE_INSTRUCTOR

```sql
-- ❌ BEFORE
-- PK: (student_id, course_id)
CREATE TABLE student_courses_bad (
    student_id    INTEGER,
    course_id     INTEGER,
    grade         VARCHAR(3),
    instructor_id INTEGER,
    instructor_name VARCHAR(100),  -- ← Partial (depends on instructor_id)
    instructor_email VARCHAR(255), -- ← Partial (depends on instructor_id)
    course_name   VARCHAR(200),    -- ← Partial (depends on course_id)
    credits       INTEGER,         -- ← Partial (depends on course_id)
    student_name  VARCHAR(100),    -- ← Partial (depends on student_id)
    PRIMARY KEY (student_id, course_id)
);
```

```
FD Diagram:
student_id ────────────────────────────────────→ student_name
course_id ─────────────────────────────────────→ course_name, credits
instructor_id ─────────────────────────────────→ instructor_name, instructor_email
(student_id, course_id) ───────────────────────→ grade, instructor_id
```

```sql
-- ✅ AFTER (2NF)
CREATE TABLE students (
    student_id    INTEGER PRIMARY KEY,
    student_name  VARCHAR(100) NOT NULL,
    email         VARCHAR(255)
);

CREATE TABLE courses (
    course_id     INTEGER PRIMARY KEY,
    course_name   VARCHAR(200) NOT NULL,
    credits       INTEGER NOT NULL
);

CREATE TABLE instructors (
    instructor_id   INTEGER PRIMARY KEY,
    instructor_name VARCHAR(100) NOT NULL,
    email           VARCHAR(255) UNIQUE
);

CREATE TABLE enrollments (
    student_id    INTEGER REFERENCES students,
    course_id     INTEGER REFERENCES courses,
    instructor_id INTEGER REFERENCES instructors,
    grade         VARCHAR(3),
    PRIMARY KEY (student_id, course_id)
);
```

---

## ตัวอย่างที่ 3: EMPLOYEE_PROJECT

```sql
-- ❌ BEFORE
-- ระบบ HR - พนักงานทำงานในหลาย Project
-- PK: (emp_id, project_id)
CREATE TABLE emp_project_bad (
    emp_id          INTEGER,
    project_id      INTEGER,
    hours_worked    DECIMAL(5,1),
    emp_name        VARCHAR(100),    -- ← Partial (emp_id only)
    emp_dept        VARCHAR(50),     -- ← Partial (emp_id only)
    emp_salary      DECIMAL(10,2),   -- ← Partial (emp_id only)
    project_name    VARCHAR(200),    -- ← Partial (project_id only)
    project_budget  DECIMAL(15,2),   -- ← Partial (project_id only)
    project_manager VARCHAR(100),    -- ← Partial (project_id only)
    PRIMARY KEY (emp_id, project_id)
);
```

```sql
-- ✅ AFTER (2NF)
CREATE TABLE employees (
    emp_id    INTEGER PRIMARY KEY,
    name      VARCHAR(100) NOT NULL,
    dept      VARCHAR(50),
    salary    DECIMAL(10,2)
);

CREATE TABLE projects (
    project_id   INTEGER PRIMARY KEY,
    name         VARCHAR(200) NOT NULL,
    budget       DECIMAL(15,2),
    manager_name VARCHAR(100)
);

CREATE TABLE project_assignments (
    emp_id       INTEGER REFERENCES employees,
    project_id   INTEGER REFERENCES projects,
    hours_worked DECIMAL(5,1) NOT NULL DEFAULT 0,
    start_date   DATE,
    end_date     DATE,
    PRIMARY KEY (emp_id, project_id)
);
```

---

## ตัวอย่างที่ 4: WAREHOUSE_INVENTORY

```sql
-- ❌ BEFORE
-- PK: (product_id, warehouse_id)
CREATE TABLE inventory_bad (
    product_id      INTEGER,
    warehouse_id    INTEGER,
    quantity        INTEGER,
    product_name    VARCHAR(200),    -- ← Partial (product_id)
    unit_price      DECIMAL(10,2),   -- ← Partial (product_id)
    supplier_name   VARCHAR(100),    -- ← Partial (product_id)
    warehouse_name  VARCHAR(100),    -- ← Partial (warehouse_id)
    warehouse_city  VARCHAR(100),    -- ← Partial (warehouse_id)
    warehouse_capacity INTEGER,      -- ← Partial (warehouse_id)
    PRIMARY KEY (product_id, warehouse_id)
);
```

```sql
-- ✅ AFTER (2NF)
CREATE TABLE products (
    product_id    INTEGER PRIMARY KEY,
    name          VARCHAR(200) NOT NULL,
    unit_price    DECIMAL(10,2),
    supplier_name VARCHAR(100)
);

CREATE TABLE warehouses (
    warehouse_id  INTEGER PRIMARY KEY,
    name          VARCHAR(100) NOT NULL,
    city          VARCHAR(100),
    capacity      INTEGER
);

CREATE TABLE inventory (
    product_id   INTEGER REFERENCES products,
    warehouse_id INTEGER REFERENCES warehouses,
    quantity     INTEGER NOT NULL DEFAULT 0,
    min_stock    INTEGER DEFAULT 0,
    last_updated TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (product_id, warehouse_id)
);
```

---

## ตัวอย่างที่ 5: FLIGHT_BOOKING

```sql
-- ❌ BEFORE (ระบบสายการบิน)
-- PK: (flight_id, passenger_id)
CREATE TABLE flight_booking_bad (
    flight_id         INTEGER,
    passenger_id      INTEGER,
    seat_number       VARCHAR(5),
    ticket_price      DECIMAL(10,2),
    booking_status    VARCHAR(20),
    passenger_name    VARCHAR(100),   -- ← Partial
    passport_number   VARCHAR(20),    -- ← Partial
    flight_number     VARCHAR(10),    -- ← Partial
    origin_airport    VARCHAR(3),     -- ← Partial
    dest_airport      VARCHAR(3),     -- ← Partial
    departure_time    TIMESTAMP,      -- ← Partial
    arrival_time      TIMESTAMP,      -- ← Partial
    airline_name      VARCHAR(100),   -- ← Partial
    PRIMARY KEY (flight_id, passenger_id)
);
```

```sql
-- ✅ AFTER (2NF)
CREATE TABLE passengers (
    passenger_id    INTEGER PRIMARY KEY,
    name            VARCHAR(100) NOT NULL,
    passport_number VARCHAR(20) UNIQUE
);

CREATE TABLE flights (
    flight_id       INTEGER PRIMARY KEY,
    flight_number   VARCHAR(10) NOT NULL,
    origin_airport  VARCHAR(3) NOT NULL,
    dest_airport    VARCHAR(3) NOT NULL,
    departure_time  TIMESTAMP NOT NULL,
    arrival_time    TIMESTAMP,
    airline_name    VARCHAR(100)
);

CREATE TABLE bookings (
    flight_id       INTEGER REFERENCES flights,
    passenger_id    INTEGER REFERENCES passengers,
    seat_number     VARCHAR(5),
    ticket_price    DECIMAL(10,2) NOT NULL,
    booking_status  VARCHAR(20) DEFAULT 'confirmed',
    booked_at       TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (flight_id, passenger_id)
);
```

---

## ตัวอย่างที่ 6: EXAM_SCORES

```sql
-- ❌ BEFORE
-- PK: (student_id, exam_id)
CREATE TABLE exam_scores_bad (
    student_id    INTEGER,
    exam_id       INTEGER,
    score         DECIMAL(5,2),
    student_name  VARCHAR(100),   -- ← Partial (student_id)
    student_class VARCHAR(20),    -- ← Partial (student_id)
    exam_name     VARCHAR(200),   -- ← Partial (exam_id)
    max_score     DECIMAL(5,2),   -- ← Partial (exam_id)
    exam_date     DATE,           -- ← Partial (exam_id)
    subject       VARCHAR(100),   -- ← Partial (exam_id)
    PRIMARY KEY (student_id, exam_id)
);
```

```sql
-- ✅ AFTER (2NF)
CREATE TABLE students (
    student_id  INTEGER PRIMARY KEY,
    name        VARCHAR(100) NOT NULL,
    class       VARCHAR(20)
);

CREATE TABLE exams (
    exam_id     INTEGER PRIMARY KEY,
    name        VARCHAR(200) NOT NULL,
    subject     VARCHAR(100),
    max_score   DECIMAL(5,2) NOT NULL,
    exam_date   DATE
);

CREATE TABLE exam_scores (
    student_id  INTEGER REFERENCES students,
    exam_id     INTEGER REFERENCES exams,
    score       DECIMAL(5,2) NOT NULL,
    PRIMARY KEY (student_id, exam_id)
);
```

---

## ตัวอย่างที่ 7: DOCTOR_PATIENT_VISIT

```sql
-- ❌ BEFORE
-- PK: (doctor_id, patient_id, visit_date)
CREATE TABLE doctor_visits_bad (
    doctor_id     INTEGER,
    patient_id    INTEGER,
    visit_date    DATE,
    diagnosis     TEXT,
    prescription  TEXT,
    doctor_name   VARCHAR(100),   -- ← Partial (doctor_id)
    doctor_spec   VARCHAR(100),   -- ← Partial (doctor_id)
    patient_name  VARCHAR(100),   -- ← Partial (patient_id)
    patient_dob   DATE,           -- ← Partial (patient_id)
    patient_blood VARCHAR(3),     -- ← Partial (patient_id)
    PRIMARY KEY (doctor_id, patient_id, visit_date)
);
```

```sql
-- ✅ AFTER (2NF)
CREATE TABLE doctors (
    doctor_id   INTEGER PRIMARY KEY,
    name        VARCHAR(100) NOT NULL,
    specialty   VARCHAR(100)
);

CREATE TABLE patients (
    patient_id  INTEGER PRIMARY KEY,
    name        VARCHAR(100) NOT NULL,
    dob         DATE,
    blood_type  VARCHAR(3)
);

CREATE TABLE visits (
    visit_id    SERIAL PRIMARY KEY,
    doctor_id   INTEGER REFERENCES doctors,
    patient_id  INTEGER REFERENCES patients,
    visit_date  DATE NOT NULL,
    diagnosis   TEXT,
    prescription TEXT,
    UNIQUE (doctor_id, patient_id, visit_date)
);
```

---

## ตัวอย่างที่ 8: PRODUCT_SUPPLIER_WAREHOUSE

```sql
-- ❌ BEFORE
-- PK: (product_id, supplier_id, warehouse_id)
CREATE TABLE supply_inventory_bad (
    product_id       INTEGER,
    supplier_id      INTEGER,
    warehouse_id     INTEGER,
    quantity_on_hand INTEGER,
    reorder_point    INTEGER,
    product_name     VARCHAR(200),   -- ← Partial (product_id)
    product_category VARCHAR(100),   -- ← Partial (product_id)
    supplier_name    VARCHAR(200),   -- ← Partial (supplier_id)
    supplier_country VARCHAR(50),    -- ← Partial (supplier_id)
    warehouse_name   VARCHAR(100),   -- ← Partial (warehouse_id)
    warehouse_city   VARCHAR(100),   -- ← Partial (warehouse_id)
    PRIMARY KEY (product_id, supplier_id, warehouse_id)
);
```

```sql
-- ✅ AFTER (2NF)
CREATE TABLE products (
    product_id  INTEGER PRIMARY KEY,
    name        VARCHAR(200) NOT NULL,
    category    VARCHAR(100)
);

CREATE TABLE suppliers (
    supplier_id INTEGER PRIMARY KEY,
    name        VARCHAR(200) NOT NULL,
    country     VARCHAR(50)
);

CREATE TABLE warehouses (
    warehouse_id INTEGER PRIMARY KEY,
    name         VARCHAR(100) NOT NULL,
    city         VARCHAR(100)
);

-- ความสัมพันธ์ Product-Supplier
CREATE TABLE product_suppliers (
    product_id  INTEGER REFERENCES products,
    supplier_id INTEGER REFERENCES suppliers,
    unit_cost   DECIMAL(10,2),
    lead_time_days INTEGER,
    PRIMARY KEY (product_id, supplier_id)
);

-- Inventory (ternary: product + warehouse)
CREATE TABLE inventory (
    product_id    INTEGER REFERENCES products,
    warehouse_id  INTEGER REFERENCES warehouses,
    quantity      INTEGER NOT NULL DEFAULT 0,
    reorder_point INTEGER DEFAULT 0,
    PRIMARY KEY (product_id, warehouse_id)
);
```

---

## ตัวอย่างที่ 9: COURSE_SCHEDULE

```sql
-- ❌ BEFORE (ตาราง Schedule ของมหาวิทยาลัย)
-- PK: (course_id, room_id, time_slot)
CREATE TABLE schedule_bad (
    course_id     INTEGER,
    room_id       INTEGER,
    time_slot     VARCHAR(30),  -- "Mon 09:00-10:30"
    enrolled_count INTEGER,
    course_name   VARCHAR(200),  -- ← Partial (course_id)
    instructor_id INTEGER,       -- ← Partial (course_id)
    credits       INTEGER,       -- ← Partial (course_id)
    room_name     VARCHAR(50),   -- ← Partial (room_id)
    building      VARCHAR(50),   -- ← Partial (room_id)
    capacity      INTEGER,       -- ← Partial (room_id)
    PRIMARY KEY (course_id, room_id, time_slot)
);
```

```sql
-- ✅ AFTER (2NF)
CREATE TABLE courses (
    course_id     INTEGER PRIMARY KEY,
    name          VARCHAR(200) NOT NULL,
    instructor_id INTEGER REFERENCES instructors,
    credits       INTEGER NOT NULL
);

CREATE TABLE rooms (
    room_id   INTEGER PRIMARY KEY,
    name      VARCHAR(50) NOT NULL,
    building  VARCHAR(50),
    capacity  INTEGER
);

CREATE TABLE schedule (
    course_id       INTEGER REFERENCES courses,
    room_id         INTEGER REFERENCES rooms,
    time_slot       VARCHAR(30) NOT NULL,
    enrolled_count  INTEGER DEFAULT 0,
    PRIMARY KEY (course_id, room_id, time_slot)
);
```

---

## ตัวอย่างที่ 10: TRANSACTION_LOG

```sql
-- ❌ BEFORE (ระบบธนาคาร)
-- PK: (transaction_id, account_id)
CREATE TABLE txn_log_bad (
    transaction_id  BIGINT,
    account_id      INTEGER,
    amount          DECIMAL(15,2),
    balance_after   DECIMAL(15,2),
    account_no      VARCHAR(20),   -- ← Partial (account_id)
    account_type    VARCHAR(20),   -- ← Partial (account_id)
    customer_name   VARCHAR(100),  -- ← Partial (account_id)
    customer_phone  VARCHAR(20),   -- ← Partial (account_id)
    PRIMARY KEY (transaction_id, account_id)
);
-- หมายเหตุ: PK นี้ซ้ำซ้อน transaction_id น่าจะเป็น PK เพียงพอ
```

```sql
-- ✅ AFTER (2NF) + ปรับ PK
CREATE TABLE customers (
    customer_id  INTEGER PRIMARY KEY,
    name         VARCHAR(100) NOT NULL,
    phone        VARCHAR(20)
);

CREATE TABLE accounts (
    account_id   INTEGER PRIMARY KEY,
    customer_id  INTEGER NOT NULL REFERENCES customers,
    account_no   VARCHAR(20) UNIQUE NOT NULL,
    account_type VARCHAR(20),
    balance      DECIMAL(15,2) DEFAULT 0
);

CREATE TABLE transactions (
    transaction_id BIGSERIAL PRIMARY KEY,  -- Simple PK, no composite
    account_id     INTEGER NOT NULL REFERENCES accounts,
    amount         DECIMAL(15,2) NOT NULL,
    balance_after  DECIMAL(15,2) NOT NULL,
    txn_type       VARCHAR(20),
    created_at     TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

---

## กระบวนการแปลงสู่ 2NF

### ขั้นตอนที่ 1: ตรวจสอบ 1NF ก่อน
### ขั้นตอนที่ 2: ระบุ Composite Primary Key
### ขั้นตอนที่ 3: วิเคราะห์ Functional Dependencies
### ขั้นตอนที่ 4: หา Partial Dependencies
### ขั้นตอนที่ 5: แยกตาราง

```sql
-- Algorithm สำหรับแปลงเป็น 2NF

-- Input: T(K1, K2, A, B, C) where PK = (K1, K2)
-- FDs: K1 → A, K2 → B, (K1,K2) → C

-- Step 1: สร้างตารางสำหรับ Full Dependencies
-- T3(K1, K2, C)  ← C depends on full PK

-- Step 2: สร้างตารางสำหรับ Partial Dependencies
-- T1(K1, A)      ← A depends only on K1
-- T2(K2, B)      ← B depends only on K2

-- ผลลัพธ์: 3 ตาราง แทน 1 ตาราง
```

---

## การแก้ Lossless Join Decomposition

การ Decompose ต้องทำให้ "ไม่สูญเสียข้อมูล" (Lossless Join)

```sql
-- ตัวอย่าง: ตาราง R(A, B, C, D) ที่ละเมิด 2NF
-- PK: (A, B)
-- FDs: A → C, (A,B) → D, B → E

-- ❌ Wrong Decomposition (Lossy):
-- R1(A, C, D), R2(B, E)
-- JOIN R1 ⋈ R2 ไม่ได้ผลเหมือนเดิม!

-- ✅ Correct Decomposition (Lossless):
-- R1(A, B, D)  ← PK: (A,B), FD: (A,B) → D
-- R2(A, C)     ← PK: A, FD: A → C
-- R3(B, E)     ← PK: B, FD: B → E

-- ตรวจสอบ: JOIN R1 ⋈ R2 ON R1.A = R2.A ⋈ R3 ON R1.B = R3.B
-- ได้ผลเหมือน R เดิม ✓

-- กฎ: Decomposition เป็น Lossless ถ้า
-- Join Attribute เป็น PK ของอย่างน้อยหนึ่งตาราง
```

---

## FD Diagrams สำหรับ 2NF

```
ตัวอย่างที่ 1: ORDER_ITEMS

ก่อน 2NF:
┌──────────────────────────────────────────────────────────┐
│  order_id ──────────────────────────────→ order_date     │
│  order_id ──────────────────────────────→ customer_id    │
│  product_id ────────────────────────────→ product_name   │ ← Partial
│  product_id ────────────────────────────→ unit_price     │ ← Partial
│  (order_id, product_id) ────────────────→ quantity       │ ← Full
└──────────────────────────────────────────────────────────┘

หลัง 2NF:
ORDERS: order_id → order_date, customer_id
PRODUCTS: product_id → product_name, unit_price
ORDER_ITEMS: (order_id, product_id) → quantity


ตัวอย่างที่ 2: STUDENT_ENROLLMENT

ก่อน 2NF:
┌──────────────────────────────────────────────────────────┐
│  student_id ───────────────────────────→ student_name    │ ← Partial
│  student_id ───────────────────────────→ major           │ ← Partial
│  course_id ────────────────────────────→ course_name     │ ← Partial
│  course_id ────────────────────────────→ instructor_id   │ ← Partial
│  (student_id, course_id) ──────────────→ grade           │ ← Full
│  (student_id, course_id) ──────────────→ semester        │ ← Full
└──────────────────────────────────────────────────────────┘

หลัง 2NF:
STUDENTS: student_id → student_name, major
COURSES: course_id → course_name, instructor_id
ENROLLMENTS: (student_id, course_id) → grade, semester
```

---

## เมื่อ 2NF ไม่เพียงพอ (Teaser สำหรับ 3NF)

แม้แปลงเป็น 2NF แล้ว ยังอาจมี Anomalies จาก Transitive Dependencies

```sql
-- ตาราง EMPLOYEES ที่อยู่ใน 2NF แต่ยังมีปัญหา
-- PK: emp_id (Simple Key → ไม่มี Partial Dependency)
CREATE TABLE employees_2nf (
    emp_id      INTEGER PRIMARY KEY,
    name        VARCHAR(100),
    dept_id     INTEGER,
    dept_name   VARCHAR(100),   -- ← Transitive! dept_id → dept_name
    dept_manager VARCHAR(100)   -- ← Transitive! dept_id → dept_manager
);

-- FD: emp_id → dept_id → dept_name
-- Transitive Dependency: emp_id → dept_name (ผ่าน dept_id)

-- ปัญหา:
-- 1. เปลี่ยนชื่อแผนก ต้องแก้หลาย rows
-- 2. ลบพนักงานคนสุดท้ายของแผนก → ข้อมูลแผนกหาย

-- แก้ใน 3NF: แยกตาราง DEPARTMENTS
-- (จะเรียนใน Part 055)
```

---

## สรุป 2NF

```
2NF Rules:
1. ต้องเป็น 1NF ก่อน
2. ทุก Non-key Attribute ต้อง Fully Depend บน Entire Primary Key

2NF Violations เกิดเมื่อ:
- PK เป็น Composite Key
- Non-key Columns depend บน PART ของ PK

วิธีแก้:
- แยก Partial Dependencies ออกเป็นตารางใหม่
- ตารางใหม่ใช้ Partial Key เป็น PK

ผลลัพธ์:
- ไม่มี Insert/Update/Delete Anomalies จาก Partial Dependencies
- ข้อมูลซ้ำซ้อนลดลง

หมายเหตุ:
- ถ้า PK เป็น Simple (Single Column) → ไม่สามารถมี Partial Dependency
- 2NF สำคัญเฉพาะกับ Composite PK เท่านั้น
```

---

## แบบฝึกหัด (10 ข้อ)

### ข้อ 1
ระบุ Partial Dependencies ในตารางนี้:
```
RENTAL(rental_id, video_id, customer_id, rental_date, return_date,
       video_title, video_genre, customer_name, customer_phone)
PK: (rental_id, video_id)
```

**เฉลย**:
```
Full Dependencies:
(rental_id, video_id) → rental_date, return_date

Partial Dependencies:
video_id → video_title, video_genre    ← Partial!
rental_id → customer_id, rental_date  ← เกิดคำถาม: rental_id อาจเป็น PK เพียงพอ
customer_id → customer_name, customer_phone ← Transitive ผ่าน customer_id

แก้ไข:
RENTALS(rental_id PK, customer_id FK, rental_date, return_date)
VIDEOS(video_id PK, title, genre)
RENTAL_VIDEOS(rental_id FK, video_id FK, PK(rental_id, video_id))
CUSTOMERS(customer_id PK, name, phone)
```

### ข้อ 2
อธิบาย Functional Dependency คืออะไร และให้ตัวอย่าง 5 FD จากระบบ E-Commerce

**เฉลย**:
```
Functional Dependency (FD): A → B หมายความว่า
"ถ้ารู้ค่า A จะสามารถกำหนดค่า B ได้เสมอ"

5 FD จาก E-Commerce:
1. customer_id → customer_name, email, phone
   (รู้ customer_id รู้ข้อมูลลูกค้าทั้งหมด)

2. product_id → product_name, price, category_id
   (รู้ product_id รู้ข้อมูลสินค้า)

3. order_id → customer_id, order_date, total_amount
   (รู้ order_id รู้ข้อมูล order)

4. (order_id, product_id) → quantity, unit_price
   (ต้องรู้ทั้ง order และสินค้า ถึงรู้ quantity)

5. zip_code → city, province
   (รู้รหัสไปรษณีย์ รู้เมืองและจังหวัด)
```

### ข้อ 3
แปลงตารางนี้ให้อยู่ใน 2NF:
```
LIBRARY_BOOK(member_id, book_id, checkout_date, due_date,
             member_name, member_email, book_title, book_author, isbn)
PK: (member_id, book_id)
```

**เฉลย**:
```sql
CREATE TABLE members (
    member_id   INTEGER PRIMARY KEY,
    name        VARCHAR(100) NOT NULL,
    email       VARCHAR(255)
);

CREATE TABLE books (
    book_id     INTEGER PRIMARY KEY,
    title       VARCHAR(300) NOT NULL,
    author      VARCHAR(200),
    isbn        VARCHAR(13) UNIQUE
);

CREATE TABLE checkouts (
    member_id     INTEGER REFERENCES members,
    book_id       INTEGER REFERENCES books,
    checkout_date DATE NOT NULL,
    due_date      DATE NOT NULL,
    return_date   DATE,
    PRIMARY KEY (member_id, book_id, checkout_date)
    -- เพิ่ม checkout_date ใน PK เพราะยืมได้หลายครั้ง
);
```

### ข้อ 4
อธิบาย Anomalies 3 ประเภทที่เกิดจาก 2NF Violations

**เฉลย**:
```
1. Insert Anomaly:
   ไม่สามารถ Insert ข้อมูลบางส่วนได้
   ตัวอย่าง: ไม่สามารถเพิ่มสินค้าใหม่ใน ORDER_DETAILS
   โดยไม่มี order_id เพราะ order_id เป็นส่วนหนึ่งของ PK

2. Update Anomaly:
   ต้องแก้ไขหลาย rows เมื่อข้อมูลเปลี่ยน
   ตัวอย่าง: เปลี่ยนชื่อสินค้าต้องแก้ทุก row ใน ORDER_DETAILS

3. Delete Anomaly:
   การลบข้อมูลทำให้ข้อมูลอื่นหายไปด้วย
   ตัวอย่าง: ลบ order สุดท้ายของสินค้า
   ทำให้ข้อมูลสินค้าหายไปจากระบบ
```

### ข้อ 5
ตาราง GRADE_REPORT มี PK = (student_id, course_id) ระบุว่า FD ใดเป็น Partial Dependency:
```
FDs:
a) student_id → student_name
b) (student_id, course_id) → grade
c) course_id → course_name, instructor
d) student_id → major, gpa
e) (student_id, course_id) → semester_taken
```

**เฉลย**:
```
Full Dependencies (Full PK):
b) (student_id, course_id) → grade ✓
e) (student_id, course_id) → semester_taken ✓

Partial Dependencies (ส่วนหนึ่งของ PK เพียงพอ):
a) student_id → student_name ✗ PARTIAL (depends on student_id only)
c) course_id → course_name, instructor ✗ PARTIAL (depends on course_id only)
d) student_id → major, gpa ✗ PARTIAL (depends on student_id only)

แก้ไข: แยกเป็น 3 ตาราง
STUDENTS(student_id, student_name, major, gpa)
COURSES(course_id, course_name, instructor)
GRADES(student_id FK, course_id FK, grade, semester_taken)
```

### ข้อ 6
เขียน SQL เพื่อ Migrate ข้อมูลจาก Unnormalized ไปยัง 2NF Tables

**เฉลย**:
```sql
-- สมมติมีตาราง order_details_bad ที่ละเมิด 2NF
-- แปลงเป็น 2NF โดย Migrate ข้อมูล

-- Step 1: สร้างตาราง products จาก Partial Dependencies
INSERT INTO products (product_id, product_name, unit_price, category)
SELECT DISTINCT product_id, product_name, unit_price, category
FROM order_details_bad;

-- Step 2: สร้างตาราง orders จาก order-specific data
INSERT INTO orders (order_id, customer_id, order_date)
SELECT DISTINCT order_id, customer_id, order_date
FROM order_details_bad;

-- Step 3: สร้างตาราง order_items จาก Full Dependencies
INSERT INTO order_items (order_id, product_id, quantity, unit_price)
SELECT order_id, product_id, quantity, unit_price
FROM order_details_bad;

-- Step 4: ตรวจสอบจำนวน records
SELECT
    (SELECT COUNT(*) FROM order_details_bad) AS original,
    (SELECT COUNT(*) FROM order_items) AS new_items,
    (SELECT COUNT(DISTINCT product_id) FROM order_details_bad) AS expected_products,
    (SELECT COUNT(*) FROM products) AS actual_products;
```

### ข้อ 7
ออกแบบ Schema ที่อยู่ใน 2NF สำหรับ "ระบบจัดการ Conference" ที่มี: Paper, Author, Reviewer, Session

**เฉลย**:
```sql
CREATE TABLE authors (
    author_id   SERIAL PRIMARY KEY,
    name        VARCHAR(100) NOT NULL,
    affiliation VARCHAR(200),
    email       VARCHAR(255) UNIQUE
);

CREATE TABLE papers (
    paper_id    SERIAL PRIMARY KEY,
    title       VARCHAR(300) NOT NULL,
    abstract    TEXT,
    status      VARCHAR(20) DEFAULT 'submitted'
);

-- M:N: Papers have multiple Authors
CREATE TABLE paper_authors (
    paper_id    INTEGER REFERENCES papers,
    author_id   INTEGER REFERENCES authors,
    is_corresponding BOOLEAN DEFAULT FALSE,
    author_order SMALLINT DEFAULT 1,
    PRIMARY KEY (paper_id, author_id)
);

CREATE TABLE reviewers (
    reviewer_id INTEGER PRIMARY KEY,
    name        VARCHAR(100) NOT NULL,
    expertise   VARCHAR(200)
);

-- M:N: Papers have multiple Reviewers
CREATE TABLE reviews (
    paper_id    INTEGER REFERENCES papers,
    reviewer_id INTEGER REFERENCES reviewers,
    score       SMALLINT CHECK (score BETWEEN 1 AND 10),
    comments    TEXT,
    decision    VARCHAR(20),
    submitted_at TIMESTAMP,
    PRIMARY KEY (paper_id, reviewer_id)
);

CREATE TABLE sessions (
    session_id  SERIAL PRIMARY KEY,
    title       VARCHAR(200) NOT NULL,
    chair_id    INTEGER REFERENCES reviewers,
    room        VARCHAR(50),
    start_time  TIMESTAMP,
    end_time    TIMESTAMP
);

-- M:N: Sessions contain Papers
CREATE TABLE session_papers (
    session_id  INTEGER REFERENCES sessions,
    paper_id    INTEGER REFERENCES papers,
    presentation_order SMALLINT,
    duration_minutes SMALLINT DEFAULT 20,
    PRIMARY KEY (session_id, paper_id)
);
```

### ข้อ 8
จงอธิบายว่าทำไม 2NF จึงสำคัญเฉพาะกับ Composite Primary Key

**เฉลย**:
```
คำอธิบาย:
- Partial Dependency หมายถึง Non-key Attribute depend on
  "PART of" the Primary Key
- ถ้า PK มีแค่ Column เดียว (Simple Key) 
  ไม่มีทาง Attribute ใดจะ depend บน "ส่วนหนึ่ง" ของ PK
  เพราะ PK มีแค่ส่วนเดียว!

ดังนั้น:
- ตาราง with Simple PK → เป็น 2NF โดยอัตโนมัติ (ถ้าเป็น 1NF)
- ตาราง with Composite PK → ต้องตรวจสอบ Partial Dependencies

ตัวอย่าง:
EMPLOYEES(emp_id PK, name, dept_id, salary)
→ เป็น 2NF เสมอเพราะ PK เป็น Simple Key

ORDER_ITEMS(order_id, product_id, quantity, product_name PK:(order_id,product_id))
→ ละเมิด 2NF เพราะ product_name depends on product_id เพียงอย่างเดียว
```

### ข้อ 9
แก้ไขตารางนี้ให้อยู่ใน 2NF และเขียน SQL เต็ม:
```
SPORT_PLAYER_TEAM(player_id, team_id, season, player_name, 
                   position, team_name, team_city, goals_scored)
PK: (player_id, team_id, season)
```

**เฉลย**:
```sql
-- FD Analysis:
-- player_id → player_name, position (Partial!)
-- team_id → team_name, team_city (Partial!)
-- (player_id, team_id, season) → goals_scored (Full)

CREATE TABLE players (
    player_id   INTEGER PRIMARY KEY,
    name        VARCHAR(100) NOT NULL,
    position    VARCHAR(30)
);

CREATE TABLE teams (
    team_id     INTEGER PRIMARY KEY,
    name        VARCHAR(100) NOT NULL,
    city        VARCHAR(100)
);

CREATE TABLE player_team_seasons (
    player_id     INTEGER REFERENCES players,
    team_id       INTEGER REFERENCES teams,
    season        CHAR(7) NOT NULL,  -- '2023-24'
    goals_scored  INTEGER DEFAULT 0,
    games_played  INTEGER DEFAULT 0,
    PRIMARY KEY (player_id, team_id, season)
);
```

### ข้อ 10
เขียน Query เพื่อตรวจหา Potential 2NF Violations ในตาราง

**เฉลย**:
```sql
-- เมื่อเห็น Repeating Values ในคอลัมน์ที่ควรจะ Unique ต่อ Part ของ PK
-- นั่นอาจบ่งชี้ Partial Dependency

-- ตัวอย่าง: ตรวจสอบ order_details_bad
-- ถ้า product_name ซ้ำสำหรับ product_id เดียวกัน = ข้อมูลดี
-- ถ้า product_name แตกต่างกันสำหรับ product_id เดียวกัน = Inconsistency!

-- ตรวจสอบ Inconsistency (บ่งชี้ว่าควรแยกตาราง)
SELECT
    product_id,
    COUNT(DISTINCT product_name) AS distinct_names
FROM order_details_bad
GROUP BY product_id
HAVING COUNT(DISTINCT product_name) > 1;
-- ถ้าได้ผลลัพธ์ = มี Inconsistency → ควรแยกตาราง

-- ตรวจสอบ Storage Redundancy
SELECT
    product_id,
    product_name,
    COUNT(*) AS occurrences
FROM order_details_bad
GROUP BY product_id, product_name
ORDER BY occurrences DESC;
-- occurrences > 1 แสดงว่า product_name ซ้ำในหลาย orders
-- = Data Redundancy จาก Partial Dependency
```

---

*จบ Part 054: Second Normal Form (2NF)*

**ในส่วนต่อไป (Part 055)**: เราจะเรียนรู้เรื่อง Third Normal Form (3NF) และ Transitive Dependencies
