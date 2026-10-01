# Part 056: BCNF, 4NF, 5NF
# Normal Forms ระดับสูง

---

## บทนำ: Beyond 3NF

3NF เพียงพอสำหรับกรณีส่วนใหญ่ แต่ยังมีกรณีพิเศษที่ 3NF ไม่เพียงพอ

```
Normal Form Summary:
1NF → 2NF → 3NF → BCNF → 4NF → 5NF → DKNF
 ↑      ↑      ↑      ↑      ↑      ↑
พื้นฐาน  Partial Trans  Super  MVD   JD
       Dep    Dep    Key
```

---

## Boyce-Codd Normal Form (BCNF)

### BCNF vs 3NF

**3NF** อนุญาตให้มี FD: X → Y ได้ถ้า Y เป็น Prime Attribute
**BCNF** เข้มกว่า: X ต้องเป็น Superkey สำหรับ **ทุก** FD

```
BCNF Definition:
ตาราง R อยู่ใน BCNF ก็ต่อเมื่อ:
สำหรับทุก FD: X → Y (ที่ Y ไม่ใช่ subset ของ X)
X ต้องเป็น Superkey ของ R

กล่าวคือ: ถ้า X → Y แล้ว X ต้องสามารถ determine ทุกอย่างใน R
```

### ตัวอย่าง 1: BCNF vs 3NF

```
ตาราง COURSE_TEACHER:
COURSE_TEACHER(student, course, teacher)

กำหนด Business Rules:
- นักเรียนหนึ่งคนในหนึ่งวิชา มีครูสอนได้หนึ่งคน
- ครูหนึ่งคนสอนวิชาได้หนึ่งวิชา

FDs:
(student, course) → teacher     ... [FD1]
teacher → course                 ... [FD2]

Candidate Keys:
- (student, course): เพราะ [FD1] กำหนดได้ทุกอย่าง
- (student, teacher): เพราะ teacher → course [FD2]

Prime Attributes: student, course, teacher (ทุกตัวเป็น Prime!)

3NF Check:
FD1: (student, course) → teacher: Superkey ✓
FD2: teacher → course: teacher ไม่ใช่ Superkey แต่ course เป็น Prime Attribute
→ 3NF: OK ✓

BCNF Check:
FD2: teacher → course: teacher ไม่ใช่ Superkey
→ BCNF: VIOLATED ✗
```

```sql
-- ❌ ละเมิด BCNF (แต่อยู่ใน 3NF)
CREATE TABLE course_teacher_bad (
    student  VARCHAR(100),
    course   VARCHAR(100),
    teacher  VARCHAR(100),
    PRIMARY KEY (student, course)
);

-- ปัญหา (Update Anomaly):
-- ถ้า Teacher Smith เปลี่ยนไปสอนวิชา Physics แทน Math
-- ต้องแก้หลาย rows ทุก Row ที่มี teacher='Smith'

-- ✅ แปลงเป็น BCNF
-- Decompose โดยแยก FD2 ออกมา

CREATE TABLE teacher_course (
    teacher  VARCHAR(100) PRIMARY KEY,
    course   VARCHAR(100) NOT NULL
);

CREATE TABLE student_teacher (
    student  VARCHAR(100),
    teacher  VARCHAR(100) REFERENCES teacher_course,
    PRIMARY KEY (student, teacher)
);

-- หมายเหตุ: FD (student,course) → teacher ไม่ได้รับการ Preserve อีกต่อไป
-- ต้องใช้ JOIN เพื่อตรวจสอบ
```

---

### ตัวอย่าง 2: BCNF ในระบบ Booking

```
ตาราง BOOKING:
BOOKING(room_no, date, guest_id, rate_per_night)

Business Rules:
- ห้องหนึ่งวันหนึ่งมีแขกได้หนึ่งคน
- ราคาขึ้นอยู่กับห้องและวันที่

FDs:
(room_no, date) → guest_id      ... Candidate Key
(room_no, date) → rate_per_night ... FD
guest_id → ? (guest ไม่ determine อื่นๆ ในบริบทนี้)

BCNF Check:
(room_no, date) → guest_id: (room_no, date) เป็น Superkey ✓
(room_no, date) → rate_per_night: (room_no, date) เป็น Superkey ✓
→ BCNF: OK ✓
```

```sql
-- ตาราง BOOKING ที่อยู่ใน BCNF
CREATE TABLE room_rates (
    room_no          INTEGER,
    date             DATE,
    rate_per_night   DECIMAL(10,2) NOT NULL,
    PRIMARY KEY (room_no, date)
);

CREATE TABLE bookings (
    room_no    INTEGER,
    date       DATE,
    guest_id   INTEGER NOT NULL REFERENCES guests,
    FOREIGN KEY (room_no, date) REFERENCES room_rates,
    PRIMARY KEY (room_no, date)
);
```

---

### ตัวอย่าง 3: BCNF กับ Multiple Candidate Keys

```sql
-- ❌ ตาราง EMPLOYEE_DEPARTMENT_PROJECT
-- Candidate Keys: (emp_id, project) และ (dept, project)
-- FDs:
-- emp_id → dept            (พนักงานอยู่ในแผนกเดียว)
-- (emp_id, project) → ?    
-- (dept, project) → emp_id (แต่ละแผนกมีพนักงานหนึ่งคนต่อ project)

CREATE TABLE emp_dept_project_bad (
    emp_id  INTEGER,
    dept    VARCHAR(50),
    project VARCHAR(100),
    PRIMARY KEY (emp_id, project)
    -- emp_id → dept: Transitive!
);

-- ✅ BCNF
CREATE TABLE employees (
    emp_id INTEGER PRIMARY KEY,
    dept   VARCHAR(50) NOT NULL
);

CREATE TABLE project_assignments (
    emp_id  INTEGER REFERENCES employees,
    project VARCHAR(100),
    PRIMARY KEY (emp_id, project)
);
```

---

### ตัวอย่าง 4: BCNF ในระบบ Academic

```sql
-- ❌ BCNF Violation
-- SEMINAR(seminar_id, room_id, instructor_id, time_slot)
-- FDs:
-- (room_id, time_slot) → instructor_id  (ห้องหนึ่งเวลาหนึ่งมีผู้สอนหนึ่งคน)
-- (instructor_id, time_slot) → room_id  (ผู้สอนหนึ่งคนเวลาหนึ่งอยู่ในห้องเดียว)
-- Candidate Keys: (room_id, time_slot) และ (instructor_id, time_slot)

-- ทุก FD มี Superkey เป็น Determinant → BCNF OK ✓
-- (กรณีนี้อยู่ใน BCNF แล้ว)

CREATE TABLE seminars (
    room_id        INTEGER NOT NULL REFERENCES rooms,
    instructor_id  INTEGER NOT NULL REFERENCES instructors,
    time_slot      VARCHAR(30) NOT NULL,
    PRIMARY KEY (room_id, time_slot),
    UNIQUE (instructor_id, time_slot)
);
```

---

### ตัวอย่าง 5: BCNF กับ Lookup Tables

```sql
-- ❌ VIOLATION: Status → Description (ในตาราง Orders)
CREATE TABLE orders_bad (
    order_id    INTEGER PRIMARY KEY,
    status_code VARCHAR(20),
    status_desc TEXT,    -- status_code → status_desc: BCNF Violation!
    total       DECIMAL
);

-- ✅ BCNF: แยก Lookup Table
CREATE TABLE order_statuses (
    code        VARCHAR(20) PRIMARY KEY,
    description TEXT NOT NULL
);

CREATE TABLE orders (
    order_id    INTEGER PRIMARY KEY,
    status_code VARCHAR(20) REFERENCES order_statuses,
    total       DECIMAL(10,2)
);
```

---

### ตัวอย่าง 6: BCNF กับ Geographic Data

```sql
-- ❌ Violation: zip_code → city → state
CREATE TABLE addresses_bad (
    address_id INTEGER PRIMARY KEY,
    street     VARCHAR(200),
    zip_code   VARCHAR(10),
    city       VARCHAR(100),  -- zip_code → city (zip_code ไม่ใช่ Superkey ของตาราง)
    state      VARCHAR(50)    -- zip_code → state
);

-- ✅ BCNF
CREATE TABLE zip_codes (
    zip_code VARCHAR(10) PRIMARY KEY,
    city     VARCHAR(100) NOT NULL,
    state    VARCHAR(50) NOT NULL
);

CREATE TABLE addresses (
    address_id INTEGER PRIMARY KEY,
    street     VARCHAR(200),
    zip_code   VARCHAR(10) REFERENCES zip_codes
);
```

---

### ตัวอย่าง 7: BCNF ในระบบ Insurance

```sql
-- ❌ Violation
-- POLICY(policy_no, vehicle_id, owner_id, premium_rate)
-- FDs:
-- policy_no → vehicle_id, owner_id, premium_rate
-- vehicle_id → premium_rate  (ประเภทรถกำหนดเบี้ยประกัน)
-- → vehicle_id ไม่ใช่ Superkey แต่ determine premium_rate

CREATE TABLE policies_bad (
    policy_no    VARCHAR(20) PRIMARY KEY,
    vehicle_id   INTEGER,
    owner_id     INTEGER,
    premium_rate DECIMAL(5,4)  -- vehicle_id → premium_rate: BCNF Violation!
);

-- ✅ BCNF
CREATE TABLE vehicle_rates (
    vehicle_id   INTEGER PRIMARY KEY REFERENCES vehicles,
    premium_rate DECIMAL(5,4) NOT NULL
);

CREATE TABLE policies (
    policy_no  VARCHAR(20) PRIMARY KEY,
    vehicle_id INTEGER NOT NULL REFERENCES vehicle_rates,
    owner_id   INTEGER NOT NULL REFERENCES owners
);
```

---

### ตัวอย่าง 8: BCNF กับ Skill Assessment

```sql
-- ❌ Violation
-- ASSESSMENT(employee, skill, assessor)
-- Business Rule: assessor สามารถ assess ได้เฉพาะ skills ที่กำหนด
-- FDs:
-- (employee, skill) → assessor
-- assessor → skill  ← BCNF Violation! (assessor ไม่ใช่ Superkey)

CREATE TABLE assessments_bad (
    employee  INTEGER,
    skill     VARCHAR(100),
    assessor  INTEGER,
    PRIMARY KEY (employee, skill)
);

-- ✅ BCNF
CREATE TABLE assessor_skills (
    assessor  INTEGER REFERENCES employees,
    skill     VARCHAR(100),
    PRIMARY KEY (assessor, skill)
);

-- หรือถ้า assessor สอน skill เดียว:
CREATE TABLE assessor_specialties (
    assessor  INTEGER PRIMARY KEY REFERENCES employees,
    skill     VARCHAR(100) NOT NULL
);

CREATE TABLE assessments (
    employee  INTEGER REFERENCES employees,
    assessor  INTEGER REFERENCES assessor_specialties,
    score     INTEGER,
    PRIMARY KEY (employee, assessor)
);
```

---

## 4NF และ Multivalued Dependencies (MVD)

### Multivalued Dependency (MVD) คืออะไร?

MVD เกิดขึ้นเมื่อ Column หนึ่งกำหนดชุดของค่า (Set of Values) ของอีก Column โดยไม่ขึ้นกับ Column อื่น

```
Notation: A →→ B (A multi-determines B)

ความหมาย:
สำหรับทุกคู่ของ Row ที่ A เท่ากัน
ชุดของ B ที่ associated กับ A จะเหมือนกัน ไม่ว่า C จะเป็นอะไร

ตัวอย่าง:
ตาราง EMPLOYEE_SKILLS_LANGUAGES:
emp_id →→ skill    (พนักงานมีหลาย skills ไม่ขึ้นกับ language)
emp_id →→ language (พนักงานพูดหลายภาษา ไม่ขึ้นกับ skill)
```

### ตัวอย่าง MVD

```sql
-- ❌ ละเมิด 4NF: Multivalued Dependency
CREATE TABLE emp_skills_languages_bad (
    emp_id   INTEGER,
    skill    VARCHAR(100),
    language VARCHAR(50),
    PRIMARY KEY (emp_id, skill, language)
);

-- ข้อมูล (พนักงาน Alice มี 2 skills และ 2 languages):
-- (Alice, Python, English)
-- (Alice, Python, Thai)
-- (Alice, SQL, English)
-- (Alice, SQL, Thai)
-- ← ต้องเพิ่ม 4 rows (Cartesian Product!)

-- ปัญหา:
-- ถ้า Alice เรียน Java ต้องเพิ่ม:
-- (Alice, Java, English)
-- (Alice, Java, Thai)
-- ← 2 rows แทนที่จะเป็น 1!

-- ถ้าลืมเพิ่ม (Alice, Java, Thai) ข้อมูลขัดแย้ง!
```

```sql
-- ✅ 4NF: แยก MVDs ออกเป็นตารางแยก

CREATE TABLE employee_skills (
    emp_id INTEGER NOT NULL REFERENCES employees,
    skill  VARCHAR(100) NOT NULL,
    PRIMARY KEY (emp_id, skill)
);

CREATE TABLE employee_languages (
    emp_id   INTEGER NOT NULL REFERENCES employees,
    language VARCHAR(50) NOT NULL,
    PRIMARY KEY (emp_id, language)
);

-- ข้อมูล:
-- employee_skills: (Alice, Python), (Alice, SQL)
-- employee_languages: (Alice, English), (Alice, Thai)

-- เพิ่ม Java: เพิ่มแค่ (Alice, Java) ใน employee_skills
-- ไม่ต้องแก้ employee_languages!
```

### 4NF Definition

ตารางอยู่ใน **4NF** เมื่อ:
1. อยู่ใน BCNF
2. ไม่มี Non-trivial Multivalued Dependency ที่ไม่ใช่ Functional Dependency

```
4NF Rule:
สำหรับทุก MVD: A →→ B ใน R
A ต้องเป็น Superkey ของ R
(หรือ MVD นั้นเป็น trivial: B ⊆ A หรือ A ∪ B = R)
```

### ตัวอย่าง 4NF เพิ่มเติม

```sql
-- ❌ ละเมิด 4NF
-- PRODUCT_COLORS_SIZES(product_id, color, size)
-- MVD: product_id →→ color (color ไม่ขึ้นกับ size)
--      product_id →→ size (size ไม่ขึ้นกับ color)

CREATE TABLE product_colors_sizes_bad (
    product_id INTEGER,
    color      VARCHAR(30),
    size       VARCHAR(20),
    PRIMARY KEY (product_id, color, size)
);
-- ถ้า product มี 3 colors และ 5 sizes = 15 rows!

-- ✅ 4NF
CREATE TABLE product_colors (
    product_id INTEGER REFERENCES products,
    color      VARCHAR(30),
    PRIMARY KEY (product_id, color)
);

CREATE TABLE product_sizes (
    product_id INTEGER REFERENCES products,
    size       VARCHAR(20),
    PRIMARY KEY (product_id, size)
);
```

---

## 5NF และ Join Dependencies

### Join Dependency (JD) คืออะไร?

Join Dependency เกิดขึ้นเมื่อตาราง R สามารถแยกเป็นหลายตารางย่อย และ JOIN กลับมาได้เหมือนเดิมเสมอ

```
JD Notation: *(R1, R2, R3)
หมายความว่า R = R1 ⋈ R2 ⋈ R3 (Lossless)

ถ้า JD ไม่ได้เกิดจาก Candidate Key = ละเมิด 5NF
```

### ตัวอย่าง 5NF

```sql
-- ตัวอย่าง: ระบบ Supply Chain
-- SUPPLY(supplier, part, project)
-- Business Rules:
-- ถ้า supplier S จัดหา part P AND part P ใช้ใน project J
-- AND supplier S จัดหาของให้ project J
-- แล้ว S จัดหา P ให้ J

-- ❌ ละเมิด 5NF (ถ้า JD ไม่เกิดจาก Key)
CREATE TABLE supply_bad (
    supplier  INTEGER,
    part      INTEGER,
    project   INTEGER,
    PRIMARY KEY (supplier, part, project)
);

-- ✅ 5NF: แยกเป็น 3 ตาราง
CREATE TABLE supplier_parts (
    supplier INTEGER REFERENCES suppliers,
    part     INTEGER REFERENCES parts,
    PRIMARY KEY (supplier, part)
);

CREATE TABLE supplier_projects (
    supplier INTEGER REFERENCES suppliers,
    project  INTEGER REFERENCES projects,
    PRIMARY KEY (supplier, project)
);

CREATE TABLE part_projects (
    part    INTEGER REFERENCES parts,
    project INTEGER REFERENCES projects,
    PRIMARY KEY (part, project)
);

-- สร้าง supply ด้วย JOIN:
SELECT sp.supplier, sp.part, spj.project
FROM supplier_parts sp
JOIN supplier_projects spj ON sp.supplier = spj.supplier
JOIN part_projects pp ON sp.part = pp.part AND spj.project = pp.project;
```

---

## เมื่อใดควรหยุด Normalize?

### Trade-offs ของการ Normalize

```
Advantages of Higher NF:
✓ ลด Data Redundancy
✓ ลด Update/Delete/Insert Anomalies
✓ ข้อมูลสอดคล้องมากขึ้น

Disadvantages of Over-Normalization:
✗ ต้อง JOIN มากขึ้น (ช้าลง)
✗ Query ซับซ้อนขึ้น
✗ Application Code ซับซ้อนขึ้น
✗ อาจ Lose Business Context
```

### แนวทาง Practical

```
ในทางปฏิบัติ:
- OLTP Systems: เป้าหมายคือ 3NF หรือ BCNF
- Reporting/Analytics: บางครั้ง Denormalize เพื่อ Performance
- Data Warehouses: Star Schema (Denormalized by design)

หยุดที่ 3NF เมื่อ:
- ระบบทำงานได้ดี
- Performance ยอมรับได้
- ไม่มีปัญหา Anomalies

พิจารณา BCNF เมื่อ:
- ยังมี Update Anomalies แม้อยู่ใน 3NF
- มี Multiple Candidate Keys

พิจารณา 4NF เมื่อ:
- มี Multivalued Dependencies ชัดเจน
- ข้อมูลเป็น Cartesian Product ของ Independent Sets

แทบไม่ต้องใช้ 5NF:
- ในทางปฏิบัติ 5NF พบน้อยมาก
- ถ้าต้องใช้ 5NF แสดงว่า Business Rule ซับซ้อนมาก
```

---

## DKNF (Domain-Key Normal Form) - Overview

**Domain-Key Normal Form (DKNF)** เป็น Normal Form ระดับสูงสุดที่ใช้กันในเชิงทฤษฎี

### นิยาม DKNF

```
ตาราง R อยู่ใน DKNF เมื่อ:
ทุก Constraint ใน R เป็นผลมาจาก:
1. Domain Constraints (ข้อกำหนดของ Data Type)
2. Key Constraints (Primary Key/Unique Constraints)

กล่าวคือ ถ้าเรากำหนด Domain และ Key อย่างถูกต้อง
Constraint ทั้งหมดจะถูก enforce โดยอัตโนมัติ
```

### ตัวอย่าง DKNF

```sql
-- ตัวอย่าง: ตารางที่อยู่ใน DKNF
CREATE TABLE employees (
    emp_id      INTEGER PRIMARY KEY,         -- Key Constraint
    name        VARCHAR(100) NOT NULL,
    birth_date  DATE CHECK (birth_date < CURRENT_DATE), -- Domain Constraint
    salary      DECIMAL(10,2) CHECK (salary >= 0),       -- Domain Constraint
    gender      CHAR(1) CHECK (gender IN ('M','F','O')), -- Domain Constraint
    hire_date   DATE NOT NULL DEFAULT CURRENT_DATE
);

-- ทุก Constraint เป็น Domain Constraint หรือ Key Constraint
-- → อยู่ใน DKNF ✓

-- ตัวอย่างที่ไม่อยู่ใน DKNF:
-- ถ้ามี Rule: "ถ้า job_grade = 'Manager' แล้ว salary ≥ 50000"
-- นี่คือ Multi-Column Constraint ที่ไม่ใช่ Domain หรือ Key Constraint
-- → ไม่อยู่ใน DKNF (ต้องใช้ CHECK Constraint หรือ Trigger)
```

---

## ตัวอย่าง BCNF เพิ่มเติม

### ตัวอย่าง 9: BCNF ในระบบ Real Estate

```sql
-- ❌ ละเมิด BCNF
-- PROPERTY(property_id, agent_id, client_id, commission_rate)
-- FDs:
-- property_id → agent_id, commission_rate
-- agent_id → commission_rate  ← BCNF Violation!

CREATE TABLE property_listings_bad (
    property_id      INTEGER,
    agent_id         INTEGER,
    client_id        INTEGER,
    commission_rate  DECIMAL(4,2),  -- agent_id → commission_rate
    PRIMARY KEY (property_id, client_id)
);

-- ✅ BCNF
CREATE TABLE agent_rates (
    agent_id         INTEGER PRIMARY KEY REFERENCES agents,
    commission_rate  DECIMAL(4,2) NOT NULL
);

CREATE TABLE property_listings (
    property_id  INTEGER REFERENCES properties,
    agent_id     INTEGER REFERENCES agent_rates,
    client_id    INTEGER REFERENCES clients,
    PRIMARY KEY (property_id, client_id)
);
```

### ตัวอย่าง 10: BCNF ในระบบ Healthcare

```sql
-- ❌ ละเมิด BCNF
-- PRESCRIPTION(patient, drug, doctor)
-- Business Rules:
-- - คนไข้ได้รับยาจากหมอคนหนึ่ง
-- - หมอแต่ละคนสั่งยาได้ชุดหนึ่ง (doctor → drug)

-- FDs:
-- (patient, drug) → doctor
-- (patient, doctor) → drug  ← ถ้าหมอสั่งยาได้คนละตัว
-- doctor →→ drug (MVD?) หรือ doctor → drug (FD?)

CREATE TABLE prescriptions_bad (
    patient VARCHAR(100),
    drug    VARCHAR(100),
    doctor  VARCHAR(100),
    PRIMARY KEY (patient, drug)
);

-- ถ้า doctor → drug เป็น FD:
-- ✅ BCNF
CREATE TABLE doctor_prescriptions (
    doctor VARCHAR(100) PRIMARY KEY,
    drug   VARCHAR(100) NOT NULL
);

CREATE TABLE patient_prescriptions (
    patient VARCHAR(100),
    doctor  VARCHAR(100) REFERENCES doctor_prescriptions,
    PRIMARY KEY (patient, doctor)
);
```

---

## Summary: Normal Forms Comparison

```
┌────────────────────────────────────────────────────────────────────────────┐
│ Normal  │ Requirements          │ Eliminates              │ Use Case       │
│ Form    │                       │                         │                │
├─────────┼───────────────────────┼─────────────────────────┼────────────────┤
│ 1NF     │ Atomic values,        │ Multi-valued columns,   │ All tables     │
│         │ No repeating groups   │ Repeating groups        │                │
├─────────┼───────────────────────┼─────────────────────────┼────────────────┤
│ 2NF     │ 1NF +                 │ Partial Dependencies    │ Tables with    │
│         │ No Partial Deps       │                         │ Composite PK   │
├─────────┼───────────────────────┼─────────────────────────┼────────────────┤
│ 3NF     │ 2NF +                 │ Transitive              │ Most OLTP      │
│         │ No Transitive Deps    │ Dependencies            │ Systems        │
├─────────┼───────────────────────┼─────────────────────────┼────────────────┤
│ BCNF    │ 3NF +                 │ FDs where               │ Multiple       │
│         │ All FD: X→Y, X=Superkey│ Determinant ≠ Superkey │ Candidate Keys │
├─────────┼───────────────────────┼─────────────────────────┼────────────────┤
│ 4NF     │ BCNF +                │ Non-trivial MVDs        │ Independent    │
│         │ No non-trivial MVDs   │                         │ Multi-values   │
├─────────┼───────────────────────┼─────────────────────────┼────────────────┤
│ 5NF     │ 4NF +                 │ Non-trivial JDs         │ Complex Join   │
│         │ No non-trivial JDs    │                         │ Dependencies   │
├─────────┼───────────────────────┼─────────────────────────┼────────────────┤
│ DKNF    │ All Constraints from  │ Application-level       │ Theoretical    │
│         │ Domain + Key only     │ Constraints             │                │
└────────────────────────────────────────────────────────────────────────────┘
```

---

## แบบฝึกหัด (10 ข้อ)

### ข้อ 1
อธิบายความแตกต่างระหว่าง 3NF และ BCNF พร้อมตัวอย่าง

**เฉลย**:
```
3NF: ยอมให้ FD: X → Y ถ้า Y เป็น Prime Attribute
BCNF: X ต้องเป็น Superkey สำหรับทุก FD: X → Y (ไม่มีข้อยกเว้น)

BCNF เข้มกว่า 3NF

ตัวอย่างที่ 3NF แต่ไม่ใช่ BCNF:
SCHEDULE(student, subject, teacher)
FDs: (student, subject) → teacher, teacher → subject
Candidate Keys: (student, subject) และ (student, teacher)

3NF: teacher → subject OK เพราะ subject เป็น Prime Attribute
BCNF: teacher → subject VIOLATION เพราะ teacher ไม่ใช่ Superkey
```

### ข้อ 2
ระบุว่าตารางนี้ละเมิด Normal Form ระดับใด:
```
SPORTS_CLUB(member_id, sport, coach, day_of_week)
FDs: (member_id, sport) → coach, coach → sport
```

**เฉลย**:
```
Candidate Keys:
- (member_id, sport): กำหนดได้ทั้งหมด ✓
- (member_id, coach): เพราะ coach → sport, ดังนั้น (member_id, coach) → sport ✓

Prime Attributes: member_id, sport, coach (ทุกตัวเป็น Prime!)
Non-prime: day_of_week

3NF Check:
coach → sport: coach ไม่ใช่ Superkey แต่ sport เป็น Prime Attribute
→ 3NF: OK ✓

BCNF Check:
coach → sport: coach ไม่ใช่ Superkey
→ BCNF: VIOLATED ✗

สรุป: อยู่ใน 3NF แต่ละเมิด BCNF
```

### ข้อ 3
แปลงตาราง COURSE_TEACHER ให้อยู่ใน BCNF และอธิบาย Trade-off

**เฉลย**:
```sql
-- Original: COURSE_TEACHER(student, course, teacher)
-- FDs: (student, course) → teacher, teacher → course

-- BCNF Decomposition:
CREATE TABLE teacher_courses (
    teacher  VARCHAR(100) PRIMARY KEY,
    course   VARCHAR(100) NOT NULL
);

CREATE TABLE student_teachers (
    student  VARCHAR(100),
    teacher  VARCHAR(100) REFERENCES teacher_courses,
    PRIMARY KEY (student, teacher)
);

-- Trade-offs:
-- ✓ แก้ BCNF Violation
-- ✓ เปลี่ยน course ของ teacher แค่ที่เดียว
-- ✗ FD (student, course) → teacher ไม่ได้รับการ Preserve
--   ต้องใช้ Query เพื่อตรวจสอบ:
SELECT st.student, tc.course, st.teacher
FROM student_teachers st
JOIN teacher_courses tc ON st.teacher = tc.teacher;
-- ✗ อาจ Lose Business Rule ว่า "นักเรียนเรียนวิชาหนึ่งกับครูเพียงคนเดียว"
```

### ข้อ 4
อธิบาย Multivalued Dependency (MVD) คืออะไร และให้ตัวอย่าง

**เฉลย**:
```
MVD: A →→ B (A multi-determines B)
หมายความว่า: ค่า A กำหนดชุดของค่า B โดยไม่ขึ้นกับ Column อื่น

ตัวอย่าง 1: emp_id →→ skill
- Alice มี skills: {Python, SQL, Java}
- skills เหล่านี้ไม่ขึ้นกับ language ที่ Alice พูด

ตัวอย่าง 2: product_id →→ color
- สินค้า P001 มี colors: {red, blue, green}
- colors ไม่ขึ้นกับ sizes ที่มี

ตัวอย่าง 3: course_id →→ textbook
- วิชา CS101 ใช้ textbooks: {Book A, Book B}
- ไม่ขึ้นกับว่า professor สอนใคร

MVD เกิดเมื่อ: ตารางแสดง Cartesian Product ของ Independent Sets
4NF กำจัด MVD โดยแยกออกเป็นตารางแยก
```

### ข้อ 5
แปลง Schema นี้ให้อยู่ใน 4NF:
```
EMPLOYEE_HOBBY_LANGUAGE(emp_id, hobby, language)
MVDs: emp_id →→ hobby, emp_id →→ language
```

**เฉลย**:
```sql
-- Original Problem:
-- Alice มี hobby: running, reading
-- Alice พูดภาษา: Thai, English
-- ต้องเก็บ: (Alice, running, Thai), (Alice, running, English),
--            (Alice, reading, Thai), (Alice, reading, English)
-- = Redundancy! (4 rows สำหรับ 2+2 values)

-- 4NF Solution: แยกเป็น 2 ตาราง
CREATE TABLE employee_hobbies (
    emp_id  INTEGER NOT NULL REFERENCES employees,
    hobby   VARCHAR(100) NOT NULL,
    PRIMARY KEY (emp_id, hobby)
);

CREATE TABLE employee_languages (
    emp_id   INTEGER NOT NULL REFERENCES employees,
    language VARCHAR(50) NOT NULL,
    PRIMARY KEY (emp_id, language)
);

-- ข้อมูล:
-- employee_hobbies: (Alice, running), (Alice, reading)
-- employee_languages: (Alice, Thai), (Alice, English)

-- ถ้าต้องการ Cartesian Product (ในบางกรณี):
SELECT eh.emp_id, eh.hobby, el.language
FROM employee_hobbies eh
JOIN employee_languages el ON eh.emp_id = el.emp_id;
```

### ข้อ 6
เมื่อใดที่ควร Stop Normalize ที่ 3NF และไม่ไป BCNF?

**เฉลย**:
```
ควร Stop ที่ 3NF เมื่อ:

1. ไม่มี Multiple Candidate Keys
   → ถ้า PK เป็น Single Column และไม่มี Candidate Keys อื่น
     3NF = BCNF (เหมือนกัน)

2. Dependency Preservation สำคัญกว่า
   → การแปลงเป็น BCNF บางครั้งทำให้ FD บางตัวไม่ได้รับการ Preserve
   → ถ้า Business Rules ต้องการ Enforce FD เหล่านั้น อาจ Stick กับ 3NF

3. Performance เป็นข้อกังวลหลัก
   → การแยก Table เพิ่มทำให้ต้อง JOIN มากขึ้น
   → ถ้า Query Performance สำคัญมาก อาจยอม Violation เล็กน้อย

4. Table ซับซ้อนน้อย
   → ถ้าตารางไม่ซับซ้อน การแยกอาจทำให้ Schema ดูยุ่งยากโดยไม่จำเป็น

5. ใน Real World:
   → ส่วนใหญ่ 3NF เพียงพอ
   → BCNF สำคัญเมื่อมีปัญหาจริงๆ (Anomalies ที่ชัดเจน)
```

### ข้อ 7
ออกแบบ Schema ระดับ BCNF สำหรับระบบ "Tutor Assignment" ที่มี:
- นักเรียนเรียนวิชาจากติวเตอร์คนหนึ่ง
- ติวเตอร์สอนวิชาได้หลายวิชา แต่นักเรียนหนึ่งคนต่อวิชาหนึ่งมีติวเตอร์คนเดียว
- ติวเตอร์ต้องการ Certificate สำหรับแต่ละวิชาที่สอน

**เฉลย**:
```sql
-- FDs:
-- (student, subject) → tutor     [student กับ subject กำหนด tutor]
-- tutor → {subjects they can teach} (MVD: tutor →→ subject)
-- (tutor, subject) → certificate [tutor+subject กำหนด certificate]

CREATE TABLE tutors (
    tutor_id  SERIAL PRIMARY KEY,
    name      VARCHAR(100) NOT NULL,
    phone     VARCHAR(20)
);

CREATE TABLE subjects (
    subject_id INTEGER PRIMARY KEY,
    name       VARCHAR(100) UNIQUE NOT NULL
);

-- BCNF: tutor qualifications (separate from student assignments)
CREATE TABLE tutor_qualifications (
    tutor_id    INTEGER REFERENCES tutors,
    subject_id  INTEGER REFERENCES subjects,
    certificate VARCHAR(200),
    cert_date   DATE,
    PRIMARY KEY (tutor_id, subject_id)
);

CREATE TABLE students (
    student_id INTEGER PRIMARY KEY,
    name       VARCHAR(100) NOT NULL
);

-- Student-Subject-Tutor assignments
-- BCNF: (student_id, subject_id) is Superkey
CREATE TABLE tutoring_sessions (
    student_id INTEGER REFERENCES students,
    subject_id INTEGER REFERENCES subjects,
    tutor_id   INTEGER NOT NULL,
    PRIMARY KEY (student_id, subject_id),
    FOREIGN KEY (tutor_id, subject_id) REFERENCES tutor_qualifications(tutor_id, subject_id)
);
```

### ข้อ 8
อธิบาย Join Dependency และเมื่อใดจะเกิด 5NF Violation

**เฉลย**:
```
Join Dependency (JD): *(R1, R2, ..., Rn)
ตาราง R มี JD *(R1, R2) ก็ต่อเมื่อ R = R1 ⋈ R2 (Lossless)

5NF Violation เกิดเมื่อ:
- R มี JD ที่ไม่ได้เกิดจาก Candidate Keys

ตัวอย่าง 5NF Violation:
SUPPLY(supplier, part, project)
Business Rule:
"ถ้า supplier S จัดหา part P
 AND part P ใช้ใน project J
 AND supplier S จัดหาของให้ project J
 แล้ว S จัดหา P ให้ J"

นี่คือ JD: *(supplier_part, part_project, supplier_project)
ไม่ได้เกิดจาก Candidate Key → ละเมิด 5NF

แก้ไข: แยกเป็น 3 Binary Relations
- supplier_parts(supplier, part)
- part_projects(part, project)
- supplier_projects(supplier, project)

ในทางปฏิบัติ: 5NF พบน้อยมาก ส่วนใหญ่เกิดใน Supply Chain และ M:N:P Relations
```

### ข้อ 9
วิเคราะห์ตาราง PRODUCT_SUPPLIER_CONTRACT ว่าอยู่ใน Normal Form ระดับใด:
```
PRODUCT_SUPPLIER_CONTRACT(product_id, supplier_id, contract_id, 
                           product_name, supplier_name, contract_start,
                           discount_rate)
FDs: product_id → product_name
     supplier_id → supplier_name  
     contract_id → contract_start, discount_rate, product_id, supplier_id
PK: contract_id
```

**เฉลย**:
```
1NF Check:
- Atomic values ✓
- No repeating groups ✓
→ 1NF: OK ✓

2NF Check:
- PK = contract_id (Simple Key)
- Simple Key → No Partial Dependencies possible
→ 2NF: OK ✓

3NF Check:
- contract_id → product_id → product_name (Transitive!)
- contract_id → supplier_id → supplier_name (Transitive!)
→ 3NF: VIOLATED ✗

แก้ไขให้เป็น 3NF:
CREATE TABLE products (product_id PK, product_name);
CREATE TABLE suppliers (supplier_id PK, supplier_name);
CREATE TABLE contracts (
    contract_id PK,
    product_id FK,
    supplier_id FK,
    contract_start,
    discount_rate
);

หลังแก้: ตรวจสอบ BCNF
- contract_id → product_id: Superkey ✓
- contract_id → supplier_id: Superkey ✓
- product_id → product_name: Superkey ใน products ✓
→ BCNF: OK ✓ (ทุกตาราง)
```

### ข้อ 10
ออกแบบ Schema ระดับ 4NF สำหรับ "Product Catalog" ที่ Product หนึ่งมี:
- หลาย Colors (ไม่ขึ้นกับ Size)
- หลาย Sizes (ไม่ขึ้นกับ Color)
- หลาย Warehouse locations (ไม่ขึ้นกับ Color หรือ Size)

**เฉลย**:
```sql
-- Problem: product_id →→ color, product_id →→ size, product_id →→ warehouse
-- ถ้าเก็บในตารางเดียว:
-- product_colors_sizes_warehouses(product_id, color, size, warehouse)
-- = 3 MVDs → ต้อง Cartesian Product ของทั้ง 3!
-- 3 colors × 5 sizes × 4 warehouses = 60 rows!

-- 4NF Solution:
CREATE TABLE products (
    product_id  SERIAL PRIMARY KEY,
    name        VARCHAR(200) NOT NULL,
    description TEXT
);

CREATE TABLE product_colors (
    product_id  INTEGER NOT NULL REFERENCES products,
    color       VARCHAR(50) NOT NULL,
    hex_code    CHAR(7),  -- #FF5733
    PRIMARY KEY (product_id, color)
);

CREATE TABLE product_sizes (
    product_id  INTEGER NOT NULL REFERENCES products,
    size_code   VARCHAR(10) NOT NULL,  -- 'XS', 'S', 'M', 'L', 'XL', '38', etc.
    size_display VARCHAR(20),
    PRIMARY KEY (product_id, size_code)
);

CREATE TABLE product_warehouses (
    product_id   INTEGER NOT NULL REFERENCES products,
    warehouse_id INTEGER NOT NULL REFERENCES warehouses,
    PRIMARY KEY (product_id, warehouse_id)
);

-- ถ้าต้องการ SKU ที่เฉพาะเจาะจงมากขึ้น (color+size specific):
CREATE TABLE product_variants (
    variant_id   SERIAL PRIMARY KEY,
    product_id   INTEGER NOT NULL REFERENCES products,
    color        VARCHAR(50) NOT NULL,
    size_code    VARCHAR(10) NOT NULL,
    sku          VARCHAR(50) UNIQUE NOT NULL,
    stock_qty    INTEGER DEFAULT 0,
    FOREIGN KEY (product_id, color) REFERENCES product_colors,
    FOREIGN KEY (product_id, size_code) REFERENCES product_sizes
);

-- Inventory per variant per warehouse:
CREATE TABLE variant_inventory (
    variant_id   INTEGER NOT NULL REFERENCES product_variants,
    warehouse_id INTEGER NOT NULL REFERENCES warehouses,
    quantity     INTEGER NOT NULL DEFAULT 0,
    PRIMARY KEY (variant_id, warehouse_id)
);
```

---

*จบ Part 056: BCNF, 4NF, 5NF*

**ในส่วนต่อไป (Part 057)**: เราจะเรียนรู้เรื่อง Denormalization - เมื่อการ Normalize มากเกินไปอาจไม่ดี และวิธีการ Denormalize อย่างมีประสิทธิภาพ
