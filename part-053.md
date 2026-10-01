# Part 053: First Normal Form (1NF)
# Normal Form ระดับแรก

---

## บทนำ: Normalization คืออะไร?

Database Normalization คือกระบวนการจัดโครงสร้างตารางในฐานข้อมูลเพื่อ:
1. **ลดการซ้ำซ้อนของข้อมูล** (Reduce Data Redundancy)
2. **เพิ่มความสอดคล้องของข้อมูล** (Improve Data Integrity)
3. **ทำให้ง่ายต่อการ Query** (Simplify Queries)
4. **รองรับการเปลี่ยนแปลง** (Support Updates)

Normal Forms มีระดับต่างๆ ดังนี้:
```
1NF (First Normal Form)   - พื้นฐาน
2NF (Second Normal Form)  - สร้างบน 1NF
3NF (Third Normal Form)   - สร้างบน 2NF
BCNF (Boyce-Codd NF)     - เข้มกว่า 3NF
4NF (Fourth Normal Form)  - Multivalued Dependencies
5NF (Fifth Normal Form)   - Join Dependencies
```

---

## 1NF คืออะไร?

**First Normal Form (1NF)** คือระดับพื้นฐานของ Normalization ที่ต้องการ:

```
กฎของ 1NF:
1. ทุก Column ต้องมีค่าเป็น Atomic (ไม่สามารถแบ่งย่อยได้อีก)
2. ทุก Column ต้องมีชนิดข้อมูลเดียวกัน (Single Type)
3. แต่ละ Column ต้องมีชื่อที่ไม่ซ้ำกัน
4. ลำดับการเก็บข้อมูลไม่มีความสำคัญ
5. ไม่มี Repeating Groups (กลุ่มคอลัมน์ซ้ำ)
6. ทุก Row สามารถระบุได้ด้วย Primary Key
```

---

## กฎที่ 1: Atomic Values (ค่าที่แบ่งย่อยไม่ได้)

### การละเมิดกฎ Atomic Values

```sql
-- ❌ ละเมิด 1NF: เก็บหลายค่าในคอลัมน์เดียว
CREATE TABLE student_courses (
    student_id   INTEGER,
    student_name VARCHAR(100),
    courses      VARCHAR(500)  -- "Math,Science,English" ← ไม่ Atomic!
);

-- ข้อมูล:
-- 1, สมชาย, "คณิตศาสตร์,วิทยาศาสตร์,ภาษาอังกฤษ"
-- 2, วิไล, "ประวัติศาสตร์,ศิลปะ"

-- ปัญหา:
-- 1. ค้นหานักเรียนที่เรียนวิชา 'คณิตศาสตร์' ยากมาก
-- 2. ไม่สามารถ JOIN กับตาราง courses ได้
-- 3. ข้อมูลอาจมีรูปแบบไม่สอดคล้อง (ช่องว่าง, ตัวพิมพ์ใหญ่/เล็ก)
-- 4. นับจำนวนวิชาต่อนักเรียนยาก
```

```sql
-- ✅ แก้ให้อยู่ใน 1NF
CREATE TABLE students (
    student_id   INTEGER PRIMARY KEY,
    student_name VARCHAR(100) NOT NULL
);

CREATE TABLE courses (
    course_id    INTEGER PRIMARY KEY,
    course_name  VARCHAR(100) NOT NULL
);

CREATE TABLE enrollments (
    student_id   INTEGER REFERENCES students,
    course_id    INTEGER REFERENCES courses,
    PRIMARY KEY (student_id, course_id)
);

-- ข้อมูล:
-- students: (1, 'สมชาย'), (2, 'วิไล')
-- courses: (1, 'คณิตศาสตร์'), (2, 'วิทยาศาสตร์'), ...
-- enrollments: (1,1), (1,2), (1,3), (2,4), (2,5)
```

---

## กฎที่ 2: ไม่มี Repeating Groups

### ตัวอย่าง Repeating Groups

```sql
-- ❌ ละเมิด 1NF: Repeating Groups (คอลัมน์ที่มีโครงสร้างซ้ำ)
CREATE TABLE orders_bad (
    order_id     INTEGER,
    customer_id  INTEGER,
    product1_id  INTEGER,    -- ← Repeating Group!
    product1_qty INTEGER,
    product1_price DECIMAL,
    product2_id  INTEGER,    -- ← Repeating Group!
    product2_qty INTEGER,
    product2_price DECIMAL,
    product3_id  INTEGER,    -- ← Repeating Group!
    product3_qty INTEGER,
    product3_price DECIMAL
    -- จะทำอย่างไรถ้า order มี 10 สินค้า?
);
```

```sql
-- ✅ แก้ให้อยู่ใน 1NF: แยกตาราง
CREATE TABLE orders (
    order_id    INTEGER PRIMARY KEY,
    customer_id INTEGER NOT NULL,
    order_date  DATE NOT NULL
);

CREATE TABLE order_items (
    order_id    INTEGER REFERENCES orders,
    item_seq    INTEGER,
    product_id  INTEGER NOT NULL,
    quantity    INTEGER NOT NULL,
    unit_price  DECIMAL(10,2) NOT NULL,
    PRIMARY KEY (order_id, item_seq)
);
```

---

## ตัวอย่างที่ 1: ตารางนักเรียนกับวิชาเลือก

```sql
-- ❌ BEFORE (ละเมิด 1NF)
CREATE TABLE student_electives_bad (
    student_id    INTEGER,
    student_name  VARCHAR(100),
    elective1     VARCHAR(50),
    elective2     VARCHAR(50),
    elective3     VARCHAR(50)
);

INSERT INTO student_electives_bad VALUES
(1, 'สมชาย', 'ดนตรี', 'กีฬา', NULL),
(2, 'วิไล', 'ศิลปะ', NULL, NULL),
(3, 'ประสิทธิ์', 'ดนตรี', 'ศิลปะ', 'กีฬา');
```

```sql
-- ✅ AFTER (1NF)
CREATE TABLE students (
    student_id    INTEGER PRIMARY KEY,
    student_name  VARCHAR(100) NOT NULL
);

CREATE TABLE elective_courses (
    course_id   INTEGER PRIMARY KEY,
    course_name VARCHAR(50) NOT NULL UNIQUE
);

CREATE TABLE student_electives (
    student_id  INTEGER REFERENCES students,
    course_id   INTEGER REFERENCES elective_courses,
    PRIMARY KEY (student_id, course_id)
);

-- ข้อดี: ค้นหาได้ง่าย
SELECT s.student_name
FROM students s
JOIN student_electives se ON s.student_id = se.student_id
JOIN elective_courses ec ON se.course_id = ec.course_id
WHERE ec.course_name = 'ดนตรี';
```

---

## ตัวอย่างที่ 2: ตารางสินค้ากับ Tags

```sql
-- ❌ BEFORE (ละเมิด 1NF)
CREATE TABLE products_bad (
    product_id   INTEGER,
    name         VARCHAR(200),
    tags         VARCHAR(500),  -- "electronics,sale,new,featured"
    category_ids VARCHAR(200)   -- "1,5,12"
);
```

```sql
-- ✅ AFTER (1NF)
CREATE TABLE products (
    product_id  INTEGER PRIMARY KEY,
    name        VARCHAR(200) NOT NULL
);

CREATE TABLE tags (
    tag_id   INTEGER PRIMARY KEY,
    tag_name VARCHAR(50) UNIQUE NOT NULL
);

CREATE TABLE product_tags (
    product_id  INTEGER REFERENCES products,
    tag_id      INTEGER REFERENCES tags,
    PRIMARY KEY (product_id, tag_id)
);

CREATE TABLE categories (
    category_id INTEGER PRIMARY KEY,
    name        VARCHAR(100) NOT NULL
);

CREATE TABLE product_categories (
    product_id  INTEGER REFERENCES products,
    category_id INTEGER REFERENCES categories,
    PRIMARY KEY (product_id, category_id)
);
```

---

## ตัวอย่างที่ 3: ข้อมูลที่อยู่รวมกัน

```sql
-- ❌ BEFORE (ละเมิด 1NF)
CREATE TABLE employees_bad (
    emp_id      INTEGER,
    name        VARCHAR(100),
    full_address TEXT,  -- "123 ถ.สุขุมวิท แขวงคลองเตย เขตคลองเตย กรุงเทพฯ 10110"
    phone_list  VARCHAR(200)  -- "0891234567,0812345678"
);
```

```sql
-- ✅ AFTER (1NF)
CREATE TABLE employees (
    emp_id      INTEGER PRIMARY KEY,
    name        VARCHAR(100) NOT NULL
);

-- Composite address → atomic columns
CREATE TABLE employee_addresses (
    address_id  SERIAL PRIMARY KEY,
    emp_id      INTEGER REFERENCES employees,
    street      VARCHAR(200),
    subdistrict VARCHAR(100),
    district    VARCHAR(100),
    province    VARCHAR(50),
    postal_code VARCHAR(10),
    address_type VARCHAR(20) DEFAULT 'home'
);

-- Multivalued phones → separate table
CREATE TABLE employee_phones (
    emp_id       INTEGER REFERENCES employees,
    phone_number VARCHAR(20),
    phone_type   VARCHAR(20),
    PRIMARY KEY (emp_id, phone_number)
);
```

---

## ตัวอย่างที่ 4: ตารางสต็อกสินค้าหลายคลัง

```sql
-- ❌ BEFORE (ละเมิด 1NF): Repeating Groups สำหรับคลัง
CREATE TABLE product_stock_bad (
    product_id     INTEGER,
    product_name   VARCHAR(200),
    warehouse1_qty INTEGER,
    warehouse2_qty INTEGER,
    warehouse3_qty INTEGER,
    warehouse4_qty INTEGER,
    warehouse5_qty INTEGER
);
-- ปัญหา: ถ้ามีคลังที่ 6 ต้อง ALTER TABLE
```

```sql
-- ✅ AFTER (1NF)
CREATE TABLE products (
    product_id   INTEGER PRIMARY KEY,
    product_name VARCHAR(200) NOT NULL
);

CREATE TABLE warehouses (
    warehouse_id   INTEGER PRIMARY KEY,
    warehouse_name VARCHAR(100) NOT NULL,
    location       VARCHAR(200)
);

CREATE TABLE inventory (
    product_id   INTEGER REFERENCES products,
    warehouse_id INTEGER REFERENCES warehouses,
    quantity     INTEGER NOT NULL DEFAULT 0,
    PRIMARY KEY (product_id, warehouse_id)
);
```

---

## ตัวอย่างที่ 5: ใบแจ้งหนี้ (Invoice)

```sql
-- ❌ BEFORE (ละเมิด 1NF)
CREATE TABLE invoices_bad (
    invoice_no   VARCHAR(20),
    customer     VARCHAR(200),
    item1_name   VARCHAR(100),
    item1_qty    INTEGER,
    item1_price  DECIMAL,
    item2_name   VARCHAR(100),
    item2_qty    INTEGER,
    item2_price  DECIMAL,
    item3_name   VARCHAR(100),
    item3_qty    INTEGER,
    item3_price  DECIMAL
);
```

```sql
-- ✅ AFTER (1NF)
CREATE TABLE invoices (
    invoice_id   SERIAL PRIMARY KEY,
    invoice_no   VARCHAR(20) UNIQUE NOT NULL,
    customer_id  INTEGER NOT NULL,
    invoice_date DATE NOT NULL,
    due_date     DATE
);

CREATE TABLE invoice_lines (
    invoice_id  INTEGER REFERENCES invoices,
    line_no     INTEGER NOT NULL,
    item_name   VARCHAR(100) NOT NULL,
    quantity    INTEGER NOT NULL,
    unit_price  DECIMAL(10,2) NOT NULL,
    PRIMARY KEY (invoice_id, line_no)
);
```

---

## ตัวอย่างที่ 6: ตารางพนักงานกับการฝึกอบรม

```sql
-- ❌ BEFORE (ละเมิด 1NF)
CREATE TABLE employee_training_bad (
    emp_id        INTEGER,
    emp_name      VARCHAR(100),
    training_dates VARCHAR(300),   -- "2024-01-15,2024-03-20,2024-06-10"
    training_types VARCHAR(300)    -- "safety,leadership,technical"
);
```

```sql
-- ✅ AFTER (1NF)
CREATE TABLE employees (
    emp_id   INTEGER PRIMARY KEY,
    emp_name VARCHAR(100) NOT NULL
);

CREATE TABLE training_types (
    type_id   INTEGER PRIMARY KEY,
    type_name VARCHAR(100) NOT NULL
);

CREATE TABLE employee_trainings (
    emp_id        INTEGER REFERENCES employees,
    training_date DATE NOT NULL,
    type_id       INTEGER REFERENCES training_types,
    trainer       VARCHAR(100),
    score         DECIMAL(5,2),
    PRIMARY KEY (emp_id, training_date, type_id)
);
```

---

## ตัวอย่างที่ 7: ระบบจอง (Booking System)

```sql
-- ❌ BEFORE (ละเมิด 1NF)
CREATE TABLE bookings_bad (
    booking_id   INTEGER,
    guest_name   VARCHAR(100),
    room_numbers VARCHAR(100),  -- "101,102,103" - จองหลายห้อง
    services     VARCHAR(200)   -- "breakfast,gym,spa"
);
```

```sql
-- ✅ AFTER (1NF)
CREATE TABLE bookings (
    booking_id   INTEGER PRIMARY KEY,
    guest_id     INTEGER NOT NULL,
    check_in     DATE NOT NULL,
    check_out    DATE NOT NULL
);

CREATE TABLE booking_rooms (
    booking_id   INTEGER REFERENCES bookings,
    room_id      INTEGER REFERENCES rooms,
    PRIMARY KEY (booking_id, room_id)
);

CREATE TABLE services (
    service_id   INTEGER PRIMARY KEY,
    service_name VARCHAR(100) NOT NULL,
    price        DECIMAL(10,2)
);

CREATE TABLE booking_services (
    booking_id   INTEGER REFERENCES bookings,
    service_id   INTEGER REFERENCES services,
    quantity     INTEGER DEFAULT 1,
    PRIMARY KEY (booking_id, service_id)
);
```

---

## ตัวอย่างที่ 8: ตาราง Legacy Database (ระบบเก่า)

```sql
-- ❌ ตาราง Legacy ที่พบบ่อยในระบบเก่า
CREATE TABLE sales_legacy (
    sale_id    CHAR(10),
    cust_info  VARCHAR(300),  -- "John Doe|john@email.com|0891234567"
    prod_list  TEXT,           -- "P001:2:100.00,P002:1:250.00,P003:3:75.00"
    sale_date  CHAR(8),        -- "20240115" (YYYYMMDD)
    total      CHAR(10)        -- "725.00"
);
```

```sql
-- ✅ แปลงเป็น 1NF
-- ขั้นตอนที่ 1: สร้างตารางใหม่
CREATE TABLE customers (
    customer_id SERIAL PRIMARY KEY,
    full_name   VARCHAR(100) NOT NULL,
    email       VARCHAR(255) UNIQUE,
    phone       VARCHAR(20)
);

CREATE TABLE products (
    product_id  SERIAL PRIMARY KEY,
    product_code VARCHAR(20) UNIQUE NOT NULL,
    name        VARCHAR(200) NOT NULL,
    unit_price  DECIMAL(10,2) NOT NULL
);

CREATE TABLE sales (
    sale_id     VARCHAR(10) PRIMARY KEY,
    customer_id INTEGER NOT NULL REFERENCES customers,
    sale_date   DATE NOT NULL,
    total_amount DECIMAL(10,2) NOT NULL
);

CREATE TABLE sale_items (
    sale_id    VARCHAR(10) REFERENCES sales,
    product_id INTEGER REFERENCES products,
    quantity   INTEGER NOT NULL,
    unit_price DECIMAL(10,2) NOT NULL,
    PRIMARY KEY (sale_id, product_id)
);

-- ขั้นตอนที่ 2: Migrate ข้อมูลจาก Legacy (ตัวอย่าง Logic)
-- (ในทางปฏิบัติต้องใช้ Script แปลงข้อมูล)
```

---

## ตัวอย่างที่ 9: ตารางที่อยู่จัดส่ง

```sql
-- ❌ BEFORE (ละเมิด 1NF)
CREATE TABLE orders_bad (
    order_id       INTEGER,
    customer_name  VARCHAR(100),
    delivery_info  TEXT  -- "สมชาย ใจดี, 123 ถ.สุขุมวิท, คลองเตย, กรุงเทพ, 10110, 0891234567"
);
```

```sql
-- ✅ AFTER (1NF)
CREATE TABLE orders (
    order_id       INTEGER PRIMARY KEY,
    customer_id    INTEGER NOT NULL,
    recipient_name VARCHAR(100) NOT NULL,
    street         VARCHAR(200) NOT NULL,
    district       VARCHAR(100),
    city           VARCHAR(100) NOT NULL,
    postal_code    VARCHAR(10) NOT NULL,
    phone          VARCHAR(20)
);
```

---

## ตัวอย่างที่ 10: ตารางผลการสอบ

```sql
-- ❌ BEFORE (ละเมิด 1NF)
CREATE TABLE exam_results_bad (
    student_id  INTEGER,
    name        VARCHAR(100),
    math_q1     CHAR(1),  -- A, B, C, D
    math_q2     CHAR(1),
    math_q3     CHAR(1),
    math_q4     CHAR(1),
    math_q5     CHAR(1),
    sci_q1      CHAR(1),
    sci_q2      CHAR(1),
    sci_q3      CHAR(1),
    sci_q4      CHAR(1),
    sci_q5      CHAR(1)
    -- Repeating pattern สำหรับแต่ละวิชา
);
```

```sql
-- ✅ AFTER (1NF)
CREATE TABLE students (
    student_id  INTEGER PRIMARY KEY,
    name        VARCHAR(100) NOT NULL
);

CREATE TABLE subjects (
    subject_id  INTEGER PRIMARY KEY,
    name        VARCHAR(50) NOT NULL
);

CREATE TABLE questions (
    question_id INTEGER PRIMARY KEY,
    subject_id  INTEGER REFERENCES subjects,
    question_no INTEGER NOT NULL,
    question_text TEXT,
    correct_answer CHAR(1),
    UNIQUE (subject_id, question_no)
);

CREATE TABLE exam_answers (
    student_id  INTEGER REFERENCES students,
    question_id INTEGER REFERENCES questions,
    answer      CHAR(1),
    is_correct  BOOLEAN,
    PRIMARY KEY (student_id, question_id)
);

-- Query: คะแนนของนักเรียนแต่ละคนต่อวิชา
SELECT
    s.name,
    sub.name AS subject,
    SUM(CASE WHEN ea.is_correct THEN 1 ELSE 0 END) AS score,
    COUNT(*) AS total_questions
FROM students s
JOIN exam_answers ea ON s.student_id = ea.student_id
JOIN questions q ON ea.question_id = q.question_id
JOIN subjects sub ON q.subject_id = sub.subject_id
GROUP BY s.name, sub.name;
```

---

## เมื่อใดที่การละเมิด 1NF อาจยอมรับได้?

### กรณีที่ 1: Arrays ใน PostgreSQL

PostgreSQL รองรับ Array Type ซึ่งเทคนิคอาจละเมิด 1NF แต่มีประโยชน์ในบางกรณี

```sql
-- PostgreSQL Array Type
CREATE TABLE articles (
    article_id  SERIAL PRIMARY KEY,
    title       VARCHAR(300) NOT NULL,
    tags        TEXT[] NOT NULL DEFAULT '{}',  -- Array
    keywords    TEXT[]
);

-- เพิ่มข้อมูล
INSERT INTO articles (title, tags) VALUES
('SQL Tutorial', ARRAY['sql', 'database', 'tutorial']),
('Python Tips', ARRAY['python', 'programming', 'tips']);

-- Query ที่ใช้ Array
-- ค้นหาบทความที่มี tag 'sql'
SELECT title FROM articles
WHERE 'sql' = ANY(tags);

-- ค้นหาบทความที่มีทั้ง 'sql' และ 'tutorial'
SELECT title FROM articles
WHERE tags @> ARRAY['sql', 'tutorial'];

-- นับจำนวน tags
SELECT title, array_length(tags, 1) AS tag_count
FROM articles;
```

**เมื่อใดที่ควรใช้ Array แทนตารางแยก:**
- ข้อมูลเป็น "container" ไม่ใช่ entity (เช่น list ของ keywords)
- ไม่จำเป็นต้อง JOIN หรือ Query แยก
- ลำดับสำคัญ
- ต้องการ Performance ดีกว่า JOIN

### กรณีที่ 2: JSON/JSONB

```sql
-- JSON สำหรับข้อมูลที่มีโครงสร้างหลากหลาย
CREATE TABLE products (
    product_id   SERIAL PRIMARY KEY,
    name         VARCHAR(200) NOT NULL,
    attributes   JSONB DEFAULT '{}'  -- specifications ต่างๆ
);

INSERT INTO products (name, attributes) VALUES
('iPhone 15', '{
    "color": ["black", "white", "gold"],
    "storage": "128GB",
    "ram": "6GB",
    "camera": "48MP"
}'),
('Running Shoes', '{
    "sizes": [38, 39, 40, 41, 42, 43, 44],
    "material": "mesh",
    "waterproof": false
}');

-- Query JSONB
SELECT name, attributes->>'storage' AS storage
FROM products
WHERE attributes->>'storage' IS NOT NULL;

-- ค้นหาสินค้าที่มีสีดำ
SELECT name FROM products
WHERE attributes->'color' ? 'black';

-- JSONB เหมาะสำหรับ:
-- - Attributes ที่แตกต่างกันในแต่ละ Record (EAV alternative)
-- - Semi-structured data
-- - Rapid development
```

### กรณีที่ 3: ข้อมูล Time-Series

```sql
-- บางครั้ง denormalized format เหมาะกว่าสำหรับ Time-Series
CREATE TABLE daily_sales_summary (
    product_id      INTEGER,
    month           DATE,  -- 2024-01-01 = January 2024
    day_01_sales    DECIMAL(10,2),
    day_02_sales    DECIMAL(10,2),
    -- ... ถึง day_31
    day_31_sales    DECIMAL(10,2),
    PRIMARY KEY (product_id, month)
);
-- เหมาะสำหรับ OLAP/Reporting แต่ไม่ใช่ OLTP
```

---

## การตรวจสอบ 1NF Violations

### Checklist สำหรับตรวจสอบ 1NF

```sql
-- 1. ตรวจหา Columns ที่เก็บหลายค่า
-- ดู Column ที่มีชื่อแบบ: list, array, csv, items, etc.

-- 2. ตรวจหา Repeating Groups
-- ดู Column ที่มีชื่อแบบ: item1, item2, item3 หรือ product_1, product_2

-- 3. ตรวจสอบ Data Type
-- VARCHAR ที่เก็บตัวเลขหลายตัวคั่นด้วยคอมมา

-- Example: ค้นหา Rows ที่มีคอมมาในคอลัมน์ที่ไม่ควรมี
SELECT *
FROM products_bad
WHERE tags LIKE '%,%';

-- ตรวจหา Repeating Group Pattern ใน System Catalog
SELECT column_name
FROM information_schema.columns
WHERE table_name = 'my_table'
AND (column_name ~ '^.*[0-9]$'  -- ลงท้ายด้วยตัวเลข
     OR column_name ~ '^.*_[0-9]+$');  -- มีตัวเลขหลัง underscore
```

---

## กระบวนการแปลงเป็น 1NF

### ขั้นตอนการแปลง

```
Step 1: ระบุ Violations
- หา Columns ที่มีหลายค่า
- หา Repeating Groups
- หา Non-Atomic Values

Step 2: แยก Multivalued Attributes
- สร้างตารางใหม่สำหรับค่าที่มีได้หลายตัว
- ใช้ FK เชื่อมกับตารางเดิม
- กำหนด PK ที่เหมาะสม

Step 3: แก้ Repeating Groups
- แยก "กลุ่ม" ออกเป็นตารางใหม่
- เพิ่ม Sequence/Number Column ถ้าจำเป็น

Step 4: กำหนด Primary Key
- ทุกตารางต้องมี PK
- PK ต้อง Unique และ Not Null

Step 5: ตรวจสอบผล
- ทุก Column มีค่าเป็น Atomic
- ไม่มี Repeating Groups
- ทุก Row ระบุได้ด้วย PK
```

---

## ตัวอย่าง Real-World: แปลง Excel เป็น 1NF

### Excel Sheet ที่พบบ่อย

```
Excel: รายงานการขาย

| วันที่    | พนักงาน  | สินค้าที่ขาย          | ยอดขาย |
|-----------|----------|----------------------|--------|
| 01/01/24  | สมชาย   | A001:5, B002:3, C003:1 | 2500   |
| 01/01/24  | วิไล    | A001:2, D004:4         | 1200   |
| 02/01/24  | สมชาย   | B002:6                 | 1800   |
```

```sql
-- ❌ ถ้าเก็บตรงๆ จาก Excel
CREATE TABLE sales_report_bad (
    sale_date    DATE,
    salesperson  VARCHAR(100),
    items_sold   TEXT,  -- "A001:5, B002:3, C003:1"
    total_amount DECIMAL
);

-- ✅ แปลงเป็น 1NF
CREATE TABLE salespeople (
    sp_id    SERIAL PRIMARY KEY,
    name     VARCHAR(100) NOT NULL
);

CREATE TABLE products (
    product_code VARCHAR(10) PRIMARY KEY,
    name         VARCHAR(200),
    unit_price   DECIMAL(10,2)
);

CREATE TABLE sales (
    sale_id     SERIAL PRIMARY KEY,
    sale_date   DATE NOT NULL,
    sp_id       INTEGER NOT NULL REFERENCES salespeople
);

CREATE TABLE sale_items (
    sale_id      INTEGER REFERENCES sales,
    product_code VARCHAR(10) REFERENCES products,
    quantity     INTEGER NOT NULL,
    PRIMARY KEY (sale_id, product_code)
);

-- Query: รายงานยอดขายรายคน
SELECT
    sp.name,
    s.sale_date,
    SUM(si.quantity * p.unit_price) AS total_sales
FROM salespeople sp
JOIN sales s ON sp.sp_id = s.sp_id
JOIN sale_items si ON s.sale_id = si.sale_id
JOIN products p ON si.product_code = p.product_code
GROUP BY sp.name, s.sale_date
ORDER BY s.sale_date, sp.name;
```

---

## ตัวอย่างเพิ่มเติม: Contact Information

```sql
-- ❌ ละเมิด 1NF: เก็บข้อมูล Contact หลายช่องทาง
CREATE TABLE contacts_bad (
    contact_id   INTEGER,
    name         VARCHAR(100),
    contact_info TEXT  -- "email:john@example.com;phone:0891234567;line:@johndoe"
);

-- ✅ 1NF: แยก Contact Methods
CREATE TABLE contacts (
    contact_id  INTEGER PRIMARY KEY,
    name        VARCHAR(100) NOT NULL
);

CREATE TABLE contact_methods (
    method_id    SERIAL PRIMARY KEY,
    contact_id   INTEGER REFERENCES contacts,
    method_type  VARCHAR(20) CHECK (method_type IN ('email','phone','line','facebook','instagram')),
    method_value VARCHAR(200) NOT NULL,
    is_primary   BOOLEAN DEFAULT FALSE
);
```

---

## ตัวอย่างที่ 11: ตาราง Skills ของพนักงาน

```sql
-- ❌ BEFORE
CREATE TABLE employee_skills_bad (
    emp_id  INTEGER,
    skills  VARCHAR(500)  -- "Python,JavaScript,SQL,Docker,AWS"
);

-- ✅ AFTER
CREATE TABLE employees (
    emp_id   INTEGER PRIMARY KEY,
    name     VARCHAR(100) NOT NULL
);

CREATE TABLE skill_catalog (
    skill_id   INTEGER PRIMARY KEY,
    skill_name VARCHAR(100) UNIQUE NOT NULL,
    category   VARCHAR(50)  -- 'programming','database','cloud','soft'
);

CREATE TABLE employee_skills (
    emp_id      INTEGER REFERENCES employees,
    skill_id    INTEGER REFERENCES skill_catalog,
    proficiency VARCHAR(20) CHECK (proficiency IN ('beginner','intermediate','advanced','expert')),
    since_year  SMALLINT,
    PRIMARY KEY (emp_id, skill_id)
);

-- Query: หาพนักงานที่รู้ Python ระดับ advanced ขึ้นไป
SELECT e.name, sc.skill_name, es.proficiency
FROM employees e
JOIN employee_skills es ON e.emp_id = es.emp_id
JOIN skill_catalog sc ON es.skill_id = sc.skill_id
WHERE sc.skill_name = 'Python'
AND es.proficiency IN ('advanced', 'expert');
```

---

## ตัวอย่างที่ 12: ตารางประวัติการรักษา

```sql
-- ❌ BEFORE (ระบบ Legacy ของโรงพยาบาล)
CREATE TABLE patient_visits_bad (
    visit_id     INTEGER,
    patient_name VARCHAR(100),
    visit_date   DATE,
    diagnoses    TEXT,    -- "J00,J06.9,Z23"  (ICD codes คั่นด้วยคอมมา)
    medicines    TEXT,    -- "Paracetamol 500mg;Amoxicillin 250mg"
    vital_signs  TEXT     -- "BP:120/80;HR:75;Temp:37.2;SpO2:98"
);

-- ✅ AFTER
CREATE TABLE patients (
    patient_id  SERIAL PRIMARY KEY,
    name        VARCHAR(100) NOT NULL,
    dob         DATE,
    gender      CHAR(1)
);

CREATE TABLE visits (
    visit_id    SERIAL PRIMARY KEY,
    patient_id  INTEGER NOT NULL REFERENCES patients,
    visit_date  TIMESTAMP NOT NULL,
    doctor_id   INTEGER,
    chief_complaint TEXT
);

CREATE TABLE diagnoses (
    diagnosis_id SERIAL PRIMARY KEY,
    visit_id     INTEGER NOT NULL REFERENCES visits,
    icd_code     VARCHAR(10) NOT NULL,
    description  TEXT,
    is_primary   BOOLEAN DEFAULT FALSE
);

CREATE TABLE prescriptions (
    rx_id        SERIAL PRIMARY KEY,
    visit_id     INTEGER NOT NULL REFERENCES visits,
    medicine     VARCHAR(200) NOT NULL,
    dosage       VARCHAR(100),
    frequency    VARCHAR(50),
    duration     VARCHAR(50)
);

CREATE TABLE vital_signs (
    vital_id    SERIAL PRIMARY KEY,
    visit_id    INTEGER NOT NULL REFERENCES visits,
    sign_type   VARCHAR(30) CHECK (sign_type IN ('BP_systolic','BP_diastolic','HR','Temp','SpO2','RR','Weight','Height')),
    value       DECIMAL(6,2) NOT NULL,
    unit        VARCHAR(20),
    recorded_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);
```

---

## สรุปกฎของ 1NF

```
1NF Checklist:
□ ทุก Column มีค่าที่ Atomic (ไม่ใช่ List หรือ Set)
□ ไม่มี Repeating Groups (column_1, column_2, column_3)
□ ทุก Column มีชนิดข้อมูลเดียวสม่ำเสมอ
□ ทุก Row ระบุได้ด้วย Primary Key
□ ลำดับ Row และ Column ไม่มีความสำคัญในตัวเอง

สิ่งที่ต้องแก้:
- "tags" VARCHAR → แยกตาราง tags + junction table
- "phone1,phone2,phone3" → แยกตาราง phones
- "item1_name, item2_name, item3_name" → แยกตาราง items
- "address" TEXT → แยก street, city, zip columns

ข้อยกเว้น (ควรใช้อย่างระมัดระวัง):
- PostgreSQL ARRAY สำหรับ simple lists
- JSONB สำหรับ semi-structured data
- Time-series denormalized tables
```

---

## แบบฝึกหัด (10 ข้อ)

### ข้อ 1
จงชี้ว่าตารางนี้ละเมิด 1NF อย่างไร และแก้ไข:
```sql
CREATE TABLE employee_benefits (
    emp_id      INTEGER,
    name        VARCHAR(100),
    benefits    VARCHAR(500)  -- "health_insurance,dental,vision,gym"
);
```

**เฉลย**:
```
ละเมิด 1NF: benefits เก็บหลายค่าในคอลัมน์เดียว (ไม่ Atomic)

แก้ไข:
CREATE TABLE employees (emp_id INTEGER PK, name VARCHAR);
CREATE TABLE benefit_types (benefit_id INTEGER PK, name VARCHAR UNIQUE);
CREATE TABLE employee_benefits (
    emp_id INTEGER REFERENCES employees,
    benefit_id INTEGER REFERENCES benefit_types,
    start_date DATE,
    PRIMARY KEY (emp_id, benefit_id)
);
```

### ข้อ 2
แปลงตารางนี้ให้อยู่ใน 1NF:
```sql
CREATE TABLE class_schedule (
    class_id   INTEGER,
    class_name VARCHAR(100),
    mon_time   VARCHAR(20),
    tue_time   VARCHAR(20),
    wed_time   VARCHAR(20),
    thu_time   VARCHAR(20),
    fri_time   VARCHAR(20)
);
```

**เฉลย**:
```sql
CREATE TABLE classes (
    class_id   INTEGER PRIMARY KEY,
    class_name VARCHAR(100) NOT NULL
);

CREATE TABLE class_sessions (
    session_id   SERIAL PRIMARY KEY,
    class_id     INTEGER REFERENCES classes,
    day_of_week  CHAR(3) CHECK (day_of_week IN ('Mon','Tue','Wed','Thu','Fri')),
    start_time   TIME NOT NULL,
    end_time     TIME,
    UNIQUE (class_id, day_of_week)
);
```

### ข้อ 3
ข้อมูลใดในตารางนี้ละเมิด 1NF:
```sql
CREATE TABLE projects (
    project_id    INTEGER,
    name          VARCHAR(200),
    team_members  TEXT,    -- "Alice,Bob,Charlie"
    tech_stack    TEXT,    -- "Python,PostgreSQL,Docker"
    milestones    TEXT     -- "2024-01-01:Design,2024-03-01:Development"
);
```

**เฉลย**:
```
ละเมิด 1NF ทั้ง 3 columns:
1. team_members - Multivalued (หลายคน)
2. tech_stack - Multivalued (หลายเทคโนโลยี)
3. milestones - Multivalued + Composite (วันที่ + ชื่อ)

แก้ไข: แยกเป็น 3 ตาราง:
- project_members (project_id FK, member_id FK)
- project_technologies (project_id FK, tech_id FK)
- project_milestones (project_id FK, milestone_date DATE, description TEXT)
```

### ข้อ 4
อธิบายว่า Repeating Groups คืออะไร และให้ตัวอย่าง

**เฉลย**:
```
Repeating Groups คือกลุ่มของ Columns ที่มีโครงสร้างซ้ำกัน
โดยปกติตั้งชื่อด้วยตัวเลขหรือ suffix ที่เพิ่มขึ้น

ตัวอย่าง:
phone1, phone2, phone3 ← Repeating Group
item1_name, item2_name, item3_name ← Repeating Group
q1_answer, q2_answer, q3_answer ← Repeating Group

วิธีแก้: แยกเป็นตารางใหม่
CREATE TABLE order_items (
    order_id INTEGER REFERENCES orders,
    item_seq INTEGER,
    -- item attributes
    PRIMARY KEY (order_id, item_seq)
);
```

### ข้อ 5
เมื่อใดที่ควรใช้ PostgreSQL Array แทนการแยกตาราง?

**เฉลย**:
```
ควรใช้ Array เมื่อ:
1. ข้อมูลเป็นเพียง list/container ไม่ใช่ Entity ที่มีตัวตน
2. ไม่จำเป็นต้อง JOIN กับตารางอื่น
3. Query ส่วนใหญ่ใช้ทั้ง Array พร้อมกัน
4. ลำดับของ Elements มีความสำคัญ
5. Elements เป็น Simple Values (string, int)

ควรแยกตาราง เมื่อ:
1. ต้องการ Query/Filter ตาม Element
2. Element มี Attributes เพิ่มเติม
3. ต้องการ Referential Integrity
4. จำนวน Elements มาก/ไม่แน่นอน

ตัวอย่าง Array ที่เหมาะสม:
tags TEXT[], keywords TEXT[], allowed_ips INET[]
```

### ข้อ 6
แปลง Legacy Table นี้ให้อยู่ใน 1NF:
```sql
CREATE TABLE student_grades_legacy (
    student_no VARCHAR(10),
    student_name VARCHAR(100),
    semester VARCHAR(10),
    grades VARCHAR(500)  -- "MATH:A,SCI:B+,ENG:A-,HIST:B,ART:A"
);
```

**เฉลย**:
```sql
CREATE TABLE students (
    student_no   VARCHAR(10) PRIMARY KEY,
    student_name VARCHAR(100) NOT NULL
);

CREATE TABLE subjects (
    subject_code VARCHAR(10) PRIMARY KEY,
    subject_name VARCHAR(100) NOT NULL
);

CREATE TABLE semesters (
    semester_id SERIAL PRIMARY KEY,
    semester    VARCHAR(10) NOT NULL,
    year        SMALLINT NOT NULL,
    UNIQUE (semester, year)
);

CREATE TABLE grade_records (
    student_no   VARCHAR(10) REFERENCES students,
    semester_id  INTEGER REFERENCES semesters,
    subject_code VARCHAR(10) REFERENCES subjects,
    grade        VARCHAR(3) CHECK (grade IN ('A','B+','B','C+','C','D+','D','F')),
    PRIMARY KEY (student_no, semester_id, subject_code)
);
```

### ข้อ 7
ให้ตัวอย่าง Derived Attribute และอธิบายว่าควรเก็บหรือไม่เก็บในตาราง

**เฉลย**:
```
Derived Attributes ที่ไม่ควรเก็บ (คำนวณจาก Columns ที่มีอยู่):
- age (คำนวณจาก birth_date)
- full_name (คำนวณจาก first_name + last_name)
- order_total (คำนวณจาก sum of items)
- years_of_service (คำนวณจาก hire_date)

เหตุผล: เก็บแล้วอาจเกิด Inconsistency ถ้าข้อมูลต้นทางเปลี่ยน

วิธีแก้: ใช้ VIEW หรือ Generated Column
CREATE VIEW order_summary AS
SELECT order_id, SUM(quantity * unit_price) AS total
FROM order_items GROUP BY order_id;

-- หรือ Generated Column (ข้อมูลอัพเดทอัตโนมัติ)
ALTER TABLE employees
ADD COLUMN full_name VARCHAR(100) GENERATED ALWAYS AS (first_name || ' ' || last_name) STORED;

ข้อยกเว้น: เก็บ Derived Value เมื่อ:
- คำนวณแพงมาก (pre-compute)
- ต้องการ snapshot (เช่น ราคา ณ วันที่ซื้อ)
```

### ข้อ 8
อธิบายความแตกต่างระหว่าง Composite Attribute และ Multivalued Attribute

**เฉลย**:
```
Composite Attribute:
- ประกอบด้วย Sub-attributes
- แต่ละ Instance มีค่าเดียว (just structured)
- ตัวอย่าง: address = {street, city, zip}
- แปลง: แยกเป็นหลาย Columns ในตารางเดียวกัน

Multivalued Attribute:
- มีได้หลายค่าสำหรับ Entity Instance เดียว
- ตัวอย่าง: phone = {0891234567, 0812345678}
- แปลง: แยกเป็นตารางใหม่ + FK

ตัวอย่างรวม:
phones = [{type:mobile, number:089...}, {type:work, number:02...}]
← Both: Multivalued + Composite

แปลง:
CREATE TABLE phones (
    entity_id INTEGER,
    phone_type VARCHAR(20),
    phone_number VARCHAR(20),
    PRIMARY KEY (entity_id, phone_number)
);
```

### ข้อ 9
แปลง Schema ของระบบ Library Catalog ให้อยู่ใน 1NF:
```
BOOK: isbn, title, authors(multiple), subjects(multiple), editions(year:publisher:pages)
```

**เฉลย**:
```sql
CREATE TABLE books (
    isbn    VARCHAR(13) PRIMARY KEY,
    title   VARCHAR(300) NOT NULL
);

CREATE TABLE authors (
    author_id SERIAL PRIMARY KEY,
    name      VARCHAR(100) NOT NULL
);

CREATE TABLE book_authors (
    isbn      VARCHAR(13) REFERENCES books,
    author_id INTEGER REFERENCES authors,
    author_order SMALLINT DEFAULT 1,
    PRIMARY KEY (isbn, author_id)
);

CREATE TABLE subjects (
    subject_id SERIAL PRIMARY KEY,
    name       VARCHAR(100) UNIQUE NOT NULL
);

CREATE TABLE book_subjects (
    isbn       VARCHAR(13) REFERENCES books,
    subject_id INTEGER REFERENCES subjects,
    PRIMARY KEY (isbn, subject_id)
);

CREATE TABLE editions (
    edition_id  SERIAL PRIMARY KEY,
    isbn        VARCHAR(13) REFERENCES books,
    edition_year SMALLINT NOT NULL,
    publisher   VARCHAR(200),
    page_count  SMALLINT,
    edition_no  SMALLINT DEFAULT 1
);
```

### ข้อ 10
เขียน SQL Query เพื่อตรวจหา 1NF Violations ในตาราง products ที่มี column `categories` VARCHAR

**เฉลย**:
```sql
-- ตรวจหา rows ที่มีหลาย categories (คั่นด้วยคอมมา)
SELECT product_id, name, categories
FROM products
WHERE categories LIKE '%,%';

-- นับจำนวน categories ต่อ product
SELECT
    product_id,
    name,
    categories,
    LENGTH(categories) - LENGTH(REPLACE(categories, ',', '')) + 1 AS category_count
FROM products
WHERE categories LIKE '%,%'
ORDER BY category_count DESC;

-- หา products ที่มี categories มากกว่า 1
SELECT COUNT(*) AS violation_count
FROM products
WHERE categories LIKE '%,%';

-- สรุปปัญหา
SELECT
    'categories column has multi-values' AS issue,
    COUNT(*) AS affected_rows
FROM products WHERE categories LIKE '%,%'
UNION ALL
SELECT
    'empty categories',
    COUNT(*)
FROM products WHERE categories IS NULL OR categories = '';
```

---

*จบ Part 053: First Normal Form (1NF)*

**ในส่วนต่อไป (Part 054)**: เราจะเรียนรู้เรื่อง Second Normal Form (2NF) และ Functional Dependencies ที่เป็นพื้นฐานสำคัญของ Normalization
