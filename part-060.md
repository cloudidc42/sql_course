# Part 060: Real-World Database Design Projects
# โปรเจกต์การออกแบบฐานข้อมูลจริง

---

## บทนำ

ในบทนี้เราจะนำความรู้ทั้งหมดจาก Parts 051-059 มาประยุกต์ใช้ออกแบบระบบจริง 4 ระบบ:

1. **E-Commerce Platform** - ระบบร้านค้าออนไลน์
2. **Hospital Management System** - ระบบบริหารโรงพยาบาล
3. **Social Media Platform** - ระบบ Social Network
4. **School Management System** - ระบบบริหารโรงเรียน

---

## Project 1: E-Commerce Platform

### Requirements

```
ฟีเจอร์หลัก:
- สินค้าหลากหลาย categories พร้อม attributes ที่แตกต่างกัน
- Customers สมัครสมาชิก, Login, จัดการ Addresses
- Shopping Cart
- Orders: place → pay → pack → ship → deliver
- Payment: Credit Card, Promptpay, COD
- Reviews และ Ratings
- Sellers/Merchants หลายราย (Marketplace model)
- Inventory Management
- Promotions และ Coupons
```

### Full SQL DDL

```sql
-- ============================================================
-- E-Commerce Platform Schema
-- ============================================================

-- Extensions
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION IF NOT EXISTS pg_trgm;

-- ============================================================
-- Users & Authentication
-- ============================================================

CREATE TABLE users (
    user_id       SERIAL PRIMARY KEY,
    email         VARCHAR(255) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    role          VARCHAR(20) NOT NULL DEFAULT 'customer'
                  CHECK (role IN ('customer','seller','admin','staff')),
    is_verified   BOOLEAN DEFAULT FALSE,
    is_active     BOOLEAN DEFAULT TRUE,
    created_at    TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    last_login    TIMESTAMP
);

CREATE TABLE user_profiles (
    user_id       INTEGER PRIMARY KEY REFERENCES users ON DELETE CASCADE,
    first_name    VARCHAR(100),
    last_name     VARCHAR(100),
    phone         VARCHAR(20),
    birthdate     DATE,
    gender        VARCHAR(10),
    avatar_url    TEXT,
    updated_at    TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE user_addresses (
    address_id    SERIAL PRIMARY KEY,
    user_id       INTEGER NOT NULL REFERENCES users ON DELETE CASCADE,
    label         VARCHAR(50) DEFAULT 'home',  -- 'home','office','other'
    recipient_name VARCHAR(100) NOT NULL,
    phone         VARCHAR(20) NOT NULL,
    street        VARCHAR(200) NOT NULL,
    district      VARCHAR(100),
    province      VARCHAR(100) NOT NULL,
    postal_code   VARCHAR(10) NOT NULL,
    country_code  CHAR(2) DEFAULT 'TH',
    is_default    BOOLEAN DEFAULT FALSE,
    created_at    TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_addresses_user ON user_addresses(user_id);

-- ============================================================
-- Sellers / Merchants
-- ============================================================

CREATE TABLE sellers (
    seller_id     SERIAL PRIMARY KEY,
    user_id       INTEGER UNIQUE NOT NULL REFERENCES users,
    shop_name     VARCHAR(200) NOT NULL,
    shop_slug     VARCHAR(200) UNIQUE NOT NULL,
    description   TEXT,
    logo_url      TEXT,
    banner_url    TEXT,
    rating        DECIMAL(3,2) DEFAULT 0,
    review_count  INTEGER DEFAULT 0,
    -- Business info
    business_name VARCHAR(200),
    tax_id        VARCHAR(20),
    bank_account  VARCHAR(20),  -- Should be encrypted
    bank_name     VARCHAR(100),
    -- Status
    status        VARCHAR(20) DEFAULT 'pending'
                  CHECK (status IN ('pending','active','suspended','closed')),
    joined_at     TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- ============================================================
-- Categories (Closure Table for hierarchy)
-- ============================================================

CREATE TABLE categories (
    category_id   SERIAL PRIMARY KEY,
    name          VARCHAR(100) NOT NULL,
    slug          VARCHAR(100) UNIQUE NOT NULL,
    description   TEXT,
    image_url     TEXT,
    is_active     BOOLEAN DEFAULT TRUE
);

CREATE TABLE category_paths (
    ancestor_id   INTEGER NOT NULL REFERENCES categories,
    descendant_id INTEGER NOT NULL REFERENCES categories,
    depth         INTEGER NOT NULL,
    PRIMARY KEY (ancestor_id, descendant_id)
);

-- Root categories
INSERT INTO categories (category_id, name, slug) VALUES
(1, 'Electronics', 'electronics'),
(2, 'Fashion', 'fashion'),
(3, 'Home & Living', 'home-living');

-- Seed category_paths for root nodes
INSERT INTO category_paths VALUES (1,1,0),(2,2,0),(3,3,0);

-- ============================================================
-- Products
-- ============================================================

CREATE TABLE products (
    product_id    SERIAL PRIMARY KEY,
    seller_id     INTEGER NOT NULL REFERENCES sellers,
    category_id   INTEGER NOT NULL REFERENCES categories,
    name          VARCHAR(300) NOT NULL,
    slug          VARCHAR(300) UNIQUE NOT NULL,
    description   TEXT,
    brand         VARCHAR(100),
    specs         JSONB DEFAULT '{}',  -- Flexible product attributes
    status        VARCHAR(20) DEFAULT 'draft'
                  CHECK (status IN ('draft','active','paused','deleted')),
    rating        DECIMAL(3,2) DEFAULT 0,
    review_count  INTEGER DEFAULT 0,
    sold_count    INTEGER DEFAULT 0,
    created_at    TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at    TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    -- Full-text search
    search_vector TSVECTOR GENERATED ALWAYS AS (
        setweight(to_tsvector('simple', name), 'A') ||
        setweight(to_tsvector('simple', COALESCE(brand, '')), 'B') ||
        setweight(to_tsvector('simple', COALESCE(description, '')), 'C')
    ) STORED
);

CREATE INDEX idx_products_seller ON products(seller_id, status);
CREATE INDEX idx_products_category ON products(category_id, status);
CREATE INDEX idx_products_search ON products USING gin(search_vector);
CREATE INDEX idx_products_rating ON products(rating DESC) WHERE status = 'active';

-- Product Variants (Size, Color, etc.)
CREATE TABLE product_variants (
    variant_id    SERIAL PRIMARY KEY,
    product_id    INTEGER NOT NULL REFERENCES products ON DELETE CASCADE,
    sku           VARCHAR(100) UNIQUE NOT NULL,
    -- Variant attributes
    options       JSONB NOT NULL DEFAULT '{}',  -- {"color":"red","size":"M"}
    price         DECIMAL(10,2) NOT NULL,
    compare_price DECIMAL(10,2),  -- Original price (for showing discount)
    cost_price    DECIMAL(10,2),  -- Cost of goods
    stock_qty     INTEGER NOT NULL DEFAULT 0,
    weight_g      INTEGER,
    image_url     TEXT,
    is_active     BOOLEAN DEFAULT TRUE,
    CONSTRAINT positive_price CHECK (price > 0),
    CONSTRAINT sufficient_stock CHECK (stock_qty >= 0)
);

CREATE INDEX idx_variants_product ON product_variants(product_id, is_active);

-- Product Images
CREATE TABLE product_images (
    image_id      SERIAL PRIMARY KEY,
    product_id    INTEGER NOT NULL REFERENCES products ON DELETE CASCADE,
    image_url     TEXT NOT NULL,
    alt_text      VARCHAR(200),
    sort_order    SMALLINT DEFAULT 0,
    is_primary    BOOLEAN DEFAULT FALSE
);

-- ============================================================
-- Shopping Cart
-- ============================================================

CREATE TABLE carts (
    cart_id       SERIAL PRIMARY KEY,
    user_id       INTEGER UNIQUE REFERENCES users ON DELETE CASCADE,
    session_id    VARCHAR(100),  -- For guest users
    created_at    TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at    TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE cart_items (
    cart_item_id  SERIAL PRIMARY KEY,
    cart_id       INTEGER NOT NULL REFERENCES carts ON DELETE CASCADE,
    variant_id    INTEGER NOT NULL REFERENCES product_variants,
    quantity      INTEGER NOT NULL DEFAULT 1 CHECK (quantity > 0),
    added_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UNIQUE (cart_id, variant_id)
);

-- ============================================================
-- Orders
-- ============================================================

CREATE TABLE order_statuses (
    code         VARCHAR(30) PRIMARY KEY,
    label        VARCHAR(100) NOT NULL,
    sort_order   SMALLINT DEFAULT 0,
    is_terminal  BOOLEAN DEFAULT FALSE
);

INSERT INTO order_statuses VALUES
('pending_payment', 'รอชำระเงิน',     1, FALSE),
('payment_confirmed','ยืนยันการชำระ',   2, FALSE),
('processing',       'กำลังเตรียมสินค้า',3, FALSE),
('shipped',          'จัดส่งแล้ว',      4, FALSE),
('delivered',        'ได้รับสินค้าแล้ว', 5, TRUE),
('cancelled',        'ยกเลิก',           6, TRUE),
('refunded',         'คืนเงินแล้ว',      7, TRUE);

CREATE TABLE orders (
    order_id      SERIAL PRIMARY KEY,
    order_no      VARCHAR(30) UNIQUE NOT NULL DEFAULT 'ORD-' || TO_CHAR(NOW(),'YYYYMMDD') || '-' || LPAD(nextval('order_seq')::TEXT, 6, '0'),
    user_id       INTEGER NOT NULL REFERENCES users,
    status        VARCHAR(30) NOT NULL DEFAULT 'pending_payment'
                  REFERENCES order_statuses(code),
    -- Amounts
    subtotal      DECIMAL(12,2) NOT NULL,
    shipping_fee  DECIMAL(8,2) NOT NULL DEFAULT 0,
    discount      DECIMAL(10,2) NOT NULL DEFAULT 0,
    total         DECIMAL(12,2) NOT NULL,
    -- Shipping info (snapshot at order time)
    ship_name     VARCHAR(100) NOT NULL,
    ship_phone    VARCHAR(20) NOT NULL,
    ship_address  TEXT NOT NULL,
    ship_province VARCHAR(100),
    ship_postal   VARCHAR(10),
    -- Metadata
    note          TEXT,
    coupon_code   VARCHAR(50),
    created_at    TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at    TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    -- Computed
    CONSTRAINT order_total_correct CHECK (total = subtotal + shipping_fee - discount)
);

CREATE SEQUENCE order_seq START 1;
CREATE INDEX idx_orders_user ON orders(user_id, created_at DESC);
CREATE INDEX idx_orders_status ON orders(status, created_at DESC);

CREATE TABLE order_items (
    order_item_id  SERIAL PRIMARY KEY,
    order_id       INTEGER NOT NULL REFERENCES orders ON DELETE CASCADE,
    variant_id     INTEGER NOT NULL REFERENCES product_variants,
    seller_id      INTEGER NOT NULL REFERENCES sellers,
    -- Snapshot at order time
    product_name   VARCHAR(300) NOT NULL,
    variant_options JSONB,
    sku            VARCHAR(100),
    unit_price     DECIMAL(10,2) NOT NULL,
    quantity       INTEGER NOT NULL CHECK (quantity > 0),
    subtotal       DECIMAL(12,2) NOT NULL,
    -- Fulfillment
    fulfillment_status VARCHAR(20) DEFAULT 'pending',
    tracking_no    VARCHAR(100),
    shipped_at     TIMESTAMP,
    delivered_at   TIMESTAMP
);

-- ============================================================
-- Payments
-- ============================================================

CREATE TABLE payments (
    payment_id     SERIAL PRIMARY KEY,
    order_id       INTEGER NOT NULL REFERENCES orders,
    payment_method VARCHAR(20) NOT NULL
                   CHECK (payment_method IN ('credit_card','promptpay','cod','wallet')),
    amount         DECIMAL(12,2) NOT NULL,
    status         VARCHAR(20) DEFAULT 'pending'
                   CHECK (status IN ('pending','processing','completed','failed','refunded')),
    -- Gateway info
    gateway        VARCHAR(50),  -- 'stripe', 'omise', 'gbprimepay'
    transaction_id VARCHAR(200),
    gateway_response JSONB,
    -- Timestamps
    initiated_at   TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    completed_at   TIMESTAMP
);

CREATE INDEX idx_payments_order ON payments(order_id);

-- ============================================================
-- Reviews
-- ============================================================

CREATE TABLE reviews (
    review_id     SERIAL PRIMARY KEY,
    product_id    INTEGER NOT NULL REFERENCES products,
    user_id       INTEGER NOT NULL REFERENCES users,
    order_item_id INTEGER REFERENCES order_items,  -- Verified purchase
    rating        SMALLINT NOT NULL CHECK (rating BETWEEN 1 AND 5),
    title         VARCHAR(200),
    body          TEXT,
    images        TEXT[],  -- Array of image URLs
    is_approved   BOOLEAN DEFAULT TRUE,
    helpful_count INTEGER DEFAULT 0,
    created_at    TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UNIQUE (product_id, user_id, order_item_id)  -- One review per purchase
);

CREATE INDEX idx_reviews_product ON reviews(product_id, rating DESC);

-- Update product rating when review added/changed
CREATE OR REPLACE FUNCTION update_product_rating()
RETURNS TRIGGER AS $$
BEGIN
    UPDATE products
    SET
        rating = (SELECT AVG(rating) FROM reviews WHERE product_id = COALESCE(NEW.product_id, OLD.product_id) AND is_approved = TRUE),
        review_count = (SELECT COUNT(*) FROM reviews WHERE product_id = COALESCE(NEW.product_id, OLD.product_id) AND is_approved = TRUE)
    WHERE product_id = COALESCE(NEW.product_id, OLD.product_id);
    RETURN COALESCE(NEW, OLD);
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_update_product_rating
    AFTER INSERT OR UPDATE OR DELETE ON reviews
    FOR EACH ROW EXECUTE FUNCTION update_product_rating();

-- ============================================================
-- Promotions & Coupons
-- ============================================================

CREATE TABLE coupons (
    coupon_id     SERIAL PRIMARY KEY,
    code          VARCHAR(50) UNIQUE NOT NULL,
    discount_type VARCHAR(20) NOT NULL CHECK (discount_type IN ('fixed','percent','free_shipping')),
    discount_value DECIMAL(10,2) NOT NULL,
    min_order     DECIMAL(10,2) DEFAULT 0,
    max_discount  DECIMAL(10,2),  -- Cap for percent type
    usage_limit   INTEGER,
    used_count    INTEGER DEFAULT 0,
    valid_from    TIMESTAMP NOT NULL,
    valid_until   TIMESTAMP NOT NULL,
    is_active     BOOLEAN DEFAULT TRUE,
    CONSTRAINT valid_period CHECK (valid_until > valid_from)
);

CREATE INDEX idx_coupons_code ON coupons(code) WHERE is_active = TRUE;
```

---

## Project 2: Hospital Management System

### Requirements

```
ฟีเจอร์หลัก:
- Patient Registration และ Medical History
- Doctors และ Departments
- Appointments: ทำนัด → ตรวจ → Diagnosis → Prescriptions
- Inpatient: Admission → Ward → Discharge
- Pharmacy: ยาและการจ่ายยา
- Lab Tests และ Results
- Billing
```

### Full SQL DDL

```sql
-- ============================================================
-- Hospital Management System Schema
-- ============================================================

-- ============================================================
-- Departments & Staff
-- ============================================================

CREATE TABLE departments (
    dept_id       SERIAL PRIMARY KEY,
    name          VARCHAR(200) NOT NULL,
    code          VARCHAR(10) UNIQUE NOT NULL,
    description   TEXT,
    floor         SMALLINT,
    phone         VARCHAR(20),
    head_doctor_id INTEGER,  -- FK added later (circular ref)
    is_active     BOOLEAN DEFAULT TRUE
);

CREATE TABLE staff (
    staff_id      SERIAL PRIMARY KEY,
    staff_type    VARCHAR(20) NOT NULL
                  CHECK (staff_type IN ('doctor','nurse','pharmacist','admin','lab_tech')),
    employee_no   VARCHAR(20) UNIQUE NOT NULL,
    first_name    VARCHAR(100) NOT NULL,
    last_name     VARCHAR(100) NOT NULL,
    email         VARCHAR(255) UNIQUE NOT NULL,
    phone         VARCHAR(20),
    gender        VARCHAR(10),
    birthdate     DATE,
    dept_id       INTEGER REFERENCES departments,
    hire_date     DATE NOT NULL,
    is_active     BOOLEAN DEFAULT TRUE,
    -- Login
    password_hash VARCHAR(255),
    last_login    TIMESTAMP
);

CREATE TABLE doctors (
    staff_id         INTEGER PRIMARY KEY REFERENCES staff ON DELETE CASCADE,
    license_no       VARCHAR(50) UNIQUE NOT NULL,
    specialization   VARCHAR(200) NOT NULL,
    sub_specialization VARCHAR(200),
    qualification    VARCHAR(300),  -- 'MD, FRCS'
    consultation_fee DECIMAL(8,2) DEFAULT 0,
    years_experience SMALLINT,
    bio              TEXT
);

-- Resolve circular reference
ALTER TABLE departments
ADD CONSTRAINT fk_head_doctor FOREIGN KEY (head_doctor_id) REFERENCES doctors(staff_id);

-- Doctor Schedule
CREATE TABLE doctor_schedules (
    schedule_id    SERIAL PRIMARY KEY,
    doctor_id      INTEGER NOT NULL REFERENCES doctors,
    day_of_week    SMALLINT NOT NULL CHECK (day_of_week BETWEEN 0 AND 6),  -- 0=Sunday
    start_time     TIME NOT NULL,
    end_time       TIME NOT NULL,
    slot_minutes   SMALLINT DEFAULT 30,
    max_patients   SMALLINT DEFAULT 20,
    room           VARCHAR(20),
    is_active      BOOLEAN DEFAULT TRUE,
    CONSTRAINT valid_time CHECK (end_time > start_time)
);

-- ============================================================
-- Patients
-- ============================================================

CREATE TABLE patients (
    patient_id    SERIAL PRIMARY KEY,
    hn            VARCHAR(20) UNIQUE NOT NULL,  -- Hospital Number
    -- Personal info
    first_name    VARCHAR(100) NOT NULL,
    last_name     VARCHAR(100) NOT NULL,
    birthdate     DATE NOT NULL,
    gender        VARCHAR(10) NOT NULL CHECK (gender IN ('male','female','other')),
    id_card       VARCHAR(20) UNIQUE,  -- National ID (encrypted ideally)
    nationality   VARCHAR(50) DEFAULT 'Thai',
    blood_type    VARCHAR(5),
    -- Contact
    phone         VARCHAR(20),
    email         VARCHAR(255),
    address       TEXT,
    -- Emergency Contact
    emergency_name  VARCHAR(100),
    emergency_phone VARCHAR(20),
    emergency_relation VARCHAR(50),
    -- Medical flags
    has_allergies BOOLEAN DEFAULT FALSE,
    has_chronic   BOOLEAN DEFAULT FALSE,
    -- System
    registered_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    is_active     BOOLEAN DEFAULT TRUE
);

CREATE INDEX idx_patients_hn ON patients(hn);
CREATE INDEX idx_patients_name ON patients(last_name, first_name);
CREATE INDEX idx_patients_id_card ON patients(id_card);

-- Allergies
CREATE TABLE patient_allergies (
    allergy_id    SERIAL PRIMARY KEY,
    patient_id    INTEGER NOT NULL REFERENCES patients ON DELETE CASCADE,
    allergen      VARCHAR(200) NOT NULL,  -- drug name, food, substance
    allergen_type VARCHAR(30) CHECK (allergen_type IN ('drug','food','environmental','other')),
    reaction      TEXT,
    severity      VARCHAR(20) CHECK (severity IN ('mild','moderate','severe','anaphylaxis')),
    noted_at      DATE,
    noted_by      INTEGER REFERENCES staff
);

-- Chronic Conditions
CREATE TABLE patient_chronic (
    chronic_id    SERIAL PRIMARY KEY,
    patient_id    INTEGER NOT NULL REFERENCES patients ON DELETE CASCADE,
    icd10_code    VARCHAR(10),
    condition     VARCHAR(200) NOT NULL,
    diagnosed_at  DATE,
    diagnosed_by  INTEGER REFERENCES doctors,
    notes         TEXT
);

-- ============================================================
-- Appointments
-- ============================================================

CREATE TABLE appointments (
    appt_id       SERIAL PRIMARY KEY,
    patient_id    INTEGER NOT NULL REFERENCES patients,
    doctor_id     INTEGER NOT NULL REFERENCES doctors,
    dept_id       INTEGER NOT NULL REFERENCES departments,
    appt_type     VARCHAR(20) DEFAULT 'outpatient'
                  CHECK (appt_type IN ('outpatient','follow_up','emergency','telemedicine')),
    appt_date     DATE NOT NULL,
    appt_time     TIME NOT NULL,
    status        VARCHAR(20) DEFAULT 'scheduled'
                  CHECK (status IN ('scheduled','confirmed','checked_in','in_progress','completed','cancelled','no_show')),
    chief_complaint TEXT,
    room          VARCHAR(20),
    queue_no      INTEGER,
    note          TEXT,
    created_at    TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_appts_doctor ON appointments(doctor_id, appt_date);
CREATE INDEX idx_appts_patient ON appointments(patient_id, appt_date DESC);
CREATE INDEX idx_appts_date ON appointments(appt_date, status);

-- ============================================================
-- Medical Encounters (Visits)
-- ============================================================

CREATE TABLE encounters (
    encounter_id   SERIAL PRIMARY KEY,
    appt_id        INTEGER REFERENCES appointments,
    patient_id     INTEGER NOT NULL REFERENCES patients,
    doctor_id      INTEGER NOT NULL REFERENCES doctors,
    encounter_type VARCHAR(20) DEFAULT 'outpatient',
    encounter_date DATE NOT NULL DEFAULT CURRENT_DATE,
    -- Vitals
    weight_kg      DECIMAL(5,2),
    height_cm      DECIMAL(5,2),
    bp_systolic    SMALLINT,
    bp_diastolic   SMALLINT,
    heart_rate     SMALLINT,
    temp_celsius   DECIMAL(4,2),
    oxygen_sat     SMALLINT,
    -- Clinical
    chief_complaint TEXT,
    hpi            TEXT,  -- History of Present Illness
    physical_exam  TEXT,
    assessment     TEXT,
    plan           TEXT,
    -- Status
    status         VARCHAR(20) DEFAULT 'open',
    started_at     TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    ended_at       TIMESTAMP
);

-- Diagnoses (ICD-10)
CREATE TABLE diagnoses (
    diagnosis_id   SERIAL PRIMARY KEY,
    encounter_id   INTEGER NOT NULL REFERENCES encounters ON DELETE CASCADE,
    icd10_code     VARCHAR(10) NOT NULL,
    description    VARCHAR(300) NOT NULL,
    is_primary     BOOLEAN DEFAULT FALSE,
    diagnosis_type VARCHAR(20) DEFAULT 'working'
                   CHECK (diagnosis_type IN ('working','confirmed','differential','ruled_out'))
);

-- ============================================================
-- Medications & Prescriptions
-- ============================================================

CREATE TABLE medications (
    med_id        SERIAL PRIMARY KEY,
    name          VARCHAR(200) NOT NULL,
    generic_name  VARCHAR(200),
    form          VARCHAR(50),  -- 'tablet', 'capsule', 'syrup', 'injection'
    strength      VARCHAR(50),  -- '500mg', '10mg/5ml'
    manufacturer  VARCHAR(100),
    unit_price    DECIMAL(8,2),
    stock_qty     INTEGER DEFAULT 0,
    min_stock     INTEGER DEFAULT 10,
    is_active     BOOLEAN DEFAULT TRUE
);

CREATE TABLE prescriptions (
    prescription_id SERIAL PRIMARY KEY,
    encounter_id   INTEGER NOT NULL REFERENCES encounters ON DELETE CASCADE,
    doctor_id      INTEGER NOT NULL REFERENCES doctors,
    issued_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    dispensed_at   TIMESTAMP,
    dispensed_by   INTEGER REFERENCES staff,
    total_cost     DECIMAL(10,2),
    status         VARCHAR(20) DEFAULT 'pending'
                   CHECK (status IN ('pending','dispensed','cancelled'))
);

CREATE TABLE prescription_items (
    item_id         SERIAL PRIMARY KEY,
    prescription_id INTEGER NOT NULL REFERENCES prescriptions ON DELETE CASCADE,
    med_id          INTEGER NOT NULL REFERENCES medications,
    -- Dosage
    dose            VARCHAR(50) NOT NULL,  -- '500mg'
    frequency       VARCHAR(50) NOT NULL,  -- 'twice daily', '3 times/day'
    route           VARCHAR(30),           -- 'oral', 'IV', 'topical'
    duration_days   INTEGER,
    quantity        INTEGER NOT NULL,
    instructions    TEXT,
    unit_price      DECIMAL(8,2) NOT NULL,
    total_price     DECIMAL(10,2) NOT NULL
);

-- ============================================================
-- Inpatient (Admission / Ward)
-- ============================================================

CREATE TABLE wards (
    ward_id       SERIAL PRIMARY KEY,
    name          VARCHAR(100) NOT NULL,
    dept_id       INTEGER REFERENCES departments,
    total_beds    SMALLINT NOT NULL,
    ward_type     VARCHAR(30)  -- 'general', 'icu', 'surgery', 'maternity'
);

CREATE TABLE beds (
    bed_id        SERIAL PRIMARY KEY,
    ward_id       INTEGER NOT NULL REFERENCES wards,
    bed_no        VARCHAR(10) NOT NULL,
    bed_type      VARCHAR(20) DEFAULT 'standard',
    is_occupied   BOOLEAN DEFAULT FALSE,
    UNIQUE (ward_id, bed_no)
);

CREATE TABLE admissions (
    admission_id   SERIAL PRIMARY KEY,
    patient_id     INTEGER NOT NULL REFERENCES patients,
    admitting_doctor INTEGER NOT NULL REFERENCES doctors,
    bed_id         INTEGER NOT NULL REFERENCES beds,
    admission_date TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    expected_discharge DATE,
    discharge_date TIMESTAMP,
    diagnosis      TEXT,
    discharge_summary TEXT,
    status         VARCHAR(20) DEFAULT 'admitted'
                   CHECK (status IN ('admitted','transferred','discharged','deceased'))
);

-- ============================================================
-- Lab Tests
-- ============================================================

CREATE TABLE lab_tests (
    test_id       SERIAL PRIMARY KEY,
    code          VARCHAR(20) UNIQUE NOT NULL,
    name          VARCHAR(200) NOT NULL,
    category      VARCHAR(50),  -- 'hematology','biochemistry','microbiology'
    sample_type   VARCHAR(50),  -- 'blood','urine','stool','culture'
    turnaround_hours SMALLINT DEFAULT 24,
    price         DECIMAL(8,2) NOT NULL
);

CREATE TABLE lab_orders (
    order_id      SERIAL PRIMARY KEY,
    encounter_id  INTEGER NOT NULL REFERENCES encounters,
    test_id       INTEGER NOT NULL REFERENCES lab_tests,
    ordered_by    INTEGER NOT NULL REFERENCES doctors,
    ordered_at    TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    status        VARCHAR(20) DEFAULT 'ordered'
                  CHECK (status IN ('ordered','collected','processing','completed','cancelled')),
    urgency       VARCHAR(10) DEFAULT 'routine' CHECK (urgency IN ('routine','urgent','stat')),
    -- Results
    result_value  TEXT,
    result_unit   VARCHAR(30),
    ref_range     VARCHAR(100),
    interpretation VARCHAR(20) CHECK (interpretation IN ('normal','low','high','critical')),
    reported_at   TIMESTAMP,
    reported_by   INTEGER REFERENCES staff
);

-- ============================================================
-- Billing
-- ============================================================

CREATE TABLE bills (
    bill_id        SERIAL PRIMARY KEY,
    patient_id     INTEGER NOT NULL REFERENCES patients,
    encounter_id   INTEGER REFERENCES encounters,
    admission_id   INTEGER REFERENCES admissions,
    bill_date      DATE NOT NULL DEFAULT CURRENT_DATE,
    due_date       DATE NOT NULL,
    subtotal       DECIMAL(12,2) NOT NULL,
    discount       DECIMAL(10,2) DEFAULT 0,
    tax            DECIMAL(10,2) DEFAULT 0,
    total          DECIMAL(12,2) NOT NULL,
    paid_amount    DECIMAL(12,2) DEFAULT 0,
    status         VARCHAR(20) DEFAULT 'unpaid'
                   CHECK (status IN ('draft','unpaid','partial','paid','overdue','waived'))
);

CREATE TABLE bill_items (
    item_id       SERIAL PRIMARY KEY,
    bill_id       INTEGER NOT NULL REFERENCES bills ON DELETE CASCADE,
    description   VARCHAR(300) NOT NULL,
    item_type     VARCHAR(30),  -- 'consultation','lab','medication','room','procedure'
    quantity      INTEGER DEFAULT 1,
    unit_price    DECIMAL(10,2) NOT NULL,
    total_price   DECIMAL(10,2) NOT NULL
);
```

---

## Project 3: Social Media Platform

### Requirements

```
ฟีเจอร์หลัก:
- User Profiles และ Follow System
- Posts (Text, Image, Video)
- Stories (24-hour expiry)
- Comments (nested)
- Reactions (Like, Love, Haha, Sad, Angry)
- Messaging (DM)
- Notifications
- Hashtags
- Explore/Feed Algorithm support
```

### Full SQL DDL

```sql
-- ============================================================
-- Social Media Platform Schema
-- ============================================================

-- ============================================================
-- Users
-- ============================================================

CREATE TABLE users (
    user_id        SERIAL PRIMARY KEY,
    username       VARCHAR(50) UNIQUE NOT NULL,
    email          VARCHAR(255) UNIQUE NOT NULL,
    password_hash  VARCHAR(255) NOT NULL,
    is_verified    BOOLEAN DEFAULT FALSE,
    is_active      BOOLEAN DEFAULT TRUE,
    created_at     TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE user_profiles (
    user_id         INTEGER PRIMARY KEY REFERENCES users ON DELETE CASCADE,
    display_name    VARCHAR(100),
    bio             TEXT,
    website         VARCHAR(300),
    avatar_url      TEXT,
    cover_url       TEXT,
    location        VARCHAR(100),
    birthdate       DATE,
    gender          VARCHAR(20),
    is_private      BOOLEAN DEFAULT FALSE,
    -- Verified blue check
    is_official     BOOLEAN DEFAULT FALSE,
    -- Counts (Denormalized)
    post_count      INTEGER DEFAULT 0,
    follower_count  INTEGER DEFAULT 0,
    following_count INTEGER DEFAULT 0,
    updated_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- ============================================================
-- Follow System
-- ============================================================

CREATE TABLE follows (
    follower_id   INTEGER NOT NULL REFERENCES users ON DELETE CASCADE,
    following_id  INTEGER NOT NULL REFERENCES users ON DELETE CASCADE,
    created_at    TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    status        VARCHAR(20) DEFAULT 'active'
                  CHECK (status IN ('active','pending','blocked')),
    PRIMARY KEY (follower_id, following_id),
    CONSTRAINT no_self_follow CHECK (follower_id != following_id)
);

CREATE INDEX idx_follows_following ON follows(following_id, status);

-- Maintain follower/following counts
CREATE OR REPLACE FUNCTION update_follow_counts()
RETURNS TRIGGER AS $$
BEGIN
    IF TG_OP = 'INSERT' AND NEW.status = 'active' THEN
        UPDATE user_profiles SET following_count = following_count + 1 WHERE user_id = NEW.follower_id;
        UPDATE user_profiles SET follower_count = follower_count + 1 WHERE user_id = NEW.following_id;
    ELSIF TG_OP = 'DELETE' AND OLD.status = 'active' THEN
        UPDATE user_profiles SET following_count = GREATEST(0, following_count - 1) WHERE user_id = OLD.follower_id;
        UPDATE user_profiles SET follower_count = GREATEST(0, follower_count - 1) WHERE user_id = OLD.following_id;
    END IF;
    RETURN COALESCE(NEW, OLD);
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_follow_counts
    AFTER INSERT OR DELETE ON follows
    FOR EACH ROW EXECUTE FUNCTION update_follow_counts();

-- ============================================================
-- Posts
-- ============================================================

CREATE TABLE posts (
    post_id       BIGSERIAL PRIMARY KEY,
    user_id       INTEGER NOT NULL REFERENCES users ON DELETE CASCADE,
    post_type     VARCHAR(20) DEFAULT 'standard'
                  CHECK (post_type IN ('standard','story','reel','live')),
    caption       TEXT,
    -- Counts (Denormalized for performance)
    like_count    INTEGER DEFAULT 0,
    comment_count INTEGER DEFAULT 0,
    share_count   INTEGER DEFAULT 0,
    view_count    INTEGER DEFAULT 0,
    -- Settings
    is_public     BOOLEAN DEFAULT TRUE,
    comments_disabled BOOLEAN DEFAULT FALSE,
    -- Status
    status        VARCHAR(20) DEFAULT 'active'
                  CHECK (status IN ('active','hidden','deleted')),
    created_at    TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    -- Story expiry
    expires_at    TIMESTAMP,
    -- Full-text
    search_vector TSVECTOR GENERATED ALWAYS AS (
        to_tsvector('simple', COALESCE(caption, ''))
    ) STORED
);

CREATE INDEX idx_posts_user ON posts(user_id, created_at DESC) WHERE status = 'active';
CREATE INDEX idx_posts_public ON posts(created_at DESC) WHERE status = 'active' AND is_public = TRUE;
CREATE INDEX idx_posts_story ON posts(user_id, expires_at) WHERE post_type = 'story';
CREATE INDEX idx_posts_search ON posts USING gin(search_vector);

-- Post Media (Images/Videos)
CREATE TABLE post_media (
    media_id      BIGSERIAL PRIMARY KEY,
    post_id       BIGINT NOT NULL REFERENCES posts ON DELETE CASCADE,
    media_type    VARCHAR(10) NOT NULL CHECK (media_type IN ('image','video')),
    url           TEXT NOT NULL,
    thumbnail_url TEXT,
    width         INTEGER,
    height        INTEGER,
    duration_sec  INTEGER,  -- For video
    alt_text      VARCHAR(300),
    sort_order    SMALLINT DEFAULT 0
);

-- ============================================================
-- Hashtags
-- ============================================================

CREATE TABLE hashtags (
    hashtag_id    SERIAL PRIMARY KEY,
    name          VARCHAR(100) UNIQUE NOT NULL,
    post_count    INTEGER DEFAULT 0,
    created_at    TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE post_hashtags (
    post_id       BIGINT NOT NULL REFERENCES posts ON DELETE CASCADE,
    hashtag_id    INTEGER NOT NULL REFERENCES hashtags ON DELETE CASCADE,
    PRIMARY KEY (post_id, hashtag_id)
);

CREATE INDEX idx_post_hashtags_tag ON post_hashtags(hashtag_id);

-- ============================================================
-- Reactions
-- ============================================================

CREATE TABLE reactions (
    reaction_id   BIGSERIAL PRIMARY KEY,
    user_id       INTEGER NOT NULL REFERENCES users ON DELETE CASCADE,
    target_type   VARCHAR(20) NOT NULL CHECK (target_type IN ('post','comment','story')),
    target_id     BIGINT NOT NULL,
    reaction_type VARCHAR(20) NOT NULL
                  CHECK (reaction_type IN ('like','love','haha','wow','sad','angry')),
    created_at    TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UNIQUE (user_id, target_type, target_id)
);

CREATE INDEX idx_reactions_target ON reactions(target_type, target_id);

-- Maintain post like_count
CREATE OR REPLACE FUNCTION update_post_like_count()
RETURNS TRIGGER AS $$
BEGIN
    IF TG_OP = 'INSERT' AND NEW.target_type = 'post' THEN
        UPDATE posts SET like_count = like_count + 1 WHERE post_id = NEW.target_id;
    ELSIF TG_OP = 'DELETE' AND OLD.target_type = 'post' THEN
        UPDATE posts SET like_count = GREATEST(0, like_count - 1) WHERE post_id = OLD.target_id;
    END IF;
    RETURN COALESCE(NEW, OLD);
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_post_like_count
    AFTER INSERT OR DELETE ON reactions
    FOR EACH ROW EXECUTE FUNCTION update_post_like_count();

-- ============================================================
-- Comments (Nested - Adjacency List)
-- ============================================================

CREATE TABLE comments (
    comment_id    BIGSERIAL PRIMARY KEY,
    post_id       BIGINT NOT NULL REFERENCES posts ON DELETE CASCADE,
    user_id       INTEGER NOT NULL REFERENCES users ON DELETE CASCADE,
    parent_id     BIGINT REFERENCES comments,  -- NULL = top-level comment
    content       TEXT NOT NULL,
    like_count    INTEGER DEFAULT 0,
    reply_count   INTEGER DEFAULT 0,
    status        VARCHAR(20) DEFAULT 'active' CHECK (status IN ('active','hidden','deleted')),
    created_at    TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_comments_post ON comments(post_id, created_at) WHERE status = 'active';
CREATE INDEX idx_comments_parent ON comments(parent_id) WHERE parent_id IS NOT NULL;

-- ============================================================
-- Direct Messages
-- ============================================================

CREATE TABLE conversations (
    convo_id      BIGSERIAL PRIMARY KEY,
    created_at    TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    last_message_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE conversation_participants (
    convo_id      BIGINT NOT NULL REFERENCES conversations ON DELETE CASCADE,
    user_id       INTEGER NOT NULL REFERENCES users ON DELETE CASCADE,
    joined_at     TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    last_read_at  TIMESTAMP,
    is_muted      BOOLEAN DEFAULT FALSE,
    PRIMARY KEY (convo_id, user_id)
);

CREATE TABLE messages (
    message_id    BIGSERIAL PRIMARY KEY,
    convo_id      BIGINT NOT NULL REFERENCES conversations ON DELETE CASCADE,
    sender_id     INTEGER NOT NULL REFERENCES users,
    content       TEXT,
    media_url     TEXT,
    media_type    VARCHAR(10),
    is_read       BOOLEAN DEFAULT FALSE,
    created_at    TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    deleted_at    TIMESTAMP
);

CREATE INDEX idx_messages_convo ON messages(convo_id, created_at DESC) WHERE deleted_at IS NULL;

-- ============================================================
-- Notifications
-- ============================================================

CREATE TABLE notifications (
    notif_id      BIGSERIAL PRIMARY KEY,
    user_id       INTEGER NOT NULL REFERENCES users ON DELETE CASCADE,
    actor_id      INTEGER REFERENCES users ON DELETE SET NULL,
    notif_type    VARCHAR(30) NOT NULL,  -- 'follow','like','comment','mention','tag'
    entity_type   VARCHAR(20),  -- 'post', 'comment'
    entity_id     BIGINT,
    message       TEXT,
    is_read       BOOLEAN DEFAULT FALSE,
    created_at    TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_notifications_user ON notifications(user_id, is_read, created_at DESC);
```

---

## Project 4: School Management System

### Requirements

```
ฟีเจอร์หลัก:
- Students, Teachers, Parents
- Academic Terms และ Classes
- Enrollments
- Grades และ Report Cards
- Attendance Tracking
- Homework/Assignments
- School Events
- Fee Management
```

### Full SQL DDL

```sql
-- ============================================================
-- School Management System Schema
-- ============================================================

-- ============================================================
-- Academic Structure
-- ============================================================

CREATE TABLE schools (
    school_id     SERIAL PRIMARY KEY,
    name          VARCHAR(300) NOT NULL,
    code          VARCHAR(20) UNIQUE NOT NULL,
    address       TEXT,
    phone         VARCHAR(20),
    email         VARCHAR(255),
    principal_id  INTEGER,  -- FK added later
    established   DATE,
    is_active     BOOLEAN DEFAULT TRUE
);

CREATE TABLE academic_years (
    year_id       SERIAL PRIMARY KEY,
    name          VARCHAR(20) NOT NULL,  -- '2567', '2567-2568'
    start_date    DATE NOT NULL,
    end_date      DATE NOT NULL,
    is_current    BOOLEAN DEFAULT FALSE,
    CONSTRAINT valid_year CHECK (end_date > start_date)
);

-- Only one current year
CREATE UNIQUE INDEX idx_one_current_year ON academic_years(is_current) WHERE is_current = TRUE;

CREATE TABLE terms (
    term_id       SERIAL PRIMARY KEY,
    year_id       INTEGER NOT NULL REFERENCES academic_years,
    term_no       SMALLINT NOT NULL CHECK (term_no BETWEEN 1 AND 3),
    name          VARCHAR(50),  -- 'Term 1', 'Summer'
    start_date    DATE NOT NULL,
    end_date      DATE NOT NULL,
    UNIQUE (year_id, term_no)
);

-- ============================================================
-- Rooms & Facilities
-- ============================================================

CREATE TABLE rooms (
    room_id       SERIAL PRIMARY KEY,
    name          VARCHAR(50) NOT NULL,
    capacity      SMALLINT,
    room_type     VARCHAR(20) DEFAULT 'classroom'
                  CHECK (room_type IN ('classroom','lab','gym','hall','library','admin'))
);

-- ============================================================
-- Subjects
-- ============================================================

CREATE TABLE subjects (
    subject_id    SERIAL PRIMARY KEY,
    code          VARCHAR(20) UNIQUE NOT NULL,
    name          VARCHAR(200) NOT NULL,
    credits       SMALLINT DEFAULT 1,
    grade_level   SMALLINT,  -- 1-12 หรือ NULL=all grades
    subject_type  VARCHAR(20) DEFAULT 'core'
                  CHECK (subject_type IN ('core','elective','activity','special')),
    is_active     BOOLEAN DEFAULT TRUE
);

-- ============================================================
-- Staff & Teachers
-- ============================================================

CREATE TABLE staff (
    staff_id      SERIAL PRIMARY KEY,
    employee_no   VARCHAR(20) UNIQUE NOT NULL,
    first_name    VARCHAR(100) NOT NULL,
    last_name     VARCHAR(100) NOT NULL,
    email         VARCHAR(255) UNIQUE NOT NULL,
    phone         VARCHAR(20),
    role          VARCHAR(30) NOT NULL
                  CHECK (role IN ('teacher','admin','counselor','principal','support')),
    hire_date     DATE NOT NULL,
    is_active     BOOLEAN DEFAULT TRUE,
    password_hash VARCHAR(255)
);

CREATE TABLE teachers (
    staff_id        INTEGER PRIMARY KEY REFERENCES staff ON DELETE CASCADE,
    specialization  VARCHAR(200),
    qualification   VARCHAR(300),
    homeroom_class  INTEGER  -- FK to classes (added later)
);

ALTER TABLE schools
ADD CONSTRAINT fk_principal FOREIGN KEY (principal_id) REFERENCES staff(staff_id);

-- ============================================================
-- Classes & Sections
-- ============================================================

CREATE TABLE classes (
    class_id      SERIAL PRIMARY KEY,
    year_id       INTEGER NOT NULL REFERENCES academic_years,
    grade_level   SMALLINT NOT NULL CHECK (grade_level BETWEEN 1 AND 12),
    section       VARCHAR(10) NOT NULL,  -- 'A', 'B', '1', '2'
    name          VARCHAR(50),  -- 'Grade 10B'
    homeroom_teacher_id INTEGER REFERENCES teachers,
    room_id       INTEGER REFERENCES rooms,
    max_students  SMALLINT DEFAULT 40,
    UNIQUE (year_id, grade_level, section)
);

ALTER TABLE teachers
ADD CONSTRAINT fk_homeroom FOREIGN KEY (homeroom_class) REFERENCES classes(class_id);

-- Subject-Class assignments
CREATE TABLE class_subjects (
    class_subject_id SERIAL PRIMARY KEY,
    class_id      INTEGER NOT NULL REFERENCES classes,
    subject_id    INTEGER NOT NULL REFERENCES subjects,
    teacher_id    INTEGER NOT NULL REFERENCES teachers,
    term_id       INTEGER NOT NULL REFERENCES terms,
    schedule      JSONB,  -- [{"day": 1, "start": "08:00", "end": "09:00", "room_id": 5}]
    UNIQUE (class_id, subject_id, term_id)
);

-- ============================================================
-- Students
-- ============================================================

CREATE TABLE students (
    student_id    SERIAL PRIMARY KEY,
    student_no    VARCHAR(20) UNIQUE NOT NULL,
    first_name    VARCHAR(100) NOT NULL,
    last_name     VARCHAR(100) NOT NULL,
    birthdate     DATE NOT NULL,
    gender        VARCHAR(10) NOT NULL,
    id_card       VARCHAR(20),
    blood_type    VARCHAR(5),
    nationality   VARCHAR(50) DEFAULT 'Thai',
    -- Contact
    phone         VARCHAR(20),
    email         VARCHAR(255),
    address       TEXT,
    -- System
    enrolled_date DATE NOT NULL DEFAULT CURRENT_DATE,
    status        VARCHAR(20) DEFAULT 'active'
                  CHECK (status IN ('active','transferred','graduated','dropped','suspended')),
    password_hash VARCHAR(255),
    created_at    TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_students_no ON students(student_no);
CREATE INDEX idx_students_name ON students(last_name, first_name);

-- Parents/Guardians
CREATE TABLE guardians (
    guardian_id   SERIAL PRIMARY KEY,
    first_name    VARCHAR(100) NOT NULL,
    last_name     VARCHAR(100) NOT NULL,
    relationship  VARCHAR(30) NOT NULL,  -- 'father','mother','guardian','grandparent'
    phone         VARCHAR(20) NOT NULL,
    email         VARCHAR(255),
    occupation    VARCHAR(100),
    password_hash VARCHAR(255)
);

CREATE TABLE student_guardians (
    student_id    INTEGER NOT NULL REFERENCES students ON DELETE CASCADE,
    guardian_id   INTEGER NOT NULL REFERENCES guardians ON DELETE CASCADE,
    is_primary    BOOLEAN DEFAULT FALSE,
    can_pickup    BOOLEAN DEFAULT TRUE,
    emergency_contact BOOLEAN DEFAULT FALSE,
    PRIMARY KEY (student_id, guardian_id)
);

-- Class Enrollment
CREATE TABLE enrollments (
    enrollment_id SERIAL PRIMARY KEY,
    student_id    INTEGER NOT NULL REFERENCES students,
    class_id      INTEGER NOT NULL REFERENCES classes,
    enrolled_at   DATE NOT NULL DEFAULT CURRENT_DATE,
    status        VARCHAR(20) DEFAULT 'active'
                  CHECK (status IN ('active','transferred','withdrawn')),
    UNIQUE (student_id, class_id)
);

-- ============================================================
-- Attendance
-- ============================================================

CREATE TABLE attendance_statuses (
    code        VARCHAR(10) PRIMARY KEY,
    label       VARCHAR(50) NOT NULL,
    is_present  BOOLEAN NOT NULL,
    affects_grade BOOLEAN DEFAULT FALSE
);

INSERT INTO attendance_statuses VALUES
('P', 'มาเรียน', TRUE, FALSE),
('L', 'มาสาย', TRUE, FALSE),
('A', 'ขาดเรียน', FALSE, TRUE),
('E', 'ลาป่วย', FALSE, FALSE),
('V', 'ลากิจ', FALSE, FALSE),
('S', 'ไปราชการ', TRUE, FALSE);

CREATE TABLE attendance (
    attendance_id   BIGSERIAL PRIMARY KEY,
    class_subject_id INTEGER NOT NULL REFERENCES class_subjects,
    student_id      INTEGER NOT NULL REFERENCES students,
    attend_date     DATE NOT NULL,
    status          VARCHAR(10) NOT NULL REFERENCES attendance_statuses,
    noted_by        INTEGER REFERENCES staff,
    note            TEXT,
    recorded_at     TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UNIQUE (class_subject_id, student_id, attend_date)
);

CREATE INDEX idx_attendance_class ON attendance(class_subject_id, attend_date);
CREATE INDEX idx_attendance_student ON attendance(student_id, attend_date DESC);

-- ============================================================
-- Grades
-- ============================================================

CREATE TABLE assessments (
    assessment_id   SERIAL PRIMARY KEY,
    class_subject_id INTEGER NOT NULL REFERENCES class_subjects,
    title           VARCHAR(200) NOT NULL,
    assessment_type VARCHAR(20) NOT NULL
                    CHECK (assessment_type IN ('homework','quiz','midterm','final','project','lab','participation')),
    max_score       DECIMAL(6,2) NOT NULL,
    weight_percent  DECIMAL(5,2),  -- Weight in total grade
    due_date        DATE,
    created_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE grades (
    grade_id        SERIAL PRIMARY KEY,
    assessment_id   INTEGER NOT NULL REFERENCES assessments ON DELETE CASCADE,
    student_id      INTEGER NOT NULL REFERENCES students,
    score           DECIMAL(6,2),
    grade_letter    VARCHAR(5),  -- 'A', 'B+', 'F'
    remarks         TEXT,
    graded_by       INTEGER REFERENCES teachers,
    graded_at       TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UNIQUE (assessment_id, student_id),
    CONSTRAINT valid_score CHECK (score IS NULL OR (score >= 0 AND score <= (SELECT max_score FROM assessments WHERE assessment_id = grades.assessment_id)))
);

-- Report Cards: Materialized View (Update each term)
CREATE MATERIALIZED VIEW term_report_cards AS
SELECT
    e.student_id,
    cs.class_id,
    cs.subject_id,
    cs.term_id,
    COUNT(g.grade_id) AS assessments_completed,
    AVG(g.score / a.max_score * 100) AS avg_percentage,
    CASE
        WHEN AVG(g.score / a.max_score * 100) >= 80 THEN 'A'
        WHEN AVG(g.score / a.max_score * 100) >= 75 THEN 'B+'
        WHEN AVG(g.score / a.max_score * 100) >= 70 THEN 'B'
        WHEN AVG(g.score / a.max_score * 100) >= 65 THEN 'C+'
        WHEN AVG(g.score / a.max_score * 100) >= 60 THEN 'C'
        WHEN AVG(g.score / a.max_score * 100) >= 55 THEN 'D+'
        WHEN AVG(g.score / a.max_score * 100) >= 50 THEN 'D'
        ELSE 'F'
    END AS grade_letter
FROM enrollments e
JOIN class_subjects cs ON e.class_id = cs.class_id
JOIN assessments a ON cs.class_subject_id = a.class_subject_id
LEFT JOIN grades g ON a.assessment_id = g.assessment_id AND g.student_id = e.student_id
WHERE e.status = 'active'
GROUP BY e.student_id, cs.class_id, cs.subject_id, cs.term_id
WITH DATA;

-- ============================================================
-- Fee Management
-- ============================================================

CREATE TABLE fee_types (
    fee_type_id   SERIAL PRIMARY KEY,
    name          VARCHAR(200) NOT NULL,
    amount        DECIMAL(10,2) NOT NULL,
    frequency     VARCHAR(20) DEFAULT 'term'
                  CHECK (frequency IN ('once','term','annual')),
    is_mandatory  BOOLEAN DEFAULT TRUE,
    description   TEXT
);

CREATE TABLE fee_invoices (
    invoice_id    SERIAL PRIMARY KEY,
    student_id    INTEGER NOT NULL REFERENCES students,
    term_id       INTEGER NOT NULL REFERENCES terms,
    fee_type_id   INTEGER NOT NULL REFERENCES fee_types,
    amount        DECIMAL(10,2) NOT NULL,
    due_date      DATE NOT NULL,
    paid_at       TIMESTAMP,
    paid_amount   DECIMAL(10,2) DEFAULT 0,
    status        VARCHAR(20) DEFAULT 'unpaid'
                  CHECK (status IN ('draft','unpaid','partial','paid','waived','overdue')),
    UNIQUE (student_id, term_id, fee_type_id)
);
```

---

## Schema Review Checklist

เมื่อออกแบบ Schema เสร็จแล้ว ควร Review ด้วย Checklist ต่อไปนี้:

```
Normalization:
□ ทุก Non-key column ขึ้นอยู่กับ Primary Key ทั้งหมด (2NF)
□ ไม่มี Transitive Dependencies (3NF)
□ ไม่มี Redundant Data โดยไม่จำเป็น

Constraints:
□ Primary Keys กำหนดครบทุกตาราง
□ Foreign Keys สำหรับทุก Relationships
□ NOT NULL สำหรับ Required Columns
□ CHECK Constraints สำหรับ Valid Values
□ UNIQUE Constraints ป้องกัน Duplicate data
□ Indexes บน FK Columns ทุกตัว

Naming:
□ Table names: lowercase_with_underscore
□ Column names: consistent (snake_case)
□ Primary Keys: table_id หรือ id
□ Timestamps: created_at, updated_at

Audit/Security:
□ Sensitive data encrypted หรือ hashed (passwords, IDs)
□ Audit columns: created_at, updated_at (ถ้าจำเป็น)
□ Soft Delete (ถ้า Business ต้องการ)

Performance:
□ Indexes บน Frequently queried columns
□ Indexes บน FK columns
□ Partial Indexes สำหรับ filtered queries
□ Denormalization ที่จำเป็น (counters, summaries)
□ Partitioning สำหรับ Large tables
```

---

## แบบฝึกหัด (10 ข้อ)

### ข้อ 1
ออกแบบ Inventory Management Module สำหรับ E-Commerce ที่รองรับ Multiple Warehouses

**เฉลย**:
```sql
CREATE TABLE warehouses (
    warehouse_id  SERIAL PRIMARY KEY,
    name          VARCHAR(200) NOT NULL,
    code          VARCHAR(20) UNIQUE NOT NULL,
    address       TEXT,
    manager_id    INTEGER REFERENCES staff,
    is_active     BOOLEAN DEFAULT TRUE
);

CREATE TABLE inventory (
    inventory_id  SERIAL PRIMARY KEY,
    warehouse_id  INTEGER NOT NULL REFERENCES warehouses,
    variant_id    INTEGER NOT NULL REFERENCES product_variants,
    qty_on_hand   INTEGER NOT NULL DEFAULT 0,
    qty_reserved  INTEGER NOT NULL DEFAULT 0,
    qty_available INTEGER GENERATED ALWAYS AS (qty_on_hand - qty_reserved) STORED,
    reorder_point INTEGER DEFAULT 10,
    updated_at    TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UNIQUE (warehouse_id, variant_id),
    CONSTRAINT non_negative_stock CHECK (qty_on_hand >= 0 AND qty_reserved >= 0)
);

CREATE TABLE inventory_movements (
    movement_id   BIGSERIAL PRIMARY KEY,
    inventory_id  INTEGER NOT NULL REFERENCES inventory,
    movement_type VARCHAR(20) NOT NULL CHECK (movement_type IN ('in','out','transfer','adjustment')),
    quantity      INTEGER NOT NULL,
    reference_type VARCHAR(30),  -- 'order', 'return', 'purchase_order'
    reference_id  INTEGER,
    notes         TEXT,
    created_by    INTEGER REFERENCES staff,
    created_at    TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

### ข้อ 2
เพิ่ม Review Reply System ใน E-Commerce (ผู้ขายตอบ Review ได้)

**เฉลย**:
```sql
CREATE TABLE review_replies (
    reply_id      SERIAL PRIMARY KEY,
    review_id     INTEGER NOT NULL REFERENCES reviews ON DELETE CASCADE,
    replier_id    INTEGER NOT NULL REFERENCES users,
    replier_type  VARCHAR(10) NOT NULL CHECK (replier_type IN ('seller','admin')),
    content       TEXT NOT NULL,
    created_at    TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at    TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_review_replies ON review_replies(review_id);

-- Query: Reviews with replies
SELECT
    r.rating,
    r.title,
    r.body,
    r.created_at,
    u.username AS reviewer,
    rr.content AS seller_reply,
    rr.created_at AS replied_at
FROM reviews r
JOIN users u ON r.user_id = u.user_id
LEFT JOIN review_replies rr ON r.review_id = rr.review_id AND rr.replier_type = 'seller'
WHERE r.product_id = 1001
ORDER BY r.created_at DESC;
```

### ข้อ 3
ออกแบบ Appointment Reminder System สำหรับ Hospital

**เฉลย**:
```sql
CREATE TABLE appointment_reminders (
    reminder_id    SERIAL PRIMARY KEY,
    appt_id        INTEGER NOT NULL REFERENCES appointments ON DELETE CASCADE,
    channel        VARCHAR(10) NOT NULL CHECK (channel IN ('sms','email','push','line')),
    send_before_hours INTEGER NOT NULL DEFAULT 24,
    scheduled_at   TIMESTAMP NOT NULL,
    status         VARCHAR(20) DEFAULT 'pending'
                   CHECK (status IN ('pending','sent','failed','cancelled')),
    sent_at        TIMESTAMP,
    error_message  TEXT,
    UNIQUE (appt_id, channel, send_before_hours)
);

-- Auto-create reminders when appointment is scheduled
CREATE OR REPLACE FUNCTION create_appointment_reminders()
RETURNS TRIGGER AS $$
BEGIN
    IF NEW.status = 'scheduled' THEN
        -- 24-hour reminder
        INSERT INTO appointment_reminders (appt_id, channel, send_before_hours, scheduled_at)
        VALUES (NEW.appt_id, 'sms', 24,
                (NEW.appt_date + NEW.appt_time)::TIMESTAMP - INTERVAL '24 hours');
        -- 2-hour reminder
        INSERT INTO appointment_reminders (appt_id, channel, send_before_hours, scheduled_at)
        VALUES (NEW.appt_id, 'sms', 2,
                (NEW.appt_date + NEW.appt_time)::TIMESTAMP - INTERVAL '2 hours');
    END IF;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_appt_reminders
    AFTER INSERT ON appointments
    FOR EACH ROW EXECUTE FUNCTION create_appointment_reminders();
```

### ข้อ 4
สร้าง Feed System สำหรับ Social Media ที่ Efficient

**เฉลย**:
```sql
-- Fan-out on Write: Pre-compute feeds
CREATE TABLE user_feeds (
    feed_id       BIGSERIAL PRIMARY KEY,
    user_id       INTEGER NOT NULL REFERENCES users,
    post_id       BIGINT NOT NULL REFERENCES posts,
    score         DECIMAL(10,4) DEFAULT 0,  -- Engagement score
    seen          BOOLEAN DEFAULT FALSE,
    added_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UNIQUE (user_id, post_id)
);

CREATE INDEX idx_feeds_user ON user_feeds(user_id, score DESC, added_at DESC) WHERE NOT seen;

-- When someone posts, push to followers' feeds
CREATE OR REPLACE FUNCTION push_post_to_feeds()
RETURNS TRIGGER AS $$
BEGIN
    INSERT INTO user_feeds (user_id, post_id, score)
    SELECT
        f.follower_id,
        NEW.post_id,
        EXTRACT(EPOCH FROM NEW.created_at) / 3600.0  -- Recency score
    FROM follows f
    WHERE f.following_id = NEW.user_id
    AND f.status = 'active'
    ON CONFLICT (user_id, post_id) DO NOTHING;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_push_feed
    AFTER INSERT ON posts
    FOR EACH ROW WHEN (NEW.status = 'active' AND NEW.is_public = TRUE)
    EXECUTE FUNCTION push_post_to_feeds();

-- Get user's feed
SELECT p.*, up.display_name, up.avatar_url
FROM user_feeds uf
JOIN posts p ON uf.post_id = p.post_id
JOIN user_profiles up ON p.user_id = up.user_id
WHERE uf.user_id = :current_user_id
AND p.status = 'active'
ORDER BY uf.score DESC, uf.added_at DESC
LIMIT 20;
```

### ข้อ 5
ออกแบบ Scholarship Management สำหรับ School System

**เฉลย**:
```sql
CREATE TABLE scholarships (
    scholarship_id SERIAL PRIMARY KEY,
    name           VARCHAR(300) NOT NULL,
    description    TEXT,
    amount         DECIMAL(10,2) NOT NULL,
    frequency      VARCHAR(20) DEFAULT 'term',
    eligibility    TEXT,
    max_recipients INTEGER,
    budget         DECIMAL(12,2),
    is_active      BOOLEAN DEFAULT TRUE,
    year_id        INTEGER REFERENCES academic_years
);

CREATE TABLE scholarship_applications (
    application_id SERIAL PRIMARY KEY,
    scholarship_id INTEGER NOT NULL REFERENCES scholarships,
    student_id     INTEGER NOT NULL REFERENCES students,
    term_id        INTEGER NOT NULL REFERENCES terms,
    gpa            DECIMAL(4,2),
    household_income DECIMAL(12,2),
    reason         TEXT,
    status         VARCHAR(20) DEFAULT 'pending'
                   CHECK (status IN ('pending','approved','rejected','cancelled')),
    reviewed_by    INTEGER REFERENCES staff,
    reviewed_at    TIMESTAMP,
    notes          TEXT,
    applied_at     TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UNIQUE (scholarship_id, student_id, term_id)
);

CREATE TABLE scholarship_disbursements (
    disbursement_id SERIAL PRIMARY KEY,
    application_id  INTEGER NOT NULL REFERENCES scholarship_applications,
    amount          DECIMAL(10,2) NOT NULL,
    disbursed_at    DATE NOT NULL,
    payment_method  VARCHAR(20),
    reference_no    VARCHAR(50)
);
```

### ข้อ 6
เพิ่ม Product Bundle/Set Feature ใน E-Commerce

**เฉลย**:
```sql
CREATE TABLE product_bundles (
    bundle_id     SERIAL PRIMARY KEY,
    name          VARCHAR(300) NOT NULL,
    description   TEXT,
    bundle_price  DECIMAL(10,2) NOT NULL,
    original_price DECIMAL(10,2),  -- Sum of individual prices
    discount_percent DECIMAL(5,2),
    stock_limit   INTEGER,  -- NULL = unlimited
    valid_from    TIMESTAMP,
    valid_until   TIMESTAMP,
    is_active     BOOLEAN DEFAULT TRUE
);

CREATE TABLE bundle_items (
    bundle_id     INTEGER NOT NULL REFERENCES product_bundles ON DELETE CASCADE,
    variant_id    INTEGER NOT NULL REFERENCES product_variants,
    quantity      INTEGER NOT NULL DEFAULT 1 CHECK (quantity > 0),
    PRIMARY KEY (bundle_id, variant_id)
);

-- Check if bundle is in stock (all items must have enough stock)
CREATE OR REPLACE FUNCTION bundle_available(p_bundle_id INTEGER, p_qty INTEGER DEFAULT 1)
RETURNS BOOLEAN AS $$
BEGIN
    RETURN NOT EXISTS (
        SELECT 1 FROM bundle_items bi
        JOIN product_variants pv ON bi.variant_id = pv.variant_id
        WHERE bi.bundle_id = p_bundle_id
        AND pv.stock_qty < (bi.quantity * p_qty)
    );
END;
$$ LANGUAGE plpgsql;
```

### ข้อ 7
ออกแบบ Telemedicine Feature สำหรับ Hospital

**เฉลย**:
```sql
CREATE TABLE telemedicine_sessions (
    session_id     SERIAL PRIMARY KEY,
    appt_id        INTEGER NOT NULL REFERENCES appointments,
    patient_id     INTEGER NOT NULL REFERENCES patients,
    doctor_id      INTEGER NOT NULL REFERENCES doctors,
    session_url    TEXT,  -- Video call link
    session_token  VARCHAR(200),
    scheduled_at   TIMESTAMP NOT NULL,
    started_at     TIMESTAMP,
    ended_at       TIMESTAMP,
    duration_min   INTEGER GENERATED ALWAYS AS (
        CASE WHEN started_at IS NOT NULL AND ended_at IS NOT NULL
        THEN EXTRACT(EPOCH FROM (ended_at - started_at)) / 60
        ELSE NULL END
    ) STORED,
    status         VARCHAR(20) DEFAULT 'scheduled'
                   CHECK (status IN ('scheduled','in_progress','completed','missed','cancelled')),
    -- Recording
    recording_url  TEXT,
    recording_consent BOOLEAN DEFAULT FALSE,
    -- Notes
    doctor_notes   TEXT,
    prescription_id INTEGER REFERENCES prescriptions
);

CREATE INDEX idx_tele_doctor ON telemedicine_sessions(doctor_id, scheduled_at);
CREATE INDEX idx_tele_patient ON telemedicine_sessions(patient_id, scheduled_at DESC);
```

### ข้อ 8
สร้าง Notification System สำหรับ School ที่แจ้งผู้ปกครอง

**เฉลย**:
```sql
CREATE TABLE school_notifications (
    notif_id      SERIAL PRIMARY KEY,
    school_id     INTEGER REFERENCES schools,
    notif_type    VARCHAR(30) NOT NULL,
    -- Recipients
    target_type   VARCHAR(20) NOT NULL CHECK (target_type IN ('all','class','student','staff','parents')),
    target_id     INTEGER,  -- class_id, student_id, staff_id (NULL if all)
    -- Content
    title         VARCHAR(300) NOT NULL,
    body          TEXT NOT NULL,
    attachments   TEXT[],
    -- Schedule
    send_at       TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    -- Status
    sent_at       TIMESTAMP,
    recipient_count INTEGER DEFAULT 0,
    created_by    INTEGER REFERENCES staff,
    created_at    TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE notification_deliveries (
    delivery_id   BIGSERIAL PRIMARY KEY,
    notif_id      INTEGER NOT NULL REFERENCES school_notifications,
    recipient_type VARCHAR(10) NOT NULL CHECK (recipient_type IN ('student','guardian','staff')),
    recipient_id  INTEGER NOT NULL,
    channel       VARCHAR(10) NOT NULL,
    status        VARCHAR(20) DEFAULT 'pending',
    sent_at       TIMESTAMP,
    read_at       TIMESTAMP
);

CREATE INDEX idx_notif_delivery ON notification_deliveries(recipient_type, recipient_id, read_at);
```

### ข้อ 9
ออกแบบ Event/Activity Tracking สำหรับ Social Media Analytics

**เฉลย**:
```sql
-- Event Store สำหรับ Analytics
CREATE TABLE events (
    event_id      BIGSERIAL,
    user_id       INTEGER,
    session_id    VARCHAR(100),
    event_type    VARCHAR(50) NOT NULL,  -- 'post_view','post_like','profile_visit','search'
    properties    JSONB DEFAULT '{}',
    ip_address    INET,
    user_agent    TEXT,
    device_type   VARCHAR(20),  -- 'mobile', 'desktop', 'tablet'
    occurred_at   TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (event_id, occurred_at)
) PARTITION BY RANGE (occurred_at);

-- Monthly Partitions
CREATE TABLE events_2024_01 PARTITION OF events
FOR VALUES FROM ('2024-01-01') TO ('2024-02-01');

CREATE TABLE events_2024_02 PARTITION OF events
FOR VALUES FROM ('2024-02-01') TO ('2024-03-01');

-- Summary: Daily Active Users
CREATE MATERIALIZED VIEW daily_active_users AS
SELECT
    DATE(occurred_at) AS event_date,
    COUNT(DISTINCT user_id) AS dau
FROM events
WHERE user_id IS NOT NULL
AND occurred_at >= CURRENT_DATE - INTERVAL '90 days'
GROUP BY 1
ORDER BY 1 DESC
WITH DATA;

-- Refresh daily
-- SELECT cron.schedule('0 1 * * *', 'REFRESH MATERIALIZED VIEW CONCURRENTLY daily_active_users');
```

### ข้อ 10
สร้าง Complete Report Card Generation สำหรับ School System

**เฉลย**:
```sql
-- Function สร้าง Report Card ต่อ Student ต่อ Term
CREATE OR REPLACE FUNCTION generate_report_card(
    p_student_id INTEGER,
    p_term_id    INTEGER
) RETURNS TABLE (
    subject_code    VARCHAR,
    subject_name    VARCHAR,
    teacher_name    VARCHAR,
    assessments_count INTEGER,
    average_score   DECIMAL,
    grade_letter    VARCHAR,
    attendance_pct  DECIMAL,
    status          VARCHAR
) AS $$
BEGIN
    RETURN QUERY
    SELECT
        s.code::VARCHAR,
        s.name::VARCHAR,
        (st.first_name || ' ' || st.last_name)::VARCHAR AS teacher_name,
        COUNT(DISTINCT g.grade_id)::INTEGER,
        ROUND(AVG(g.score / a.max_score * 100)::DECIMAL, 2),
        CASE
            WHEN AVG(g.score / a.max_score * 100) >= 80 THEN 'A'
            WHEN AVG(g.score / a.max_score * 100) >= 75 THEN 'B+'
            WHEN AVG(g.score / a.max_score * 100) >= 70 THEN 'B'
            WHEN AVG(g.score / a.max_score * 100) >= 65 THEN 'C+'
            WHEN AVG(g.score / a.max_score * 100) >= 60 THEN 'C'
            WHEN AVG(g.score / a.max_score * 100) >= 55 THEN 'D+'
            WHEN AVG(g.score / a.max_score * 100) >= 50 THEN 'D'
            ELSE 'F'
        END::VARCHAR AS grade_letter,
        ROUND(
            100.0 * SUM(CASE WHEN att.status IN ('P','L','S') THEN 1 ELSE 0 END) /
            NULLIF(COUNT(DISTINCT att.attendance_id), 0)
        , 2)::DECIMAL AS attendance_pct,
        CASE
            WHEN AVG(g.score / a.max_score * 100) >= 50 THEN 'PASS'
            ELSE 'FAIL'
        END::VARCHAR AS status
    FROM enrollments e
    JOIN classes c ON e.class_id = c.class_id
    JOIN class_subjects cs ON c.class_id = cs.class_id AND cs.term_id = p_term_id
    JOIN subjects s ON cs.subject_id = s.subject_id
    JOIN teachers t ON cs.teacher_id = t.staff_id
    JOIN staff st ON t.staff_id = st.staff_id
    LEFT JOIN assessments a ON cs.class_subject_id = a.class_subject_id
    LEFT JOIN grades g ON a.assessment_id = g.assessment_id AND g.student_id = p_student_id
    LEFT JOIN attendance att ON cs.class_subject_id = att.class_subject_id AND att.student_id = p_student_id
    WHERE e.student_id = p_student_id AND e.status = 'active'
    GROUP BY s.code, s.name, st.first_name, st.last_name;
END;
$$ LANGUAGE plpgsql;

-- Usage:
SELECT * FROM generate_report_card(1001, 3);
```

---

*จบ Part 060: Real-World Database Design Projects*

**ยินดีด้วย!** คุณได้เรียนรู้ครบ 10 Parts ของ Database Design และ Normalization (Parts 051-060)

**สรุปสิ่งที่เรียนมา**:
- Part 051: หลักการออกแบบ Database (Design Phases, Top-Down/Bottom-Up)
- Part 052: Entity-Relationship Modeling (ER Diagrams, Cardinality)
- Part 053: First Normal Form (1NF) - Atomic Values
- Part 054: Second Normal Form (2NF) - No Partial Dependencies
- Part 055: Third Normal Form (3NF) - No Transitive Dependencies
- Part 056: BCNF, 4NF, 5NF - Higher Normal Forms
- Part 057: Denormalization - When and How
- Part 058: Schema Design Patterns
- Part 059: Inheritance Patterns
- Part 060: Real-World Projects

**ในส่วนต่อไป (Part 061+)**: Indexes, Query Optimization, และ Performance Tuning
