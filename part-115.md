# Part 115: HR Management System

## บทนำ (Introduction)

ระบบ HR Management System (HRMS) ครอบคลุมตั้งแต่วันที่พนักงานเข้าทำงานจนถึงวันเกษียณ (Employee Lifecycle) บทนี้จะออกแบบ Schema ที่ครบถ้วนและเขียน Queries สำหรับการจัดการทรัพยากรมนุษย์

## 1. Complete HR Schema

```sql
-- =========================================
-- HR MANAGEMENT SYSTEM - COMPLETE DDL
-- =========================================

CREATE DATABASE hr_db
    CHARACTER SET utf8mb4
    COLLATE utf8mb4_unicode_ci;

USE hr_db;

-- =========================================
-- SECTION 1: ORGANIZATION STRUCTURE
-- =========================================

-- ตาราง Companies (บริษัทในเครือ)
CREATE TABLE companies (
    company_id      INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    company_code    VARCHAR(10) NOT NULL UNIQUE,
    name            VARCHAR(300) NOT NULL,
    tax_id          VARCHAR(20),
    address         TEXT,
    country         VARCHAR(100) DEFAULT 'Thailand',
    is_active       BOOLEAN DEFAULT TRUE
) ENGINE=InnoDB;

-- ตาราง Departments (แผนก)
CREATE TABLE departments (
    dept_id         INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    company_id      INT UNSIGNED NOT NULL,
    parent_dept_id  INT UNSIGNED,
    code            VARCHAR(20) NOT NULL UNIQUE,
    name            VARCHAR(200) NOT NULL,
    description     TEXT,
    cost_center     VARCHAR(20),
    head_employee_id INT UNSIGNED,
    is_active       BOOLEAN DEFAULT TRUE,
    FOREIGN KEY (company_id) REFERENCES companies(company_id),
    FOREIGN KEY (parent_dept_id) REFERENCES departments(dept_id) ON DELETE SET NULL,
    INDEX idx_parent (parent_dept_id),
    INDEX idx_company (company_id)
) ENGINE=InnoDB;

-- ตาราง Job Positions (ตำแหน่งงาน)
CREATE TABLE job_positions (
    position_id     INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    dept_id         INT UNSIGNED NOT NULL,
    title           VARCHAR(200) NOT NULL,
    job_code        VARCHAR(20) NOT NULL UNIQUE,
    job_level       ENUM('C1','C2','C3','M1','M2','M3','E1','E2','E3') NOT NULL COMMENT 'C=Clerical,M=Management,E=Executive',
    min_salary      DECIMAL(12,2) NOT NULL,
    max_salary      DECIMAL(12,2) NOT NULL,
    headcount_budget INT DEFAULT 1,
    current_headcount INT DEFAULT 0,
    is_active       BOOLEAN DEFAULT TRUE,
    FOREIGN KEY (dept_id) REFERENCES departments(dept_id)
) ENGINE=InnoDB;

-- =========================================
-- SECTION 2: EMPLOYEES
-- =========================================

-- ตาราง Employees (พนักงาน)
CREATE TABLE employees (
    employee_id     INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    employee_number VARCHAR(20) NOT NULL UNIQUE,
    company_id      INT UNSIGNED NOT NULL,
    dept_id         INT UNSIGNED NOT NULL,
    position_id     INT UNSIGNED NOT NULL,
    manager_id      INT UNSIGNED,
    -- Personal Info
    first_name      VARCHAR(100) NOT NULL,
    last_name       VARCHAR(100) NOT NULL,
    first_name_en   VARCHAR(100),
    last_name_en    VARCHAR(100),
    date_of_birth   DATE NOT NULL,
    gender          ENUM('male','female','other') NOT NULL,
    national_id     VARCHAR(20) UNIQUE,
    passport_number VARCHAR(30),
    nationality     VARCHAR(100) DEFAULT 'Thai',
    marital_status  ENUM('single','married','divorced','widowed') DEFAULT 'single',
    -- Contact
    email           VARCHAR(255) UNIQUE,
    email_work      VARCHAR(255) UNIQUE,
    phone           VARCHAR(20),
    phone_work      VARCHAR(20),
    address         TEXT,
    -- Employment
    hire_date       DATE NOT NULL,
    probation_end   DATE,
    confirmation_date DATE,
    contract_type   ENUM('permanent','contract','part_time','internship') DEFAULT 'permanent',
    employment_type ENUM('full_time','part_time') DEFAULT 'full_time',
    -- Status
    status          ENUM('active','on_leave','resigned','terminated','retired','deceased') DEFAULT 'active',
    termination_date DATE,
    termination_reason TEXT,
    -- Bank for Payroll
    bank_name       VARCHAR(100),
    bank_account    VARCHAR(30),
    tax_id          VARCHAR(20),
    -- Education
    education_level ENUM('below_bachelor','bachelor','master','phd','other'),
    created_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    FOREIGN KEY (company_id) REFERENCES companies(company_id),
    FOREIGN KEY (dept_id) REFERENCES departments(dept_id),
    FOREIGN KEY (position_id) REFERENCES job_positions(position_id),
    FOREIGN KEY (manager_id) REFERENCES employees(employee_id) ON DELETE SET NULL,
    INDEX idx_dept (dept_id),
    INDEX idx_manager (manager_id),
    INDEX idx_status (status),
    INDEX idx_hire_date (hire_date)
) ENGINE=InnoDB;

-- =========================================
-- SECTION 3: SALARY & COMPENSATION
-- =========================================

-- ตาราง Salary History (ประวัติเงินเดือน)
CREATE TABLE salary_history (
    salary_id       INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    employee_id     INT UNSIGNED NOT NULL,
    effective_date  DATE NOT NULL,
    base_salary     DECIMAL(12,2) NOT NULL,
    -- Allowances
    housing_allowance DECIMAL(10,2) DEFAULT 0,
    transport_allowance DECIMAL(10,2) DEFAULT 0,
    meal_allowance  DECIMAL(10,2) DEFAULT 0,
    phone_allowance DECIMAL(10,2) DEFAULT 0,
    other_allowance DECIMAL(10,2) DEFAULT 0,
    -- Total
    total_compensation DECIMAL(12,2) GENERATED ALWAYS AS (
        base_salary + housing_allowance + transport_allowance + 
        meal_allowance + phone_allowance + other_allowance
    ) STORED,
    change_reason   ENUM('initial','annual_review','promotion','transfer','adjustment','market_correction'),
    change_pct      DECIMAL(5,2),
    approved_by     INT UNSIGNED,
    notes           TEXT,
    created_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (employee_id) REFERENCES employees(employee_id),
    FOREIGN KEY (approved_by) REFERENCES employees(employee_id) ON DELETE SET NULL,
    INDEX idx_employee (employee_id),
    INDEX idx_effective (effective_date)
) ENGINE=InnoDB;

-- ตาราง Bonus Records (โบนัส)
CREATE TABLE bonus_records (
    bonus_id        INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    employee_id     INT UNSIGNED NOT NULL,
    bonus_type      ENUM('annual','performance','signing','spot','project','holiday') NOT NULL,
    amount          DECIMAL(12,2) NOT NULL,
    bonus_period    VARCHAR(20) COMMENT 'e.g., 2024-H1',
    performance_rating DECIMAL(3,1),
    paid_date       DATE NOT NULL,
    notes           TEXT,
    FOREIGN KEY (employee_id) REFERENCES employees(employee_id),
    INDEX idx_employee (employee_id)
) ENGINE=InnoDB;

-- =========================================
-- SECTION 4: PERFORMANCE MANAGEMENT
-- =========================================

-- ตาราง Performance Reviews (การประเมินผลงาน)
CREATE TABLE performance_reviews (
    review_id       INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    employee_id     INT UNSIGNED NOT NULL,
    reviewer_id     INT UNSIGNED NOT NULL,
    review_period   VARCHAR(20) NOT NULL COMMENT 'e.g., 2024-H1, 2024-Annual',
    review_type     ENUM('annual','mid_year','probation','360','peer') DEFAULT 'annual',
    -- Ratings (1-5 scale)
    performance_score DECIMAL(3,1),
    attitude_score  DECIMAL(3,1),
    leadership_score DECIMAL(3,1),
    teamwork_score  DECIMAL(3,1),
    overall_rating  DECIMAL(3,1),
    rating_label    ENUM('outstanding','exceeds_expectations','meets_expectations','needs_improvement','unsatisfactory'),
    -- KPIs
    kpi_achievement_pct DECIMAL(5,2),
    -- Narrative
    strengths       TEXT,
    improvements    TEXT,
    goals_next_period TEXT,
    reviewer_comments TEXT,
    employee_comments TEXT,
    -- Status
    status          ENUM('draft','submitted','approved','acknowledged') DEFAULT 'draft',
    submitted_at    TIMESTAMP NULL,
    approved_at     TIMESTAMP NULL,
    FOREIGN KEY (employee_id) REFERENCES employees(employee_id),
    FOREIGN KEY (reviewer_id) REFERENCES employees(employee_id),
    INDEX idx_employee (employee_id),
    INDEX idx_period (review_period)
) ENGINE=InnoDB;

-- =========================================
-- SECTION 5: LEAVE MANAGEMENT
-- =========================================

-- ตาราง Leave Types (ประเภทการลา)
CREATE TABLE leave_types (
    leave_type_id   INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    name            VARCHAR(100) NOT NULL,
    code            VARCHAR(20) NOT NULL UNIQUE,
    days_per_year   DECIMAL(4,1) DEFAULT 0,
    is_paid         BOOLEAN DEFAULT TRUE,
    requires_approval BOOLEAN DEFAULT TRUE,
    advance_notice_days INT DEFAULT 1,
    carry_forward   BOOLEAN DEFAULT FALSE,
    max_carry_forward_days DECIMAL(4,1) DEFAULT 0,
    description     TEXT
) ENGINE=InnoDB;

-- ตาราง Leave Balances (ยอดวันลาคงเหลือ)
CREATE TABLE leave_balances (
    balance_id      INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    employee_id     INT UNSIGNED NOT NULL,
    leave_type_id   INT UNSIGNED NOT NULL,
    year            YEAR NOT NULL,
    entitlement_days DECIMAL(4,1) NOT NULL,
    taken_days      DECIMAL(4,1) DEFAULT 0,
    pending_days    DECIMAL(4,1) DEFAULT 0,
    carried_forward DECIMAL(4,1) DEFAULT 0,
    remaining_days  DECIMAL(4,1) GENERATED ALWAYS AS (
        entitlement_days + carried_forward - taken_days - pending_days
    ) STORED,
    FOREIGN KEY (employee_id) REFERENCES employees(employee_id),
    FOREIGN KEY (leave_type_id) REFERENCES leave_types(leave_type_id),
    UNIQUE KEY uk_emp_type_year (employee_id, leave_type_id, year),
    INDEX idx_employee (employee_id)
) ENGINE=InnoDB;

-- ตาราง Leave Requests (คำขอลา)
CREATE TABLE leave_requests (
    request_id      INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    employee_id     INT UNSIGNED NOT NULL,
    leave_type_id   INT UNSIGNED NOT NULL,
    start_date      DATE NOT NULL,
    end_date        DATE NOT NULL,
    total_days      DECIMAL(4,1) NOT NULL,
    reason          TEXT,
    status          ENUM('pending','approved','rejected','cancelled') DEFAULT 'pending',
    approved_by     INT UNSIGNED,
    approved_at     TIMESTAMP NULL,
    rejection_reason TEXT,
    created_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (employee_id) REFERENCES employees(employee_id),
    FOREIGN KEY (leave_type_id) REFERENCES leave_types(leave_type_id),
    FOREIGN KEY (approved_by) REFERENCES employees(employee_id) ON DELETE SET NULL,
    INDEX idx_employee (employee_id),
    INDEX idx_dates (start_date, end_date),
    INDEX idx_status (status)
) ENGINE=InnoDB;

-- =========================================
-- SECTION 6: TRAINING
-- =========================================

-- ตาราง Training Courses (หลักสูตรฝึกอบรม)
CREATE TABLE training_courses (
    course_id       INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    course_code     VARCHAR(20) NOT NULL UNIQUE,
    title           VARCHAR(300) NOT NULL,
    category        VARCHAR(100),
    description     TEXT,
    duration_hours  DECIMAL(5,1),
    delivery_type   ENUM('classroom','online','blended','on_job') DEFAULT 'classroom',
    provider        VARCHAR(200),
    cost_per_person DECIMAL(10,2) DEFAULT 0,
    is_mandatory    BOOLEAN DEFAULT FALSE,
    is_active       BOOLEAN DEFAULT TRUE
) ENGINE=InnoDB;

-- ตาราง Training Records (ประวัติการฝึกอบรม)
CREATE TABLE training_records (
    record_id       INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    employee_id     INT UNSIGNED NOT NULL,
    course_id       INT UNSIGNED NOT NULL,
    training_date   DATE NOT NULL,
    completion_date DATE,
    score           DECIMAL(5,2),
    pass_score      DECIMAL(5,2),
    is_passed       BOOLEAN,
    certificate_number VARCHAR(100),
    certificate_expiry DATE,
    cost            DECIMAL(10,2),
    notes           TEXT,
    FOREIGN KEY (employee_id) REFERENCES employees(employee_id),
    FOREIGN KEY (course_id) REFERENCES training_courses(course_id),
    INDEX idx_employee (employee_id),
    INDEX idx_course (course_id)
) ENGINE=InnoDB;

-- =========================================
-- SECTION 7: BENEFITS
-- =========================================

-- ตาราง Benefit Plans (แผนสวัสดิการ)
CREATE TABLE benefit_plans (
    plan_id         INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    name            VARCHAR(200) NOT NULL,
    benefit_type    ENUM('health_insurance','life_insurance','dental','vision','pension','provident_fund','other') NOT NULL,
    provider        VARCHAR(200),
    employer_contribution_pct DECIMAL(5,2) DEFAULT 0,
    employee_contribution_pct DECIMAL(5,2) DEFAULT 0,
    annual_limit    DECIMAL(12,2),
    description     TEXT,
    is_active       BOOLEAN DEFAULT TRUE
) ENGINE=InnoDB;

-- ตาราง Employee Benefits (สวัสดิการพนักงาน)
CREATE TABLE employee_benefits (
    enrollment_id   INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    employee_id     INT UNSIGNED NOT NULL,
    plan_id         INT UNSIGNED NOT NULL,
    effective_date  DATE NOT NULL,
    end_date        DATE,
    employee_contribution DECIMAL(10,2) DEFAULT 0,
    employer_contribution DECIMAL(10,2) DEFAULT 0,
    is_active       BOOLEAN DEFAULT TRUE,
    FOREIGN KEY (employee_id) REFERENCES employees(employee_id),
    FOREIGN KEY (plan_id) REFERENCES benefit_plans(plan_id),
    INDEX idx_employee (employee_id)
) ENGINE=InnoDB;

-- =========================================
-- SECTION 8: PAYROLL
-- =========================================

-- ตาราง Payroll Runs (รอบการจ่ายเงินเดือน)
CREATE TABLE payroll_runs (
    run_id          INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    period_year     YEAR NOT NULL,
    period_month    TINYINT NOT NULL,
    run_date        DATE NOT NULL,
    status          ENUM('draft','calculated','approved','paid','reversed') DEFAULT 'draft',
    total_employees INT DEFAULT 0,
    total_gross     DECIMAL(15,2) DEFAULT 0,
    total_deductions DECIMAL(15,2) DEFAULT 0,
    total_net       DECIMAL(15,2) DEFAULT 0,
    approved_by     INT UNSIGNED,
    approved_at     TIMESTAMP NULL,
    paid_at         TIMESTAMP NULL,
    UNIQUE KEY uk_period (period_year, period_month)
) ENGINE=InnoDB;

-- ตาราง Payslips (สลิปเงินเดือน)
CREATE TABLE payslips (
    payslip_id      INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    run_id          INT UNSIGNED NOT NULL,
    employee_id     INT UNSIGNED NOT NULL,
    -- Earnings
    base_salary     DECIMAL(12,2) NOT NULL,
    housing_allowance DECIMAL(10,2) DEFAULT 0,
    transport_allowance DECIMAL(10,2) DEFAULT 0,
    meal_allowance  DECIMAL(10,2) DEFAULT 0,
    other_allowance DECIMAL(10,2) DEFAULT 0,
    overtime_pay    DECIMAL(10,2) DEFAULT 0,
    bonus           DECIMAL(12,2) DEFAULT 0,
    gross_pay       DECIMAL(12,2) NOT NULL,
    -- Deductions
    social_security DECIMAL(10,2) DEFAULT 0,
    income_tax      DECIMAL(12,2) DEFAULT 0,
    provident_fund  DECIMAL(10,2) DEFAULT 0,
    loan_deduction  DECIMAL(10,2) DEFAULT 0,
    other_deductions DECIMAL(10,2) DEFAULT 0,
    total_deductions DECIMAL(12,2) NOT NULL,
    -- Net
    net_pay         DECIMAL(12,2) NOT NULL,
    -- Working days
    working_days    DECIMAL(4,1) DEFAULT 0,
    leave_taken_days DECIMAL(4,1) DEFAULT 0,
    FOREIGN KEY (run_id) REFERENCES payroll_runs(run_id),
    FOREIGN KEY (employee_id) REFERENCES employees(employee_id),
    UNIQUE KEY uk_run_employee (run_id, employee_id),
    INDEX idx_employee (employee_id)
) ENGINE=InnoDB;
```

## 2. Sample Data

```sql
-- Insert Companies
INSERT INTO companies (company_id, company_code, name) VALUES
(1, 'ABC', 'ABC Company Limited'),
(2, 'XYZ', 'XYZ Technology Co., Ltd.');

-- Insert Departments (Hierarchical)
INSERT INTO departments (dept_id, company_id, parent_dept_id, code, name, cost_center) VALUES
(1, 1, NULL, 'EXEC', 'Executive', 'CC001'),
(2, 1, 1, 'HR', 'Human Resources', 'CC002'),
(3, 1, 1, 'FIN', 'Finance & Accounting', 'CC003'),
(4, 1, 1, 'IT', 'Information Technology', 'CC004'),
(5, 1, 4, 'DEV', 'Software Development', 'CC005'),
(6, 1, 4, 'INFRA', 'Infrastructure', 'CC006'),
(7, 1, 1, 'SALES', 'Sales & Marketing', 'CC007'),
(8, 1, 7, 'BSALES', 'Business Sales', 'CC008'),
(9, 1, 7, 'MKTG', 'Marketing', 'CC009'),
(10, 1, 1, 'OPS', 'Operations', 'CC010');

-- Insert Job Positions
INSERT INTO job_positions (position_id, dept_id, title, job_code, job_level, min_salary, max_salary, headcount_budget) VALUES
(1, 1, 'Chief Executive Officer', 'CEO', 'E3', 500000, 1000000, 1),
(2, 2, 'HR Manager', 'HRM', 'M2', 80000, 150000, 1),
(3, 2, 'HR Officer', 'HRO', 'C3', 25000, 50000, 3),
(4, 3, 'Finance Manager', 'FINM', 'M2', 90000, 170000, 1),
(5, 3, 'Accountant', 'ACC', 'C2', 20000, 45000, 4),
(6, 5, 'Senior Software Engineer', 'SSE', 'M1', 80000, 150000, 5),
(7, 5, 'Software Engineer', 'SE', 'C3', 35000, 70000, 10),
(8, 7, 'Sales Manager', 'SALM', 'M2', 70000, 130000, 1),
(9, 8, 'Sales Executive', 'SE2', 'C2', 25000, 60000, 8),
(10, 9, 'Marketing Manager', 'MKTM', 'M2', 75000, 140000, 1);

-- Insert Employees
INSERT INTO employees (employee_id, employee_number, company_id, dept_id, position_id, manager_id,
    first_name, last_name, date_of_birth, gender, national_id, email_work, phone, 
    hire_date, probation_end, confirmation_date, contract_type, status, education_level) VALUES
(1, 'EMP-000001', 1, 1, 1, NULL, 'วิชัย', 'วงศ์สกุล', '1970-01-15', 'male', '1001234567890', 'vichai.c@abc.com', '0812345678', '2015-01-01', NULL, '2015-01-01', 'permanent', 'active', 'master'),
(2, 'EMP-000002', 1, 2, 2, 1, 'สุนัน', 'ทองทวี', '1980-05-20', 'female', '2001234567890', 'sunan.t@abc.com', '0823456789', '2018-03-01', '2018-06-01', '2018-06-01', 'permanent', 'active', 'bachelor'),
(3, 'EMP-000003', 1, 2, 3, 2, 'มานะ', 'ขยันดี', '1992-08-10', 'male', '3001234567890', 'mana.k@abc.com', '0834567890', '2020-07-01', '2020-10-01', '2020-10-01', 'permanent', 'active', 'bachelor'),
(4, 'EMP-000004', 1, 3, 4, 1, 'ธนพร', 'เงินทอง', '1978-11-25', 'female', '4001234567890', 'tanaporn.n@abc.com', '0845678901', '2016-06-01', NULL, '2016-06-01', 'permanent', 'active', 'master'),
(5, 'EMP-000005', 1, 5, 6, 1, 'กฤษฎา', 'โปรแกรมเมอร์', '1988-03-14', 'male', '5001234567890', 'kritsada.p@abc.com', '0856789012', '2019-09-01', '2019-12-01', '2019-12-01', 'permanent', 'active', 'bachelor'),
(6, 'EMP-000006', 1, 5, 7, 5, 'พิมพ์นารา', 'โค้ดสวย', '1995-07-07', 'female', '6001234567890', 'pimnara.k@abc.com', '0867890123', '2022-01-10', '2022-04-10', '2022-04-10', 'permanent', 'active', 'bachelor'),
(7, 'EMP-000007', 1, 8, 9, 1, 'สมศักดิ์', 'ขายเก่ง', '1985-09-22', 'male', '7001234567890', 'somsak.s@abc.com', '0878901234', '2017-04-01', NULL, '2017-04-01', 'permanent', 'active', 'bachelor'),
(8, 'EMP-000008', 1, 5, 7, 5, 'วันใหม่', 'นักพัฒนา', '1997-12-03', 'female', '8001234567890', 'wannmai.n@abc.com', '0889012345', '2023-06-01', '2023-09-01', NULL, 'permanent', 'active', 'bachelor'),
(9, 'EMP-000009', 1, 3, 5, 4, 'อำพล', 'บัญชีดี', '1990-02-18', 'male', '9001234567890', 'ampon.b@abc.com', '0890123456', '2021-02-15', '2021-05-15', '2021-05-15', 'permanent', 'active', 'bachelor'),
(10, 'EMP-000010', 1, 2, 3, 2, 'นฤมล', 'คนรู้ใจ', '1993-06-28', 'female', '1101234567890', 'naruemon.k@abc.com', '0801234567', '2021-08-01', '2021-11-01', '2021-11-01', 'permanent', 'active', 'bachelor');

-- Set department heads
UPDATE departments SET head_employee_id = 1 WHERE dept_id = 1;
UPDATE departments SET head_employee_id = 2 WHERE dept_id = 2;
UPDATE departments SET head_employee_id = 4 WHERE dept_id = 3;
UPDATE departments SET head_employee_id = 5 WHERE dept_id = 5;

-- Insert Salary History
INSERT INTO salary_history (employee_id, effective_date, base_salary, housing_allowance, transport_allowance, meal_allowance, change_reason, change_pct, approved_by) VALUES
(1, '2015-01-01', 500000, 50000, 10000, 5000, 'initial', NULL, NULL),
(1, '2024-01-01', 550000, 50000, 10000, 5000, 'annual_review', 10.0, NULL),
(2, '2018-03-01', 80000, 10000, 3000, 2000, 'initial', NULL, NULL),
(2, '2020-01-01', 90000, 10000, 3000, 2000, 'annual_review', 12.5, 1),
(2, '2023-01-01', 100000, 12000, 3000, 2000, 'annual_review', 11.1, 1),
(3, '2020-07-01', 30000, 5000, 2000, 1500, 'initial', NULL, NULL),
(3, '2023-01-01', 35000, 5000, 2000, 1500, 'annual_review', 16.7, 2),
(4, '2016-06-01', 90000, 15000, 5000, 2000, 'initial', NULL, NULL),
(4, '2024-01-01', 110000, 15000, 5000, 2000, 'annual_review', 22.2, 1),
(5, '2019-09-01', 80000, 8000, 3000, 2000, 'initial', NULL, NULL),
(5, '2022-01-01', 95000, 8000, 3000, 2000, 'promotion', 18.75, 1),
(5, '2024-01-01', 105000, 8000, 3000, 2000, 'annual_review', 10.5, 1),
(6, '2022-01-10', 45000, 5000, 2000, 1500, 'initial', NULL, NULL),
(6, '2024-01-01', 52000, 5000, 2000, 1500, 'annual_review', 15.6, 5),
(7, '2017-04-01', 40000, 5000, 2000, 1500, 'initial', NULL, NULL),
(7, '2023-01-01', 55000, 8000, 3000, 1500, 'annual_review', 37.5, 1),
(8, '2023-06-01', 40000, 5000, 2000, 1500, 'initial', NULL, NULL),
(9, '2021-02-15', 30000, 4000, 2000, 1500, 'initial', NULL, NULL),
(9, '2024-01-01', 38000, 4000, 2000, 1500, 'annual_review', 26.7, 4),
(10, '2021-08-01', 28000, 4000, 2000, 1500, 'initial', NULL, NULL),
(10, '2024-01-01', 33000, 4000, 2000, 1500, 'annual_review', 17.9, 2);

-- Insert Leave Types
INSERT INTO leave_types (leave_type_id, name, code, days_per_year, is_paid, carry_forward, max_carry_forward_days) VALUES
(1, 'Annual Leave', 'AL', 10, TRUE, TRUE, 5),
(2, 'Sick Leave', 'SL', 30, TRUE, FALSE, 0),
(3, 'Personal Leave', 'PL', 3, TRUE, FALSE, 0),
(4, 'Maternity Leave', 'ML', 90, TRUE, FALSE, 0),
(5, 'Paternity Leave', 'PTL', 15, TRUE, FALSE, 0),
(6, 'Ordination Leave', 'OL', 120, FALSE, FALSE, 0),
(7, 'Unpaid Leave', 'UPL', 0, FALSE, FALSE, 0);

-- Insert Leave Balances (2024)
INSERT INTO leave_balances (employee_id, leave_type_id, year, entitlement_days, taken_days, carried_forward) VALUES
(1, 1, 2024, 15, 3, 0),
(1, 2, 2024, 30, 0, 0),
(2, 1, 2024, 12, 5, 2),
(2, 2, 2024, 30, 2, 0),
(3, 1, 2024, 10, 2, 0),
(3, 2, 2024, 30, 5, 0),
(5, 1, 2024, 12, 8, 0),
(5, 2, 2024, 30, 1, 0),
(6, 1, 2024, 10, 3, 0),
(7, 1, 2024, 10, 7, 0),
(7, 2, 2024, 30, 10, 0);

-- Insert Performance Reviews
INSERT INTO performance_reviews (employee_id, reviewer_id, review_period, review_type, performance_score, attitude_score, leadership_score, teamwork_score, overall_rating, rating_label, kpi_achievement_pct, status) VALUES
(2, 1, '2023-Annual', 'annual', 4.5, 4.5, 4.0, 5.0, 4.5, 'outstanding', 115, 'approved'),
(3, 2, '2023-Annual', 'annual', 4.0, 4.5, 3.5, 4.5, 4.1, 'exceeds_expectations', 105, 'approved'),
(5, 1, '2023-Annual', 'annual', 4.8, 4.5, 4.5, 5.0, 4.7, 'outstanding', 120, 'approved'),
(6, 5, '2023-Annual', 'annual', 3.5, 4.5, 3.0, 4.5, 3.9, 'meets_expectations', 95, 'approved'),
(7, 1, '2023-Annual', 'annual', 4.0, 4.0, 3.5, 4.0, 3.9, 'meets_expectations', 100, 'approved'),
(9, 4, '2023-Annual', 'annual', 4.2, 4.0, 3.5, 4.5, 4.1, 'exceeds_expectations', 108, 'approved');

-- Insert Training Records
INSERT INTO training_courses (course_id, course_code, title, category, duration_hours, cost_per_person, is_mandatory) VALUES
(1, 'TRN001', 'บทบาทและหน้าที่ตาม PDPA', 'Compliance', 3, 0, TRUE),
(2, 'TRN002', 'Advanced SQL for Data Analysis', 'Technical', 16, 5000, FALSE),
(3, 'TRN003', 'Leadership Skills for Managers', 'Leadership', 8, 8000, FALSE),
(4, 'TRN004', 'Customer Service Excellence', 'Soft Skills', 6, 3000, TRUE),
(5, 'TRN005', 'Agile & Scrum Methodology', 'Technical', 8, 4000, FALSE);

INSERT INTO training_records (employee_id, course_id, training_date, completion_date, score, pass_score, is_passed, cost) VALUES
(1, 1, '2024-01-15', '2024-01-15', 92, 70, TRUE, 0),
(2, 1, '2024-01-15', '2024-01-15', 88, 70, TRUE, 0),
(2, 3, '2023-06-10', '2023-06-10', 95, 70, TRUE, 8000),
(3, 1, '2024-01-15', '2024-01-15', 85, 70, TRUE, 0),
(5, 2, '2023-09-20', '2023-09-21', 90, 70, TRUE, 5000),
(5, 5, '2024-02-10', '2024-02-11', 88, 70, TRUE, 4000),
(6, 2, '2023-09-20', '2023-09-21', 75, 70, TRUE, 5000),
(6, 5, '2024-02-10', '2024-02-11', 82, 70, TRUE, 4000),
(7, 4, '2024-01-20', '2024-01-20', 92, 70, TRUE, 3000),
(9, 1, '2024-01-15', '2024-01-15', 80, 70, TRUE, 0);

-- Payroll Run for January 2024
INSERT INTO payroll_runs (run_id, period_year, period_month, run_date, status, total_employees) VALUES
(1, 2024, 1, '2024-01-25', 'paid', 10);

-- Payslips (January 2024)
INSERT INTO payslips (run_id, employee_id, base_salary, housing_allowance, transport_allowance, meal_allowance,
    gross_pay, social_security, income_tax, provident_fund, total_deductions, net_pay, working_days) VALUES
(1, 1, 550000, 50000, 10000, 5000, 615000, 750, 148270, 55000, 204020, 410980, 23),
(1, 2, 100000, 12000, 3000, 2000, 117000, 750, 19450, 10000, 30200, 86800, 23),
(1, 3, 35000, 5000, 2000, 1500, 43500, 750, 2500, 3500, 6750, 36750, 23),
(1, 4, 110000, 15000, 5000, 2000, 132000, 750, 22650, 11000, 34400, 97600, 23),
(1, 5, 105000, 8000, 3000, 2000, 118000, 750, 19750, 10500, 31000, 87000, 23),
(1, 6, 52000, 5000, 2000, 1500, 60500, 750, 6350, 5200, 12300, 48200, 23),
(1, 7, 55000, 8000, 3000, 1500, 67500, 750, 8250, 5500, 14500, 53000, 23),
(1, 8, 40000, 5000, 2000, 1500, 48500, 750, 4050, 4000, 8800, 39700, 23),
(1, 9, 38000, 4000, 2000, 1500, 45500, 750, 3350, 3800, 7900, 37600, 23),
(1, 10, 33000, 4000, 2000, 1500, 40500, 750, 2550, 3300, 6600, 33900, 23);
```

## 3. HR Queries

### Query 1: Organization Chart with Recursive CTE

```sql
-- แสดง Org Chart แบบ Hierarchical
WITH RECURSIVE org_chart AS (
    -- Root: CEO
    SELECT 
        e.employee_id,
        e.employee_number,
        CONCAT(e.first_name, ' ', e.last_name) AS employee_name,
        jp.title AS position_title,
        d.name AS department,
        e.manager_id,
        1 AS level,
        CAST(CONCAT(e.first_name, ' ', e.last_name) AS CHAR(1000)) AS hierarchy_path,
        LPAD('', 0, ' ') AS indent
    FROM employees e
    JOIN job_positions jp ON e.position_id = jp.position_id
    JOIN departments d ON e.dept_id = d.dept_id
    WHERE e.manager_id IS NULL
      AND e.status = 'active'
    
    UNION ALL
    
    SELECT 
        e.employee_id,
        e.employee_number,
        CONCAT(e.first_name, ' ', e.last_name),
        jp.title,
        d.name,
        e.manager_id,
        oc.level + 1,
        CONCAT(oc.hierarchy_path, ' > ', e.first_name, ' ', e.last_name),
        LPAD('', (oc.level) * 4, ' ')
    FROM employees e
    JOIN job_positions jp ON e.position_id = jp.position_id
    JOIN departments d ON e.dept_id = d.dept_id
    JOIN org_chart oc ON e.manager_id = oc.employee_id
    WHERE e.status = 'active'
)
SELECT 
    level,
    CONCAT(indent, '└─ ', employee_name) AS org_chart,
    position_title,
    department,
    employee_number,
    hierarchy_path
FROM org_chart
ORDER BY hierarchy_path;
```

---

### Query 2: Payroll Calculation Query

```sql
-- คำนวณเงินเดือนพร้อมการหักและภาษี (ไทย 2024)
WITH current_salary AS (
    SELECT 
        sh.employee_id,
        sh.base_salary,
        sh.housing_allowance,
        sh.transport_allowance,
        sh.meal_allowance,
        sh.phone_allowance,
        sh.other_allowance,
        sh.total_compensation
    FROM salary_history sh
    WHERE sh.effective_date = (
        SELECT MAX(sh2.effective_date)
        FROM salary_history sh2
        WHERE sh2.employee_id = sh.employee_id
          AND sh2.effective_date <= CURDATE()
    )
),
tax_calculation AS (
    SELECT 
        e.employee_id,
        cs.total_compensation AS monthly_gross,
        cs.total_compensation * 12 AS annual_gross,
        -- Standard deductions (Thai)
        LEAST(cs.total_compensation * 0.5, 100000) AS monthly_50pct_deduction,
        750 AS monthly_personal_allowance_adjustment,  -- monthly portion
        -- SSO (5% max 750/month)
        LEAST(cs.base_salary * 0.05, 750) AS sso_employee,
        -- Income Tax (simplified progressive)
        CASE 
            WHEN cs.total_compensation * 12 <= 150000 THEN 0
            WHEN cs.total_compensation * 12 <= 300000 THEN (cs.total_compensation * 12 - 150000) * 0.05 / 12
            WHEN cs.total_compensation * 12 <= 500000 THEN ((cs.total_compensation * 12 - 300000) * 0.10 + 7500) / 12
            WHEN cs.total_compensation * 12 <= 750000 THEN ((cs.total_compensation * 12 - 500000) * 0.15 + 27500) / 12
            WHEN cs.total_compensation * 12 <= 1000000 THEN ((cs.total_compensation * 12 - 750000) * 0.20 + 65000) / 12
            WHEN cs.total_compensation * 12 <= 2000000 THEN ((cs.total_compensation * 12 - 1000000) * 0.25 + 115000) / 12
            WHEN cs.total_compensation * 12 <= 5000000 THEN ((cs.total_compensation * 12 - 2000000) * 0.30 + 365000) / 12
            ELSE ((cs.total_compensation * 12 - 5000000) * 0.35 + 1265000) / 12
        END AS income_tax,
        -- Provident Fund (employee 5% of base salary, employer matches)
        cs.base_salary * 0.05 AS provident_fund_employee
    FROM employees e
    JOIN current_salary cs ON e.employee_id = cs.employee_id
    WHERE e.status = 'active'
)
SELECT 
    e.employee_number,
    CONCAT(e.first_name, ' ', e.last_name) AS employee_name,
    d.name AS department,
    jp.title AS position,
    ROUND(tc.monthly_gross, 2) AS gross_pay,
    ROUND(tc.sso_employee, 2) AS social_security,
    ROUND(tc.income_tax, 2) AS income_tax,
    ROUND(tc.provident_fund_employee, 2) AS provident_fund,
    ROUND(tc.sso_employee + tc.income_tax + tc.provident_fund_employee, 2) AS total_deductions,
    ROUND(tc.monthly_gross - tc.sso_employee - tc.income_tax - tc.provident_fund_employee, 2) AS net_pay
FROM tax_calculation tc
JOIN employees e ON tc.employee_id = e.employee_id
JOIN departments d ON e.dept_id = d.dept_id
JOIN job_positions jp ON e.position_id = jp.position_id
ORDER BY e.dept_id, tc.monthly_gross DESC;
```

---

### Query 3: Employee Headcount Report

```sql
SELECT 
    d.name AS department,
    COUNT(DISTINCT e.employee_id) AS total_headcount,
    SUM(CASE WHEN e.gender = 'male' THEN 1 ELSE 0 END) AS male,
    SUM(CASE WHEN e.gender = 'female' THEN 1 ELSE 0 END) AS female,
    SUM(CASE WHEN e.contract_type = 'permanent' THEN 1 ELSE 0 END) AS permanent,
    SUM(CASE WHEN e.contract_type = 'contract' THEN 1 ELSE 0 END) AS contract,
    -- Tenure breakdown
    SUM(CASE WHEN DATEDIFF(CURDATE(), e.hire_date) < 365 THEN 1 ELSE 0 END) AS less_1yr,
    SUM(CASE WHEN DATEDIFF(CURDATE(), e.hire_date) BETWEEN 365 AND 1095 THEN 1 ELSE 0 END) AS yr1_3,
    SUM(CASE WHEN DATEDIFF(CURDATE(), e.hire_date) > 1095 THEN 1 ELSE 0 END) AS more_3yr,
    -- Average tenure in years
    ROUND(AVG(DATEDIFF(CURDATE(), e.hire_date) / 365.25), 1) AS avg_tenure_years,
    -- Budget vs Actual
    SUM(jp.headcount_budget) AS headcount_budget,
    SUM(jp.headcount_budget) - COUNT(DISTINCT e.employee_id) AS vacant_positions
FROM departments d
LEFT JOIN employees e ON d.dept_id = e.dept_id AND e.status = 'active'
LEFT JOIN job_positions jp ON e.position_id = jp.position_id
WHERE d.is_active = TRUE
GROUP BY d.dept_id, d.name
ORDER BY total_headcount DESC;
```

---

### Query 4: Turnover Analysis (การวิเคราะห์อัตราการลาออก)

```sql
-- อัตราการลาออกรายปี
WITH yearly_headcount AS (
    SELECT 
        YEAR(hire_date) AS year,
        COUNT(*) AS hired,
        SUM(CASE WHEN status IN ('resigned', 'terminated') 
             AND YEAR(termination_date) = YEAR(hire_date) THEN 1 ELSE 0 END) AS terminated_same_year
    FROM employees
    GROUP BY YEAR(hire_date)
),
terminations_by_year AS (
    SELECT 
        YEAR(termination_date) AS year,
        COUNT(*) AS total_terminations,
        SUM(CASE WHEN status = 'resigned' THEN 1 ELSE 0 END) AS resignations,
        SUM(CASE WHEN status = 'terminated' THEN 1 ELSE 0 END) AS terminations,
        AVG(DATEDIFF(termination_date, hire_date) / 365.25) AS avg_tenure_at_departure
    FROM employees
    WHERE termination_date IS NOT NULL
    GROUP BY YEAR(termination_date)
)
SELECT 
    t.year,
    -- Active employees at start of year (simplified)
    (SELECT COUNT(*) FROM employees e2 
     WHERE YEAR(e2.hire_date) <= t.year 
       AND (e2.termination_date IS NULL OR YEAR(e2.termination_date) >= t.year)) AS avg_headcount,
    t.total_terminations,
    t.resignations,
    t.terminations,
    ROUND(t.avg_tenure_at_departure, 1) AS avg_tenure_at_departure_yrs,
    -- Turnover rate = (separations / avg headcount) * 100
    ROUND(
        t.total_terminations * 100.0 / NULLIF(
            (SELECT COUNT(*) FROM employees e2 
             WHERE YEAR(e2.hire_date) <= t.year 
               AND (e2.termination_date IS NULL OR YEAR(e2.termination_date) >= t.year)),
            0
        ),
        1
    ) AS turnover_rate_pct
FROM terminations_by_year t
ORDER BY t.year DESC;
```

---

### Query 5: Leave Utilization Report

```sql
SELECT 
    CONCAT(e.first_name, ' ', e.last_name) AS employee_name,
    d.name AS department,
    lt.name AS leave_type,
    lb.year,
    lb.entitlement_days + lb.carried_forward AS total_available,
    lb.taken_days,
    lb.pending_days,
    lb.remaining_days,
    ROUND(lb.taken_days * 100.0 / NULLIF(lb.entitlement_days + lb.carried_forward, 0), 1) AS utilization_pct,
    CASE 
        WHEN lb.remaining_days < 0 THEN 'OVER LIMIT!'
        WHEN lb.remaining_days = 0 THEN 'FULLY USED'
        WHEN (lb.taken_days / NULLIF(lb.entitlement_days, 0)) < 0.5 
          AND MONTH(CURDATE()) >= 10 THEN 'RISK: EXPIRING'
        ELSE 'NORMAL'
    END AS status
FROM leave_balances lb
JOIN employees e ON lb.employee_id = e.employee_id
JOIN departments d ON e.dept_id = d.dept_id
JOIN leave_types lt ON lb.leave_type_id = lt.leave_type_id
WHERE lb.year = YEAR(CURDATE())
  AND e.status = 'active'
ORDER BY d.name, e.first_name, lt.name;
```

---

### Query 6: Salary Benchmarking

```sql
-- เปรียบเทียบเงินเดือนกับ Range ของตำแหน่ง
WITH current_salary AS (
    SELECT DISTINCT ON (employee_id)
        employee_id,
        base_salary,
        total_compensation
    FROM salary_history
    ORDER BY employee_id, effective_date DESC
)
SELECT 
    CONCAT(e.first_name, ' ', e.last_name) AS employee_name,
    jp.title AS position,
    jp.job_level,
    jp.min_salary AS salary_range_min,
    jp.max_salary AS salary_range_max,
    ROUND((jp.min_salary + jp.max_salary) / 2, 0) AS salary_midpoint,
    cs.base_salary AS current_salary,
    -- Compa-ratio: current salary / midpoint * 100
    ROUND(cs.base_salary * 100.0 / ((jp.min_salary + jp.max_salary) / 2), 1) AS compa_ratio,
    -- Position in range (0-100%)
    ROUND(
        (cs.base_salary - jp.min_salary) * 100.0 / NULLIF(jp.max_salary - jp.min_salary, 0),
        1
    ) AS range_position_pct,
    CASE 
        WHEN cs.base_salary > jp.max_salary THEN 'ABOVE RANGE'
        WHEN cs.base_salary < jp.min_salary THEN 'BELOW RANGE'
        WHEN cs.base_salary <= jp.min_salary + (jp.max_salary - jp.min_salary) * 0.33 THEN 'LOWER THIRD'
        WHEN cs.base_salary <= jp.min_salary + (jp.max_salary - jp.min_salary) * 0.67 THEN 'MIDDLE THIRD'
        ELSE 'UPPER THIRD'
    END AS range_quartile
FROM employees e
JOIN job_positions jp ON e.position_id = jp.position_id
JOIN current_salary cs ON e.employee_id = cs.employee_id
WHERE e.status = 'active'
ORDER BY jp.job_level, compa_ratio DESC;
```

---

### Query 7: Training Compliance Report

```sql
-- ตรวจสอบว่าพนักงานทำ Training ครบตาม Requirement
SELECT 
    CONCAT(e.first_name, ' ', e.last_name) AS employee_name,
    d.name AS department,
    -- Mandatory training status
    COUNT(DISTINCT tc.course_id) AS mandatory_courses_total,
    COUNT(DISTINCT CASE WHEN tr.is_passed = TRUE THEN tc.course_id END) AS mandatory_courses_completed,
    COUNT(DISTINCT tc.course_id) - COUNT(DISTINCT CASE WHEN tr.is_passed = TRUE THEN tc.course_id END) AS mandatory_courses_pending,
    -- Overall training hours this year
    COALESCE(SUM(CASE WHEN YEAR(tr.training_date) = YEAR(CURDATE()) 
                       AND tr.is_passed = TRUE 
                 THEN tc_all.duration_hours ELSE 0 END), 0) AS training_hours_ytd,
    -- Non-mandatory courses
    COUNT(DISTINCT CASE WHEN NOT tc_all.is_mandatory AND tr2.is_passed = TRUE THEN tr2.course_id END) AS optional_courses_completed,
    -- Compliance status
    CASE 
        WHEN COUNT(DISTINCT tc.course_id) = COUNT(DISTINCT CASE WHEN tr.is_passed = TRUE THEN tc.course_id END) 
        THEN 'COMPLIANT'
        ELSE CONCAT('NON-COMPLIANT (', 
            COUNT(DISTINCT tc.course_id) - COUNT(DISTINCT CASE WHEN tr.is_passed = TRUE THEN tc.course_id END),
            ' pending)')
    END AS compliance_status
FROM employees e
JOIN departments d ON e.dept_id = d.dept_id
CROSS JOIN training_courses tc ON tc.is_mandatory = TRUE AND tc.is_active = TRUE
LEFT JOIN training_records tr ON e.employee_id = tr.employee_id 
    AND tc.course_id = tr.course_id
    AND tr.is_passed = TRUE
-- For optional courses
LEFT JOIN training_records tr2 ON e.employee_id = tr2.employee_id
LEFT JOIN training_courses tc_all ON tr2.course_id = tc_all.course_id
WHERE e.status = 'active'
GROUP BY e.employee_id, e.first_name, e.last_name, d.name
ORDER BY compliance_status, d.name;
```

---

### Query 8: Performance Distribution

```sql
SELECT 
    d.name AS department,
    COUNT(pr.review_id) AS total_reviews,
    ROUND(AVG(pr.overall_rating), 2) AS avg_rating,
    SUM(CASE WHEN pr.rating_label = 'outstanding' THEN 1 ELSE 0 END) AS outstanding,
    SUM(CASE WHEN pr.rating_label = 'exceeds_expectations' THEN 1 ELSE 0 END) AS exceeds_exp,
    SUM(CASE WHEN pr.rating_label = 'meets_expectations' THEN 1 ELSE 0 END) AS meets_exp,
    SUM(CASE WHEN pr.rating_label = 'needs_improvement' THEN 1 ELSE 0 END) AS needs_improvement,
    SUM(CASE WHEN pr.rating_label = 'unsatisfactory' THEN 1 ELSE 0 END) AS unsatisfactory,
    ROUND(AVG(pr.kpi_achievement_pct), 1) AS avg_kpi_achievement
FROM performance_reviews pr
JOIN employees e ON pr.employee_id = e.employee_id
JOIN departments d ON e.dept_id = d.dept_id
WHERE pr.review_period LIKE '2023%'
  AND pr.status = 'approved'
GROUP BY d.dept_id, d.name
ORDER BY avg_rating DESC;
```

---

### Query 9: Payroll Summary by Department

```sql
SELECT 
    d.name AS department,
    COUNT(p.payslip_id) AS employee_count,
    SUM(p.gross_pay) AS total_gross,
    SUM(p.base_salary) AS total_base,
    SUM(p.housing_allowance + p.transport_allowance + p.meal_allowance + p.other_allowance) AS total_allowances,
    SUM(p.bonus) AS total_bonus,
    SUM(p.social_security) AS total_sso,
    SUM(p.income_tax) AS total_income_tax,
    SUM(p.provident_fund) AS total_provident_fund,
    SUM(p.total_deductions) AS total_deductions,
    SUM(p.net_pay) AS total_net_pay,
    ROUND(AVG(p.gross_pay), 2) AS avg_gross_pay,
    ROUND(AVG(p.net_pay), 2) AS avg_net_pay,
    -- Cost per employee
    ROUND(SUM(p.gross_pay + p.social_security * 2) / COUNT(p.payslip_id), 2) AS cost_per_employee
    -- Note: employer SSO = employee SSO (5% each)
FROM payslips p
JOIN payroll_runs pr ON p.run_id = pr.run_id
JOIN employees e ON p.employee_id = e.employee_id
JOIN departments d ON e.dept_id = d.dept_id
WHERE pr.period_year = 2024 AND pr.period_month = 1
GROUP BY d.dept_id, d.name
ORDER BY total_gross DESC;
```

---

### Query 10: Employee Tenure and Retention Analysis

```sql
WITH tenure_data AS (
    SELECT 
        employee_id,
        DATEDIFF(COALESCE(termination_date, CURDATE()), hire_date) / 365.25 AS tenure_years,
        status,
        hire_date,
        termination_date,
        dept_id
    FROM employees
)
SELECT 
    d.name AS department,
    COUNT(*) AS total_employees,
    ROUND(AVG(td.tenure_years), 1) AS avg_tenure_years,
    MIN(td.tenure_years) AS min_tenure_years,
    MAX(td.tenure_years) AS max_tenure_years,
    -- Tenure buckets
    SUM(CASE WHEN td.tenure_years < 1 THEN 1 ELSE 0 END) AS less_1yr,
    SUM(CASE WHEN td.tenure_years BETWEEN 1 AND 3 THEN 1 ELSE 0 END) AS yr_1_3,
    SUM(CASE WHEN td.tenure_years BETWEEN 3 AND 5 THEN 1 ELSE 0 END) AS yr_3_5,
    SUM(CASE WHEN td.tenure_years > 5 THEN 1 ELSE 0 END) AS more_5yr,
    -- Currently on probation
    SUM(CASE WHEN e.probation_end > CURDATE() THEN 1 ELSE 0 END) AS on_probation
FROM tenure_data td
JOIN employees e ON td.employee_id = e.employee_id
JOIN departments d ON e.dept_id = d.dept_id
WHERE e.status = 'active'
GROUP BY d.dept_id, d.name
ORDER BY avg_tenure_years DESC;
```

---

## แบบฝึกหัด (Challenge Exercises)

1. **Succession Planning**: เขียน Query หาพนักงานที่พร้อม (High Performer + ประสบการณ์ > 3 ปี) สำหรับตำแหน่ง Management ที่ว่างในปีหน้า

2. **Salary Equity Analysis**: วิเคราะห์ช่องว่างเงินเดือนระหว่างเพศ (Gender Pay Gap) แบบควบคุม Position Level และ Department

3. **Leave Forecasting**: เขียน Query ที่ทำนายจำนวนวันลาของทีมในแต่ละเดือนโดยอิงจาก Historical Pattern

4. **Training ROI**: คำนวณ Return on Investment ของการฝึกอบรม โดยเปรียบเทียบ Performance Score ก่อนและหลังผ่านการ Training

5. **Retirement Planning**: หาพนักงานที่จะเกษียณใน 5 ปีข้างหน้า และ Position ที่ต้องวางแผน Succession

6. **Benefits Cost Analysis**: คำนวณต้นทุนสวัสดิการรวมต่อพนักงาน (Total Compensation = Salary + Benefits + SSO + PF) และเปรียบเทียบระหว่าง Levels

7. **Absence Rate**: คำนวณ Absence Rate (จำนวนวันลา / จำนวนวันทำงานทั้งหมด) รายเดือนและแยกตาม Department

8. **Internal Mobility**: วิเคราะห์ Internal Transfer Pattern ว่าพนักงานย้ายจาก Department ไหนไป Department ไหนบ่อยที่สุด

9. **Hire vs Target**: เปรียบเทียบ Actual Headcount กับ Approved Headcount Budget แยกตาม Department และ Quarter

10. **High Performer Retention**: เขียน Query หาพนักงานที่มี Rating "Outstanding" แต่ไม่ได้รับการขึ้นเงินเดือนในช่วง 2 ปีที่ผ่านมา (Retention Risk)
