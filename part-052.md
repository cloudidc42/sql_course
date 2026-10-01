# Part 052: Entity-Relationship Modeling
# การสร้างแบบจำลอง Entity-Relationship

---

## บทนำ: ER Modeling คืออะไร?

Entity-Relationship (ER) Modeling เป็นเทคนิคการออกแบบฐานข้อมูลแบบ Visual ที่ช่วยให้นักออกแบบและผู้ใช้งานเข้าใจโครงสร้างข้อมูลร่วมกัน โดยใช้แนวคิดของ:

- **Entity**: สิ่งที่เราต้องการเก็บข้อมูล
- **Attribute**: คุณสมบัติของ Entity
- **Relationship**: ความสัมพันธ์ระหว่าง Entity

ER Diagram ถูกคิดค้นโดย Peter Chen ในปี 1976 และยังคงเป็นมาตรฐานการออกแบบฐานข้อมูลที่ใช้กันอย่างแพร่หลาย

---

## Entities และ Entity Types

### Entity คืออะไร?

Entity คือ "สิ่ง" หรือ "แนวคิด" ที่มีตัวตนและเราต้องการเก็บข้อมูลเกี่ยวกับมัน

```
ตัวอย่าง Entities:
- คน: Customer, Employee, Student
- สถานที่: Store, Warehouse, City
- สิ่งของ: Product, Book, Car
- แนวคิด: Order, Account, Course, Transaction
- เหตุการณ์: Purchase, Login, Meeting
```

### Entity Type vs Entity Instance

```
Entity Type:  CUSTOMER (แม่แบบ/Class)
Entity Instance: Customer #1001 - สมชาย ใจดี
                Customer #1002 - วิไล รักเรียน
                Customer #1003 - ประสิทธิ์ ขยันทำ
```

### Strong Entity vs Weak Entity

```
Strong Entity (Regular Entity):
- มีตัวตนเป็นอิสระ
- มี Primary Key เป็นของตัวเอง
- สามารถดำรงอยู่ได้โดยไม่พึ่ง Entity อื่น

ตัวอย่าง Strong Entity:
┌─────────────┐
│   CUSTOMER  │  ← มี customer_id เป็น PK
│             │  ← ดำรงอยู่ได้อิสระ
└─────────────┘

Weak Entity:
- ไม่มีตัวตนเป็นอิสระ
- ต้องพึ่งพา Strong Entity (Owner Entity)
- ใช้ Partial Key ร่วมกับ PK ของ Owner

ตัวอย่าง Weak Entity:
┌═════════════╗
║  ORDER_ITEM  ║  ← ต้องมี order_id (จาก ORDER)
║             ║  ← Partial Key: item_sequence
╚═════════════╝
ความสัมพันธ์: ORDER_ITEM ไม่มีความหมายโดยไม่มี ORDER
```

---

## Attributes และประเภทของ Attributes

### 1. Simple Attribute (Atomic Attribute)

ไม่สามารถแบ่งย่อยได้อีก

```
ตัวอย่าง Simple Attributes:
- Customer.first_name
- Product.price
- Employee.hire_date
- Order.quantity
```

### 2. Composite Attribute

ประกอบด้วย Attributes ย่อยหลายตัว

```
Composite Attribute:
name = {first_name, middle_name, last_name}
address = {street, city, state, zip_code, country}

ตัวอย่างในฐานข้อมูล:
```
```sql
-- Composite Attribute แปลงเป็นหลาย Columns
CREATE TABLE customers (
    customer_id   SERIAL PRIMARY KEY,
    -- name (composite)
    first_name    VARCHAR(50) NOT NULL,
    middle_name   VARCHAR(50),
    last_name     VARCHAR(50) NOT NULL,
    -- address (composite)
    street        VARCHAR(200),
    city          VARCHAR(100),
    state         VARCHAR(50),
    postal_code   VARCHAR(20),
    country       VARCHAR(50) DEFAULT 'TH'
);
```

### 3. Multivalued Attribute

มีได้หลายค่าสำหรับ Entity Instance เดียว

```
Multivalued Attributes:
- Customer.phone (มีหลายเบอร์โทร)
- Employee.skills (มีหลายทักษะ)
- Product.images (มีหลายรูป)
- Person.degrees (มีหลายวุฒิการศึกษา)
```
```sql
-- Multivalued Attribute → แยกตาราง
CREATE TABLE customer_phones (
    phone_id     SERIAL PRIMARY KEY,
    customer_id  INTEGER NOT NULL REFERENCES customers,
    phone_number VARCHAR(20) NOT NULL,
    phone_type   VARCHAR(20) CHECK (phone_type IN ('mobile','home','work','fax')),
    is_primary   BOOLEAN DEFAULT FALSE
);

CREATE TABLE employee_skills (
    emp_id       INTEGER NOT NULL REFERENCES employees,
    skill        VARCHAR(100) NOT NULL,
    level        VARCHAR(20) CHECK (level IN ('beginner','intermediate','advanced','expert')),
    PRIMARY KEY (emp_id, skill)
);
```

### 4. Derived Attribute

คำนวณได้จาก Attributes อื่น

```
Derived Attributes:
- Employee.age → คำนวณจาก birth_date
- Order.total → คำนวณจาก sum(items)
- Employee.years_of_service → คำนวณจาก hire_date
- Customer.order_count → นับจาก orders
```
```sql
-- Derived Attribute → ใช้ VIEW หรือ Generated Column

-- วิธีที่ 1: VIEW
CREATE VIEW employee_info AS
SELECT
    employee_id,
    first_name,
    last_name,
    birth_date,
    EXTRACT(YEAR FROM AGE(birth_date)) AS age,
    hire_date,
    EXTRACT(YEAR FROM AGE(hire_date)) AS years_of_service
FROM employees;

-- วิธีที่ 2: Generated Column (PostgreSQL 12+)
CREATE TABLE employees (
    employee_id  SERIAL PRIMARY KEY,
    first_name   VARCHAR(50),
    birth_date   DATE,
    age_years    INTEGER GENERATED ALWAYS AS (
        EXTRACT(YEAR FROM AGE(birth_date))::INTEGER
    ) STORED
);
```

### 5. Key Attribute

ใช้ระบุ Entity Instance ได้เฉพาะตัว

```
Key Attributes:
- Customer.customer_id (Surrogate Key)
- Employee.employee_id
- Product.sku (Natural Key)
- Student.student_number

Candidate Keys:
- Employee: employee_id, national_id, email
  (ทุกตัวระบุพนักงานได้ แต่เลือก employee_id เป็น Primary Key)
```

---

## Relationships และ Relationship Types

### Relationship คืออะไร?

Relationship คือความสัมพันธ์ที่มีความหมายระหว่าง Entity หรือกลุ่มของ Entity

```
ตัวอย่าง Relationships:
- Customer PLACES Order (ลูกค้าสั่งซื้อ)
- Employee WORKS_IN Department (พนักงานทำงานในแผนก)
- Student TAKES Course (นักเรียนลงทะเบียนวิชา)
- Doctor TREATS Patient (หมอรักษาคนไข้)
```

### Degree of Relationship

```
Unary (1 Entity): Self-referencing
┌──────────┐    manages    ┌──────────┐
│ EMPLOYEE │──────────────>│ EMPLOYEE │
└──────────┘               └──────────┘

Binary (2 Entities): ที่พบบ่อยที่สุด
┌──────────┐    places    ┌──────────┐
│ CUSTOMER │─────────────>│  ORDER   │
└──────────┘              └──────────┘

Ternary (3 Entities): ซับซ้อนกว่า
┌──────────┐              ┌──────────┐
│ SUPPLIER │              │ PROJECT  │
└────┬─────┘              └────┬─────┘
     │      ┌──────────┐       │
     └──────>│  SUPPLY  │<──────┘
             │ (Part,   │
             │  Qty)    │
             └────┬─────┘
                  │
             ┌────▼─────┐
             │   PART   │
             └──────────┘
```

---

## Cardinality Notation

### Chen Notation (Original)

Peter Chen คิดค้นสัญลักษณ์ดั้งเดิมที่ใช้ตัวเลข 1, N, M

```
One-to-One (1:1)
┌──────────┐   1        1   ┌──────────────┐
│ EMPLOYEE │────────────────│    PARKING   │
└──────────┘                │     SPOT     │
                            └──────────────┘

One-to-Many (1:N)
┌──────────┐   1        N   ┌──────────┐
│ CUSTOMER │────────────────│  ORDER   │
└──────────┘                └──────────┘

Many-to-Many (M:N)
┌──────────┐   M        N   ┌──────────┐
│ STUDENT  │────────────────│  COURSE  │
└──────────┘                └──────────┘
```

### Crow's Foot Notation (Modern Standard)

ใช้กันอย่างแพร่หลายในเครื่องมือ CASE ปัจจุบัน

```
สัญลักษณ์:
  ──|   = Exactly One (หนึ่งตัว)
  ──O   = Zero or One (ไม่มีหรือหนึ่งตัว)
  ──<   = One or More (หนึ่งหรือมากกว่า) ← รูปแบบ crow's foot
  ──O<  = Zero or More (ไม่มีหรือมากกว่า)
  ──|<  = One or More (mandatory many)

ตัวอย่าง One-to-Many:
┌──────────┐           ┌──────────┐
│ CUSTOMER │─────|──O< │  ORDER   │
└──────────┘           └──────────┘
                 ↑     ↑
           (1 customer)(0 or more orders)

ตัวอย่าง Many-to-Many:
┌──────────┐           ┌──────────┐
│ STUDENT  │───O<──O<──│  COURSE  │
└──────────┘           └──────────┘
           ↑           ↑
    (0+ students) (0+ courses)
```

### ตัวอย่าง Crow's Foot ทุกรูปแบบ

```
1:1 (One-to-One Mandatory)
PERSON ──|──|── PASSPORT
(คนหนึ่งมี Passport หนึ่งเล่ม, Passport เป็นของคนหนึ่งคน)

1:1 Optional
EMPLOYEE ──|──O── PARKING_SPOT
(พนักงานอาจมีหรือไม่มีที่จอดรถ)

1:N (One Customer, Many Orders)
CUSTOMER ──|──O<── ORDER
(ลูกค้ามีคำสั่งซื้อ 0 หรือมากกว่า)

M:N (Many-to-Many)
PRODUCT ──O<──O<── CATEGORY
(สินค้าหนึ่งอยู่ในหลายหมวดหมู่ หมวดหมู่หนึ่งมีหลายสินค้า)
```

---

## Participation Constraints

### Total Participation (Mandatory)

ทุก Entity Instance ต้องมีส่วนร่วมใน Relationship

```
ตัวอย่าง Total Participation:
ORDER ──════════── CUSTOMER
(คำสั่งซื้อทุกอันต้องมีลูกค้า ← Total Participation ของ ORDER)

ใน SQL: NOT NULL constraint บน Foreign Key
CREATE TABLE orders (
    order_id    SERIAL PRIMARY KEY,
    customer_id INTEGER NOT NULL REFERENCES customers  -- Total!
);
```

### Partial Participation (Optional)

บาง Entity Instance อาจไม่มีส่วนร่วมใน Relationship

```
ตัวอย่าง Partial Participation:
CUSTOMER ──────── ORDER
(ลูกค้าบางคนอาจไม่เคยสั่งซื้อ ← Partial Participation ของ CUSTOMER)

ใน SQL: NULL allowed
-- ไม่มี constraint ว่า customer ต้องมี order
```

---

## ER Diagram Examples - 5+ ตัวอย่าง

### ตัวอย่างที่ 1: ระบบมหาวิทยาลัย

```
UNIVERSITY DATABASE ER DIAGRAM

┌────────────────────────────────────────────────────────────────────┐
│                                                                    │
│  ┌────────────┐    1      N    ┌─────────────┐                    │
│  │ DEPARTMENT │────────────────│   FACULTY   │                    │
│  │            │  belongs_to    │             │                    │
│  │ dept_id PK │                │ faculty_id  │                    │
│  │ name       │                │ first_name  │                    │
│  │ location   │                │ last_name   │                    │
│  └────────────┘                │ title       │                    │
│        │                       └──────┬──────┘                    │
│        │ offers 1:N                   │ teaches M:N               │
│        ▼                              ▼                            │
│  ┌────────────┐    M      N    ┌─────────────┐                    │
│  │   COURSE   │────────────────│   SECTION   │                    │
│  │            │    has         │             │                    │
│  │ course_id  │                │ section_id  │                    │
│  │ title      │                │ semester    │                    │
│  │ credits    │                │ year        │                    │
│  └────────────┘                │ room        │                    │
│                                └──────┬──────┘                    │
│                                       │ enrolls M:N               │
│                                       ▼                            │
│                               ┌─────────────┐                    │
│                               │   STUDENT   │                    │
│                               │             │                    │
│                               │ student_id  │                    │
│                               │ first_name  │                    │
│                               │ major       │                    │
│                               │ gpa         │                    │
│                               └─────────────┘                    │
└────────────────────────────────────────────────────────────────────┘
```

```sql
-- SQL Implementation
CREATE TABLE departments (
    dept_id   SERIAL PRIMARY KEY,
    name      VARCHAR(100) NOT NULL,
    location  VARCHAR(100),
    head_id   INTEGER  -- FK to faculty (set after)
);

CREATE TABLE faculty (
    faculty_id  SERIAL PRIMARY KEY,
    dept_id     INTEGER NOT NULL REFERENCES departments,
    first_name  VARCHAR(50) NOT NULL,
    last_name   VARCHAR(50) NOT NULL,
    title       VARCHAR(50),
    email       VARCHAR(255) UNIQUE
);

CREATE TABLE courses (
    course_id   SERIAL PRIMARY KEY,
    dept_id     INTEGER NOT NULL REFERENCES departments,
    code        VARCHAR(10) UNIQUE NOT NULL,
    title       VARCHAR(200) NOT NULL,
    credits     SMALLINT NOT NULL CHECK (credits > 0)
);

CREATE TABLE sections (
    section_id  SERIAL PRIMARY KEY,
    course_id   INTEGER NOT NULL REFERENCES courses,
    faculty_id  INTEGER REFERENCES faculty,
    semester    VARCHAR(10) NOT NULL,
    year        SMALLINT NOT NULL,
    room        VARCHAR(20),
    capacity    SMALLINT NOT NULL DEFAULT 30
);

CREATE TABLE students (
    student_id  SERIAL PRIMARY KEY,
    dept_id     INTEGER REFERENCES departments,
    student_no  VARCHAR(20) UNIQUE NOT NULL,
    first_name  VARCHAR(50) NOT NULL,
    last_name   VARCHAR(50) NOT NULL,
    gpa         DECIMAL(3,2) CHECK (gpa BETWEEN 0 AND 4)
);

CREATE TABLE enrollments (
    enrollment_id SERIAL PRIMARY KEY,
    student_id  INTEGER NOT NULL REFERENCES students,
    section_id  INTEGER NOT NULL REFERENCES sections,
    grade       CHAR(2),
    UNIQUE (student_id, section_id)
);
```

### ตัวอย่างที่ 2: ระบบโรงพยาบาล

```
HOSPITAL DATABASE ER DIAGRAM

┌──────────────────────────────────────────────────────────────────┐
│                                                                  │
│  ┌──────────┐         ┌──────────────┐         ┌──────────┐    │
│  │ PATIENT  │         │ APPOINTMENT  │         │  DOCTOR  │    │
│  │          │1      N │              │N       1│          │    │
│  │ pat_id   │─────────│ appt_id      │─────────│ doc_id   │    │
│  │ name     │ books   │ date_time    │ attends │ name     │    │
│  │ dob      │         │ reason       │         │ specialty│    │
│  │ blood_type│        │ status       │         │ license  │    │
│  └──────────┘         └──────┬───────┘         └──────────┘    │
│                              │                                  │
│                              │ results_in 1:N                   │
│                              ▼                                  │
│                       ┌──────────────┐                         │
│                       │  DIAGNOSIS   │                         │
│                       │              │                         │
│                       │ diag_id      │                         │
│                       │ icd_code     │                         │
│                       │ description  │                         │
│                       └──────┬───────┘                         │
│                              │ requires M:N                    │
│                              ▼                                  │
│                       ┌──────────────┐                         │
│                       │ PRESCRIPTION │                         │
│                       │              │                         │
│                       │ rx_id        │─────────────────────┐  │
│                       │ dosage       │                     │  │
│                       │ duration     │                     │  │
│                       └──────────────┘                     │  │
│                                                            │  │
│                                               ┌────────────┘  │
│                                               ▼               │
│                                        ┌──────────────┐       │
│                                        │ MEDICATION   │       │
│                                        │              │       │
│                                        │ med_id       │       │
│                                        │ name         │       │
│                                        │ manufacturer │       │
│                                        └──────────────┘       │
└──────────────────────────────────────────────────────────────────┘
```

```sql
CREATE TABLE doctors (
    doctor_id   SERIAL PRIMARY KEY,
    license_no  VARCHAR(20) UNIQUE NOT NULL,
    first_name  VARCHAR(50) NOT NULL,
    last_name   VARCHAR(50) NOT NULL,
    specialty   VARCHAR(100),
    phone       VARCHAR(20)
);

CREATE TABLE patients (
    patient_id  SERIAL PRIMARY KEY,
    hn          VARCHAR(20) UNIQUE NOT NULL,  -- Hospital Number
    first_name  VARCHAR(50) NOT NULL,
    last_name   VARCHAR(50) NOT NULL,
    dob         DATE,
    blood_type  VARCHAR(3) CHECK (blood_type IN ('A','B','AB','O','A+','A-','B+','B-','AB+','AB-','O+','O-')),
    phone       VARCHAR(20),
    address     TEXT
);

CREATE TABLE appointments (
    appt_id     SERIAL PRIMARY KEY,
    patient_id  INTEGER NOT NULL REFERENCES patients,
    doctor_id   INTEGER NOT NULL REFERENCES doctors,
    appt_datetime TIMESTAMP NOT NULL,
    reason      TEXT,
    status      VARCHAR(20) DEFAULT 'scheduled'
                CHECK (status IN ('scheduled','completed','cancelled','no-show'))
);

CREATE TABLE diagnoses (
    diagnosis_id SERIAL PRIMARY KEY,
    appt_id     INTEGER NOT NULL REFERENCES appointments,
    icd_code    VARCHAR(10) NOT NULL,
    description TEXT NOT NULL,
    is_primary  BOOLEAN DEFAULT FALSE
);

CREATE TABLE medications (
    med_id      SERIAL PRIMARY KEY,
    name        VARCHAR(200) NOT NULL,
    generic_name VARCHAR(200),
    manufacturer VARCHAR(100),
    unit        VARCHAR(20)
);

CREATE TABLE prescriptions (
    rx_id       SERIAL PRIMARY KEY,
    diagnosis_id INTEGER NOT NULL REFERENCES diagnoses,
    med_id      INTEGER NOT NULL REFERENCES medications,
    dosage      VARCHAR(100) NOT NULL,
    frequency   VARCHAR(50) NOT NULL,
    duration_days SMALLINT NOT NULL,
    instructions TEXT
);
```

### ตัวอย่างที่ 3: ระบบ Social Media

```
SOCIAL MEDIA ER DIAGRAM

┌────────────────────────────────────────────────────────┐
│                                                        │
│  ┌──────────┐  follows  ┌──────────┐                  │
│  │  USER    │──M:N──────│  USER    │ (Self-referencing)│
│  │          │           │          │                  │
│  │ user_id  │           └──────────┘                  │
│  │ username │                                          │
│  │ email    │                                          │
│  │ bio      │                                          │
│  └────┬─────┘                                          │
│       │ creates 1:N                                    │
│       ▼                                                │
│  ┌──────────┐  has  ┌──────────────┐                  │
│  │   POST   │──1:N──│   COMMENT    │                  │
│  │          │       │              │                  │
│  │ post_id  │       │ comment_id   │                  │
│  │ content  │       │ content      │                  │
│  │ image    │       │ user_id FK   │                  │
│  └────┬─────┘       └──────────────┘                  │
│       │ has M:N                                        │
│       ▼                                                │
│  ┌──────────┐                                          │
│  │  LIKE    │                                          │
│  │          │                                          │
│  │ user_id  │                                          │
│  │ post_id  │                                          │
│  │ created  │                                          │
│  └──────────┘                                          │
└────────────────────────────────────────────────────────┘
```

```sql
CREATE TABLE users (
    user_id     SERIAL PRIMARY KEY,
    username    VARCHAR(50) UNIQUE NOT NULL,
    email       VARCHAR(255) UNIQUE NOT NULL,
    display_name VARCHAR(100),
    bio         TEXT,
    avatar_url  VARCHAR(500),
    created_at  TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    is_active   BOOLEAN DEFAULT TRUE
);

-- Self-referencing M:N for follows
CREATE TABLE follows (
    follower_id INTEGER NOT NULL REFERENCES users,
    following_id INTEGER NOT NULL REFERENCES users,
    created_at  TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (follower_id, following_id),
    CHECK (follower_id <> following_id)  -- ห้ามติดตามตัวเอง
);

CREATE TABLE posts (
    post_id     SERIAL PRIMARY KEY,
    user_id     INTEGER NOT NULL REFERENCES users,
    content     TEXT,
    image_url   VARCHAR(500),
    created_at  TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    is_visible  BOOLEAN DEFAULT TRUE
);

CREATE TABLE comments (
    comment_id  SERIAL PRIMARY KEY,
    post_id     INTEGER NOT NULL REFERENCES posts,
    user_id     INTEGER NOT NULL REFERENCES users,
    parent_id   INTEGER REFERENCES comments,  -- nested comments
    content     TEXT NOT NULL,
    created_at  TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE likes (
    user_id     INTEGER NOT NULL REFERENCES users,
    post_id     INTEGER NOT NULL REFERENCES posts,
    created_at  TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (user_id, post_id)
);
```

### ตัวอย่างที่ 4: ระบบโรงแรม

```
HOTEL BOOKING SYSTEM ER DIAGRAM

┌────────────────────────────────────────────────────────────────┐
│                                                                │
│  ┌──────────────┐         ┌────────────────┐                  │
│  │    GUEST     │1      N │    BOOKING     │                  │
│  │              │─────────│                │                  │
│  │ guest_id     │  makes  │ booking_id     │                  │
│  │ name         │         │ check_in       │                  │
│  │ passport     │         │ check_out      │                  │
│  │ nationality  │         │ total_price    │                  │
│  │ email        │         │ status         │                  │
│  └──────────────┘         └───────┬────────┘                  │
│                                   │ for 1:1 or 1:N            │
│                                   │                            │
│  ┌──────────────┐                 │                            │
│  │  ROOM_TYPE   │1              N │                            │
│  │              │                 ▼                            │
│  │ type_id      │         ┌────────────────┐                  │
│  │ name         │─────────│      ROOM      │                  │
│  │ price/night  │  is type │                │                  │
│  │ max_guests   │         │ room_id        │                  │
│  │ amenities    │         │ room_number    │                  │
│  └──────────────┘         │ floor         │                  │
│                           │ status        │                  │
│                           └────────────────┘                  │
└────────────────────────────────────────────────────────────────┘
```

```sql
CREATE TABLE room_types (
    type_id     SERIAL PRIMARY KEY,
    name        VARCHAR(50) NOT NULL,  -- 'standard', 'deluxe', 'suite'
    price_per_night DECIMAL(10,2) NOT NULL,
    max_guests  SMALLINT NOT NULL,
    description TEXT,
    amenities   TEXT[]  -- PostgreSQL array
);

CREATE TABLE rooms (
    room_id     SERIAL PRIMARY KEY,
    type_id     INTEGER NOT NULL REFERENCES room_types,
    room_number VARCHAR(10) UNIQUE NOT NULL,
    floor       SMALLINT NOT NULL,
    status      VARCHAR(20) DEFAULT 'available'
                CHECK (status IN ('available','occupied','maintenance','cleaning'))
);

CREATE TABLE guests (
    guest_id    SERIAL PRIMARY KEY,
    first_name  VARCHAR(50) NOT NULL,
    last_name   VARCHAR(50) NOT NULL,
    passport_no VARCHAR(20) UNIQUE,
    nationality VARCHAR(50),
    email       VARCHAR(255),
    phone       VARCHAR(20)
);

CREATE TABLE bookings (
    booking_id  SERIAL PRIMARY KEY,
    guest_id    INTEGER NOT NULL REFERENCES guests,
    room_id     INTEGER NOT NULL REFERENCES rooms,
    check_in    DATE NOT NULL,
    check_out   DATE NOT NULL,
    adults      SMALLINT NOT NULL DEFAULT 1,
    children    SMALLINT NOT NULL DEFAULT 0,
    total_price DECIMAL(10,2) NOT NULL,
    status      VARCHAR(20) DEFAULT 'confirmed',
    special_requests TEXT,
    CONSTRAINT chk_dates CHECK (check_out > check_in)
);
```

### ตัวอย่างที่ 5: ระบบ Inventory (คลังสินค้า)

```
INVENTORY MANAGEMENT ER DIAGRAM

  ┌──────────────┐         ┌──────────────┐
  │   SUPPLIER   │1      N │   PRODUCT    │
  │              │─────────│              │
  │ supplier_id  │supplies │ product_id   │
  │ name         │         │ name         │
  │ contact      │         │ sku          │
  │ country      │         │ unit_price   │
  └──────────────┘         └──────┬───────┘
                                  │ stored_in M:N
                                  ▼
  ┌──────────────┐         ┌──────────────┐
  │  WAREHOUSE   │1      N │   INVENTORY  │
  │              │─────────│              │
  │ warehouse_id │contains │ product_id   │
  │ name         │         │ warehouse_id │
  │ location     │         │ quantity     │
  │ capacity     │         │ min_stock    │
  └──────────────┘         └──────────────┘
```

```sql
CREATE TABLE suppliers (
    supplier_id SERIAL PRIMARY KEY,
    name        VARCHAR(200) NOT NULL,
    contact_name VARCHAR(100),
    email       VARCHAR(255),
    phone       VARCHAR(20),
    country     VARCHAR(50),
    is_active   BOOLEAN DEFAULT TRUE
);

CREATE TABLE warehouses (
    warehouse_id SERIAL PRIMARY KEY,
    name        VARCHAR(100) NOT NULL,
    location    VARCHAR(300),
    manager_id  INTEGER,
    capacity    INTEGER
);

CREATE TABLE products (
    product_id  SERIAL PRIMARY KEY,
    supplier_id INTEGER REFERENCES suppliers,
    name        VARCHAR(200) NOT NULL,
    sku         VARCHAR(50) UNIQUE NOT NULL,
    description TEXT,
    unit_price  DECIMAL(10,2) NOT NULL,
    unit        VARCHAR(20),  -- 'piece', 'kg', 'liter'
    is_active   BOOLEAN DEFAULT TRUE
);

CREATE TABLE inventory (
    inventory_id SERIAL PRIMARY KEY,
    product_id  INTEGER NOT NULL REFERENCES products,
    warehouse_id INTEGER NOT NULL REFERENCES warehouses,
    quantity    INTEGER NOT NULL DEFAULT 0,
    min_stock   INTEGER NOT NULL DEFAULT 0,
    max_stock   INTEGER,
    last_updated TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UNIQUE (product_id, warehouse_id)
);

CREATE TABLE stock_movements (
    movement_id  SERIAL PRIMARY KEY,
    product_id   INTEGER NOT NULL REFERENCES products,
    warehouse_id INTEGER NOT NULL REFERENCES warehouses,
    movement_type VARCHAR(20) CHECK (movement_type IN ('in','out','transfer','adjustment')),
    quantity     INTEGER NOT NULL,
    reference_no VARCHAR(50),
    notes        TEXT,
    created_at   TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

---

## Identifying Relationships (ความสัมพันธ์แบบ Identifying)

### Identifying vs Non-Identifying

```
Identifying Relationship:
- PK ของ Weak Entity ประกอบด้วย PK ของ Owner Entity ด้วย
- Weak Entity ขึ้นอยู่กับ Owner Entity อย่างสมบูรณ์
- แสดงด้วยเส้นทึบ (solid line) ใน ER Diagram

ตัวอย่าง:
ORDER ═══════ ORDER_ITEM
PK: order_id  PK: (order_id, item_seq) ← ต้องมี order_id

Non-Identifying Relationship:
- FK ของ Child Entity ไม่ได้เป็นส่วนหนึ่งของ PK
- Child Entity มีตัวตนอิสระ
- แสดงด้วยเส้นประ (dashed line)

ตัวอย่าง:
CUSTOMER ─────── ORDER
PK: customer_id  PK: order_id (อิสระ, customer_id เป็นแค่ FK)
```

```sql
-- Identifying Relationship
CREATE TABLE orders (
    order_id    SERIAL PRIMARY KEY
);

CREATE TABLE order_items (
    order_id    INTEGER NOT NULL REFERENCES orders,
    item_seq    SMALLINT NOT NULL,
    product_id  INTEGER NOT NULL REFERENCES products,
    quantity    INTEGER NOT NULL,
    PRIMARY KEY (order_id, item_seq)  -- Composite PK ที่รวม order_id
);

-- Non-Identifying Relationship
CREATE TABLE customers (
    customer_id SERIAL PRIMARY KEY
);

CREATE TABLE orders (
    order_id    SERIAL PRIMARY KEY,     -- Independent PK
    customer_id INTEGER REFERENCES customers  -- FK เท่านั้น
);
```

---

## ER to Relational Mapping Rules

### กฎที่ 1: Strong Entity → Table

```
ER:   CUSTOMER (customer_id, name, email)
SQL:  CREATE TABLE customers (
          customer_id SERIAL PRIMARY KEY,
          name        VARCHAR(100) NOT NULL,
          email       VARCHAR(255) UNIQUE NOT NULL
      );
```

### กฎที่ 2: Weak Entity → Table with Composite PK

```
ER:   ORDER_ITEM (item_seq, quantity, price) — Weak, depends on ORDER
SQL:  CREATE TABLE order_items (
          order_id  INTEGER NOT NULL REFERENCES orders,
          item_seq  SMALLINT NOT NULL,
          quantity  INTEGER NOT NULL,
          price     DECIMAL(10,2),
          PRIMARY KEY (order_id, item_seq)
      );
```

### กฎที่ 3: Multivalued Attribute → Separate Table

```
ER:   EMPLOYEE {phones: [mobile, home, work]}
SQL:  CREATE TABLE employee_phones (
          employee_id  INTEGER NOT NULL REFERENCES employees,
          phone_number VARCHAR(20) NOT NULL,
          phone_type   VARCHAR(20),
          PRIMARY KEY (employee_id, phone_number)
      );
```

### กฎที่ 4: 1:1 Relationship

```
ER:   EMPLOYEE ──1:1── EMPLOYEE_DETAIL
SQL:  -- Option A: Merge into one table (ถ้า Always exists)
      CREATE TABLE employees (
          emp_id   SERIAL PRIMARY KEY,
          name     VARCHAR(100),
          -- detail attributes here
          ssn      VARCHAR(11) UNIQUE
      );
      
      -- Option B: Separate tables with shared PK
      CREATE TABLE employees (...);
      CREATE TABLE employee_details (
          emp_id   INTEGER PRIMARY KEY REFERENCES employees,  -- PK=FK
          ssn      VARCHAR(11) UNIQUE,
          emergency_contact VARCHAR(100)
      );
```

### กฎที่ 5: 1:N Relationship

```
ER:   DEPARTMENT ──1:N── EMPLOYEE
SQL:  CREATE TABLE departments (dept_id SERIAL PRIMARY KEY, ...);
      CREATE TABLE employees (
          emp_id  SERIAL PRIMARY KEY,
          dept_id INTEGER REFERENCES departments,  -- FK บนฝั่ง N
          ...
      );
```

### กฎที่ 6: M:N Relationship → Junction Table

```
ER:   STUDENT ──M:N── COURSE
SQL:  CREATE TABLE students (...);
      CREATE TABLE courses (...);
      CREATE TABLE enrollments (  -- Junction/Bridge table
          student_id INTEGER NOT NULL REFERENCES students,
          course_id  INTEGER NOT NULL REFERENCES courses,
          grade      CHAR(2),
          enrolled_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
          PRIMARY KEY (student_id, course_id)
      );
```

### กฎที่ 7: Ternary Relationship

```
ER:   SUPPLIER ──ternary── PART ──── PROJECT
SQL:  CREATE TABLE supply_assignments (
          supplier_id INTEGER NOT NULL REFERENCES suppliers,
          part_id     INTEGER NOT NULL REFERENCES parts,
          project_id  INTEGER NOT NULL REFERENCES projects,
          quantity    INTEGER NOT NULL,
          PRIMARY KEY (supplier_id, part_id, project_id)
      );
```

---

## ER Diagram สมบูรณ์: ระบบ E-Commerce

```
COMPLETE E-COMMERCE ER DIAGRAM
═══════════════════════════════════════════════════════════════════

  ┌─────────────┐     places      ┌─────────────────┐
  │  CUSTOMER   │─────1:N────────>│      ORDER      │
  │             │                 │                 │
  │ customer_id │                 │ order_id        │
  │ first_name  │                 │ customer_id FK  │
  │ last_name   │                 │ address_id FK   │
  │ email UNIQUE│                 │ order_number    │
  │ phone       │                 │ status          │
  │ created_at  │                 │ total_amount    │
  └──────┬──────┘                 │ created_at      │
         │                        └────────┬────────┘
         │ has                             │ contains
         │ 1:N                             │ 1:N
         ▼                                 ▼
  ┌─────────────┐              ┌──════════════════╗
  │   ADDRESS   │              ║   ORDER_ITEM     ║ (Weak Entity)
  │             │              ║                 ║
  │ address_id  │              ║ order_id FK     ║
  │ customer_id │              ║ item_seq        ║
  │ street      │              ║ variant_id FK   ║
  │ city        │              ║ quantity        ║
  │ is_default  │              ║ unit_price      ║
  └─────────────┘              ╚════════┬════════╝
                                        │ is
                                        │ M:1
                                        ▼
  ┌─────────────┐    belongs     ┌─────────────────┐
  │  CATEGORY   │──────M:N──────│PRODUCT_VARIANT  │
  │             │     to         │                 │
  │ category_id │                │ variant_id      │
  │ parent_id   │                │ product_id FK   │
  │ name        │                │ sku UNIQUE      │
  │ slug        │                │ color           │
  └─────────────┘                │ size            │
                                 │ price           │
                                 │ stock_qty       │
                                 └────────┬────────┘
                                          │ is variant of
                                          │ M:1
                                          ▼
                                 ┌─────────────────┐
                                 │     PRODUCT     │
                                 │                 │
                                 │ product_id      │
                                 │ name            │
                                 │ description     │
                                 │ brand           │
                                 └─────────────────┘
                                          │ has
                                          │ 1:N
                                          ▼
                                 ┌─────────────────┐
                                 │     REVIEW      │
                                 │                 │
                                 │ review_id       │
                                 │ customer_id FK  │
                                 │ product_id FK   │
                                 │ rating (1-5)    │
                                 │ body            │
                                 └─────────────────┘

═══════════════════════════════════════════════════════════════════
```

---

## ER Diagram: ระบบ HR (Human Resources)

```
HR MANAGEMENT SYSTEM ER DIAGRAM

  ┌───────────────┐   manages   ┌───────────────┐
  │  DEPARTMENT   │─────1:1─────│   EMPLOYEE    │ (manager)
  │               │             │               │
  │ dept_id       │             │ emp_id        │
  │ name          │◄────1:N─────│ dept_id FK    │
  │ budget        │  belongs to │ manager_id FK │ (self-ref)
  │ location      │             │ first_name    │
  └───────────────┘             │ last_name     │
                                │ hire_date     │
                                │ salary        │
                                └───────┬───────┘
                                        │
                     ┌──────────────────┼──────────────────┐
                     │                  │                  │
                     │ 1:N              │ M:N              │ 1:N
                     ▼                  ▼                  ▼
             ┌──────────────┐  ┌──────────────┐  ┌──────────────┐
             │ JOB_HISTORY  │  │   PROJECT    │  │  DEPENDENT   │
             │              │  │              │  │              │
             │ history_id   │  │ project_id   │  │ dep_id       │
             │ emp_id FK    │  │ name         │  │ emp_id FK    │
             │ job_title    │  │ start_date   │  │ name         │
             │ start_date   │  │ end_date     │  │ relation     │
             │ end_date     │  │ budget       │  │ birth_date   │
             └──────────────┘  └──────────────┘  └──────────────┘
```

```sql
CREATE TABLE departments (
    dept_id      SERIAL PRIMARY KEY,
    name         VARCHAR(100) NOT NULL,
    budget       DECIMAL(15,2),
    location     VARCHAR(100),
    manager_id   INTEGER  -- FK to employees (set after)
);

CREATE TABLE employees (
    emp_id       SERIAL PRIMARY KEY,
    dept_id      INTEGER REFERENCES departments,
    manager_id   INTEGER REFERENCES employees,  -- self-referencing
    first_name   VARCHAR(50) NOT NULL,
    last_name    VARCHAR(50) NOT NULL,
    email        VARCHAR(255) UNIQUE NOT NULL,
    hire_date    DATE NOT NULL,
    salary       DECIMAL(10,2),
    job_title    VARCHAR(100),
    is_active    BOOLEAN DEFAULT TRUE
);

-- Add manager FK after employees table exists
ALTER TABLE departments
    ADD CONSTRAINT fk_dept_manager
    FOREIGN KEY (manager_id) REFERENCES employees(emp_id);

CREATE TABLE job_history (
    history_id   SERIAL PRIMARY KEY,
    emp_id       INTEGER NOT NULL REFERENCES employees,
    dept_id      INTEGER REFERENCES departments,
    job_title    VARCHAR(100) NOT NULL,
    start_date   DATE NOT NULL,
    end_date     DATE,
    salary       DECIMAL(10,2),
    CONSTRAINT chk_dates CHECK (end_date IS NULL OR end_date >= start_date)
);

CREATE TABLE projects (
    project_id   SERIAL PRIMARY KEY,
    name         VARCHAR(200) NOT NULL,
    description  TEXT,
    start_date   DATE,
    end_date     DATE,
    budget       DECIMAL(15,2),
    status       VARCHAR(20) DEFAULT 'active'
);

CREATE TABLE project_assignments (
    emp_id       INTEGER NOT NULL REFERENCES employees,
    project_id   INTEGER NOT NULL REFERENCES projects,
    role         VARCHAR(100),
    hours_per_week DECIMAL(4,1),
    start_date   DATE,
    end_date     DATE,
    PRIMARY KEY (emp_id, project_id)
);

CREATE TABLE dependents (
    dep_id       SERIAL PRIMARY KEY,
    emp_id       INTEGER NOT NULL REFERENCES employees,
    name         VARCHAR(100) NOT NULL,
    relationship VARCHAR(20) CHECK (relationship IN ('spouse','child','parent')),
    birth_date   DATE,
    gender       CHAR(1) CHECK (gender IN ('M','F'))
);
```

---

## สรุป ER Modeling

```
สิ่งที่ต้องจำ:

1. Entity Types:
   - Strong Entity: มีตัวตนอิสระ
   - Weak Entity: ต้องพึ่ง Owner Entity

2. Attribute Types:
   - Simple: ค่าเดี่ยว
   - Composite: รวมจากหลาย attributes
   - Multivalued: มีได้หลายค่า → แยกตาราง
   - Derived: คำนวณได้ → ใช้ VIEW

3. Cardinality:
   - 1:1, 1:N, M:N
   - Chen Notation vs Crow's Foot

4. Participation:
   - Total (Mandatory): NOT NULL
   - Partial (Optional): NULL allowed

5. Mapping Rules:
   - Entity → Table
   - Multivalued → Separate Table
   - M:N → Junction Table
   - 1:N → FK บนฝั่ง N
```

---

## แบบฝึกหัด (10 ข้อ)

### ข้อ 1
วาด ER Diagram สำหรับระบบ Airline (สายการบิน) ที่มี: เที่ยวบิน, เครื่องบิน, นักบิน, ผู้โดยสาร, ที่นั่ง

**เฉลย**:
```
AIRCRAFT ──N:1──FLIGHT ──M:N──PASSENGER
  │                │               │
  │ aircraft_id    │ flight_id      │ passenger_id
  │ model          │ flight_no      │ name
  │ capacity       │ departure      │ passport
                   │ arrival        
                   │               
PILOT ──M:N── FLIGHT (Pilot flies Flight)

SEAT ──Weak──AIRCRAFT (seat_no, class)
BOOKING ──M:N── FLIGHT + PASSENGER (with seat assignment)
```

### ข้อ 2
ระบุว่า Attribute ต่อไปนี้เป็นประเภทใด:
a) Student.grades (Array of grades)
b) Person.age (คำนวณจาก birth_date)
c) Address.full_address (street + city + zip)
d) Employee.employee_id

**เฉลย**:
```
a) Multivalued Attribute → แยกตาราง student_grades
b) Derived Attribute → คำนวณใน VIEW หรือ Generated Column
c) Composite Attribute → แยก columns (street, city, zip)
d) Key Attribute (Primary Key)
```

### ข้อ 3
แปลง M:N Relationship ระหว่าง PRODUCT และ CATEGORY เป็น SQL

**เฉลย**:
```sql
CREATE TABLE products (
    product_id  SERIAL PRIMARY KEY,
    name        VARCHAR(200) NOT NULL
);

CREATE TABLE categories (
    category_id SERIAL PRIMARY KEY,
    name        VARCHAR(100) NOT NULL
);

-- Junction table for M:N
CREATE TABLE product_categories (
    product_id  INTEGER NOT NULL REFERENCES products ON DELETE CASCADE,
    category_id INTEGER NOT NULL REFERENCES categories ON DELETE CASCADE,
    is_primary  BOOLEAN DEFAULT FALSE,
    PRIMARY KEY (product_id, category_id)
);
```

### ข้อ 4
อธิบาย Weak Entity พร้อมตัวอย่างใน SQL

**เฉลย**:
```sql
-- Weak Entity ต้องพึ่งพา Strong Entity
-- ตัวอย่าง: ORDER_ITEM พึ่งพา ORDER

CREATE TABLE orders (
    order_id SERIAL PRIMARY KEY,
    customer_id INTEGER NOT NULL REFERENCES customers,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- ORDER_ITEM เป็น Weak Entity
-- มีได้ก็ต่อเมื่อมี ORDER
-- PK ประกอบด้วย order_id (จาก Owner) + item_seq
CREATE TABLE order_items (
    order_id    INTEGER NOT NULL REFERENCES orders ON DELETE CASCADE,
    item_seq    SMALLINT NOT NULL,
    product_id  INTEGER NOT NULL REFERENCES products,
    quantity    INTEGER NOT NULL,
    unit_price  DECIMAL(10,2) NOT NULL,
    PRIMARY KEY (order_id, item_seq)
);
```

### ข้อ 5
สร้าง ER Diagram (ASCII) สำหรับ Ternary Relationship: STUDENT, COURSE, INSTRUCTOR (ผู้สอน assign นักเรียนเรียนวิชา)

**เฉลย**:
```
  ┌──────────┐
  │ STUDENT  │
  └────┬─────┘
       │
       └──────────────────┐
                          ▼
  ┌──────────┐     ┌────────────────┐     ┌──────────────┐
  │  COURSE  │─────│  ASSIGNMENT   │─────│  INSTRUCTOR  │
  └──────────┘     │               │     └──────────────┘
                   │ student_id FK │
                   │ course_id FK  │
                   │ instructor_id │
                   │ semester      │
                   │ grade         │
                   └───────────────┘
```
```sql
CREATE TABLE course_assignments (
    student_id    INTEGER NOT NULL REFERENCES students,
    course_id     INTEGER NOT NULL REFERENCES courses,
    instructor_id INTEGER NOT NULL REFERENCES instructors,
    semester      VARCHAR(20) NOT NULL,
    grade         CHAR(2),
    PRIMARY KEY (student_id, course_id, instructor_id, semester)
);
```

### ข้อ 6
จงอธิบาย Crow's Foot Notation และวาดสัญลักษณ์สำหรับ:
a) "ทุก Order ต้องมี Customer (mandatory)" แต่ "Customer อาจไม่มี Order ก็ได้"

**เฉลย**:
```
CUSTOMER ──|──O<── ORDER

|  = exactly one (Customer ต้องมีอย่างน้อย 1 สำหรับ Order)
O< = zero or more (Customer มี 0 หรือมากกว่า Orders)

อ่านว่า: "Customer หนึ่งคนมี 0 หรือมากกว่า Orders"
         "Order หนึ่งอันต้องมี Customer อย่างน้อย 1 คน"

SQL:
ORDER.customer_id INTEGER NOT NULL REFERENCES customers
                  ↑ NOT NULL = mandatory (|)
```

### ข้อ 7
ออกแบบ ER Diagram สำหรับระบบ Banking ที่มี: ลูกค้า, บัญชี (ลูกค้าหนึ่งคนมีได้หลายบัญชี), รายการธุรกรรม

**เฉลย**:
```
CUSTOMER ──1:N── ACCOUNT ──1:N── TRANSACTION
  │
  └── ยังอาจมี M:N: CUSTOMER ──M:N── ACCOUNT (joint account)

SQL:
CREATE TABLE customers (
    customer_id SERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    id_number VARCHAR(20) UNIQUE
);

CREATE TABLE accounts (
    account_id   BIGSERIAL PRIMARY KEY,
    account_no   VARCHAR(20) UNIQUE NOT NULL,
    account_type VARCHAR(20) CHECK (account_type IN ('savings','checking','fixed')),
    balance      DECIMAL(15,2) NOT NULL DEFAULT 0 CHECK (balance >= 0),
    opened_at    TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    is_active    BOOLEAN DEFAULT TRUE
);

-- M:N for joint accounts
CREATE TABLE account_holders (
    customer_id  INTEGER NOT NULL REFERENCES customers,
    account_id   INTEGER NOT NULL REFERENCES accounts,
    role         VARCHAR(20) CHECK (role IN ('primary','joint')),
    PRIMARY KEY (customer_id, account_id)
);

CREATE TABLE transactions (
    txn_id       BIGSERIAL PRIMARY KEY,
    account_id   INTEGER NOT NULL REFERENCES accounts,
    txn_type     VARCHAR(20) NOT NULL,
    amount       DECIMAL(15,2) NOT NULL,
    balance_after DECIMAL(15,2) NOT NULL,
    created_at   TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    reference    VARCHAR(50)
);
```

### ข้อ 8
อธิบาย Self-Referencing Relationship พร้อมตัวอย่าง 3 แบบ

**เฉลย**:
```sql
-- 1. Employee manages Employee (Hierarchy)
CREATE TABLE employees (
    emp_id     SERIAL PRIMARY KEY,
    manager_id INTEGER REFERENCES employees(emp_id),
    name       VARCHAR(100) NOT NULL
);

-- 2. Category → Subcategory (Tree)
CREATE TABLE categories (
    category_id SERIAL PRIMARY KEY,
    parent_id   INTEGER REFERENCES categories(category_id),
    name        VARCHAR(100) NOT NULL
);

-- 3. User follows User (Many-to-Many)
CREATE TABLE user_follows (
    follower_id  INTEGER REFERENCES users(user_id),
    following_id INTEGER REFERENCES users(user_id),
    followed_at  TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (follower_id, following_id),
    CHECK (follower_id <> following_id)
);
```

### ข้อ 9
แปลง Composite Attribute ต่อไปนี้เป็น SQL: 
"Address = {house_no, street, district, city, province, postal_code}"

**เฉลย**:
```sql
-- Option 1: Flatten ใน table เดียวกัน (ถ้า Entity มี Address เดียว)
CREATE TABLE suppliers (
    supplier_id SERIAL PRIMARY KEY,
    name        VARCHAR(200) NOT NULL,
    -- address composite
    addr_house_no  VARCHAR(20),
    addr_street    VARCHAR(200),
    addr_district  VARCHAR(100),
    addr_city      VARCHAR(100),
    addr_province  VARCHAR(50),
    addr_postal    VARCHAR(10)
);

-- Option 2: แยกตาราง (ถ้า Entity มีได้หลาย Address)
CREATE TABLE addresses (
    address_id  SERIAL PRIMARY KEY,
    entity_type VARCHAR(20) NOT NULL,  -- 'customer', 'supplier', etc.
    entity_id   INTEGER NOT NULL,
    house_no    VARCHAR(20),
    street      VARCHAR(200),
    district    VARCHAR(100),
    city        VARCHAR(100),
    province    VARCHAR(50),
    postal_code VARCHAR(10),
    country     VARCHAR(50) DEFAULT 'TH',
    label       VARCHAR(50),  -- 'home', 'office', 'billing'
    is_default  BOOLEAN DEFAULT FALSE
);
```

### ข้อ 10
จงออกแบบ ER Diagram (ASCII) สำหรับระบบ Event Management ที่มี: Event, Venue, Organizer, Attendee, Ticket

**เฉลย**:
```
ER DIAGRAM: Event Management System

  ┌──────────────┐    organizes    ┌──────────────┐
  │  ORGANIZER   │────────1:N─────>│    EVENT     │
  │              │                 │              │
  │ org_id PK    │                 │ event_id PK  │
  │ name         │                 │ title        │
  │ email        │                 │ start_dt     │
  │ phone        │                 │ end_dt       │
  └──────────────┘                 │ venue_id FK  │
                                   │ org_id FK    │
  ┌──────────────┐    held_at      │ status       │
  │    VENUE     │────────N:1──────│              │
  │              │                 └──────┬───────┘
  │ venue_id PK  │                        │ has
  │ name         │                        │ 1:N
  │ address      │                        ▼
  │ capacity     │                 ┌──════════════╗
  └──────────────┘                 ║   TICKET     ║ (Weak)
                                   ║              ║
                                   ║ event_id FK  ║
  ┌──────────────┐    purchases    ║ ticket_seq   ║
  │  ATTENDEE    │────────1:N─────>║ attendee_id  ║
  │              │                 ║ type         ║
  │ attendee_id  │                 ║ price        ║
  │ name         │                 ║ is_used      ║
  │ email        │                 ╚══════════════╝
  └──────────────┘

SQL:
CREATE TABLE organizers (org_id SERIAL PK, name, email);
CREATE TABLE venues (venue_id SERIAL PK, name, address, capacity);
CREATE TABLE events (
    event_id SERIAL PK,
    org_id FK NOT NULL,
    venue_id FK NOT NULL,
    title VARCHAR NOT NULL,
    start_dt TIMESTAMP, end_dt TIMESTAMP
);
CREATE TABLE attendees (attendee_id SERIAL PK, name, email UNIQUE);
CREATE TABLE tickets (
    event_id FK NOT NULL,
    ticket_seq SMALLINT NOT NULL,
    attendee_id FK NOT NULL,
    ticket_type VARCHAR, price DECIMAL, is_used BOOLEAN,
    PRIMARY KEY (event_id, ticket_seq)
);
```

---

*จบ Part 052: Entity-Relationship Modeling*

**ในส่วนต่อไป (Part 053)**: เราจะเรียนรู้เรื่อง First Normal Form (1NF) อย่างละเอียด พร้อมตัวอย่างการแปลงข้อมูลให้อยู่ในรูปแบบที่ถูกต้อง
