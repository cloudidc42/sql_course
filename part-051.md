# Part 051: Database Design Principles and Methodology
# หลักการและวิธีการออกแบบฐานข้อมูล

---

## บทนำ: ทำไมการออกแบบฐานข้อมูลจึงสำคัญ?

การออกแบบฐานข้อมูลเปรียบเสมือนการวางรากฐานของบ้าน ถ้ารากฐานไม่แข็งแรง ไม่ว่าจะตกแต่งภายในสวยงามแค่ไหน บ้านก็อาจพังทลายได้ เช่นเดียวกัน ถ้าออกแบบฐานข้อมูลไม่ดี แอปพลิเคชันที่พัฒนาบนนั้นจะประสบปัญหาในระยะยาว

### ปัญหาที่เกิดจากการออกแบบฐานข้อมูลที่ไม่ดี

```
ตัวอย่างจากชีวิตจริง:
บริษัทแห่งหนึ่งมีระบบ ERP ที่ใช้มา 10 ปี
ปัญหาที่พบ:
- ข้อมูลซ้ำซ้อนในหลายตาราง
- ข้อมูลขัดแย้งกัน (inconsistent data)
- Query ที่ต้องใช้เวลานานหลายชั่วโมง
- เพิ่มฟีเจอร์ใหม่แทบเป็นไปไม่ได้

ต้นทุนในการแก้ไข: มากกว่า 10 ล้านบาท
ต้นทุนในการออกแบบที่ดีตั้งแต่แรก: น้อยกว่า 500,000 บาท
```

### ผลกระทบของการออกแบบฐานข้อมูลที่ดี

1. **ความถูกต้องของข้อมูล (Data Integrity)** - ข้อมูลถูกต้องและสอดคล้องกัน
2. **ประสิทธิภาพ (Performance)** - Query ทำงานได้เร็ว
3. **การขยายตัว (Scalability)** - รองรับการเติบโตได้
4. **การบำรุงรักษา (Maintainability)** - แก้ไขและเพิ่มเติมได้ง่าย
5. **ความปลอดภัย (Security)** - ควบคุมการเข้าถึงได้อย่างมีประสิทธิภาพ

---

## ขั้นตอนการออกแบบฐานข้อมูล (Design Phases)

การออกแบบฐานข้อมูลมี 3 ขั้นตอนหลัก:

```
┌─────────────────────────────────────────────────────────┐
│                  DATABASE DESIGN PHASES                  │
│                                                         │
│  ┌─────────────────┐                                    │
│  │  1. CONCEPTUAL   │  ← ความเข้าใจของธุรกิจ            │
│  │     DESIGN       │    (Business Understanding)        │
│  └────────┬────────┘                                    │
│           │ ER Diagrams, Entities, Relationships         │
│           ▼                                             │
│  ┌─────────────────┐                                    │
│  │  2. LOGICAL      │  ← แปลงเป็นโครงสร้างเชิงตรรกะ    │
│  │     DESIGN       │    (Tables, Keys, Constraints)     │
│  └────────┬────────┘                                    │
│           │ Normalization, Relational Model              │
│           ▼                                             │
│  ┌─────────────────┐                                    │
│  │  3. PHYSICAL     │  ← การนำไปใช้จริง                 │
│  │     DESIGN       │    (DBMS-specific, Indexes)        │
│  └─────────────────┘                                    │
└─────────────────────────────────────────────────────────┘
```

### ขั้นที่ 1: Conceptual Design (การออกแบบเชิงแนวคิด)

**เป้าหมาย**: เข้าใจปัญหาทางธุรกิจ และสร้าง ER Diagram เบื้องต้น

สิ่งที่ต้องทำ:
- ระบุ Entity (สิ่งของหรือแนวคิดสำคัญ)
- ระบุ Attribute (คุณสมบัติของแต่ละ Entity)
- ระบุ Relationship (ความสัมพันธ์ระหว่าง Entity)
- ยังไม่ต้องสนใจ DBMS ที่จะใช้

```
ตัวอย่าง Conceptual Design สำหรับระบบ E-Commerce:

Entity:
- ลูกค้า (Customer)
- สินค้า (Product)
- คำสั่งซื้อ (Order)
- หมวดหมู่สินค้า (Category)

Relationship:
- ลูกค้า "ทำ" คำสั่งซื้อ
- คำสั่งซื้อ "มี" สินค้า
- สินค้า "อยู่ใน" หมวดหมู่
```

### ขั้นที่ 2: Logical Design (การออกแบบเชิงตรรกะ)

**เป้าหมาย**: แปลง ER Diagram เป็น Relational Model

สิ่งที่ต้องทำ:
- แปลง Entity เป็น Table
- กำหนด Primary Key
- แปลง Relationship เป็น Foreign Key
- Normalize ตาราง (1NF, 2NF, 3NF)
- กำหนด Constraint ต่างๆ

```sql
-- ตัวอย่าง Logical Design
-- ยังไม่มีรายละเอียด DBMS-specific

CUSTOMERS (
    customer_id    PRIMARY KEY,
    first_name     NOT NULL,
    last_name      NOT NULL,
    email          UNIQUE NOT NULL,
    phone          
)

PRODUCTS (
    product_id     PRIMARY KEY,
    name           NOT NULL,
    price          NOT NULL,
    category_id    FOREIGN KEY → CATEGORIES
)

ORDERS (
    order_id       PRIMARY KEY,
    customer_id    FOREIGN KEY → CUSTOMERS,
    order_date     NOT NULL,
    status         NOT NULL
)

ORDER_ITEMS (
    order_id       FOREIGN KEY → ORDERS,
    product_id     FOREIGN KEY → PRODUCTS,
    quantity       NOT NULL,
    unit_price     NOT NULL,
    PRIMARY KEY (order_id, product_id)
)
```

### ขั้นที่ 3: Physical Design (การออกแบบเชิงกายภาพ)

**เป้าหมาย**: นำ Logical Design ไปใช้กับ DBMS ที่เลือก

สิ่งที่ต้องทำ:
- เลือก Data Type ที่เหมาะสม
- สร้าง Index
- กำหนด Partition (ถ้าจำเป็น)
- กำหนด Storage Parameters
- Optimize Query Performance

```sql
-- ตัวอย่าง Physical Design สำหรับ PostgreSQL
CREATE TABLE customers (
    customer_id    SERIAL PRIMARY KEY,
    first_name     VARCHAR(50) NOT NULL,
    last_name      VARCHAR(50) NOT NULL,
    email          VARCHAR(255) UNIQUE NOT NULL,
    phone          VARCHAR(20),
    created_at     TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at     TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Index สำหรับการค้นหา
CREATE INDEX idx_customers_email ON customers(email);
CREATE INDEX idx_customers_name ON customers(last_name, first_name);

CREATE TABLE products (
    product_id     SERIAL PRIMARY KEY,
    name           VARCHAR(200) NOT NULL,
    price          DECIMAL(10, 2) NOT NULL CHECK (price >= 0),
    stock_quantity INTEGER NOT NULL DEFAULT 0,
    category_id    INTEGER REFERENCES categories(category_id),
    created_at     TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_products_category ON products(category_id);
CREATE INDEX idx_products_price ON products(price);
```

---

## Top-Down vs Bottom-Up Design

### Top-Down Design (การออกแบบจากบนลงล่าง)

```
ธุรกิจ/ความต้องการระดับสูง
           │
           ▼
    ระบบใหญ่ (System)
           │
           ▼
    โมดูล (Modules)
           │
           ▼
    ตาราง (Tables)
           │
           ▼
    คอลัมน์ (Columns)
```

**ข้อดี**:
- มองภาพรวมได้ก่อน
- สอดคล้องกับความต้องการทางธุรกิจ
- ลดความเสี่ยงการลืมสิ่งสำคัญ

**ข้อเสีย**:
- อาจใช้เวลานาน
- ต้องการความรู้ธุรกิจลึกซึ้ง

**เหมาะสำหรับ**: โปรเจกต์ใหม่ขนาดใหญ่

### Bottom-Up Design (การออกแบบจากล่างขึ้นบน)

```
ข้อมูลที่มีอยู่ (Forms, Reports, Files)
           │
           ▼
    Attributes ที่ระบุได้
           │
           ▼
    จัดกลุ่มเป็น Entities
           │
           ▼
    สร้าง Relationships
           │
           ▼
    ภาพรวมระบบ
```

**ข้อดี**:
- เริ่มจากสิ่งที่มีอยู่แล้ว
- เหมาะกับการ Migrate ระบบเก่า

**ข้อเสีย**:
- อาจพลาด Requirement บางอย่าง
- อาจได้โครงสร้างที่ไม่ clean

**เหมาะสำหรับ**: การ Migrate หรือ Reverse Engineer ระบบเก่า

### แนวทางผสม (Hybrid Approach)

ในทางปฏิบัติ นักออกแบบฐานข้อมูลมักใช้แนวทางผสม:

```
1. Top-Down: วางโครงสร้างใหญ่จากความต้องการธุรกิจ
2. Bottom-Up: ตรวจสอบกับข้อมูลที่มีอยู่จริง
3. ปรับให้สอดคล้องกัน
```

---

## การระบุ Entities และ Attributes

### การระบุ Entities

Entity คือ "สิ่ง" หรือ "แนวคิด" ที่เราต้องการเก็บข้อมูล

**วิธีการระบุ Entity จาก Requirements**:
1. มองหาคำนาม (Nouns) ใน Requirements
2. ถามว่า "เราต้องการเก็บข้อมูลเกี่ยวกับอะไรบ้าง?"
3. ตรวจสอบว่าแต่ละ Entity มีหลาย Instance

```
ตัวอย่าง Requirements ระบบ E-Commerce:

"ลูกค้าสามารถเข้าสู่ระบบและสั่งซื้อสินค้า โดยสินค้าแต่ละชิ้น
มีราคาและหมวดหมู่ คำสั่งซื้อจะถูกส่งไปยังที่อยู่ที่ลูกค้ากำหนด
และชำระเงินผ่านบัตรเครดิต"

Entities ที่ระบุได้:
✓ ลูกค้า (Customer) - มีหลาย instance
✓ สินค้า (Product) - มีหลาย instance
✓ คำสั่งซื้อ (Order) - มีหลาย instance
✓ หมวดหมู่ (Category) - มีหลาย instance
✓ ที่อยู่ (Address) - มีหลาย instance
✓ บัตรเครดิต (Payment Method) - มีหลาย instance

? ราคา (Price) - เป็น Attribute ของ Product ไม่ใช่ Entity
```

### การระบุ Attributes

Attribute คือคุณสมบัติของ Entity

**ประเภทของ Attributes**:

```
1. Simple Attribute (Attribute เดี่ยว)
   Customer.first_name, Customer.age

2. Composite Attribute (Attribute รวม)
   Customer.full_name = first_name + last_name
   Address = street + city + state + zip

3. Multivalued Attribute (หลายค่า)
   Customer.phone (อาจมีหลายเบอร์)
   Product.images (อาจมีหลายรูป)

4. Derived Attribute (คำนวณได้)
   Customer.age (คำนวณจาก birth_date)
   Order.total (คำนวณจาก items)

5. Key Attribute (ใช้ระบุ Instance)
   Customer.customer_id
   Product.product_id
```

```sql
-- ตัวอย่างการแปลง Attributes เป็น Columns

-- Simple Attributes → ตรงไปตรงมา
CREATE TABLE customers (
    customer_id  SERIAL PRIMARY KEY,    -- Key Attribute
    first_name   VARCHAR(50),           -- Simple Attribute
    last_name    VARCHAR(50),           -- Simple Attribute
    birth_date   DATE,                  -- Simple Attribute
    age          INTEGER                -- Derived (ไม่ควรเก็บ!)
);

-- Composite Attribute → แยก Columns
CREATE TABLE addresses (
    address_id   SERIAL PRIMARY KEY,
    street       VARCHAR(200),          -- ส่วนหนึ่งของ Composite
    city         VARCHAR(100),          -- ส่วนหนึ่งของ Composite
    state        VARCHAR(50),           -- ส่วนหนึ่งของ Composite
    postal_code  VARCHAR(20)            -- ส่วนหนึ่งของ Composite
);

-- Multivalued Attribute → แยกตาราง
CREATE TABLE customer_phones (
    phone_id     SERIAL PRIMARY KEY,
    customer_id  INTEGER REFERENCES customers,
    phone_number VARCHAR(20),
    phone_type   VARCHAR(20)            -- mobile, home, work
);

-- Derived Attribute → คำนวณตอน Query
-- ไม่ควรเก็บในตาราง แต่ใช้ VIEW หรือ Computed Column
CREATE VIEW customers_with_age AS
SELECT
    customer_id,
    first_name,
    last_name,
    birth_date,
    EXTRACT(YEAR FROM AGE(birth_date)) AS age  -- คำนวณตอน Query
FROM customers;
```

---

## Business Rules to Database Rules

กฎของธุรกิจ (Business Rules) ต้องแปลงเป็นข้อกำหนดของฐานข้อมูล (Database Constraints)

### ประเภทของ Business Rules

```
1. กฎเกี่ยวกับข้อมูล (Data Rules)
   "อีเมลต้องไม่ซ้ำกัน"
   "ราคาต้องไม่ติดลบ"
   "วันหมดอายุต้องหลังวันผลิต"

2. กฎเกี่ยวกับความสัมพันธ์ (Relationship Rules)
   "ลูกค้าหนึ่งคนสามารถมีหลายคำสั่งซื้อ"
   "สินค้าต้องอยู่ในหมวดหมู่เสมอ"
   "พนักงานต้องมีแผนกที่สังกัด"

3. กฎเกี่ยวกับสถานะ (State Rules)
   "คำสั่งซื้อที่ส่งแล้วไม่สามารถลบได้"
   "สต็อกต้องไม่ติดลบ"
```

### การแปลง Business Rules เป็น SQL

```sql
-- Rule 1: อีเมลต้องไม่ซ้ำกัน
CREATE TABLE customers (
    email VARCHAR(255) UNIQUE NOT NULL  -- UNIQUE constraint
);

-- Rule 2: ราคาต้องไม่ติดลบ
CREATE TABLE products (
    price DECIMAL(10,2) CHECK (price >= 0)  -- CHECK constraint
);

-- Rule 3: วันหมดอายุต้องหลังวันผลิต
CREATE TABLE batches (
    manufacture_date DATE NOT NULL,
    expiry_date      DATE NOT NULL,
    CONSTRAINT chk_dates CHECK (expiry_date > manufacture_date)
);

-- Rule 4: สินค้าต้องอยู่ในหมวดหมู่เสมอ
CREATE TABLE products (
    category_id INTEGER NOT NULL REFERENCES categories(category_id)
    -- NOT NULL + FOREIGN KEY = ต้องมีหมวดหมู่เสมอ
);

-- Rule 5: ยอดในบัญชีต้องไม่ติดลบ (ใช้ Trigger)
CREATE OR REPLACE FUNCTION check_account_balance()
RETURNS TRIGGER AS $$
BEGIN
    IF NEW.balance < 0 THEN
        RAISE EXCEPTION 'Account balance cannot be negative';
    END IF;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_check_balance
    BEFORE UPDATE ON accounts
    FOR EACH ROW
    EXECUTE FUNCTION check_account_balance();
```

---

## Case Study: การออกแบบ E-Commerce จากศูนย์

### ขั้นตอนที่ 1: รวบรวม Requirements

```
System Requirements Document
ระบบ: ร้านค้าออนไลน์ขายเสื้อผ้า

Functional Requirements:
1. ลูกค้าสามารถลงทะเบียนและล็อกอิน
2. ลูกค้าสามารถค้นหาและดูรายละเอียดสินค้า
3. สินค้ามีหลายสี หลายขนาด (variants)
4. ลูกค้าสามารถเพิ่มสินค้าลงตะกร้า
5. ลูกค้าสามารถสั่งซื้อและชำระเงิน
6. ลูกค้ามีหลายที่อยู่จัดส่ง
7. ระบบส่งอีเมลยืนยันการสั่งซื้อ
8. Admin สามารถจัดการสินค้าและหมวดหมู่
9. รายงานยอดขายรายวัน/รายเดือน
10. ระบบรีวิวสินค้า

Non-Functional Requirements:
- รองรับลูกค้า 10,000 คนพร้อมกัน
- ตอบสนองภายใน 2 วินาที
- Data Backup ทุกวัน
```

### ขั้นตอนที่ 2: Conceptual Design

```
ER Diagram (Conceptual Level)

┌──────────────┐     places      ┌──────────────┐
│   CUSTOMER   │────────────────>│    ORDER     │
│              │    1 to many    │              │
│  - name      │                 │  - date      │
│  - email     │                 │  - status    │
│  - phone     │                 │  - total     │
└──────┬───────┘                 └──────┬───────┘
       │                                │
       │ has                            │ contains
       │ (1 to many)                    │ (many to many)
       ▼                                ▼
┌──────────────┐                 ┌──────────────┐
│   ADDRESS    │                 │   PRODUCT    │
│              │                 │   VARIANT    │
│  - street    │                 │              │
│  - city      │                 │  - sku       │
│  - country   │                 │  - price     │
└──────────────┘                 │  - stock     │
                                 └──────┬───────┘
                                        │
                                        │ belongs to
                                        │ (many to 1)
                                        ▼
                                 ┌──────────────┐
                                 │   PRODUCT    │
                                 │              │
                                 │  - name      │
                                 │  - desc      │
                                 │  - brand     │
                                 └──────┬───────┘
                                        │
                                        │ in category
                                        │ (many to many)
                                        ▼
                                 ┌──────────────┐
                                 │   CATEGORY   │
                                 │              │
                                 │  - name      │
                                 │  - parent    │
                                 └──────────────┘
```

### ขั้นตอนที่ 3: Logical Design

```sql
-- Logical Schema (DBMS-independent)

CUSTOMERS (
    customer_id    PK
    email          UNIQUE NOT NULL
    first_name     NOT NULL
    last_name      NOT NULL
    phone          
    created_at     
    is_active      DEFAULT TRUE
)

ADDRESSES (
    address_id     PK
    customer_id    FK → CUSTOMERS, NOT NULL
    label          (home/work/other)
    recipient_name NOT NULL
    street_line1   NOT NULL
    street_line2   
    city           NOT NULL
    province       NOT NULL
    postal_code    NOT NULL
    country        DEFAULT 'TH'
    is_default     DEFAULT FALSE
)

CATEGORIES (
    category_id    PK
    parent_id      FK → CATEGORIES (self-referencing, nullable)
    name           NOT NULL
    slug           UNIQUE NOT NULL
    sort_order     DEFAULT 0
)

PRODUCTS (
    product_id     PK
    name           NOT NULL
    description    
    brand          
    is_active      DEFAULT TRUE
    created_at     
)

PRODUCT_CATEGORIES (
    product_id     FK → PRODUCTS
    category_id    FK → CATEGORIES
    PK (product_id, category_id)
)

PRODUCT_VARIANTS (
    variant_id     PK
    product_id     FK → PRODUCTS, NOT NULL
    sku            UNIQUE NOT NULL
    color          
    size           
    price          NOT NULL, CHECK >= 0
    stock_qty      NOT NULL DEFAULT 0, CHECK >= 0
    image_url      
)

ORDERS (
    order_id       PK
    customer_id    FK → CUSTOMERS, NOT NULL
    address_id     FK → ADDRESSES, NOT NULL
    order_number   UNIQUE NOT NULL
    status         NOT NULL (pending/confirmed/shipped/delivered/cancelled)
    subtotal       NOT NULL
    shipping_fee   NOT NULL DEFAULT 0
    discount_amount DEFAULT 0
    total_amount   NOT NULL
    notes          
    created_at     
    updated_at     
)

ORDER_ITEMS (
    order_item_id  PK
    order_id       FK → ORDERS, NOT NULL
    variant_id     FK → PRODUCT_VARIANTS, NOT NULL
    quantity       NOT NULL, CHECK > 0
    unit_price     NOT NULL (snapshot of price at time of order)
    subtotal       NOT NULL
)

REVIEWS (
    review_id      PK
    customer_id    FK → CUSTOMERS, NOT NULL
    product_id     FK → PRODUCTS, NOT NULL
    order_id       FK → ORDERS (ต้องซื้อก่อนถึงรีวิวได้)
    rating         NOT NULL, CHECK BETWEEN 1 AND 5
    title          
    body           
    created_at     
    is_published   DEFAULT FALSE
    UNIQUE (customer_id, product_id, order_id)
)
```

### ขั้นตอนที่ 4: Physical Design

```sql
-- Physical Schema สำหรับ PostgreSQL

-- ตาราง CUSTOMERS
CREATE TABLE customers (
    customer_id  SERIAL PRIMARY KEY,
    email        VARCHAR(255) UNIQUE NOT NULL,
    first_name   VARCHAR(50) NOT NULL,
    last_name    VARCHAR(50) NOT NULL,
    phone        VARCHAR(20),
    password_hash VARCHAR(255) NOT NULL,
    created_at   TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at   TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    is_active    BOOLEAN NOT NULL DEFAULT TRUE
);

CREATE INDEX idx_customers_email ON customers(email);
CREATE INDEX idx_customers_name ON customers(last_name, first_name);
CREATE INDEX idx_customers_active ON customers(is_active) WHERE is_active = TRUE;

-- ตาราง CATEGORIES (Self-referencing)
CREATE TABLE categories (
    category_id  SERIAL PRIMARY KEY,
    parent_id    INTEGER REFERENCES categories(category_id),
    name         VARCHAR(100) NOT NULL,
    slug         VARCHAR(100) UNIQUE NOT NULL,
    description  TEXT,
    image_url    VARCHAR(500),
    sort_order   SMALLINT NOT NULL DEFAULT 0,
    is_active    BOOLEAN NOT NULL DEFAULT TRUE,
    created_at   TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_categories_parent ON categories(parent_id);
CREATE INDEX idx_categories_slug ON categories(slug);

-- ตาราง PRODUCTS
CREATE TABLE products (
    product_id   SERIAL PRIMARY KEY,
    name         VARCHAR(200) NOT NULL,
    slug         VARCHAR(200) UNIQUE NOT NULL,
    description  TEXT,
    brand        VARCHAR(100),
    is_active    BOOLEAN NOT NULL DEFAULT TRUE,
    created_at   TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at   TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_products_brand ON products(brand);
CREATE INDEX idx_products_active ON products(is_active) WHERE is_active = TRUE;

-- ตาราง PRODUCT_VARIANTS
CREATE TABLE product_variants (
    variant_id   SERIAL PRIMARY KEY,
    product_id   INTEGER NOT NULL REFERENCES products(product_id),
    sku          VARCHAR(50) UNIQUE NOT NULL,
    color        VARCHAR(50),
    size         VARCHAR(20),
    price        DECIMAL(10,2) NOT NULL CHECK (price >= 0),
    compare_price DECIMAL(10,2) CHECK (compare_price >= 0),
    stock_qty    INTEGER NOT NULL DEFAULT 0 CHECK (stock_qty >= 0),
    image_url    VARCHAR(500),
    is_active    BOOLEAN NOT NULL DEFAULT TRUE
);

CREATE INDEX idx_variants_product ON product_variants(product_id);
CREATE INDEX idx_variants_sku ON product_variants(sku);

-- ตาราง ORDERS
CREATE TABLE orders (
    order_id      SERIAL PRIMARY KEY,
    customer_id   INTEGER NOT NULL REFERENCES customers(customer_id),
    address_id    INTEGER NOT NULL REFERENCES addresses(address_id),
    order_number  VARCHAR(20) UNIQUE NOT NULL,
    status        VARCHAR(20) NOT NULL DEFAULT 'pending'
                  CHECK (status IN ('pending','confirmed','processing',
                                    'shipped','delivered','cancelled','refunded')),
    subtotal      DECIMAL(10,2) NOT NULL,
    shipping_fee  DECIMAL(10,2) NOT NULL DEFAULT 0,
    discount_amount DECIMAL(10,2) NOT NULL DEFAULT 0,
    total_amount  DECIMAL(10,2) NOT NULL,
    notes         TEXT,
    created_at    TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at    TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_orders_customer ON orders(customer_id);
CREATE INDEX idx_orders_status ON orders(status);
CREATE INDEX idx_orders_created ON orders(created_at DESC);
CREATE INDEX idx_orders_number ON orders(order_number);

-- ตาราง ORDER_ITEMS
CREATE TABLE order_items (
    order_item_id SERIAL PRIMARY KEY,
    order_id      INTEGER NOT NULL REFERENCES orders(order_id),
    variant_id    INTEGER NOT NULL REFERENCES product_variants(variant_id),
    quantity      INTEGER NOT NULL CHECK (quantity > 0),
    unit_price    DECIMAL(10,2) NOT NULL,  -- snapshot price
    subtotal      DECIMAL(10,2) NOT NULL,
    UNIQUE (order_id, variant_id)
);

CREATE INDEX idx_order_items_order ON order_items(order_id);
CREATE INDEX idx_order_items_variant ON order_items(variant_id);
```

---

## ข้อผิดพลาดที่พบบ่อยในการออกแบบฐานข้อมูล

### ข้อผิดพลาดที่ 1: การใช้ Generic Column Names

```sql
-- ❌ Bad Design
CREATE TABLE data_table (
    id      INTEGER,
    field1  VARCHAR(255),
    field2  VARCHAR(255),
    field3  INTEGER,
    value   TEXT
);

-- ✅ Good Design
CREATE TABLE orders (
    order_id     INTEGER,
    order_number VARCHAR(20),
    customer_id  INTEGER,
    total_amount DECIMAL(10,2),
    notes        TEXT
);
```

### ข้อผิดพลาดที่ 2: การเก็บข้อมูลหลายอย่างในคอลัมน์เดียว

```sql
-- ❌ Bad Design: เก็บหลายค่าในคอลัมน์เดียว
CREATE TABLE products (
    product_id  INTEGER,
    tags        VARCHAR(500)  -- "electronics,sale,new" ← ปัญหา!
);

-- ❌ ปัญหาของการออกแบบแบบนี้:
-- 1. ค้นหาสินค้าที่มี tag = 'sale' ยากมาก
-- 2. ไม่สามารถ JOIN กับตาราง tags ได้
-- 3. ข้อมูลอาจมีรูปแบบไม่สอดคล้อง

-- ✅ Good Design: แยกตาราง
CREATE TABLE tags (
    tag_id   SERIAL PRIMARY KEY,
    name     VARCHAR(50) UNIQUE NOT NULL
);

CREATE TABLE product_tags (
    product_id  INTEGER REFERENCES products,
    tag_id      INTEGER REFERENCES tags,
    PRIMARY KEY (product_id, tag_id)
);
```

### ข้อผิดพลาดที่ 3: ไม่มี Primary Key

```sql
-- ❌ Bad Design
CREATE TABLE log_entries (
    timestamp   TIMESTAMP,
    user_id     INTEGER,
    action      VARCHAR(100),
    details     TEXT
);
-- ไม่มี Primary Key → ไม่สามารถระบุ Row ได้

-- ✅ Good Design
CREATE TABLE log_entries (
    log_id      BIGSERIAL PRIMARY KEY,  -- หรือใช้ UUID
    timestamp   TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    user_id     INTEGER,
    action      VARCHAR(100) NOT NULL,
    details     TEXT
);
```

### ข้อผิดพลาดที่ 4: ข้อมูลที่คำนวณได้แต่เก็บแบบ Hardcode

```sql
-- ❌ Bad Design: เก็บ total ที่คำนวณจาก items
CREATE TABLE orders (
    order_id    INTEGER,
    item1_price DECIMAL,
    item2_price DECIMAL,
    item3_price DECIMAL,
    total       DECIMAL  -- คำนวณจาก item1+item2+item3
);

-- ปัญหา: ถ้าเพิ่ม item4 ต้องแก้ Schema

-- ✅ Good Design
CREATE TABLE order_items (
    order_id   INTEGER,
    product_id INTEGER,
    quantity   INTEGER,
    unit_price DECIMAL
);

-- คำนวณ total ด้วย Query
SELECT order_id, SUM(quantity * unit_price) as total
FROM order_items
GROUP BY order_id;
```

### ข้อผิดพลาดที่ 5: ไม่ใช้ Foreign Keys

```sql
-- ❌ Bad Design: ไม่มี Foreign Key Constraint
CREATE TABLE orders (
    order_id    INTEGER,
    customer_id INTEGER  -- ไม่มี FK → อาจมีค่าที่ไม่มีใน customers
);

-- ✅ Good Design
CREATE TABLE orders (
    order_id    SERIAL PRIMARY KEY,
    customer_id INTEGER NOT NULL REFERENCES customers(customer_id)
    -- FK ป้องกัน orphaned records
);
```

### ข้อผิดพลาดที่ 6: ใช้ NULL อย่างไม่เหมาะสม

```sql
-- ❌ Bad Design: ใช้ NULL แทนค่า Default
CREATE TABLE products (
    price DECIMAL  -- NULL หมายความว่าอะไร? ไม่มีราคา? ฟรี?
);

-- ✅ Good Design: ชัดเจนเรื่อง NULL
CREATE TABLE products (
    regular_price  DECIMAL(10,2) NOT NULL,          -- ต้องมีราคา
    sale_price     DECIMAL(10,2),                   -- NULL = ไม่มีโปรโมชัน
    cost_price     DECIMAL(10,2)                    -- NULL = ยังไม่รู้ต้นทุน
);
```

### ข้อผิดพลาดที่ 7: Over-Engineering (ออกแบบซับซ้อนเกินความจำเป็น)

```sql
-- ❌ Over-engineered: EAV สำหรับข้อมูลง่ายๆ
CREATE TABLE product_attributes (
    product_id  INTEGER,
    attr_name   VARCHAR(50),
    attr_value  TEXT
);
-- ยากต่อการ Query, Validate, และ Join

-- ✅ Simple และ Effective
CREATE TABLE products (
    product_id  INTEGER,
    color       VARCHAR(50),
    size        VARCHAR(20),
    weight_kg   DECIMAL(5,2)
);
```

---

## Design Review Checklist

```
การตรวจสอบ Schema ก่อน Production

□ NAMING CONVENTIONS
  □ ตั้งชื่อตารางสม่ำเสมอ (plural/singular)
  □ ตั้งชื่อคอลัมน์ชัดเจนและสม่ำเสมอ
  □ FK columns ตั้งชื่อตาม PK ของตารางที่อ้างถึง

□ PRIMARY KEYS
  □ ทุกตารางมี Primary Key
  □ เลือกชนิดข้อมูลที่เหมาะสม (INT vs UUID)
  □ พิจารณา Surrogate vs Natural Key

□ FOREIGN KEYS
  □ FK ทั้งหมดมี Constraint
  □ กำหนด ON DELETE action ที่เหมาะสม
  □ มี Index บน FK columns

□ CONSTRAINTS
  □ NOT NULL สำหรับคอลัมน์ที่จำเป็น
  □ UNIQUE สำหรับข้อมูลที่ต้องไม่ซ้ำ
  □ CHECK constraints สำหรับ Business Rules
  □ DEFAULT values ที่เหมาะสม

□ DATA TYPES
  □ เลือก Data Type ที่เหมาะสมและประหยัดพื้นที่
  □ VARCHAR มีขนาดที่เหมาะสม
  □ ใช้ DECIMAL สำหรับเงิน (ไม่ใช้ FLOAT)

□ NORMALIZATION
  □ ตรวจสอบ 1NF - ไม่มี Repeating Groups
  □ ตรวจสอบ 2NF - ไม่มี Partial Dependencies
  □ ตรวจสอบ 3NF - ไม่มี Transitive Dependencies

□ INDEXES
  □ มี Index บน FK columns
  □ มี Index บน columns ที่ใช้ใน WHERE บ่อยๆ
  □ ไม่มี Index ที่ไม่จำเป็น (กิน Storage)

□ TIMESTAMPS
  □ มี created_at ทุกตาราง
  □ มี updated_at สำหรับตารางที่มีการแก้ไข
  □ มี deleted_at สำหรับ Soft Delete

□ DOCUMENTATION
  □ มี Comment บน Tables และ Columns ที่ซับซ้อน
  □ มี ER Diagram อัปเดตล่าสุด
  □ มีเอกสาร Business Rules
```

---

## แนวปฏิบัติที่ดี (Best Practices)

### 1. Naming Conventions

```sql
-- ใช้รูปแบบที่สม่ำเสมอ

-- Tables: snake_case, plural
CREATE TABLE customers (...);
CREATE TABLE order_items (...);
CREATE TABLE product_variants (...);

-- Columns: snake_case
customer_id, first_name, created_at, is_active

-- Primary Keys: table_name + _id
customers.customer_id
orders.order_id
products.product_id

-- Foreign Keys: ชื่อเหมือน Primary Key ที่อ้างถึง
orders.customer_id → customers.customer_id

-- Indexes: idx_ + table + column
CREATE INDEX idx_customers_email ON customers(email);
CREATE INDEX idx_orders_customer ON orders(customer_id);

-- Constraints: chk_ / fk_ / uq_
CONSTRAINT chk_price_positive CHECK (price >= 0)
CONSTRAINT fk_orders_customer FOREIGN KEY (customer_id)
CONSTRAINT uq_customers_email UNIQUE (email)
```

### 2. ใช้ Surrogate Keys

```sql
-- ✅ ใช้ Surrogate Key (Auto-generated)
CREATE TABLE customers (
    customer_id SERIAL PRIMARY KEY,  -- หรือ UUID
    email       VARCHAR(255) UNIQUE NOT NULL
    -- email เป็น Natural Key แต่ไม่ใช้เป็น PK
    -- เพราะ email อาจเปลี่ยนได้
);

-- ❌ ใช้ Natural Key เป็น PK (มีความเสี่ยง)
CREATE TABLE customers (
    email    VARCHAR(255) PRIMARY KEY  -- ถ้าเปลี่ยน email ต้อง cascade ทุกที่
);
```

### 3. Audit Columns

```sql
-- เพิ่ม Audit Columns ทุกตาราง
CREATE TABLE products (
    product_id   SERIAL PRIMARY KEY,
    -- ... columns อื่นๆ ...
    created_at   TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at   TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    created_by   INTEGER REFERENCES users(user_id),
    updated_by   INTEGER REFERENCES users(user_id)
);

-- Auto-update updated_at
CREATE OR REPLACE FUNCTION update_updated_at()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = CURRENT_TIMESTAMP;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_products_updated_at
    BEFORE UPDATE ON products
    FOR EACH ROW
    EXECUTE FUNCTION update_updated_at();
```

---

## สรุป

การออกแบบฐานข้อมูลที่ดีต้องผ่านกระบวนการ:

```
1. เข้าใจ Business Requirements อย่างถ่องแท้
2. ออกแบบ Conceptual Model (ER Diagram)
3. แปลงเป็น Logical Model (Normalization)
4. นำไป Implement เป็น Physical Design
5. Review และปรับปรุงตาม Checklist
```

หลักการสำคัญที่ต้องจำ:
- **Data Integrity First** - ความถูกต้องสำคัญกว่า Performance เสมอ
- **Keep It Simple** - อย่าออกแบบซับซ้อนเกินความจำเป็น
- **Plan for Change** - ออกแบบให้รองรับการเปลี่ยนแปลงในอนาคต
- **Document Everything** - เอกสารที่ดีช่วยทีมในระยะยาว

---

## แบบฝึกหัด (10 ข้อ)

### ข้อ 1
วาด ER Diagram (แบบ ASCII) สำหรับระบบห้องสมุด ที่มี: หนังสือ, สมาชิก, การยืม-คืน

**เฉลย**:
```
┌──────────┐      borrows     ┌──────────────┐      is        ┌──────────┐
│  MEMBER  │────────────────>│   BORROWING  │──────────────>│   BOOK   │
│          │   (1 to many)   │              │   (many to 1) │          │
│ id       │                 │ borrow_id    │               │ book_id  │
│ name     │                 │ member_id FK │               │ title    │
│ email    │                 │ book_id FK   │               │ isbn     │
│ phone    │                 │ borrow_date  │               │ author   │
└──────────┘                 │ due_date     │               │ qty      │
                             │ return_date  │               └──────────┘
                             └──────────────┘
```

### ข้อ 2
ระบุ Entities, Attributes, และ Relationships จาก Requirements นี้:
"โรงแรมมีห้องพักหลายประเภท แขกสามารถจองห้องพักสำหรับช่วงวันที่ต้องการ และชำระเงินผ่านบัตรเครดิต"

**เฉลย**:
```
Entities: Hotel, Room, RoomType, Guest, Booking, Payment
Attributes:
  Room: room_id, room_number, floor, status
  RoomType: type_id, name, price_per_night, capacity
  Guest: guest_id, name, email, passport_number
  Booking: booking_id, check_in, check_out, total_price
  Payment: payment_id, amount, payment_method, transaction_id

Relationships:
  Room -[is type of]→ RoomType (many to 1)
  Guest -[makes]→ Booking (1 to many)
  Booking -[includes]→ Room (many to many? or 1 room per booking)
  Booking -[has]→ Payment (1 to 1 or 1 to many)
```

### ข้อ 3
แปลง Business Rule เหล่านี้เป็น SQL Constraints:
a) "นักเรียนต้องมีอายุระหว่าง 6-18 ปี"
b) "เกรดต้องเป็น A, B, C, D, หรือ F เท่านั้น"
c) "วันเริ่มงานต้องไม่เป็นวันในอนาคต"

**เฉลย**:
```sql
-- a)
age SMALLINT CHECK (age BETWEEN 6 AND 18)
-- หรือดีกว่า:
birth_date DATE CHECK (
    EXTRACT(YEAR FROM AGE(birth_date)) BETWEEN 6 AND 18
)

-- b)
grade CHAR(1) CHECK (grade IN ('A', 'B', 'C', 'D', 'F'))

-- c)
hire_date DATE NOT NULL CHECK (hire_date <= CURRENT_DATE)
```

### ข้อ 4
ออกแบบ Schema สำหรับระบบจัดการพนักงาน ที่ต้องเก็บ:
- ข้อมูลพนักงาน
- แผนก
- ประวัติตำแหน่ง (พนักงานเปลี่ยนตำแหน่งได้)
- เงินเดือน (ประวัติการปรับเงินเดือน)

**เฉลย**:
```sql
CREATE TABLE departments (
    dept_id     SERIAL PRIMARY KEY,
    name        VARCHAR(100) NOT NULL,
    manager_id  INTEGER  -- FK to employees (circular, set after)
);

CREATE TABLE employees (
    emp_id      SERIAL PRIMARY KEY,
    dept_id     INTEGER REFERENCES departments,
    first_name  VARCHAR(50) NOT NULL,
    last_name   VARCHAR(50) NOT NULL,
    email       VARCHAR(255) UNIQUE NOT NULL,
    hire_date   DATE NOT NULL,
    is_active   BOOLEAN DEFAULT TRUE
);

CREATE TABLE job_history (
    history_id  SERIAL PRIMARY KEY,
    emp_id      INTEGER NOT NULL REFERENCES employees,
    job_title   VARCHAR(100) NOT NULL,
    start_date  DATE NOT NULL,
    end_date    DATE,  -- NULL = ตำแหน่งปัจจุบัน
    dept_id     INTEGER REFERENCES departments
);

CREATE TABLE salary_history (
    salary_id   SERIAL PRIMARY KEY,
    emp_id      INTEGER NOT NULL REFERENCES employees,
    amount      DECIMAL(10,2) NOT NULL,
    effective_date DATE NOT NULL,
    reason      VARCHAR(100)  -- 'annual_review', 'promotion', etc.
);
```

### ข้อ 5
อธิบายความแตกต่างระหว่าง Natural Key และ Surrogate Key พร้อมตัวอย่าง

**เฉลย**:
```
Natural Key: ข้อมูลที่มีอยู่แล้วในโลกจริง ใช้ระบุตัวตนได้
ตัวอย่าง: ISBN ของหนังสือ, เลขที่บัตรประชาชน, Email

Surrogate Key: ข้อมูลที่ระบบสร้างขึ้นมาเองเพื่อระบุตัวตน
ตัวอย่าง: AUTO_INCREMENT id, UUID

ข้อดี Surrogate Key:
- ไม่เปลี่ยนแปลง (stable)
- ขนาดเล็กกว่า (เป็น integer)
- ไม่มีข้อมูล business ที่อาจเปลี่ยน

ข้อดี Natural Key:
- มีความหมายในตัวเอง
- ไม่ต้อง JOIN เพิ่มเพื่อดูความหมาย

ในทางปฏิบัติ: ใช้ Surrogate Key เป็น PK + Natural Key เป็น UNIQUE constraint
```

### ข้อ 6
จงชี้ข้อผิดพลาดใน Schema นี้และแนะนำวิธีแก้:
```sql
CREATE TABLE orders (
    id INT,
    cust VARCHAR(200),
    items TEXT,  -- "product1:2,product2:1,product3:5"
    total FLOAT,
    dt DATETIME
);
```

**เฉลย**:
```
ข้อผิดพลาด:
1. ไม่มี PRIMARY KEY
2. ชื่อคอลัมน์ไม่ชัดเจน (cust, dt)
3. เก็บหลายค่าใน items column
4. ใช้ FLOAT สำหรับเงิน (ควรใช้ DECIMAL)
5. ไม่มี Constraints

แก้ไข:
CREATE TABLE orders (
    order_id    SERIAL PRIMARY KEY,
    customer_id INTEGER NOT NULL REFERENCES customers,
    total       DECIMAL(10,2) NOT NULL,
    created_at  TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE order_items (
    item_id    SERIAL PRIMARY KEY,
    order_id   INTEGER NOT NULL REFERENCES orders,
    product_id INTEGER NOT NULL REFERENCES products,
    quantity   INTEGER NOT NULL CHECK (quantity > 0),
    unit_price DECIMAL(10,2) NOT NULL
);
```

### ข้อ 7
ออกแบบ Physical Schema สำหรับตาราง PRODUCTS ที่:
- ค้นหาสินค้าตามชื่อบ่อย
- Filter ตาม category บ่อย
- Sort ตาม price บ่อย
- มีสินค้าประมาณ 1 ล้านรายการ

**เฉลย**:
```sql
CREATE TABLE products (
    product_id   SERIAL PRIMARY KEY,
    category_id  INTEGER NOT NULL REFERENCES categories,
    name         VARCHAR(200) NOT NULL,
    name_tsv     TSVECTOR,  -- สำหรับ Full-Text Search
    price        DECIMAL(10,2) NOT NULL,
    is_active    BOOLEAN NOT NULL DEFAULT TRUE,
    created_at   TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);

-- Index สำหรับการค้นหาชื่อ
CREATE INDEX idx_products_name ON products(name);
-- หรือ Full-Text Search
CREATE INDEX idx_products_name_fts ON products USING gin(name_tsv);

-- Index สำหรับ Filter ตาม category
CREATE INDEX idx_products_category ON products(category_id);

-- Index สำหรับ Sort ตาม price (พร้อม active filter)
CREATE INDEX idx_products_price ON products(price) WHERE is_active = TRUE;

-- Composite Index สำหรับ Query ที่ใช้ทั้ง category และ price
CREATE INDEX idx_products_cat_price ON products(category_id, price) WHERE is_active = TRUE;
```

### ข้อ 8
อธิบาย 3 ขั้นตอนของ Database Design พร้อมตัวอย่างสำหรับระบบธนาคาร

**เฉลย**:
```
1. Conceptual Design:
   Entities: Account, Customer, Transaction, Branch
   Relationships:
   - Customer owns Account (1 to many)
   - Account has Transaction (1 to many)
   - Account belongs to Branch (many to 1)

2. Logical Design:
   CUSTOMERS(customer_id PK, name, id_number UNIQUE, phone)
   ACCOUNTS(account_id PK, customer_id FK, type, balance, status)
   TRANSACTIONS(txn_id PK, account_id FK, type, amount, timestamp)
   BRANCHES(branch_id PK, name, address, manager_id)

3. Physical Design (PostgreSQL):
   CREATE TABLE customers (
       customer_id  SERIAL PRIMARY KEY,
       id_number    CHAR(13) UNIQUE NOT NULL,
       first_name   VARCHAR(50) NOT NULL,
       last_name    VARCHAR(50) NOT NULL,
       phone        VARCHAR(20)
   );
   
   CREATE TABLE accounts (
       account_id   BIGSERIAL PRIMARY KEY,
       customer_id  INTEGER NOT NULL REFERENCES customers,
       account_type VARCHAR(20) CHECK (account_type IN ('savings','checking')),
       balance      DECIMAL(15,2) NOT NULL DEFAULT 0 CHECK (balance >= 0),
       is_active    BOOLEAN DEFAULT TRUE
   );
   
   -- Partition by date for performance
   CREATE TABLE transactions (
       txn_id      BIGSERIAL,
       account_id  INTEGER NOT NULL REFERENCES accounts,
       txn_type    VARCHAR(20) NOT NULL,
       amount      DECIMAL(15,2) NOT NULL,
       created_at  TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
   ) PARTITION BY RANGE (created_at);
```

### ข้อ 9
ออกแบบ Schema สำหรับระบบ Blog ที่มี: บทความ, ผู้เขียน, หมวดหมู่, แท็ก, ความคิดเห็น

**เฉลย**:
```sql
CREATE TABLE authors (
    author_id   SERIAL PRIMARY KEY,
    username    VARCHAR(50) UNIQUE NOT NULL,
    email       VARCHAR(255) UNIQUE NOT NULL,
    display_name VARCHAR(100) NOT NULL,
    bio         TEXT,
    avatar_url  VARCHAR(500)
);

CREATE TABLE categories (
    category_id SERIAL PRIMARY KEY,
    name        VARCHAR(100) UNIQUE NOT NULL,
    slug        VARCHAR(100) UNIQUE NOT NULL
);

CREATE TABLE tags (
    tag_id  SERIAL PRIMARY KEY,
    name    VARCHAR(50) UNIQUE NOT NULL,
    slug    VARCHAR(50) UNIQUE NOT NULL
);

CREATE TABLE posts (
    post_id     SERIAL PRIMARY KEY,
    author_id   INTEGER NOT NULL REFERENCES authors,
    category_id INTEGER REFERENCES categories,
    title       VARCHAR(300) NOT NULL,
    slug        VARCHAR(300) UNIQUE NOT NULL,
    content     TEXT NOT NULL,
    status      VARCHAR(20) DEFAULT 'draft' CHECK (status IN ('draft','published','archived')),
    published_at TIMESTAMP,
    created_at  TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at  TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE post_tags (
    post_id INTEGER REFERENCES posts,
    tag_id  INTEGER REFERENCES tags,
    PRIMARY KEY (post_id, tag_id)
);

CREATE TABLE comments (
    comment_id  SERIAL PRIMARY KEY,
    post_id     INTEGER NOT NULL REFERENCES posts,
    parent_id   INTEGER REFERENCES comments,  -- สำหรับ nested comments
    author_id   INTEGER REFERENCES authors,
    content     TEXT NOT NULL,
    is_approved BOOLEAN DEFAULT FALSE,
    created_at  TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

### ข้อ 10
บอกความแตกต่างและเมื่อใดควรใช้ Top-Down vs Bottom-Up Design

**เฉลย**:
```
Top-Down Design:
- เริ่มจาก Business Requirements ระดับสูง
- ลงมาสู่รายละเอียด
- เหมาะสำหรับ: โปรเจกต์ใหม่ที่ยังไม่มีข้อมูลเดิม
- ข้อดี: Schema สะอาด, สอดคล้องกับ Business
- ข้อเสีย: ต้องใช้เวลา, ต้องรู้ Business ดี

Bottom-Up Design:
- เริ่มจากข้อมูลที่มีอยู่แล้ว
- สร้างขึ้นจาก Reports, Forms, Excel files
- เหมาะสำหรับ: Migration จากระบบเก่า, Legacy database
- ข้อดี: เริ่มได้เร็ว, อิงจากของจริง
- ข้อเสีย: อาจได้ Schema ที่ไม่ clean

แนะนำ: ใช้ Top-Down เป็นหลัก + Bottom-Up เพื่อ Validate
"ออกแบบจากบนลงล่าง แต่ตรวจสอบกับข้อมูลจริงจากล่างขึ้นบน"
```

---

*จบ Part 051: Database Design Principles and Methodology*

**ในส่วนต่อไป (Part 052)**: เราจะเรียนรู้เรื่อง Entity-Relationship Modeling อย่างละเอียด รวมถึง Notations ต่างๆ และการวาด ER Diagram ที่ถูกต้อง
