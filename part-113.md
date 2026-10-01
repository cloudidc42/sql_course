# Part 113: Hospital Management System

## บทนำ (Introduction)

ระบบบริหารโรงพยาบาล (Hospital Management System) เป็นหนึ่งในระบบฐานข้อมูลที่ซับซ้อนและมีความสำคัญสูงมาก เนื่องจากเกี่ยวข้องกับชีวิตของผู้คน ข้อมูลต้องมีความถูกต้อง ครบถ้วน และปลอดภัยสูงสุด บทนี้จะออกแบบระบบที่ครอบคลุมทุกส่วนของโรงพยาบาล

## 1. Schema Design - ฐานข้อมูลโรงพยาบาล

```sql
-- =========================================
-- HOSPITAL MANAGEMENT SYSTEM - COMPLETE DDL
-- =========================================

CREATE DATABASE hospital_db
    CHARACTER SET utf8mb4
    COLLATE utf8mb4_unicode_ci;

USE hospital_db;

-- =========================================
-- SECTION 1: STAFF & ORGANIZATION
-- =========================================

-- ตาราง Departments (แผนก)
CREATE TABLE departments (
    dept_id         INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    name            VARCHAR(200) NOT NULL,
    code            VARCHAR(20) NOT NULL UNIQUE,
    description     TEXT,
    floor_number    INT,
    phone_ext       VARCHAR(20),
    head_doctor_id  INT UNSIGNED,
    is_active       BOOLEAN DEFAULT TRUE,
    created_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP
) ENGINE=InnoDB;

-- ตาราง Specialties (ความเชี่ยวชาญพิเศษ)
CREATE TABLE specialties (
    specialty_id    INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    name            VARCHAR(200) NOT NULL,
    description     TEXT
) ENGINE=InnoDB;

-- ตาราง Staff (บุคลากรทั้งหมด)
CREATE TABLE staff (
    staff_id        INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    staff_number    VARCHAR(20) NOT NULL UNIQUE,
    role            ENUM('doctor','nurse','pharmacist','lab_tech','admin','receptionist','other') NOT NULL,
    dept_id         INT UNSIGNED,
    first_name      VARCHAR(100) NOT NULL,
    last_name       VARCHAR(100) NOT NULL,
    title           VARCHAR(50),
    gender          ENUM('male','female','other'),
    date_of_birth   DATE,
    national_id     VARCHAR(20) UNIQUE,
    phone           VARCHAR(20),
    email           VARCHAR(255) UNIQUE,
    hire_date       DATE NOT NULL,
    license_number  VARCHAR(50),
    license_expiry  DATE,
    is_active       BOOLEAN DEFAULT TRUE,
    created_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (dept_id) REFERENCES departments(dept_id)
) ENGINE=InnoDB;

-- ตาราง Doctor Specialties (ความเชี่ยวชาญของแพทย์)
CREATE TABLE doctor_specialties (
    staff_id        INT UNSIGNED NOT NULL,
    specialty_id    INT UNSIGNED NOT NULL,
    is_primary      BOOLEAN DEFAULT FALSE,
    PRIMARY KEY (staff_id, specialty_id),
    FOREIGN KEY (staff_id) REFERENCES staff(staff_id),
    FOREIGN KEY (specialty_id) REFERENCES specialties(specialty_id)
) ENGINE=InnoDB;

-- =========================================
-- SECTION 2: PATIENT MANAGEMENT
-- =========================================

-- ตาราง Patients (ผู้ป่วย)
CREATE TABLE patients (
    patient_id      INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    hn              VARCHAR(20) NOT NULL UNIQUE COMMENT 'Hospital Number',
    first_name      VARCHAR(100) NOT NULL,
    last_name       VARCHAR(100) NOT NULL,
    date_of_birth   DATE NOT NULL,
    gender          ENUM('male','female','other') NOT NULL,
    blood_type      ENUM('A+','A-','B+','B-','AB+','AB-','O+','O-','unknown') DEFAULT 'unknown',
    national_id     VARCHAR(20) UNIQUE,
    passport_number VARCHAR(30),
    phone           VARCHAR(20),
    emergency_contact_name VARCHAR(200),
    emergency_contact_phone VARCHAR(20),
    emergency_contact_relation VARCHAR(100),
    address_line1   VARCHAR(255),
    city            VARCHAR(100),
    postal_code     VARCHAR(20),
    country_code    CHAR(2) DEFAULT 'TH',
    nationality     VARCHAR(100) DEFAULT 'Thai',
    occupation      VARCHAR(100),
    -- Medical Info
    allergies       TEXT COMMENT 'Known allergies',
    chronic_conditions TEXT,
    -- Insurance
    insurance_provider VARCHAR(200),
    insurance_policy_number VARCHAR(100),
    insurance_expiry DATE,
    -- System
    is_active       BOOLEAN DEFAULT TRUE,
    registered_at   TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    last_visit_at   TIMESTAMP NULL,
    FULLTEXT INDEX idx_search (first_name, last_name)
) ENGINE=InnoDB;

-- =========================================
-- SECTION 3: APPOINTMENTS
-- =========================================

-- ตาราง Doctor Schedules (ตารางเวร)
CREATE TABLE doctor_schedules (
    schedule_id     INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    staff_id        INT UNSIGNED NOT NULL,
    day_of_week     TINYINT NOT NULL COMMENT '0=Sun, 1=Mon, ..., 6=Sat',
    start_time      TIME NOT NULL,
    end_time        TIME NOT NULL,
    max_appointments INT DEFAULT 20,
    is_active       BOOLEAN DEFAULT TRUE,
    FOREIGN KEY (staff_id) REFERENCES staff(staff_id),
    INDEX idx_staff_day (staff_id, day_of_week)
) ENGINE=InnoDB;

-- ตาราง Appointments (การนัดหมาย)
CREATE TABLE appointments (
    appointment_id  INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    patient_id      INT UNSIGNED NOT NULL,
    doctor_id       INT UNSIGNED NOT NULL,
    dept_id         INT UNSIGNED NOT NULL,
    appointment_date DATE NOT NULL,
    appointment_time TIME NOT NULL,
    duration_minutes INT DEFAULT 15,
    type            ENUM('new_patient','follow_up','emergency','routine_checkup','consultation','procedure') DEFAULT 'follow_up',
    status          ENUM('scheduled','confirmed','arrived','in_progress','completed','cancelled','no_show') DEFAULT 'scheduled',
    reason          TEXT,
    notes           TEXT,
    booked_by       INT UNSIGNED COMMENT 'staff_id who booked',
    created_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    FOREIGN KEY (patient_id) REFERENCES patients(patient_id),
    FOREIGN KEY (doctor_id) REFERENCES staff(staff_id),
    FOREIGN KEY (dept_id) REFERENCES departments(dept_id),
    INDEX idx_date_doctor (appointment_date, doctor_id),
    INDEX idx_patient (patient_id),
    INDEX idx_status (status)
) ENGINE=InnoDB;

-- =========================================
-- SECTION 4: MEDICAL RECORDS
-- =========================================

-- ตาราง Visits (การเยี่ยมโรงพยาบาล)
CREATE TABLE visits (
    visit_id        INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    visit_number    VARCHAR(30) NOT NULL UNIQUE,
    patient_id      INT UNSIGNED NOT NULL,
    doctor_id       INT UNSIGNED NOT NULL,
    dept_id         INT UNSIGNED NOT NULL,
    appointment_id  INT UNSIGNED,
    visit_type      ENUM('opd','ipd','emergency','telehealth') DEFAULT 'opd',
    visit_date      DATETIME NOT NULL,
    discharge_date  DATETIME,
    -- Vital Signs at Visit
    height_cm       DECIMAL(5,1),
    weight_kg       DECIMAL(5,1),
    temperature_c   DECIMAL(4,1),
    blood_pressure_systolic  INT,
    blood_pressure_diastolic INT,
    heart_rate      INT,
    respiratory_rate INT,
    oxygen_saturation DECIMAL(4,1),
    -- Chief Complaint
    chief_complaint TEXT,
    status          ENUM('active','discharged','transferred','deceased') DEFAULT 'active',
    FOREIGN KEY (patient_id) REFERENCES patients(patient_id),
    FOREIGN KEY (doctor_id) REFERENCES staff(staff_id),
    FOREIGN KEY (dept_id) REFERENCES departments(dept_id),
    FOREIGN KEY (appointment_id) REFERENCES appointments(appointment_id) ON DELETE SET NULL,
    INDEX idx_patient (patient_id),
    INDEX idx_date (visit_date),
    INDEX idx_doctor (doctor_id)
) ENGINE=InnoDB;

-- ตาราง ICD-10 Diagnoses Reference
CREATE TABLE icd10_codes (
    icd_code        VARCHAR(10) PRIMARY KEY,
    description     VARCHAR(500) NOT NULL,
    category        VARCHAR(100),
    is_active       BOOLEAN DEFAULT TRUE
) ENGINE=InnoDB;

-- ตาราง Diagnoses (การวินิจฉัยโรค)
CREATE TABLE diagnoses (
    diagnosis_id    INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    visit_id        INT UNSIGNED NOT NULL,
    icd_code        VARCHAR(10) NOT NULL,
    diagnosis_type  ENUM('primary','secondary','complication','comorbidity') DEFAULT 'primary',
    notes           TEXT,
    diagnosed_by    INT UNSIGNED NOT NULL,
    diagnosed_at    TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (visit_id) REFERENCES visits(visit_id) ON DELETE CASCADE,
    FOREIGN KEY (icd_code) REFERENCES icd10_codes(icd_code),
    FOREIGN KEY (diagnosed_by) REFERENCES staff(staff_id),
    INDEX idx_visit (visit_id),
    INDEX idx_icd (icd_code)
) ENGINE=InnoDB;

-- =========================================
-- SECTION 5: PRESCRIPTIONS & PHARMACY
-- =========================================

-- ตาราง Medications (ยา)
CREATE TABLE medications (
    medication_id   INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    drug_code       VARCHAR(30) NOT NULL UNIQUE,
    generic_name    VARCHAR(300) NOT NULL,
    brand_name      VARCHAR(200),
    drug_class      VARCHAR(100),
    form            ENUM('tablet','capsule','syrup','injection','cream','inhaler','drops','other') NOT NULL,
    strength        VARCHAR(100) COMMENT 'e.g., 500mg, 10mg/5ml',
    unit            VARCHAR(50) DEFAULT 'tablet',
    requires_prescription BOOLEAN DEFAULT TRUE,
    is_controlled   BOOLEAN DEFAULT FALSE,
    stock_quantity  INT DEFAULT 0,
    reorder_level   INT DEFAULT 100,
    unit_cost       DECIMAL(10,2),
    unit_price      DECIMAL(10,2),
    is_active       BOOLEAN DEFAULT TRUE
) ENGINE=InnoDB;

-- ตาราง Prescriptions (ใบสั่งยา)
CREATE TABLE prescriptions (
    prescription_id INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    rx_number       VARCHAR(30) NOT NULL UNIQUE,
    visit_id        INT UNSIGNED NOT NULL,
    patient_id      INT UNSIGNED NOT NULL,
    prescribed_by   INT UNSIGNED NOT NULL,
    prescribed_at   TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    dispensed_at    TIMESTAMP NULL,
    dispensed_by    INT UNSIGNED,
    status          ENUM('pending','dispensed','partially_dispensed','cancelled') DEFAULT 'pending',
    notes           TEXT,
    FOREIGN KEY (visit_id) REFERENCES visits(visit_id),
    FOREIGN KEY (patient_id) REFERENCES patients(patient_id),
    FOREIGN KEY (prescribed_by) REFERENCES staff(staff_id),
    FOREIGN KEY (dispensed_by) REFERENCES staff(staff_id) ON DELETE SET NULL,
    INDEX idx_visit (visit_id),
    INDEX idx_patient (patient_id)
) ENGINE=InnoDB;

-- ตาราง Prescription Items (รายการยาในใบสั่งยา)
CREATE TABLE prescription_items (
    item_id         INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    prescription_id INT UNSIGNED NOT NULL,
    medication_id   INT UNSIGNED NOT NULL,
    dosage          VARCHAR(100) NOT NULL COMMENT 'e.g., 1 tablet',
    frequency       VARCHAR(100) NOT NULL COMMENT 'e.g., 3 times/day',
    duration_days   INT NOT NULL,
    total_quantity  INT NOT NULL,
    instructions    TEXT,
    FOREIGN KEY (prescription_id) REFERENCES prescriptions(prescription_id) ON DELETE CASCADE,
    FOREIGN KEY (medication_id) REFERENCES medications(medication_id),
    INDEX idx_prescription (prescription_id)
) ENGINE=InnoDB;

-- =========================================
-- SECTION 6: LAB TESTS
-- =========================================

-- ตาราง Lab Tests (รายการตรวจทางห้องปฏิบัติการ)
CREATE TABLE lab_tests (
    test_id         INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    test_code       VARCHAR(30) NOT NULL UNIQUE,
    test_name       VARCHAR(200) NOT NULL,
    category        VARCHAR(100),
    sample_type     VARCHAR(100) COMMENT 'Blood, Urine, etc.',
    normal_range    VARCHAR(200),
    unit            VARCHAR(50),
    turnaround_hours INT DEFAULT 24,
    cost            DECIMAL(10,2) DEFAULT 0
) ENGINE=InnoDB;

-- ตาราง Lab Orders (คำสั่งตรวจ)
CREATE TABLE lab_orders (
    order_id        INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    order_number    VARCHAR(30) NOT NULL UNIQUE,
    visit_id        INT UNSIGNED NOT NULL,
    patient_id      INT UNSIGNED NOT NULL,
    ordered_by      INT UNSIGNED NOT NULL,
    ordered_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    priority        ENUM('routine','urgent','stat') DEFAULT 'routine',
    status          ENUM('ordered','sample_collected','processing','completed','cancelled') DEFAULT 'ordered',
    FOREIGN KEY (visit_id) REFERENCES visits(visit_id),
    FOREIGN KEY (patient_id) REFERENCES patients(patient_id),
    FOREIGN KEY (ordered_by) REFERENCES staff(staff_id),
    INDEX idx_patient (patient_id),
    INDEX idx_visit (visit_id)
) ENGINE=InnoDB;

-- ตาราง Lab Results (ผลการตรวจ)
CREATE TABLE lab_results (
    result_id       INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    order_id        INT UNSIGNED NOT NULL,
    test_id         INT UNSIGNED NOT NULL,
    result_value    VARCHAR(200),
    result_numeric  DECIMAL(12,4),
    unit            VARCHAR(50),
    is_abnormal     BOOLEAN DEFAULT FALSE,
    abnormal_flag   ENUM('H','L','HH','LL','A') COMMENT 'High, Low, Critical High, Critical Low, Abnormal',
    reference_range VARCHAR(200),
    performed_by    INT UNSIGNED,
    performed_at    TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    reviewed_by     INT UNSIGNED,
    reviewed_at     TIMESTAMP NULL,
    notes           TEXT,
    FOREIGN KEY (order_id) REFERENCES lab_orders(order_id) ON DELETE CASCADE,
    FOREIGN KEY (test_id) REFERENCES lab_tests(test_id),
    FOREIGN KEY (performed_by) REFERENCES staff(staff_id) ON DELETE SET NULL,
    FOREIGN KEY (reviewed_by) REFERENCES staff(staff_id) ON DELETE SET NULL,
    INDEX idx_order (order_id),
    INDEX idx_test (test_id)
) ENGINE=InnoDB;

-- =========================================
-- SECTION 7: WARD & BED MANAGEMENT
-- =========================================

-- ตาราง Wards (หอผู้ป่วย)
CREATE TABLE wards (
    ward_id         INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    name            VARCHAR(100) NOT NULL,
    ward_type       ENUM('general','icu','surgery','maternity','pediatric','psychiatric','emergency') NOT NULL,
    floor           INT,
    dept_id         INT UNSIGNED,
    total_beds      INT NOT NULL,
    FOREIGN KEY (dept_id) REFERENCES departments(dept_id)
) ENGINE=InnoDB;

-- ตาราง Beds (เตียง)
CREATE TABLE beds (
    bed_id          INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    ward_id         INT UNSIGNED NOT NULL,
    bed_number      VARCHAR(20) NOT NULL,
    bed_type        ENUM('standard','private','icu','isolation') DEFAULT 'standard',
    status          ENUM('available','occupied','maintenance','reserved') DEFAULT 'available',
    current_patient_id INT UNSIGNED,
    FOREIGN KEY (ward_id) REFERENCES wards(ward_id),
    FOREIGN KEY (current_patient_id) REFERENCES patients(patient_id) ON DELETE SET NULL,
    UNIQUE KEY uk_ward_bed (ward_id, bed_number)
) ENGINE=InnoDB;

-- ตาราง Admissions (การรับผู้ป่วยใน)
CREATE TABLE admissions (
    admission_id    INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    visit_id        INT UNSIGNED NOT NULL UNIQUE,
    patient_id      INT UNSIGNED NOT NULL,
    bed_id          INT UNSIGNED,
    admitted_by     INT UNSIGNED,
    admitted_at     TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    admitted_from   ENUM('opd','emergency','transfer','direct') DEFAULT 'opd',
    discharged_at   TIMESTAMP NULL,
    discharge_type  ENUM('recovered','transferred','deceased','self_discharge','against_medical_advice'),
    primary_diagnosis_icd VARCHAR(10),
    FOREIGN KEY (visit_id) REFERENCES visits(visit_id),
    FOREIGN KEY (patient_id) REFERENCES patients(patient_id),
    FOREIGN KEY (bed_id) REFERENCES beds(bed_id) ON DELETE SET NULL,
    FOREIGN KEY (primary_diagnosis_icd) REFERENCES icd10_codes(icd_code),
    INDEX idx_patient (patient_id),
    INDEX idx_admitted (admitted_at)
) ENGINE=InnoDB;

-- =========================================
-- SECTION 8: BILLING & INSURANCE
-- =========================================

-- ตาราง Bills (ใบแจ้งหนี้)
CREATE TABLE bills (
    bill_id         INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    bill_number     VARCHAR(30) NOT NULL UNIQUE,
    visit_id        INT UNSIGNED NOT NULL UNIQUE,
    patient_id      INT UNSIGNED NOT NULL,
    billed_at       TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    due_date        DATE,
    subtotal        DECIMAL(12,2) NOT NULL DEFAULT 0,
    insurance_coverage DECIMAL(12,2) DEFAULT 0,
    discount_amount DECIMAL(10,2) DEFAULT 0,
    patient_responsibility DECIMAL(12,2) DEFAULT 0,
    paid_amount     DECIMAL(12,2) DEFAULT 0,
    status          ENUM('draft','issued','partially_paid','paid','overdue','waived','cancelled') DEFAULT 'draft',
    payment_method  ENUM('cash','card','insurance','government_scheme','instalment'),
    paid_at         TIMESTAMP NULL,
    FOREIGN KEY (visit_id) REFERENCES visits(visit_id),
    FOREIGN KEY (patient_id) REFERENCES patients(patient_id),
    INDEX idx_patient (patient_id),
    INDEX idx_status (status)
) ENGINE=InnoDB;

-- ตาราง Bill Items (รายการค่าใช้จ่าย)
CREATE TABLE bill_items (
    item_id         INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    bill_id         INT UNSIGNED NOT NULL,
    item_type       ENUM('consultation','medication','lab_test','procedure','room_charge','imaging','other') NOT NULL,
    description     VARCHAR(500) NOT NULL,
    quantity        INT DEFAULT 1,
    unit_price      DECIMAL(10,2) NOT NULL,
    total_price     DECIMAL(12,2) NOT NULL,
    insurance_covered DECIMAL(10,2) DEFAULT 0,
    FOREIGN KEY (bill_id) REFERENCES bills(bill_id) ON DELETE CASCADE,
    INDEX idx_bill (bill_id)
) ENGINE=InnoDB;
```

## 2. Sample Data

```sql
-- Insert Departments
INSERT INTO departments (dept_id, name, code, floor_number, phone_ext) VALUES
(1, 'อายุรกรรม', 'MED', 2, '1001'),
(2, 'ศัลยกรรม', 'SURG', 3, '1002'),
(3, 'กุมารเวชกรรม', 'PED', 4, '1003'),
(4, 'สูตินรีเวช', 'OB', 4, '1004'),
(5, 'กระดูกและข้อ', 'ORTHO', 3, '1005'),
(6, 'โรคหัวใจ', 'CARDIO', 2, '1006'),
(7, 'ห้องฉุกเฉิน', 'ER', 1, '1007'),
(8, 'รังสีวิทยา', 'RADIO', 1, '1008'),
(9, 'ห้องปฏิบัติการ', 'LAB', 1, '1009'),
(10, 'เภสัชกรรม', 'PHARM', 1, '1010');

-- Insert Specialties
INSERT INTO specialties (specialty_id, name) VALUES
(1, 'อายุรศาสตร์ทั่วไป'),
(2, 'โรคหัวใจและหลอดเลือด'),
(3, 'โรคทางเดินอาหาร'),
(4, 'โรคปอดและระบบหายใจ'),
(5, 'ศัลยศาสตร์ทั่วไป'),
(6, 'กระดูกและข้อ'),
(7, 'กุมารเวชศาสตร์'),
(8, 'สูตินรีเวชวิทยา'),
(9, 'จักษุวิทยา'),
(10, 'โสต ศอ นาสิก');

-- Insert Staff (Doctors & Nurses)
INSERT INTO staff (staff_id, staff_number, role, dept_id, first_name, last_name, title, gender, hire_date, license_number) VALUES
(1, 'DR001', 'doctor', 1, 'สมศักดิ์', 'แพทย์ดี', 'นพ.', 'male', '2015-01-05', 'MD-12345'),
(2, 'DR002', 'doctor', 1, 'วิมล', 'รักษาเก่ง', 'พญ.', 'female', '2016-03-10', 'MD-12346'),
(3, 'DR003', 'doctor', 2, 'ประยุทธ', 'ศัลยกรรมดี', 'นพ.', 'male', '2014-06-15', 'MD-12347'),
(4, 'DR004', 'doctor', 6, 'นพพร', 'หัวใจแข็งแรง', 'นพ.', 'male', '2013-09-01', 'MD-12348'),
(5, 'DR005', 'doctor', 3, 'กาญจนา', 'เด็กน่ารัก', 'พญ.', 'female', '2017-01-10', 'MD-12349'),
(6, 'NR001', 'nurse', 1, 'สุดา', 'พยาบาลดี', NULL, 'female', '2018-04-01', 'RN-56789'),
(7, 'NR002', 'nurse', 7, 'มนต์ตรี', 'ดูแลดี', NULL, 'female', '2019-07-01', 'RN-56790'),
(8, 'PH001', 'pharmacist', 10, 'เอกชัย', 'ยาดี', NULL, 'male', '2017-10-15', 'RPH-11111'),
(9, 'LT001', 'lab_tech', 9, 'พิไล', 'แล็บเก่ง', NULL, 'female', '2020-02-01', 'LAB-22222'),
(10, 'DR006', 'doctor', 7, 'วรรณา', 'ฉุกเฉินดี', 'พญ.', 'female', '2016-11-20', 'MD-12350');

-- Doctor Specialties
INSERT INTO doctor_specialties VALUES
(1, 1, TRUE), (1, 3, FALSE),
(2, 1, TRUE), (2, 4, FALSE),
(3, 5, TRUE),
(4, 2, TRUE),
(5, 7, TRUE),
(10, 1, TRUE);

-- Insert ICD-10 Codes
INSERT INTO icd10_codes (icd_code, description, category) VALUES
('I10', 'Essential (primary) hypertension', 'Diseases of the Circulatory System'),
('E11', 'Type 2 diabetes mellitus', 'Endocrine, Nutritional and Metabolic Diseases'),
('J18.9', 'Pneumonia, unspecified organism', 'Diseases of the Respiratory System'),
('K29.5', 'Chronic gastritis, unspecified', 'Diseases of the Digestive System'),
('M54.5', 'Low back pain', 'Diseases of the Musculoskeletal System'),
('N39.0', 'Urinary tract infection, site not specified', 'Diseases of the Genitourinary System'),
('A09', 'Other and unspecified gastroenteritis', 'Certain Infectious and Parasitic Diseases'),
('J06.9', 'Acute upper respiratory infection, unspecified', 'Diseases of the Respiratory System'),
('I25.10', 'Atherosclerotic heart disease', 'Diseases of the Circulatory System'),
('S72.0', 'Fracture of femoral neck', 'Injury, Poisoning');

-- Insert Patients
INSERT INTO patients (patient_id, hn, first_name, last_name, date_of_birth, gender, blood_type, phone, allergies, chronic_conditions, insurance_provider) VALUES
(1, 'HN-2024-000001', 'สมหมาย', 'ป่วยบ่อย', '1965-04-12', 'male', 'A+', '0891234567', 'Penicillin', 'Hypertension, Type 2 Diabetes', 'ประกันสังคม'),
(2, 'HN-2024-000002', 'มาลัย', 'สุขสบาย', '1980-08-25', 'female', 'O+', '0862345678', NULL, NULL, 'AIA'),
(3, 'HN-2024-000003', 'วิสุทธิ์', 'กาย', '1990-12-01', 'male', 'B+', '0813456789', 'Sulfa drugs', 'Asthma', 'BUPA'),
(4, 'HN-2024-000004', 'นภา', 'ท้องฟ้า', '1975-03-18', 'female', 'AB+', '0904567890', NULL, 'Hypothyroidism', 'ข้าราชการ'),
(5, 'HN-2024-000005', 'ชาติชาย', 'ใจเย็น', '1955-11-30', 'male', 'A-', '0775678901', 'NSAIDs', 'CAD, Hypertension', 'ประกันสังคม'),
(6, 'HN-2024-000006', 'อุษา', 'แสงทอง', '1995-07-22', 'female', 'O-', '0826789012', NULL, NULL, 'Cigna'),
(7, 'HN-2024-000007', 'สุรเดช', 'หาญกล้า', '1970-02-14', 'male', 'B-', '0857890123', 'Aspirin', 'CKD Stage 3', 'ประกันสังคม'),
(8, 'HN-2024-000008', 'ลลิตา', 'ทองดี', '1988-09-05', 'female', 'A+', '0838901234', NULL, NULL, 'Prudential'),
(9, 'HN-2024-000009', 'ธนากร', 'มั่งมี', '1945-06-28', 'male', 'O+', '0799012345', 'Codeine', 'HTN, DM2, CKD', 'ข้าราชการ'),
(10, 'HN-2024-000010', 'พิมพ์ใจ', 'งามวิไล', '2000-01-15', 'female', 'AB-', '0810123456', NULL, NULL, NULL);

-- Insert Visits
INSERT INTO visits (visit_id, visit_number, patient_id, doctor_id, dept_id, visit_type, visit_date,
    height_cm, weight_kg, temperature_c, blood_pressure_systolic, blood_pressure_diastolic, 
    heart_rate, chief_complaint, status) VALUES
(1, 'VN-2024-000001', 1, 1, 1, 'opd', '2024-01-15 09:30:00', 168, 78, 36.8, 145, 92, 82, 'ปวดหัว วิงเวียน ความดันสูง', 'discharged'),
(2, 'VN-2024-000002', 2, 2, 1, 'opd', '2024-01-20 10:00:00', 160, 55, 37.1, 110, 70, 75, 'ไอ เจ็บคอ 3 วัน', 'discharged'),
(3, 'VN-2024-000003', 5, 4, 6, 'opd', '2024-01-25 14:00:00', 172, 82, 36.5, 160, 100, 88, 'แน่นหน้าอก หอบเหนื่อย', 'discharged'),
(4, 'VN-2024-000004', 3, 2, 1, 'opd', '2024-02-01 09:00:00', 175, 70, 38.2, 120, 80, 90, 'ไข้สูง หนาวสั่น ไอมีเสมหะ', 'discharged'),
(5, 'VN-2024-000005', 9, 1, 1, 'ipd', '2024-02-10 15:00:00', 165, 75, 37.5, 155, 98, 85, 'ปัสสาวะน้อย ขาบวม', 'discharged'),
(6, 'VN-2024-000006', 7, 4, 6, 'emergency', '2024-02-15 02:30:00', 170, 80, 36.9, 180, 110, 95, 'เจ็บหน้าอกรุนแรง', 'discharged'),
(7, 'VN-2024-000007', 4, 5, 3, 'opd', '2024-02-20 11:00:00', 158, 60, 36.6, 118, 75, 78, 'ไอ น้ำมูกไหล ตาแดง (เด็กอายุ 5 ขวบ)', 'discharged'),
(8, 'VN-2024-000008', 6, 10, 7, 'emergency', '2024-03-01 20:00:00', 162, 52, 39.1, 115, 75, 105, 'ไข้สูงมาก ปวดท้อง', 'discharged'),
(9, 'VN-2024-000009', 1, 1, 1, 'opd', '2024-03-10 09:00:00', 168, 77, 36.7, 138, 88, 80, 'ติดตามอาการ ความดัน เบาหวาน', 'discharged'),
(10, 'VN-2024-000010', 8, 3, 2, 'opd', '2024-03-15 13:00:00', 158, 58, 36.8, 112, 72, 72, 'ปวดท้องน้อย ต้องการตรวจ', 'discharged');

-- Insert Diagnoses
INSERT INTO diagnoses (visit_id, icd_code, diagnosis_type, diagnosed_by) VALUES
(1, 'I10', 'primary', 1),
(2, 'J06.9', 'primary', 2),
(3, 'I10', 'primary', 4),
(3, 'I25.10', 'secondary', 4),
(4, 'J18.9', 'primary', 2),
(5, 'E11', 'primary', 1),
(5, 'I10', 'secondary', 1),
(6, 'I25.10', 'primary', 4),
(7, 'J06.9', 'primary', 5),
(8, 'N39.0', 'primary', 10),
(9, 'I10', 'primary', 1),
(9, 'E11', 'secondary', 1),
(10, 'N39.0', 'primary', 3);

-- Insert Medications
INSERT INTO medications (medication_id, drug_code, generic_name, brand_name, drug_class, form, strength, unit_cost, unit_price, stock_quantity) VALUES
(1, 'MED001', 'Amlodipine', 'Norvasc', 'Calcium Channel Blocker', 'tablet', '5mg', 3, 8, 5000),
(2, 'MED002', 'Metformin', 'Glucophage', 'Biguanide', 'tablet', '500mg', 2, 5, 8000),
(3, 'MED003', 'Atorvastatin', 'Lipitor', 'Statin', 'tablet', '20mg', 8, 20, 3000),
(4, 'MED004', 'Amoxicillin', 'Amoxil', 'Penicillin Antibiotic', 'capsule', '500mg', 5, 12, 10000),
(5, 'MED005', 'Paracetamol', 'Panadol', 'Analgesic/Antipyretic', 'tablet', '500mg', 0.5, 2, 50000),
(6, 'MED006', 'Omeprazole', 'Prilosec', 'Proton Pump Inhibitor', 'capsule', '20mg', 4, 10, 6000),
(7, 'MED007', 'Salbutamol', 'Ventolin', 'Bronchodilator', 'inhaler', '100mcg/dose', 80, 200, 500),
(8, 'MED008', 'Losartan', 'Cozaar', 'ARB', 'tablet', '50mg', 6, 15, 4000),
(9, 'MED009', 'Ciprofloxacin', 'Cipro', 'Fluoroquinolone Antibiotic', 'tablet', '500mg', 7, 18, 3000),
(10, 'MED010', 'Aspirin', 'Aspirin', 'Antiplatelet', 'tablet', '81mg', 1, 3, 20000);

-- Insert Prescriptions
INSERT INTO prescriptions (prescription_id, rx_number, visit_id, patient_id, prescribed_by, dispensed_by, status) VALUES
(1, 'RX-2024-000001', 1, 1, 1, 8, 'dispensed'),
(2, 'RX-2024-000002', 2, 2, 2, 8, 'dispensed'),
(3, 'RX-2024-000003', 3, 5, 4, 8, 'dispensed'),
(4, 'RX-2024-000004', 4, 3, 2, 8, 'dispensed'),
(5, 'RX-2024-000005', 9, 1, 1, 8, 'dispensed');

-- Insert Prescription Items
INSERT INTO prescription_items (prescription_id, medication_id, dosage, frequency, duration_days, total_quantity) VALUES
(1, 1, '1 tablet', 'Once daily', 30, 30),
(1, 2, '1 tablet', 'Twice daily', 30, 60),
(2, 5, '1-2 tablets', 'Every 4-6 hours as needed', 5, 10),
(2, 4, '1 capsule', 'Three times daily', 7, 21),
(3, 8, '1 tablet', 'Once daily', 30, 30),
(3, 10, '1 tablet', 'Once daily', 30, 30),
(4, 4, '1 capsule', 'Three times daily', 10, 30),
(4, 5, '2 tablets', 'Every 6 hours', 5, 20),
(5, 1, '1 tablet', 'Once daily', 30, 30),
(5, 2, '1 tablet', 'Twice daily', 30, 60),
(5, 3, '1 tablet', 'Once daily at bedtime', 30, 30);

-- Insert Lab Tests
INSERT INTO lab_tests (test_id, test_code, test_name, category, sample_type, normal_range, unit, cost) VALUES
(1, 'CBC', 'Complete Blood Count', 'Hematology', 'Blood', 'WBC: 4.5-11 x10^9/L', 'Various', 150),
(2, 'FBS', 'Fasting Blood Sugar', 'Chemistry', 'Blood', '70-100 mg/dL', 'mg/dL', 80),
(3, 'HbA1c', 'Hemoglobin A1c', 'Chemistry', 'Blood', '<5.7%', '%', 200),
(4, 'LIPID', 'Lipid Profile', 'Chemistry', 'Blood', 'Total Chol: <200 mg/dL', 'mg/dL', 350),
(5, 'CREA', 'Serum Creatinine', 'Chemistry', 'Blood', '0.6-1.2 mg/dL', 'mg/dL', 120),
(6, 'UA', 'Urinalysis', 'Urinalysis', 'Urine', 'Normal', 'N/A', 80),
(7, 'ECG', 'Electrocardiogram', 'Cardiology', 'N/A', 'Normal sinus rhythm', 'N/A', 200),
(8, 'CXR', 'Chest X-Ray', 'Radiology', 'N/A', 'Normal', 'N/A', 400),
(9, 'URIC', 'Uric Acid', 'Chemistry', 'Blood', 'Male: 3.4-7.0 mg/dL', 'mg/dL', 120),
(10, 'TROP', 'Troponin I', 'Cardiology', 'Blood', '<0.04 ng/mL', 'ng/mL', 800);

-- Insert Lab Orders
INSERT INTO lab_orders (order_id, order_number, visit_id, patient_id, ordered_by, priority, status) VALUES
(1, 'LAB-2024-000001', 1, 1, 1, 'routine', 'completed'),
(2, 'LAB-2024-000002', 3, 5, 4, 'urgent', 'completed'),
(3, 'LAB-2024-000003', 5, 9, 1, 'routine', 'completed'),
(4, 'LAB-2024-000004', 6, 7, 4, 'stat', 'completed'),
(5, 'LAB-2024-000005', 9, 1, 1, 'routine', 'completed');

-- Insert Lab Results
INSERT INTO lab_results (order_id, test_id, result_value, result_numeric, unit, is_abnormal, abnormal_flag, reference_range, performed_by) VALUES
(1, 2, '126', 126, 'mg/dL', TRUE, 'H', '70-100 mg/dL', 9),
(1, 3, '7.8', 7.8, '%', TRUE, 'H', '<5.7%', 9),
(1, 5, '1.3', 1.3, 'mg/dL', TRUE, 'H', '0.6-1.2 mg/dL', 9),
(2, 7, 'ST elevation in V1-V4', NULL, NULL, TRUE, 'A', 'Normal sinus rhythm', 9),
(2, 10, '2.5', 2.5, 'ng/mL', TRUE, 'HH', '<0.04 ng/mL', 9),
(3, 2, '145', 145, 'mg/dL', TRUE, 'H', '70-100 mg/dL', 9),
(3, 3, '8.2', 8.2, '%', TRUE, 'H', '<5.7%', 9),
(3, 5, '1.8', 1.8, 'mg/dL', TRUE, 'H', '0.6-1.2 mg/dL', 9),
(4, 6, 'WBC positive, bacteria seen', NULL, NULL, TRUE, 'A', 'No bacteria', 9),
(5, 2, '112', 112, 'mg/dL', FALSE, NULL, '70-100 mg/dL', 9),
(5, 3, '7.1', 7.1, '%', TRUE, 'H', '<5.7%', 9),
(5, 4, '210', 210, 'mg/dL', TRUE, 'H', '<200 mg/dL', 9);

-- Insert Wards and Beds
INSERT INTO wards (ward_id, name, ward_type, floor, dept_id, total_beds) VALUES
(1, 'Ward 2A - อายุรกรรม', 'general', 2, 1, 30),
(2, 'ICU', 'icu', 2, 6, 10),
(3, 'Ward 3A - ศัลยกรรม', 'general', 3, 2, 20),
(4, 'ห้องฉุกเฉิน', 'emergency', 1, 7, 15);

INSERT INTO beds (bed_id, ward_id, bed_number, bed_type, status) VALUES
(1, 1, '2A-01', 'standard', 'available'),
(2, 1, '2A-02', 'standard', 'occupied'),
(3, 1, '2A-03', 'standard', 'available'),
(4, 2, 'ICU-01', 'icu', 'occupied'),
(5, 2, 'ICU-02', 'icu', 'available');

-- Insert Bills
INSERT INTO bills (bill_id, bill_number, visit_id, patient_id, subtotal, insurance_coverage, patient_responsibility, paid_amount, status, payment_method) VALUES
(1, 'BILL-2024-000001', 1, 1, 1850, 1480, 370, 370, 'paid', 'insurance'),
(2, 'BILL-2024-000002', 2, 2, 650, 0, 650, 650, 'paid', 'cash'),
(3, 'BILL-2024-000003', 3, 5, 2200, 1760, 440, 440, 'paid', 'insurance'),
(4, 'BILL-2024-000004', 4, 3, 980, 784, 196, 196, 'paid', 'insurance'),
(5, 'BILL-2024-000005', 9, 1, 1250, 1000, 250, 250, 'paid', 'insurance');
```

## 3. Healthcare Queries (25+ Queries)

### Query 1: รายชื่อผู้ป่วยพร้อม ประวัติโรคประจำตัวและการแพ้ยา

```sql
SELECT 
    p.hn,
    CONCAT(p.first_name, ' ', p.last_name) AS patient_name,
    FLOOR(DATEDIFF(CURDATE(), p.date_of_birth) / 365.25) AS age,
    p.gender,
    p.blood_type,
    COALESCE(p.allergies, 'ไม่มี') AS allergies,
    COALESCE(p.chronic_conditions, 'ไม่มี') AS chronic_conditions,
    p.insurance_provider,
    COUNT(DISTINCT v.visit_id) AS total_visits,
    MAX(v.visit_date) AS last_visit
FROM patients p
LEFT JOIN visits v ON p.patient_id = v.patient_id
WHERE p.is_active = TRUE
GROUP BY p.patient_id, p.hn, p.first_name, p.last_name, p.date_of_birth, 
         p.gender, p.blood_type, p.allergies, p.chronic_conditions, p.insurance_provider
ORDER BY last_visit DESC NULLS LAST;
```

---

### Query 2: ตารางนัดหมายวันนี้ของแพทย์

```sql
SELECT 
    a.appointment_time,
    CONCAT(p.first_name, ' ', p.last_name) AS patient_name,
    p.hn,
    FLOOR(DATEDIFF(CURDATE(), p.date_of_birth) / 365.25) AS age,
    a.type AS visit_type,
    a.reason,
    a.status,
    COALESCE(p.allergies, '-') AS allergies,
    COALESCE(p.chronic_conditions, '-') AS chronic_conditions,
    -- จำนวนครั้งที่เคยมา
    (SELECT COUNT(*) FROM visits v2 
     WHERE v2.patient_id = p.patient_id) AS previous_visits
FROM appointments a
JOIN patients p ON a.patient_id = p.patient_id
JOIN staff dr ON a.doctor_id = dr.staff_id
WHERE a.appointment_date = CURDATE()
  AND a.doctor_id = 1  -- Doctor ID
  AND a.status NOT IN ('cancelled', 'no_show')
ORDER BY a.appointment_time;
```

---

### Query 3: ประวัติการรักษาของผู้ป่วย (Patient Medical History)

```sql
SELECT 
    v.visit_date,
    v.visit_number,
    CONCAT(dr.title, ' ', dr.first_name, ' ', dr.last_name) AS doctor,
    d.name AS department,
    v.visit_type,
    v.chief_complaint,
    -- วิตามินสัญญาณชีพ
    CONCAT(v.blood_pressure_systolic, '/', v.blood_pressure_diastolic) AS bp_mmhg,
    v.heart_rate AS hr_bpm,
    v.temperature_c AS temp_c,
    v.weight_kg,
    -- การวินิจฉัย
    GROUP_CONCAT(
        CONCAT(diag.icd_code, ': ', ic.description, ' (', diag.diagnosis_type, ')')
        ORDER BY FIELD(diag.diagnosis_type, 'primary', 'secondary', 'complication')
        SEPARATOR ' | '
    ) AS diagnoses,
    -- ยาที่สั่ง
    (SELECT GROUP_CONCAT(m.generic_name, ' ', pi.dosage SEPARATOR ', ')
     FROM prescriptions rx
     JOIN prescription_items pi ON rx.prescription_id = pi.prescription_id
     JOIN medications m ON pi.medication_id = m.medication_id
     WHERE rx.visit_id = v.visit_id) AS medications_prescribed
FROM visits v
JOIN staff dr ON v.doctor_id = dr.staff_id
JOIN departments d ON v.dept_id = d.dept_id
LEFT JOIN diagnoses diag ON v.visit_id = diag.visit_id
LEFT JOIN icd10_codes ic ON diag.icd_code = ic.icd_code
WHERE v.patient_id = 1
GROUP BY v.visit_id, v.visit_date, v.visit_number, dr.title, dr.first_name, dr.last_name, 
         d.name, v.visit_type, v.chief_complaint, v.blood_pressure_systolic, 
         v.blood_pressure_diastolic, v.heart_rate, v.temperature_c, v.weight_kg
ORDER BY v.visit_date DESC;
```

---

### Query 4: ผลตรวจทางห้องปฏิบัติการที่ผิดปกติ

```sql
SELECT 
    CONCAT(p.first_name, ' ', p.last_name) AS patient_name,
    p.hn,
    v.visit_date,
    lo.order_number,
    lt.test_name,
    lr.result_value,
    lr.unit,
    lr.reference_range,
    lr.abnormal_flag,
    CASE lr.abnormal_flag
        WHEN 'H' THEN 'สูงกว่าปกติ'
        WHEN 'L' THEN 'ต่ำกว่าปกติ'
        WHEN 'HH' THEN 'สูงวิกฤต'
        WHEN 'LL' THEN 'ต่ำวิกฤต'
        WHEN 'A' THEN 'ผิดปกติ'
    END AS flag_description,
    CONCAT(dr.title, ' ', dr.first_name, ' ', dr.last_name) AS ordering_doctor
FROM lab_results lr
JOIN lab_orders lo ON lr.order_id = lo.order_id
JOIN lab_tests lt ON lr.test_id = lt.test_id
JOIN visits v ON lo.visit_id = v.visit_id
JOIN patients p ON lo.patient_id = p.patient_id
JOIN staff dr ON lo.ordered_by = dr.staff_id
WHERE lr.is_abnormal = TRUE
ORDER BY 
    CASE lr.abnormal_flag WHEN 'HH' THEN 1 WHEN 'LL' THEN 2 ELSE 3 END,
    v.visit_date DESC;
```

---

### Query 5: รายงานสรุปผู้ป่วยใน (Inpatient Census)

```sql
SELECT 
    w.name AS ward_name,
    w.ward_type,
    COUNT(b.bed_id) AS total_beds,
    SUM(CASE WHEN b.status = 'occupied' THEN 1 ELSE 0 END) AS occupied_beds,
    SUM(CASE WHEN b.status = 'available' THEN 1 ELSE 0 END) AS available_beds,
    SUM(CASE WHEN b.status = 'maintenance' THEN 1 ELSE 0 END) AS maintenance_beds,
    ROUND(
        SUM(CASE WHEN b.status = 'occupied' THEN 1 ELSE 0 END) * 100.0 / COUNT(b.bed_id), 
        1
    ) AS occupancy_rate_pct
FROM wards w
LEFT JOIN beds b ON w.ward_id = b.ward_id
GROUP BY w.ward_id, w.name, w.ward_type
ORDER BY occupancy_rate_pct DESC;
```

---

### Query 6: สถิติโรคที่พบบ่อย (Disease Frequency Statistics)

```sql
SELECT 
    d.icd_code,
    ic.description AS diagnosis_name,
    ic.category,
    COUNT(*) AS diagnosis_count,
    COUNT(DISTINCT diag.visit_id) AS visits_with_diagnosis,
    COUNT(DISTINCT v.patient_id) AS unique_patients,
    ROUND(COUNT(*) * 100.0 / SUM(COUNT(*)) OVER (), 2) AS pct_of_all_diagnoses,
    -- อายุเฉลี่ยของผู้ป่วย
    ROUND(AVG(FLOOR(DATEDIFF(v.visit_date, p.date_of_birth) / 365.25)), 1) AS avg_patient_age,
    -- เพศ
    SUM(CASE WHEN p.gender = 'male' THEN 1 ELSE 0 END) AS male_count,
    SUM(CASE WHEN p.gender = 'female' THEN 1 ELSE 0 END) AS female_count
FROM diagnoses diag
JOIN icd10_codes ic ON diag.icd_code = ic.icd_code
JOIN visits v ON diag.visit_id = v.visit_id
JOIN patients p ON v.patient_id = p.patient_id
-- Alias for the outer use in GROUP BY
INNER JOIN icd10_codes d ON diag.icd_code = d.icd_code
GROUP BY diag.icd_code, ic.description, ic.category
ORDER BY diagnosis_count DESC;
```

---

### Query 7: ประสิทธิภาพแพทย์ (Doctor Performance Metrics)

```sql
SELECT 
    CONCAT(dr.title, ' ', dr.first_name, ' ', dr.last_name) AS doctor_name,
    dep.name AS department,
    COUNT(DISTINCT v.visit_id) AS total_visits,
    COUNT(DISTINCT v.patient_id) AS unique_patients,
    COUNT(DISTINCT a.appointment_id) AS scheduled_appointments,
    SUM(CASE WHEN a.status = 'no_show' THEN 1 ELSE 0 END) AS no_shows,
    ROUND(
        SUM(CASE WHEN a.status = 'no_show' THEN 1 ELSE 0 END) * 100.0 / 
        NULLIF(COUNT(DISTINCT a.appointment_id), 0), 
        1
    ) AS no_show_rate_pct,
    COUNT(DISTINCT rx.prescription_id) AS prescriptions_written,
    COUNT(DISTINCT lo.order_id) AS lab_orders_placed,
    -- Average consultation duration (if tracked)
    SUM(CASE WHEN v.visit_type = 'opd' THEN 1 ELSE 0 END) AS opd_count,
    SUM(CASE WHEN v.visit_type = 'ipd' THEN 1 ELSE 0 END) AS ipd_count,
    SUM(CASE WHEN v.visit_type = 'emergency' THEN 1 ELSE 0 END) AS emergency_count
FROM staff dr
JOIN departments dep ON dr.dept_id = dep.dept_id
LEFT JOIN visits v ON dr.staff_id = v.doctor_id
LEFT JOIN appointments a ON dr.staff_id = a.doctor_id
LEFT JOIN prescriptions rx ON dr.staff_id = rx.prescribed_by
LEFT JOIN lab_orders lo ON dr.staff_id = lo.ordered_by
WHERE dr.role = 'doctor'
  AND dr.is_active = TRUE
GROUP BY dr.staff_id, dr.title, dr.first_name, dr.last_name, dep.name
ORDER BY total_visits DESC;
```

---

### Query 8: รายงานยาที่ใช้บ่อย (Medication Usage Report)

```sql
SELECT 
    m.generic_name,
    m.brand_name,
    m.drug_class,
    m.form,
    m.strength,
    COUNT(pi.item_id) AS times_prescribed,
    SUM(pi.total_quantity) AS total_units_dispensed,
    ROUND(SUM(pi.total_quantity) * m.unit_cost, 2) AS total_cost,
    ROUND(SUM(pi.total_quantity) * m.unit_price, 2) AS total_revenue,
    m.stock_quantity AS current_stock,
    CASE 
        WHEN m.stock_quantity <= m.reorder_level THEN 'REORDER NEEDED'
        WHEN m.stock_quantity <= m.reorder_level * 2 THEN 'LOW STOCK'
        ELSE 'OK'
    END AS stock_status,
    -- Average duration of prescription
    ROUND(AVG(pi.duration_days), 0) AS avg_prescription_days
FROM prescription_items pi
JOIN prescriptions rx ON pi.prescription_id = rx.prescription_id
JOIN medications m ON pi.medication_id = m.medication_id
WHERE rx.status = 'dispensed'
GROUP BY m.medication_id, m.generic_name, m.brand_name, m.drug_class, 
         m.form, m.strength, m.stock_quantity, m.reorder_level, m.unit_cost, m.unit_price
ORDER BY times_prescribed DESC;
```

---

### Query 9: รายงานรายได้โรงพยาบาล (Hospital Revenue Report)

```sql
SELECT 
    DATE_FORMAT(b.billed_at, '%Y-%m') AS month,
    COUNT(DISTINCT b.bill_id) AS total_bills,
    COUNT(DISTINCT b.patient_id) AS unique_patients,
    -- Revenue breakdown
    SUM(b.subtotal) AS gross_revenue,
    SUM(b.insurance_coverage) AS insurance_paid,
    SUM(b.patient_responsibility) AS patient_responsibility,
    SUM(b.paid_amount) AS collected,
    SUM(b.patient_responsibility - b.paid_amount) AS outstanding,
    -- By payment method
    SUM(CASE WHEN b.payment_method = 'insurance' THEN b.paid_amount ELSE 0 END) AS insurance_payment,
    SUM(CASE WHEN b.payment_method = 'cash' THEN b.paid_amount ELSE 0 END) AS cash_payment,
    SUM(CASE WHEN b.payment_method = 'government_scheme' THEN b.paid_amount ELSE 0 END) AS govt_payment,
    -- Collection rate
    ROUND(SUM(b.paid_amount) * 100.0 / NULLIF(SUM(b.patient_responsibility), 0), 1) AS collection_rate_pct
FROM bills b
WHERE b.status != 'cancelled'
GROUP BY DATE_FORMAT(b.billed_at, '%Y-%m')
ORDER BY month DESC;
```

---

### Query 10: ผู้ป่วยที่มีความเสี่ยงสูง (High-Risk Patients)

```sql
-- ผู้ป่วยที่มีโรคหลายโรค ผลเลือดผิดปกติ และมาบ่อย
WITH patient_risk AS (
    SELECT 
        p.patient_id,
        CONCAT(p.first_name, ' ', p.last_name) AS patient_name,
        FLOOR(DATEDIFF(CURDATE(), p.date_of_birth) / 365.25) AS age,
        p.chronic_conditions,
        -- Visit frequency score
        COUNT(DISTINCT v.visit_id) AS total_visits_1yr,
        -- Abnormal lab count
        (SELECT COUNT(*) FROM lab_results lr
         JOIN lab_orders lo ON lr.order_id = lo.order_id
         WHERE lo.patient_id = p.patient_id
           AND lr.is_abnormal = TRUE
           AND lr.abnormal_flag IN ('HH', 'LL')) AS critical_lab_count,
        -- Chronic disease count
        CASE WHEN p.chronic_conditions IS NOT NULL THEN
            LENGTH(p.chronic_conditions) - LENGTH(REPLACE(p.chronic_conditions, ',', '')) + 1
        ELSE 0 END AS chronic_disease_count,
        -- Days since last visit
        DATEDIFF(CURDATE(), MAX(v.visit_date)) AS days_since_last_visit,
        -- Emergency visits
        SUM(CASE WHEN v.visit_type = 'emergency' THEN 1 ELSE 0 END) AS emergency_visits
    FROM patients p
    LEFT JOIN visits v ON p.patient_id = v.patient_id
        AND v.visit_date >= DATE_SUB(CURDATE(), INTERVAL 1 YEAR)
    WHERE p.is_active = TRUE
    GROUP BY p.patient_id, p.first_name, p.last_name, p.date_of_birth, 
             p.chronic_conditions
)
SELECT 
    patient_id,
    patient_name,
    age,
    chronic_conditions,
    total_visits_1yr,
    critical_lab_count,
    chronic_disease_count,
    emergency_visits,
    days_since_last_visit,
    -- Risk Score (simplified)
    (chronic_disease_count * 20) + 
    (critical_lab_count * 30) + 
    (emergency_visits * 15) +
    (CASE WHEN age > 70 THEN 20 WHEN age > 60 THEN 10 ELSE 0 END) AS risk_score,
    CASE 
        WHEN (chronic_disease_count * 20 + critical_lab_count * 30 + emergency_visits * 15) >= 80 
        THEN 'HIGH RISK'
        WHEN (chronic_disease_count * 20 + critical_lab_count * 30 + emergency_visits * 15) >= 40 
        THEN 'MEDIUM RISK'
        ELSE 'LOW RISK'
    END AS risk_level
FROM patient_risk
ORDER BY risk_score DESC;
```

---

### Query 11: ตรวจสอบ Drug Interaction (การแพ้ยา)

```sql
-- ตรวจสอบว่าผู้ป่วยที่มีประวัติแพ้ยาได้รับยาที่อาจมีปัญหา
SELECT 
    CONCAT(p.first_name, ' ', p.last_name) AS patient_name,
    p.hn,
    p.allergies,
    rx.rx_number,
    m.generic_name AS prescribed_drug,
    m.drug_class,
    -- Flag ถ้ายาที่สั่งอยู่ใน class ที่แพ้
    CASE 
        WHEN p.allergies LIKE '%Penicillin%' AND m.drug_class LIKE '%Penicillin%' 
        THEN 'ALLERGY ALERT: Penicillin!'
        WHEN p.allergies LIKE '%Sulfa%' AND m.drug_class LIKE '%Sulfonamide%' 
        THEN 'ALLERGY ALERT: Sulfa!'
        WHEN p.allergies LIKE '%NSAIDs%' AND m.drug_class LIKE '%NSAID%' 
        THEN 'ALLERGY ALERT: NSAIDs!'
        WHEN p.allergies LIKE '%Aspirin%' AND m.generic_name = 'Aspirin'
        THEN 'ALLERGY ALERT: Aspirin!'
        WHEN p.allergies LIKE '%Codeine%' AND m.drug_class LIKE '%Opioid%'
        THEN 'ALLERGY ALERT: Opioid!'
        ELSE 'No known interaction'
    END AS allergy_check
FROM prescription_items pi
JOIN prescriptions rx ON pi.prescription_id = rx.prescription_id
JOIN medications m ON pi.medication_id = m.medication_id
JOIN patients p ON rx.patient_id = p.patient_id
WHERE p.allergies IS NOT NULL
  AND p.allergies != ''
  AND rx.status != 'cancelled'
ORDER BY allergy_check DESC;
```

---

### Query 12: รายงานผู้ป่วย OPD/IPD ประจำเดือน

```sql
SELECT 
    DATE_FORMAT(v.visit_date, '%Y-%m') AS month,
    SUM(CASE WHEN v.visit_type = 'opd' THEN 1 ELSE 0 END) AS opd_visits,
    SUM(CASE WHEN v.visit_type = 'ipd' THEN 1 ELSE 0 END) AS ipd_visits,
    SUM(CASE WHEN v.visit_type = 'emergency' THEN 1 ELSE 0 END) AS er_visits,
    COUNT(DISTINCT v.visit_id) AS total_visits,
    COUNT(DISTINCT v.patient_id) AS unique_patients,
    -- New patients (first visit ever)
    COUNT(DISTINCT CASE WHEN v.visit_id = (
        SELECT MIN(v2.visit_id) FROM visits v2 WHERE v2.patient_id = v.patient_id
    ) THEN v.patient_id END) AS new_patients,
    -- Average stay for IPD (days)
    ROUND(AVG(CASE WHEN v.visit_type = 'ipd' AND v.discharge_date IS NOT NULL
        THEN TIMESTAMPDIFF(HOUR, v.visit_date, v.discharge_date) / 24.0
        ELSE NULL END), 1) AS avg_ipd_stay_days
FROM visits v
GROUP BY DATE_FORMAT(v.visit_date, '%Y-%m')
ORDER BY month DESC;
```

---

### Query 13: ตรวจสอบนัดหมายที่ขาด/ยกเลิก (No-Show Analysis)

```sql
SELECT 
    DATE_FORMAT(a.appointment_date, '%Y-%m') AS month,
    COUNT(*) AS total_scheduled,
    SUM(CASE WHEN a.status = 'completed' THEN 1 ELSE 0 END) AS completed,
    SUM(CASE WHEN a.status = 'no_show' THEN 1 ELSE 0 END) AS no_shows,
    SUM(CASE WHEN a.status = 'cancelled' THEN 1 ELSE 0 END) AS cancelled,
    ROUND(SUM(CASE WHEN a.status = 'no_show' THEN 1 ELSE 0 END) * 100.0 / COUNT(*), 1) AS no_show_rate_pct,
    ROUND(SUM(CASE WHEN a.status = 'cancelled' THEN 1 ELSE 0 END) * 100.0 / COUNT(*), 1) AS cancellation_rate_pct,
    -- Doctor with most no-shows
    (SELECT CONCAT(dr2.first_name, ' ', dr2.last_name)
     FROM appointments a2 
     JOIN staff dr2 ON a2.doctor_id = dr2.staff_id
     WHERE a2.status = 'no_show'
       AND DATE_FORMAT(a2.appointment_date, '%Y-%m') = DATE_FORMAT(a.appointment_date, '%Y-%m')
     GROUP BY a2.doctor_id
     ORDER BY COUNT(*) DESC
     LIMIT 1) AS doctor_with_most_noshows
FROM appointments a
GROUP BY DATE_FORMAT(a.appointment_date, '%Y-%m')
ORDER BY month DESC;
```

---

### Query 14: รายงาน Compliance สำหรับโรงพยาบาล (Hospital Compliance Report)

```sql
-- ตรวจสอบว่า License ของแพทย์/พยาบาลใกล้หมดอายุ
SELECT 
    s.staff_number,
    CONCAT(s.title, ' ', s.first_name, ' ', s.last_name) AS staff_name,
    s.role,
    d.name AS department,
    s.license_number,
    s.license_expiry,
    DATEDIFF(s.license_expiry, CURDATE()) AS days_until_expiry,
    CASE 
        WHEN s.license_expiry < CURDATE() THEN 'EXPIRED!'
        WHEN DATEDIFF(s.license_expiry, CURDATE()) <= 30 THEN 'EXPIRES WITHIN 30 DAYS'
        WHEN DATEDIFF(s.license_expiry, CURDATE()) <= 90 THEN 'EXPIRES WITHIN 90 DAYS'
        ELSE 'VALID'
    END AS license_status
FROM staff s
LEFT JOIN departments d ON s.dept_id = d.dept_id
WHERE s.is_active = TRUE
  AND s.license_number IS NOT NULL
ORDER BY days_until_expiry ASC;
```

---

### Query 15: Blood Type Distribution

```sql
SELECT 
    blood_type,
    COUNT(*) AS patient_count,
    ROUND(COUNT(*) * 100.0 / SUM(COUNT(*)) OVER (), 1) AS percentage,
    RPAD('', ROUND(COUNT(*) * 40.0 / MAX(COUNT(*)) OVER (), 0), '█') AS bar_chart
FROM patients
WHERE is_active = TRUE AND blood_type != 'unknown'
GROUP BY blood_type
ORDER BY patient_count DESC;
```

---

### Query 16: ค้นหาผู้ป่วยที่ควรนัดติดตาม (Follow-up Needed)

```sql
-- ผู้ป่วยที่มีโรคเรื้อรังแต่ไม่ได้มาโรงพยาบาลในรอบ 90 วัน
SELECT 
    p.hn,
    CONCAT(p.first_name, ' ', p.last_name) AS patient_name,
    p.phone,
    p.chronic_conditions,
    MAX(v.visit_date) AS last_visit_date,
    DATEDIFF(CURDATE(), MAX(v.visit_date)) AS days_since_last_visit,
    -- แพทย์ที่รักษาล่าสุด
    (SELECT CONCAT(dr.title, ' ', dr.first_name, ' ', dr.last_name)
     FROM visits v3
     JOIN staff dr ON v3.doctor_id = dr.staff_id
     WHERE v3.patient_id = p.patient_id
     ORDER BY v3.visit_date DESC
     LIMIT 1) AS last_treating_doctor,
    -- มีนัดต่อหรือไม่
    CASE WHEN EXISTS (
        SELECT 1 FROM appointments a
        WHERE a.patient_id = p.patient_id
          AND a.appointment_date >= CURDATE()
          AND a.status NOT IN ('cancelled')
    ) THEN 'มีนัด' ELSE 'ไม่มีนัด' END AS has_upcoming_appointment
FROM patients p
LEFT JOIN visits v ON p.patient_id = v.patient_id
WHERE p.chronic_conditions IS NOT NULL
  AND p.is_active = TRUE
GROUP BY p.patient_id, p.hn, p.first_name, p.last_name, p.phone, p.chronic_conditions
HAVING last_visit_date IS NULL OR DATEDIFF(CURDATE(), MAX(v.visit_date)) > 90
ORDER BY days_since_last_visit DESC;
```

---

## แบบฝึกหัด (Challenge Exercises)

1. **Patient Risk Stratification**: เขียน Query ที่ใช้ Machine Learning-style scoring เพื่อคำนวณ 30-day Readmission Risk โดยพิจารณาจาก: อายุ, จำนวนโรคเรื้อรัง, จำนวนยา, จำนวนครั้ง admit ในปีที่แล้ว

2. **HL7-style Report**: สร้าง Query ที่ออก Summary ผู้ป่วยในรูปแบบที่ใกล้เคียง HL7 FHIR Patient Summary

3. **Drug Utilization Review**: เขียน Query วิเคราะห์ว่าแพทย์แต่ละคนสั่งยา generic vs brand-name ในสัดส่วนเท่าไร

4. **Average Length of Stay**: คำนวณ Average Length of Stay (ALOS) แยกตาม Diagnosis Group และเปรียบเทียบกับ benchmark มาตรฐาน

5. **Bed Utilization Timeline**: เขียน Query แสดง % การใช้เตียงในแต่ละ Ward รายวัน สำหรับ 30 วันที่ผ่านมา

6. **Readmission Report**: หาผู้ป่วยที่ต้อง readmit ภายใน 30 วันหลัง discharge (30-day Readmission Rate)

7. **Doctor Workload Balance**: วิเคราะห์ว่า Workload ของแพทย์ในแต่ละแผนกสม่ำเสมอหรือไม่ และแนะนำการ Rebalance

8. **Lab Result Trends**: เขียน Query ที่แสดง Trend ของผล Lab (เช่น HbA1c, Creatinine) ของผู้ป่วยรายบุคคลตลอดระยะเวลา

9. **Insurance Claim Success Rate**: วิเคราะห์อัตราความสำเร็จในการเรียกเบี้ยประกัน แยกตาม Insurance Provider

10. **Mortality and Morbidity Report**: สร้าง M&M Report รายเดือนแสดงสถิติ: จำนวนผู้ป่วย, ผู้เสียชีวิต, ภาวะแทรกซ้อนสำคัญ
