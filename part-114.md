# Part 114: Banking System Database

## บทนำ (Introduction)

ระบบธนาคารเป็นหนึ่งในระบบที่มีความต้องการด้าน Data Integrity และ Security สูงที่สุด เนื่องจากเกี่ยวข้องกับเงินและข้อมูลทางการเงินของลูกค้า บทนี้จะครอบคลุม Schema สมบูรณ์, Financial Calculation Queries, Audit Trail Implementation และ Fraud Detection

## 1. Complete Banking Schema

```sql
-- =========================================
-- BANKING SYSTEM - COMPLETE DDL
-- =========================================

CREATE DATABASE banking_db
    CHARACTER SET utf8mb4
    COLLATE utf8mb4_unicode_ci;

USE banking_db;

-- =========================================
-- SECTION 1: CUSTOMER HIERARCHY
-- =========================================

-- ตาราง Customer Types (ประเภทลูกค้า)
CREATE TABLE customer_types (
    type_id         TINYINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    name            VARCHAR(50) NOT NULL,
    description     TEXT
) ENGINE=InnoDB;

-- ตาราง Customers (ลูกค้า - ทั้ง Personal และ Business)
CREATE TABLE customers (
    customer_id     INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    customer_number VARCHAR(20) NOT NULL UNIQUE,
    customer_type   ENUM('personal','business','government') DEFAULT 'personal',
    -- Personal info
    first_name      VARCHAR(100),
    last_name       VARCHAR(100),
    date_of_birth   DATE,
    national_id     VARCHAR(20) UNIQUE,
    -- Business info
    company_name    VARCHAR(300),
    tax_id          VARCHAR(20) UNIQUE,
    business_type   VARCHAR(100),
    -- Contact
    email           VARCHAR(255) UNIQUE,
    phone           VARCHAR(20),
    address_line1   VARCHAR(255),
    city            VARCHAR(100),
    country_code    CHAR(2) DEFAULT 'TH',
    -- KYC
    kyc_status      ENUM('pending','verified','rejected','expired') DEFAULT 'pending',
    kyc_verified_at TIMESTAMP NULL,
    risk_rating     ENUM('low','medium','high') DEFAULT 'low',
    -- Status
    is_active       BOOLEAN DEFAULT TRUE,
    created_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    INDEX idx_customer_number (customer_number),
    INDEX idx_national_id (national_id)
) ENGINE=InnoDB;

-- ตาราง Customer Relationships (ความสัมพันธ์ เช่น Joint Account)
CREATE TABLE customer_relationships (
    relation_id     INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    primary_customer_id INT UNSIGNED NOT NULL,
    related_customer_id INT UNSIGNED NOT NULL,
    relationship_type ENUM('spouse','child','parent','business_partner','authorized_signatory') NOT NULL,
    created_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (primary_customer_id) REFERENCES customers(customer_id),
    FOREIGN KEY (related_customer_id) REFERENCES customers(customer_id)
) ENGINE=InnoDB;

-- =========================================
-- SECTION 2: ACCOUNT MANAGEMENT
-- =========================================

-- ตาราง Account Types (ประเภทบัญชี)
CREATE TABLE account_types (
    account_type_id TINYINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    name            VARCHAR(100) NOT NULL,
    category        ENUM('deposit','credit','loan','investment') NOT NULL,
    interest_rate   DECIMAL(6,4),
    min_balance     DECIMAL(15,2) DEFAULT 0,
    monthly_fee     DECIMAL(8,2) DEFAULT 0,
    withdrawal_limit_daily DECIMAL(15,2),
    description     TEXT,
    is_active       BOOLEAN DEFAULT TRUE
) ENGINE=InnoDB;

-- ตาราง Accounts (บัญชีธนาคาร)
CREATE TABLE accounts (
    account_id      INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    account_number  VARCHAR(20) NOT NULL UNIQUE,
    account_type_id TINYINT UNSIGNED NOT NULL,
    customer_id     INT UNSIGNED NOT NULL,
    currency        CHAR(3) DEFAULT 'THB',
    balance         DECIMAL(20,4) NOT NULL DEFAULT 0.0000,
    available_balance DECIMAL(20,4) NOT NULL DEFAULT 0.0000,
    hold_amount     DECIMAL(15,4) DEFAULT 0.0000,
    interest_rate   DECIMAL(6,4),
    status          ENUM('active','frozen','closed','dormant','pending_approval') DEFAULT 'active',
    opened_at       TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    closed_at       TIMESTAMP NULL,
    last_transaction_at TIMESTAMP NULL,
    branch_code     VARCHAR(10),
    overdraft_limit DECIMAL(12,2) DEFAULT 0,
    FOREIGN KEY (account_type_id) REFERENCES account_types(account_type_id),
    FOREIGN KEY (customer_id) REFERENCES customers(customer_id),
    INDEX idx_customer (customer_id),
    INDEX idx_status (status),
    INDEX idx_account_number (account_number)
) ENGINE=InnoDB;

-- =========================================
-- SECTION 3: TRANSACTIONS
-- =========================================

-- ตาราง Transaction Categories (ประเภทธุรกรรม)
CREATE TABLE transaction_types (
    type_id         SMALLINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    name            VARCHAR(100) NOT NULL,
    code            VARCHAR(20) NOT NULL UNIQUE,
    direction       ENUM('debit','credit','both') NOT NULL,
    description     TEXT,
    fee_applicable  BOOLEAN DEFAULT FALSE
) ENGINE=InnoDB;

-- ตาราง Transactions (ธุรกรรมทางการเงิน)
CREATE TABLE transactions (
    transaction_id      BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    transaction_ref     VARCHAR(50) NOT NULL UNIQUE,
    account_id          INT UNSIGNED NOT NULL,
    related_account_id  INT UNSIGNED COMMENT 'For transfers',
    transaction_type_id SMALLINT UNSIGNED NOT NULL,
    direction           ENUM('debit','credit') NOT NULL,
    amount              DECIMAL(20,4) NOT NULL,
    currency            CHAR(3) DEFAULT 'THB',
    -- Balance tracking (crucial for banking)
    balance_before      DECIMAL(20,4) NOT NULL,
    balance_after       DECIMAL(20,4) NOT NULL,
    -- Transaction details
    description         VARCHAR(500),
    reference           VARCHAR(200) COMMENT 'External reference (cheque, etc.)',
    channel             ENUM('atm','branch','online','mobile','api','pos','swift') NOT NULL,
    -- Status & Audit
    status              ENUM('pending','completed','failed','reversed','disputed') DEFAULT 'completed',
    value_date          DATE NOT NULL COMMENT 'Settlement date',
    posted_at           TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    -- Location
    branch_code         VARCHAR(10),
    atm_id              VARCHAR(20),
    ip_address          VARCHAR(45),
    device_fingerprint  VARCHAR(255),
    -- Flags
    is_suspicious       BOOLEAN DEFAULT FALSE,
    fraud_score         DECIMAL(4,3) DEFAULT 0.000,
    FOREIGN KEY (account_id) REFERENCES accounts(account_id),
    FOREIGN KEY (related_account_id) REFERENCES accounts(account_id) ON DELETE SET NULL,
    FOREIGN KEY (transaction_type_id) REFERENCES transaction_types(type_id),
    INDEX idx_account (account_id),
    INDEX idx_posted_at (posted_at),
    INDEX idx_ref (transaction_ref),
    INDEX idx_suspicious (is_suspicious),
    INDEX idx_value_date (value_date)
) ENGINE=InnoDB;

-- =========================================
-- SECTION 4: LOANS
-- =========================================

-- ตาราง Loans (สินเชื่อ)
CREATE TABLE loans (
    loan_id         INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    loan_number     VARCHAR(20) NOT NULL UNIQUE,
    customer_id     INT UNSIGNED NOT NULL,
    account_id      INT UNSIGNED COMMENT 'Disbursement account',
    loan_type       ENUM('personal','mortgage','auto','business','student','credit_line') NOT NULL,
    principal_amount DECIMAL(15,2) NOT NULL,
    outstanding_balance DECIMAL(15,4) NOT NULL,
    interest_rate   DECIMAL(6,4) NOT NULL COMMENT 'Annual rate',
    interest_type   ENUM('fixed','variable') DEFAULT 'fixed',
    term_months     INT NOT NULL,
    installment_amount DECIMAL(12,2) NOT NULL,
    payment_day     TINYINT NOT NULL COMMENT '1-28',
    start_date      DATE NOT NULL,
    maturity_date   DATE NOT NULL,
    status          ENUM('pending','active','paid_off','defaulted','written_off','restructured') DEFAULT 'pending',
    collateral_type VARCHAR(100),
    collateral_value DECIMAL(15,2),
    disbursed_at    TIMESTAMP NULL,
    created_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (customer_id) REFERENCES customers(customer_id),
    FOREIGN KEY (account_id) REFERENCES accounts(account_id) ON DELETE SET NULL,
    INDEX idx_customer (customer_id),
    INDEX idx_status (status)
) ENGINE=InnoDB;

-- ตาราง Loan Payments (การชำระสินเชื่อ)
CREATE TABLE loan_payments (
    payment_id      INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    loan_id         INT UNSIGNED NOT NULL,
    transaction_id  BIGINT UNSIGNED,
    payment_date    DATE NOT NULL,
    due_date        DATE NOT NULL,
    total_payment   DECIMAL(12,2) NOT NULL,
    principal_portion DECIMAL(12,2) NOT NULL,
    interest_portion DECIMAL(12,2) NOT NULL,
    fee_portion     DECIMAL(10,2) DEFAULT 0,
    outstanding_after DECIMAL(15,4) NOT NULL,
    is_on_time      BOOLEAN DEFAULT TRUE,
    days_overdue    INT DEFAULT 0,
    FOREIGN KEY (loan_id) REFERENCES loans(loan_id),
    FOREIGN KEY (transaction_id) REFERENCES transactions(transaction_id) ON DELETE SET NULL,
    INDEX idx_loan (loan_id),
    INDEX idx_payment_date (payment_date)
) ENGINE=InnoDB;

-- ตาราง Loan Amortization Schedule (ตารางผ่อนชำระ)
CREATE TABLE loan_amortization (
    schedule_id     INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    loan_id         INT UNSIGNED NOT NULL,
    installment_num INT NOT NULL,
    due_date        DATE NOT NULL,
    opening_balance DECIMAL(15,4) NOT NULL,
    installment_amount DECIMAL(12,2) NOT NULL,
    principal_portion DECIMAL(12,2) NOT NULL,
    interest_portion DECIMAL(12,2) NOT NULL,
    closing_balance DECIMAL(15,4) NOT NULL,
    actual_payment_id INT UNSIGNED,
    status          ENUM('scheduled','paid','overdue','waived') DEFAULT 'scheduled',
    FOREIGN KEY (loan_id) REFERENCES loans(loan_id) ON DELETE CASCADE,
    FOREIGN KEY (actual_payment_id) REFERENCES loan_payments(payment_id) ON DELETE SET NULL,
    UNIQUE KEY uk_loan_installment (loan_id, installment_num),
    INDEX idx_due_date (due_date)
) ENGINE=InnoDB;

-- =========================================
-- SECTION 5: WIRE TRANSFERS / SWIFT
-- =========================================

-- ตาราง Wire Transfers (โอนเงินระหว่างประเทศ)
CREATE TABLE wire_transfers (
    wire_id         INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    reference_number VARCHAR(30) NOT NULL UNIQUE,
    direction       ENUM('outgoing','incoming') NOT NULL,
    sender_account_id INT UNSIGNED,
    receiver_account_id INT UNSIGNED,
    sender_name     VARCHAR(300) NOT NULL,
    sender_bank     VARCHAR(200),
    sender_bank_code VARCHAR(20),
    receiver_name   VARCHAR(300) NOT NULL,
    receiver_bank   VARCHAR(200),
    receiver_bank_code VARCHAR(20),
    amount          DECIMAL(20,4) NOT NULL,
    currency        CHAR(3) NOT NULL,
    exchange_rate   DECIMAL(12,6),
    amount_thb      DECIMAL(20,4),
    purpose_code    VARCHAR(20),
    purpose_description TEXT,
    status          ENUM('initiated','pending','processing','completed','failed','rejected','cancelled') DEFAULT 'initiated',
    transaction_id  BIGINT UNSIGNED,
    initiated_at    TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    completed_at    TIMESTAMP NULL,
    swift_message   TEXT,
    FOREIGN KEY (sender_account_id) REFERENCES accounts(account_id) ON DELETE SET NULL,
    FOREIGN KEY (receiver_account_id) REFERENCES accounts(account_id) ON DELETE SET NULL,
    FOREIGN KEY (transaction_id) REFERENCES transactions(transaction_id) ON DELETE SET NULL
) ENGINE=InnoDB;

-- =========================================
-- SECTION 6: INTEREST CALCULATIONS
-- =========================================

-- ตาราง Interest Accrual (การสะสมดอกเบี้ย)
CREATE TABLE interest_accrual (
    accrual_id      BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    account_id      INT UNSIGNED NOT NULL,
    accrual_date    DATE NOT NULL,
    daily_balance   DECIMAL(20,4) NOT NULL,
    interest_rate   DECIMAL(6,4) NOT NULL,
    daily_interest  DECIMAL(12,6) NOT NULL,
    accrued_interest DECIMAL(15,6) NOT NULL,
    FOREIGN KEY (account_id) REFERENCES accounts(account_id),
    UNIQUE KEY uk_account_date (account_id, accrual_date),
    INDEX idx_account (account_id),
    INDEX idx_date (accrual_date)
) ENGINE=InnoDB;

-- =========================================
-- SECTION 7: STATEMENTS
-- =========================================

-- ตาราง Account Statements (ใบแจ้งยอด)
CREATE TABLE account_statements (
    statement_id    INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    account_id      INT UNSIGNED NOT NULL,
    statement_period_start DATE NOT NULL,
    statement_period_end DATE NOT NULL,
    opening_balance DECIMAL(20,4) NOT NULL,
    closing_balance DECIMAL(20,4) NOT NULL,
    total_credits   DECIMAL(20,4) NOT NULL DEFAULT 0,
    total_debits    DECIMAL(20,4) NOT NULL DEFAULT 0,
    transaction_count INT DEFAULT 0,
    interest_earned DECIMAL(15,4) DEFAULT 0,
    fees_charged    DECIMAL(10,2) DEFAULT 0,
    generated_at    TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (account_id) REFERENCES accounts(account_id),
    INDEX idx_account (account_id),
    INDEX idx_period (statement_period_start, statement_period_end)
) ENGINE=InnoDB;

-- =========================================
-- SECTION 8: FRAUD & RISK MANAGEMENT
-- =========================================

-- ตาราง Fraud Alerts (การแจ้งเตือนการฉ้อโกง)
CREATE TABLE fraud_alerts (
    alert_id        INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    transaction_id  BIGINT UNSIGNED NOT NULL,
    account_id      INT UNSIGNED NOT NULL,
    alert_type      ENUM('velocity','geo_anomaly','pattern','high_amount','unusual_time','new_device') NOT NULL,
    severity        ENUM('low','medium','high','critical') NOT NULL,
    description     TEXT,
    fraud_score     DECIMAL(4,3),
    auto_blocked    BOOLEAN DEFAULT FALSE,
    is_false_positive BOOLEAN,
    investigated_by INT UNSIGNED,
    investigated_at TIMESTAMP NULL,
    resolution      TEXT,
    created_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (transaction_id) REFERENCES transactions(transaction_id),
    FOREIGN KEY (account_id) REFERENCES accounts(account_id),
    INDEX idx_account (account_id),
    INDEX idx_severity (severity),
    INDEX idx_created_at (created_at)
) ENGINE=InnoDB;

-- =========================================
-- SECTION 9: AUDIT TRAIL
-- =========================================

-- ตาราง Audit Log (บันทึก Audit)
CREATE TABLE audit_log (
    log_id          BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    table_name      VARCHAR(100) NOT NULL,
    record_id       BIGINT UNSIGNED NOT NULL,
    action          ENUM('INSERT','UPDATE','DELETE','SELECT_SENSITIVE') NOT NULL,
    old_values      JSON,
    new_values      JSON,
    changed_fields  JSON,
    user_id         INT UNSIGNED,
    username        VARCHAR(100),
    ip_address      VARCHAR(45),
    session_id      VARCHAR(255),
    performed_at    TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_table_record (table_name, record_id),
    INDEX idx_user (user_id),
    INDEX idx_performed_at (performed_at),
    INDEX idx_action (action)
) ENGINE=InnoDB;
```

## 2. Sample Data

```sql
-- Insert Account Types
INSERT INTO account_types (account_type_id, name, category, interest_rate, min_balance, monthly_fee, withdrawal_limit_daily) VALUES
(1, 'ออมทรัพย์', 'deposit', 0.5000, 0, 0, 200000),
(2, 'กระแสรายวัน', 'deposit', 0.0000, 5000, 200, 500000),
(3, 'ฝากประจำ 3 เดือน', 'deposit', 1.5000, 10000, 0, NULL),
(4, 'ฝากประจำ 6 เดือน', 'deposit', 2.0000, 10000, 0, NULL),
(5, 'ฝากประจำ 12 เดือน', 'deposit', 2.5000, 10000, 0, NULL),
(6, 'สินเชื่อส่วนบุคคล', 'loan', 15.0000, 0, 0, NULL),
(7, 'สินเชื่อบ้าน', 'loan', 6.0000, 0, 0, NULL),
(8, 'บัตรเครดิต', 'credit', 18.0000, 0, 0, NULL);

-- Insert Customers
INSERT INTO customers (customer_id, customer_number, customer_type, first_name, last_name, date_of_birth, national_id, email, phone, city, kyc_status, risk_rating) VALUES
(1, 'CUST-000001', 'personal', 'สมบัติ', 'มั่งมี', '1975-03-15', '1234567890123', 'sombat@email.com', '0891234567', 'Bangkok', 'verified', 'low'),
(2, 'CUST-000002', 'personal', 'มณี', 'ทองคำ', '1988-07-22', '2345678901234', 'manee@email.com', '0862345678', 'Bangkok', 'verified', 'low'),
(3, 'CUST-000003', 'business', NULL, NULL, NULL, NULL, 'company@abc.com', '021234567', 'Bangkok', 'verified', 'medium'),
(4, 'CUST-000004', 'personal', 'วิจิตร', 'สวยงาม', '1960-12-01', '3456789012345', 'wijit@email.com', '0813456789', 'Chiang Mai', 'verified', 'low'),
(5, 'CUST-000005', 'personal', 'อภิชาต', 'เก่งกล้า', '1992-05-28', '4567890123456', 'apichat@email.com', '0904567890', 'Bangkok', 'verified', 'medium');

-- Update Business customer
UPDATE customers SET company_name = 'ABC Co., Ltd.', tax_id = '0105554567890' WHERE customer_id = 3;

-- Insert Accounts
INSERT INTO accounts (account_id, account_number, account_type_id, customer_id, balance, available_balance, interest_rate, status, opened_at, branch_code) VALUES
(1, '0001234567890', 1, 1, 250000.0000, 250000.0000, 0.5000, 'active', '2019-01-15 09:00:00', 'BKK001'),
(2, '0001234567891', 2, 1, 85000.0000, 85000.0000, 0.0000, 'active', '2019-01-15 09:00:00', 'BKK001'),
(3, '0002345678901', 1, 2, 150000.0000, 150000.0000, 0.5000, 'active', '2020-06-01 10:00:00', 'BKK002'),
(4, '0003456789012', 2, 3, 500000.0000, 500000.0000, 0.0000, 'active', '2018-03-10 08:00:00', 'BKK001'),
(5, '0004567890123', 1, 4, 75000.0000, 75000.0000, 0.5000, 'active', '2021-09-20 11:00:00', 'CNX001'),
(6, '0005678901234', 5, 2, 300000.0000, 300000.0000, 2.5000, 'active', '2023-01-01 09:00:00', 'BKK002');

-- Insert Transaction Types
INSERT INTO transaction_types (type_id, name, code, direction, fee_applicable) VALUES
(1, 'Deposit', 'DEP', 'credit', FALSE),
(2, 'Withdrawal', 'WDR', 'debit', FALSE),
(3, 'Transfer In', 'TFR_IN', 'credit', FALSE),
(4, 'Transfer Out', 'TFR_OUT', 'debit', TRUE),
(5, 'ATM Withdrawal', 'ATM_WDR', 'debit', TRUE),
(6, 'Interest Credit', 'INT_CR', 'credit', FALSE),
(7, 'Fee Debit', 'FEE_DR', 'debit', FALSE),
(8, 'Loan Disbursement', 'LOAN_DISB', 'credit', FALSE),
(9, 'Loan Payment', 'LOAN_PAY', 'debit', FALSE),
(10, 'PromptPay', 'PMPAY', 'debit', FALSE);

-- Insert Transactions
INSERT INTO transactions (transaction_ref, account_id, related_account_id, transaction_type_id, direction, amount, balance_before, balance_after, description, channel, value_date, branch_code) VALUES
('TXN-2024-000001', 1, NULL, 1, 'credit', 50000, 200000, 250000, 'Cash deposit', 'branch', '2024-01-10', 'BKK001'),
('TXN-2024-000002', 1, 3, 4, 'debit', 30000, 250000, 220000, 'Transfer to 0002345678901', 'online', '2024-01-15', NULL),
('TXN-2024-000003', 3, 1, 3, 'credit', 30000, 120000, 150000, 'Transfer from 0001234567890', 'online', '2024-01-15', NULL),
('TXN-2024-000004', 1, NULL, 5, 'debit', 5000, 220000, 215000, 'ATM Withdrawal at BKK ATM', 'atm', '2024-01-20', NULL),
('TXN-2024-000005', 1, NULL, 1, 'credit', 35000, 215000, 250000, 'Salary deposit', 'api', '2024-02-01', NULL),
('TXN-2024-000006', 1, NULL, 10, 'debit', 1200, 250000, 248800, 'PromptPay to 0861234567', 'mobile', '2024-02-05', NULL),
('TXN-2024-000007', 2, NULL, 1, 'credit', 100000, 0, 100000, 'Initial deposit', 'branch', '2024-01-10', 'BKK001'),
('TXN-2024-000008', 4, NULL, 1, 'credit', 200000, 300000, 500000, 'Business revenue', 'api', '2024-02-10', NULL),
('TXN-2024-000009', 1, NULL, 6, 'credit', 104.17, 248800, 248904.17, 'Monthly interest January', 'api', '2024-01-31', NULL),
('TXN-2024-000010', 5, NULL, 2, 'debit', 20000, 75000, 55000, 'Cash withdrawal', 'branch', '2024-02-15', 'CNX001');

-- Insert Loans
INSERT INTO loans (loan_id, loan_number, customer_id, account_id, loan_type, principal_amount, outstanding_balance, interest_rate, term_months, installment_amount, payment_day, start_date, maturity_date, status, disbursed_at) VALUES
(1, 'LOAN-2022-000001', 1, 1, 'mortgage', 3000000, 2750000.0000, 6.0000, 360, 17990.00, 5, '2022-01-05', '2052-01-05', 'active', '2022-01-05 10:00:00'),
(2, 'LOAN-2023-000001', 2, 3, 'personal', 200000, 150000.0000, 15.0000, 48, 5568.00, 15, '2023-01-15', '2027-01-15', 'active', '2023-01-15 11:00:00'),
(3, 'LOAN-2024-000001', 5, NULL, 'auto', 500000, 480000.0000, 5.5000, 60, 9577.00, 20, '2024-01-20', '2029-01-20', 'active', '2024-01-20 14:00:00');

-- Insert some Loan Amortization entries
INSERT INTO loan_amortization (loan_id, installment_num, due_date, opening_balance, installment_amount, principal_portion, interest_portion, closing_balance, status) VALUES
(2, 1, '2023-02-15', 200000, 5568, 3068, 2500, 196932, 'paid'),
(2, 2, '2023-03-15', 196932, 5568, 3106.35, 2461.65, 193825.65, 'paid'),
(2, 3, '2023-04-15', 193825.65, 5568, 3144.93, 2423.07, 190680.72, 'paid'),
(2, 13, '2024-02-15', 163532.22, 5568, 3548.05, 2019.95, 159984.17, 'paid'),
(2, 14, '2024-03-15', 159984.17, 5568, 3592.35, 1975.65, 156391.82, 'scheduled'),
(2, 15, '2024-04-15', 156391.82, 5568, 3637.10, 1930.90, 152754.72, 'scheduled');

-- Fraud Alerts data
INSERT INTO fraud_alerts (transaction_id, account_id, alert_type, severity, description, fraud_score, auto_blocked) VALUES
(4, 1, 'unusual_time', 'low', 'ATM withdrawal at 2am', 0.250, FALSE),
(8, 4, 'high_amount', 'medium', 'Large credit transaction above threshold', 0.450, FALSE);
```

## 3. Financial Calculation Queries

### Query 1: Current Account Balance with Interest Accrual

```sql
-- คำนวณยอดเงินปัจจุบันรวมดอกเบี้ยค้างรับ
SELECT 
    a.account_number,
    at.name AS account_type,
    c.first_name, c.last_name,
    a.balance AS current_balance,
    a.interest_rate,
    -- Daily Interest
    ROUND(a.balance * a.interest_rate / 100 / 365, 4) AS daily_interest,
    -- Days since last statement (approximate accrued interest)
    DATEDIFF(CURDATE(), COALESCE(
        (SELECT MAX(statement_period_end) FROM account_statements WHERE account_id = a.account_id),
        a.opened_at
    )) AS days_since_statement,
    -- Estimated accrued interest
    ROUND(
        a.balance * a.interest_rate / 100 / 365 * 
        DATEDIFF(CURDATE(), COALESCE(
            (SELECT MAX(statement_period_end) FROM account_statements WHERE account_id = a.account_id),
            a.opened_at
        )),
        2
    ) AS estimated_accrued_interest,
    -- Total with accrued interest
    ROUND(
        a.balance + 
        a.balance * a.interest_rate / 100 / 365 * 
        DATEDIFF(CURDATE(), COALESCE(
            (SELECT MAX(statement_period_end) FROM account_statements WHERE account_id = a.account_id),
            a.opened_at
        )),
        2
    ) AS total_with_accrued_interest
FROM accounts a
JOIN account_types at ON a.account_type_id = at.account_type_id
JOIN customers c ON a.customer_id = c.customer_id
WHERE a.status = 'active'
  AND a.interest_rate > 0
ORDER BY a.balance DESC;
```

---

### Query 2: Loan Amortization Calculation (Stored Procedure)

```sql
-- คำนวณ Amortization Schedule แบบ On-the-fly
WITH RECURSIVE amortize AS (
    -- Base case: เริ่มต้น
    SELECT 
        l.loan_id,
        l.loan_number,
        1 AS installment_num,
        l.outstanding_balance AS opening_balance,
        l.installment_amount,
        -- Interest for this period: P * r/12
        ROUND(l.outstanding_balance * l.interest_rate / 100 / 12, 2) AS interest_portion,
        -- Principal = Payment - Interest
        ROUND(l.installment_amount - (l.outstanding_balance * l.interest_rate / 100 / 12), 2) AS principal_portion,
        -- Closing balance
        ROUND(l.outstanding_balance - (l.installment_amount - (l.outstanding_balance * l.interest_rate / 100 / 12)), 2) AS closing_balance,
        DATE_ADD(l.start_date, INTERVAL 1 MONTH) AS due_date,
        l.term_months,
        l.interest_rate
    FROM loans l
    WHERE l.loan_id = 2
    
    UNION ALL
    
    -- Recursive case: periods 2 to N
    SELECT
        a.loan_id,
        a.loan_number,
        a.installment_num + 1,
        GREATEST(a.closing_balance, 0) AS opening_balance,
        a.installment_amount,
        ROUND(GREATEST(a.closing_balance, 0) * a.interest_rate / 100 / 12, 2) AS interest_portion,
        ROUND(a.installment_amount - (GREATEST(a.closing_balance, 0) * a.interest_rate / 100 / 12), 2) AS principal_portion,
        ROUND(GREATEST(a.closing_balance, 0) - (a.installment_amount - (GREATEST(a.closing_balance, 0) * a.interest_rate / 100 / 12)), 2) AS closing_balance,
        DATE_ADD(a.due_date, INTERVAL 1 MONTH) AS due_date,
        a.term_months,
        a.interest_rate
    FROM amortize a
    WHERE a.installment_num < a.term_months
      AND a.closing_balance > 0
)
SELECT 
    installment_num,
    due_date,
    ROUND(opening_balance, 2) AS opening_balance,
    installment_amount AS payment,
    principal_portion,
    interest_portion,
    ROUND(GREATEST(closing_balance, 0), 2) AS closing_balance,
    -- Cumulative stats
    SUM(principal_portion) OVER (ORDER BY installment_num) AS cumulative_principal,
    SUM(interest_portion) OVER (ORDER BY installment_num) AS cumulative_interest
FROM amortize
ORDER BY installment_num;
```

---

### Query 3: Transaction History with Running Balance

```sql
-- ประวัติธุรกรรมพร้อม Running Balance
SELECT 
    t.posted_at,
    t.transaction_ref,
    tt.name AS transaction_type,
    t.description,
    CASE WHEN t.direction = 'debit' THEN -t.amount ELSE t.amount END AS amount,
    t.balance_after AS balance,
    t.channel,
    CASE WHEN t.is_suspicious THEN '⚠ SUSPICIOUS' ELSE '' END AS flags
FROM transactions t
JOIN transaction_types tt ON t.transaction_type_id = tt.type_id
WHERE t.account_id = 1
ORDER BY t.posted_at, t.transaction_id;
```

---

### Query 4: Account Summary Statement Generator

```sql
-- สร้าง Account Statement Summary
SELECT 
    a.account_number,
    at2.name AS account_type,
    CONCAT(c.first_name, ' ', c.last_name) AS account_holder,
    -- Period
    DATE_FORMAT('2024-01-01', '%d/%m/%Y') AS period_start,
    DATE_FORMAT('2024-01-31', '%d/%m/%Y') AS period_end,
    -- Opening balance (balance at start of period)
    (SELECT t_open.balance_after 
     FROM transactions t_open 
     WHERE t_open.account_id = a.account_id 
       AND t_open.posted_at < '2024-01-01'
     ORDER BY t_open.posted_at DESC, t_open.transaction_id DESC
     LIMIT 1) AS opening_balance,
    -- Transactions in period
    COUNT(t.transaction_id) AS transaction_count,
    SUM(CASE WHEN t.direction = 'credit' THEN t.amount ELSE 0 END) AS total_credits,
    SUM(CASE WHEN t.direction = 'debit' THEN t.amount ELSE 0 END) AS total_debits,
    -- Closing balance
    (SELECT t_close.balance_after 
     FROM transactions t_close 
     WHERE t_close.account_id = a.account_id 
       AND t_close.posted_at <= '2024-01-31 23:59:59'
     ORDER BY t_close.posted_at DESC, t_close.transaction_id DESC
     LIMIT 1) AS closing_balance,
    -- Interest credited
    SUM(CASE WHEN t.transaction_type_id = 6 THEN t.amount ELSE 0 END) AS interest_credited,
    -- Fees charged
    SUM(CASE WHEN t.transaction_type_id = 7 THEN t.amount ELSE 0 END) AS fees_charged
FROM accounts a
JOIN account_types at2 ON a.account_type_id = at2.account_type_id
JOIN customers c ON a.customer_id = c.customer_id
LEFT JOIN transactions t ON a.account_id = t.account_id
    AND t.posted_at BETWEEN '2024-01-01' AND '2024-01-31 23:59:59'
    AND t.status = 'completed'
WHERE a.account_id = 1
GROUP BY a.account_id, a.account_number, at2.name, c.first_name, c.last_name;
```

---

### Query 5: Interest Rate Calculation for Fixed Deposits

```sql
-- คำนวณดอกเบี้ยฝากประจำ
SELECT 
    a.account_number,
    CONCAT(c.first_name, ' ', c.last_name) AS customer_name,
    at2.name AS deposit_type,
    a.balance AS principal,
    a.interest_rate AS annual_rate_pct,
    a.opened_at::DATE AS deposit_date,
    -- Calculate maturity date based on account type
    CASE 
        WHEN at2.account_type_id = 3 THEN DATE_ADD(a.opened_at, INTERVAL 3 MONTH)
        WHEN at2.account_type_id = 4 THEN DATE_ADD(a.opened_at, INTERVAL 6 MONTH)
        WHEN at2.account_type_id = 5 THEN DATE_ADD(a.opened_at, INTERVAL 12 MONTH)
    END AS maturity_date,
    -- Simple interest calculation
    ROUND(a.balance * a.interest_rate / 100 * 
        CASE 
            WHEN at2.account_type_id = 3 THEN 3/12.0
            WHEN at2.account_type_id = 4 THEN 6/12.0
            WHEN at2.account_type_id = 5 THEN 12/12.0
        END,
        2
    ) AS gross_interest,
    -- Tax on interest (15% withholding)
    ROUND(
        a.balance * a.interest_rate / 100 * 
        CASE 
            WHEN at2.account_type_id = 3 THEN 3/12.0
            WHEN at2.account_type_id = 4 THEN 6/12.0
            WHEN at2.account_type_id = 5 THEN 12/12.0
        END * 0.15,
        2
    ) AS withholding_tax,
    -- Net interest
    ROUND(
        a.balance * a.interest_rate / 100 * 
        CASE 
            WHEN at2.account_type_id = 3 THEN 3/12.0
            WHEN at2.account_type_id = 4 THEN 6/12.0
            WHEN at2.account_type_id = 5 THEN 12/12.0
        END * 0.85,
        2
    ) AS net_interest,
    -- Total at maturity
    ROUND(
        a.balance + 
        a.balance * a.interest_rate / 100 * 
        CASE 
            WHEN at2.account_type_id = 3 THEN 3/12.0
            WHEN at2.account_type_id = 4 THEN 6/12.0
            WHEN at2.account_type_id = 5 THEN 12/12.0
        END * 0.85,
        2
    ) AS total_at_maturity
FROM accounts a
JOIN account_types at2 ON a.account_type_id = at2.account_type_id
JOIN customers c ON a.customer_id = c.customer_id
WHERE at2.category = 'deposit'
  AND at2.account_type_id IN (3, 4, 5)
  AND a.status = 'active';
```

---

### Query 6: Fraud Detection Queries

```sql
-- Pattern 1: Velocity Check - มีธุรกรรมหลายครั้งในเวลาสั้น
SELECT 
    t.account_id,
    CONCAT(c.first_name, ' ', c.last_name) AS customer_name,
    COUNT(*) AS transactions_in_1hour,
    SUM(t.amount) AS total_amount_1hour,
    MIN(t.posted_at) AS first_transaction,
    MAX(t.posted_at) AS last_transaction
FROM transactions t
JOIN accounts a ON t.account_id = a.account_id
JOIN customers c ON a.customer_id = c.customer_id
WHERE t.direction = 'debit'
  AND t.posted_at >= DATE_SUB(NOW(), INTERVAL 1 HOUR)
  AND t.status = 'completed'
GROUP BY t.account_id, c.first_name, c.last_name
HAVING transactions_in_1hour >= 5
    OR total_amount_1hour > 100000;

-- Pattern 2: Transaction Amount Anomaly (Z-score based)
WITH account_stats AS (
    SELECT 
        account_id,
        AVG(amount) AS avg_amount,
        STDDEV(amount) AS stddev_amount
    FROM transactions
    WHERE status = 'completed'
    GROUP BY account_id
)
SELECT 
    t.transaction_ref,
    t.account_id,
    t.amount,
    s.avg_amount,
    s.stddev_amount,
    ROUND((t.amount - s.avg_amount) / NULLIF(s.stddev_amount, 0), 2) AS z_score,
    CASE 
        WHEN ABS((t.amount - s.avg_amount) / NULLIF(s.stddev_amount, 0)) > 3 
        THEN 'HIGH ANOMALY (3σ)'
        WHEN ABS((t.amount - s.avg_amount) / NULLIF(s.stddev_amount, 0)) > 2 
        THEN 'MEDIUM ANOMALY (2σ)'
        ELSE 'NORMAL'
    END AS anomaly_level
FROM transactions t
JOIN account_stats s ON t.account_id = s.account_id
WHERE s.stddev_amount > 0
  AND ABS((t.amount - s.avg_amount) / NULLIF(s.stddev_amount, 0)) > 2
ORDER BY z_score DESC;

-- Pattern 3: Round-Amount Transactions (often fraudulent)
SELECT 
    t.account_id,
    t.transaction_ref,
    t.amount,
    t.description,
    t.channel,
    t.posted_at,
    CASE WHEN MOD(t.amount, 100) = 0 THEN 'ROUND_100'
         WHEN MOD(t.amount, 1000) = 0 THEN 'ROUND_1000'
         WHEN MOD(t.amount, 10000) = 0 THEN 'ROUND_10000'
    END AS amount_pattern
FROM transactions t
WHERE t.direction = 'debit'
  AND t.amount >= 50000
  AND MOD(t.amount, 100) = 0
  AND t.posted_at >= DATE_SUB(CURDATE(), INTERVAL 30 DAY)
ORDER BY t.amount DESC;
```

---

### Query 7: Loan Portfolio Analysis

```sql
SELECT 
    l.loan_type,
    COUNT(*) AS loan_count,
    SUM(l.principal_amount) AS total_principal,
    SUM(l.outstanding_balance) AS total_outstanding,
    ROUND(SUM(l.principal_amount - l.outstanding_balance) / SUM(l.principal_amount) * 100, 1) AS pct_collected,
    AVG(l.interest_rate) AS avg_interest_rate,
    -- NPL (Non-Performing Loans) - overdue > 90 days
    SUM(CASE WHEN l.status = 'defaulted' THEN l.outstanding_balance ELSE 0 END) AS npl_amount,
    ROUND(
        SUM(CASE WHEN l.status = 'defaulted' THEN l.outstanding_balance ELSE 0 END) 
        * 100.0 / SUM(l.outstanding_balance),
        2
    ) AS npl_ratio_pct,
    -- Weighted average interest
    ROUND(
        SUM(l.outstanding_balance * l.interest_rate) / SUM(l.outstanding_balance),
        4
    ) AS weighted_avg_rate,
    -- Expected annual interest income
    ROUND(SUM(l.outstanding_balance * l.interest_rate / 100), 2) AS expected_annual_income
FROM loans l
WHERE l.status IN ('active', 'defaulted')
GROUP BY l.loan_type
ORDER BY total_outstanding DESC;
```

---

### Query 8: Regulatory Reporting (Basel III Style)

```sql
-- รายงานตาม Basel III requirements: Capital Adequacy
-- (Simplified version)
SELECT 
    'Deposit Products' AS category,
    COUNT(DISTINCT a.account_id) AS account_count,
    SUM(a.balance) AS total_balance,
    SUM(a.balance) AS risk_weighted_assets,  -- Simplified: deposits have 0% risk weight
    0 AS risk_weight_pct
FROM accounts a
JOIN account_types at ON a.account_type_id = at.account_type_id
WHERE at.category = 'deposit'
  AND a.status = 'active'

UNION ALL

SELECT 
    'Personal Loans',
    COUNT(DISTINCT l.loan_id),
    SUM(l.outstanding_balance),
    SUM(l.outstanding_balance) * 1.00,  -- 100% risk weight
    100
FROM loans l
WHERE l.loan_type = 'personal' AND l.status = 'active'

UNION ALL

SELECT 
    'Mortgage Loans',
    COUNT(DISTINCT l.loan_id),
    SUM(l.outstanding_balance),
    SUM(l.outstanding_balance) * 0.35,  -- 35% risk weight (Basel III for residential mortgage)
    35
FROM loans l
WHERE l.loan_type = 'mortgage' AND l.status = 'active';
```

---

### Query 9: Customer Net Worth Analysis

```sql
SELECT 
    c.customer_number,
    CONCAT(c.first_name, ' ', c.last_name) AS customer_name,
    c.risk_rating,
    -- Assets (deposits)
    COALESCE(SUM(CASE WHEN at.category = 'deposit' THEN a.balance ELSE 0 END), 0) AS total_deposits,
    -- Liabilities (loans)
    COALESCE(SUM(CASE WHEN l.loan_id IS NOT NULL THEN l.outstanding_balance ELSE 0 END), 0) AS total_loan_outstanding,
    -- Net worth
    COALESCE(SUM(CASE WHEN at.category = 'deposit' THEN a.balance ELSE 0 END), 0) -
    COALESCE(SUM(CASE WHEN l.loan_id IS NOT NULL THEN l.outstanding_balance ELSE 0 END), 0) AS net_worth,
    -- Customer value to bank
    COALESCE(SUM(CASE WHEN l.loan_id IS NOT NULL 
                 THEN l.outstanding_balance * l.interest_rate / 100 
                 ELSE 0 END), 0) AS annual_interest_income_from_loans,
    COUNT(DISTINCT a.account_id) AS account_count,
    COUNT(DISTINCT l.loan_id) AS loan_count
FROM customers c
LEFT JOIN accounts a ON c.customer_id = a.customer_id AND a.status = 'active'
LEFT JOIN account_types at ON a.account_type_id = at.account_type_id
LEFT JOIN loans l ON c.customer_id = l.customer_id AND l.status = 'active'
WHERE c.is_active = TRUE
GROUP BY c.customer_id, c.customer_number, c.first_name, c.last_name, c.risk_rating
ORDER BY net_worth DESC;
```

---

### Query 10: Audit Trail Implementation

```sql
-- ดู Audit Trail ของบัญชีที่สำคัญ
SELECT 
    al.performed_at,
    al.action,
    al.table_name,
    al.record_id,
    al.username,
    al.ip_address,
    -- แสดงว่า field ไหนเปลี่ยน
    JSON_KEYS(al.changed_fields) AS changed_fields,
    -- แสดงค่าก่อน-หลัง สำหรับ balance
    JSON_EXTRACT(al.old_values, '$.balance') AS old_balance,
    JSON_EXTRACT(al.new_values, '$.balance') AS new_balance,
    JSON_EXTRACT(al.old_values, '$.status') AS old_status,
    JSON_EXTRACT(al.new_values, '$.status') AS new_status
FROM audit_log al
WHERE al.table_name = 'accounts'
  AND al.record_id = 1
ORDER BY al.performed_at DESC;

-- Stored Procedure สำหรับบันทึก Audit Log อัตโนมัติ
DELIMITER //

CREATE PROCEDURE log_audit(
    IN p_table_name VARCHAR(100),
    IN p_record_id BIGINT,
    IN p_action ENUM('INSERT','UPDATE','DELETE','SELECT_SENSITIVE'),
    IN p_old_values JSON,
    IN p_new_values JSON,
    IN p_username VARCHAR(100),
    IN p_ip_address VARCHAR(45)
)
BEGIN
    DECLARE v_changed_fields JSON;
    
    -- Calculate changed fields for UPDATE
    IF p_action = 'UPDATE' AND p_old_values IS NOT NULL AND p_new_values IS NOT NULL THEN
        SET v_changed_fields = (
            SELECT JSON_ARRAYAGG(k)
            FROM JSON_TABLE(
                JSON_KEYS(p_new_values),
                '$[*]' COLUMNS (k VARCHAR(100) PATH '$')
            ) keys
            WHERE JSON_EXTRACT(p_old_values, CONCAT('$.', k)) != JSON_EXTRACT(p_new_values, CONCAT('$.', k))
        );
    END IF;
    
    INSERT INTO audit_log (table_name, record_id, action, old_values, new_values, changed_fields, username, ip_address)
    VALUES (p_table_name, p_record_id, p_action, p_old_values, p_new_values, v_changed_fields, p_username, p_ip_address);
END //

-- Trigger สำหรับ Balance Changes
CREATE TRIGGER trg_accounts_audit_update
AFTER UPDATE ON accounts
FOR EACH ROW
BEGIN
    IF OLD.balance != NEW.balance OR OLD.status != NEW.status THEN
        CALL log_audit(
            'accounts',
            NEW.account_id,
            'UPDATE',
            JSON_OBJECT(
                'balance', OLD.balance,
                'available_balance', OLD.available_balance,
                'status', OLD.status
            ),
            JSON_OBJECT(
                'balance', NEW.balance,
                'available_balance', NEW.available_balance,
                'status', NEW.status
            ),
            USER(),
            NULL
        );
    END IF;
END //

DELIMITER ;
```

---

### Query 11: Daily Balance Reconciliation

```sql
-- ตรวจสอบว่า Transaction สอดคล้องกับ Balance ของบัญชี
WITH transaction_check AS (
    SELECT 
        account_id,
        SUM(CASE WHEN direction = 'credit' THEN amount ELSE -amount END) AS net_transaction_amount,
        MIN(balance_before) AS earliest_balance_before,
        MAX(balance_after) AS latest_balance_after,
        COUNT(*) AS transaction_count
    FROM transactions
    WHERE status = 'completed'
    GROUP BY account_id
)
SELECT 
    a.account_number,
    a.balance AS current_balance,
    tc.net_transaction_amount,
    -- The first transaction's balance_before should be the initial balance
    tc.earliest_balance_before AS initial_balance,
    tc.earliest_balance_before + tc.net_transaction_amount AS calculated_balance,
    -- Flag discrepancy
    CASE 
        WHEN ABS(a.balance - (tc.earliest_balance_before + tc.net_transaction_amount)) > 0.01
        THEN 'DISCREPANCY FOUND!'
        ELSE 'OK'
    END AS reconciliation_status,
    a.balance - (tc.earliest_balance_before + tc.net_transaction_amount) AS variance
FROM accounts a
JOIN transaction_check tc ON a.account_id = tc.account_id
ORDER BY ABS(a.balance - (tc.earliest_balance_before + tc.net_transaction_amount)) DESC;
```

---

### Query 12: Overdue Loan Installments

```sql
SELECT 
    l.loan_number,
    CONCAT(c.first_name, ' ', c.last_name) AS borrower_name,
    c.phone,
    c.email,
    l.loan_type,
    l.outstanding_balance,
    -- Overdue installments
    COUNT(la.schedule_id) AS overdue_installments,
    SUM(la.installment_amount) AS total_overdue_amount,
    MIN(la.due_date) AS earliest_due_date,
    DATEDIFF(CURDATE(), MIN(la.due_date)) AS max_days_overdue,
    -- NPL Classification (Thai Bank of Thailand standard)
    CASE 
        WHEN MAX(DATEDIFF(CURDATE(), la.due_date)) <= 90 THEN 'Special Mention (SM)'
        WHEN MAX(DATEDIFF(CURDATE(), la.due_date)) <= 180 THEN 'Sub-Standard (SS)'
        WHEN MAX(DATEDIFF(CURDATE(), la.due_date)) <= 365 THEN 'Doubtful (D)'
        ELSE 'Doubtful of Loss (DL)'
    END AS loan_classification,
    -- Expected Recovery (Thai BOT provisioning rates)
    ROUND(l.outstanding_balance * 
        CASE 
            WHEN MAX(DATEDIFF(CURDATE(), la.due_date)) <= 90 THEN 0.02
            WHEN MAX(DATEDIFF(CURDATE(), la.due_date)) <= 180 THEN 0.20
            WHEN MAX(DATEDIFF(CURDATE(), la.due_date)) <= 365 THEN 0.50
            ELSE 1.00
        END,
        2
    ) AS required_provision
FROM loans l
JOIN customers c ON l.customer_id = c.customer_id
JOIN loan_amortization la ON l.loan_id = la.loan_id
WHERE la.status = 'overdue'
  OR (la.status = 'scheduled' AND la.due_date < CURDATE())
GROUP BY l.loan_id, l.loan_number, c.first_name, c.last_name, c.phone, c.email,
         l.loan_type, l.outstanding_balance
ORDER BY max_days_overdue DESC;
```

---

## แบบฝึกหัด (Challenge Exercises)

1. **Interest Compounding**: เขียน Query คำนวณดอกเบี้ยแบบ Compound Interest (ดอกเบี้ยทบต้น) สำหรับ Savings Account ที่คำนวณดอกเบี้ยรายวันแต่จ่ายรายเดือน

2. **Currency Exchange**: สร้าง Schema และ Query สำหรับบัญชีเงินตราต่างประเทศ (Multi-currency Account) พร้อม FX conversion ที่ถูกต้อง

3. **Credit Score**: เขียน Query คำนวณ Internal Credit Score ของลูกค้าโดยพิจารณาจาก: ประวัติการชำระหนี้, อัตราส่วนหนี้ต่อรายได้, ระยะเวลาเป็นลูกค้า

4. **ATM Network Analysis**: สร้าง Query วิเคราะห์การใช้งาน ATM แต่ละเครื่อง (ธุรกรรมต่อวัน, จำนวนเงินที่เบิก, ช่วงเวลาที่ใช้งานสูงสุด)

5. **FATF Reporting**: เขียน Query สร้าง Suspicious Transaction Report (STR) ตาม FATF guidelines สำหรับธุรกรรมที่น่าสงสัย

6. **Portfolio Stress Testing**: จำลองสถานการณ์ว่าถ้าดอกเบี้ยขึ้น 2% จะส่งผลต่อ Net Interest Income ของธนาคารอย่างไร

7. **Customer 360 View**: เขียน Single Query ที่แสดงข้อมูลครบถ้วนของลูกค้า: บัญชีทั้งหมด, ยอดเงิน, สินเชื่อ, ประวัติธุรกรรม 3 เดือน, Risk Score

8. **Inter-bank Settlement**: ออกแบบ Query สำหรับการ Reconcile ธุรกรรม BAHTNET (ระหว่างธนาคาร) ประจำวัน

9. **KPI Dashboard**: สร้าง Banking KPI Dashboard แสดง: NIM (Net Interest Margin), ROA, ROE, CAR (Capital Adequacy Ratio), NPL Ratio

10. **Regulatory Compliance**: สร้าง Query ตรวจสอบว่าลูกค้าที่มีธุรกรรม > 1,000,000 บาทต่อวัน ได้รับการ KYC ครบถ้วนหรือไม่ (ตาม AMLO requirement)
