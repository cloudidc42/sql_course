# Part 058: Database Schema Design Patterns
# รูปแบบการออกแบบ Database Schema

---

## บทนำ: Schema Design Patterns คืออะไร?

Schema Design Patterns คือ "สูตรสำเร็จ" ที่ผ่านการทดสอบมาแล้วสำหรับปัญหาที่พบบ่อยในการออกแบบฐานข้อมูล เช่นเดียวกับ Design Patterns ในการเขียนโปรแกรม

Patterns ที่จะเรียนในบทนี้:
1. Lookup/Reference Tables
2. Audit/History Tables
3. Soft Delete
4. Multi-tenant Schema
5. EAV (Entity-Attribute-Value)
6. Polymorphic Associations
7. Category/Subcategory Trees
8. Tag/Label Systems

---

## Pattern 1: Lookup/Reference Tables

### ปัญหา
```sql
-- ❌ ใช้ String ตรงๆ ในตาราง
CREATE TABLE orders (
    order_id INTEGER PRIMARY KEY,
    status   VARCHAR(20)  -- 'pending', 'processing', 'shipped', ...
    -- ปัญหา: Typo, ไม่มีมาตรฐาน, ไม่รู้ว่ามีค่าอะไรบ้าง
);
```

### Solution: Reference Table

```sql
-- ✅ แยก Reference Table
CREATE TABLE order_statuses (
    code        VARCHAR(20) PRIMARY KEY,
    label       VARCHAR(50) NOT NULL,
    description TEXT,
    sort_order  SMALLINT DEFAULT 0,
    is_terminal BOOLEAN DEFAULT FALSE,  -- final states: 'delivered', 'cancelled'
    is_active   BOOLEAN DEFAULT TRUE
);

INSERT INTO order_statuses (code, label, sort_order, is_terminal) VALUES
('pending',    'รอการยืนยัน',     1, FALSE),
('confirmed',  'ยืนยันแล้ว',       2, FALSE),
('processing', 'กำลังเตรียมสินค้า', 3, FALSE),
('shipped',    'จัดส่งแล้ว',       4, FALSE),
('delivered',  'ได้รับสินค้าแล้ว', 5, TRUE),
('cancelled',  'ยกเลิก',           6, TRUE),
('refunded',   'คืนเงินแล้ว',      7, TRUE);

CREATE TABLE orders (
    order_id    INTEGER PRIMARY KEY,
    customer_id INTEGER REFERENCES customers,
    status      VARCHAR(20) NOT NULL DEFAULT 'pending'
                REFERENCES order_statuses(code),
    created_at  TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- ข้อดี:
-- ✓ Referential Integrity: ไม่มี Typo
-- ✓ Metadata เพิ่มได้ (label, description, sort_order)
-- ✓ ค้นหา status ที่มีได้ง่าย
-- ✓ UI สามารถ load options จาก database ได้

-- Query
SELECT o.*, os.label AS status_label
FROM orders o
JOIN order_statuses os ON o.status = os.code
WHERE os.is_terminal = FALSE;
```

### Pattern 1a: Hierarchical Reference Tables

```sql
-- ตัวอย่าง: ประเทศ > จังหวัด > อำเภอ > ตำบล
CREATE TABLE geo_locations (
    location_id INTEGER PRIMARY KEY,
    parent_id   INTEGER REFERENCES geo_locations,
    level       SMALLINT NOT NULL,  -- 1=country, 2=province, 3=district, 4=subdistrict
    code        VARCHAR(20) UNIQUE NOT NULL,
    name_th     VARCHAR(100) NOT NULL,
    name_en     VARCHAR(100),
    postal_code VARCHAR(10),
    is_active   BOOLEAN DEFAULT TRUE
);

INSERT INTO geo_locations (location_id, level, code, name_th) VALUES
(1, 1, 'TH', 'ประเทศไทย'),
(10, 2, 'TH-10', 'กรุงเทพมหานคร'),
(1001, 3, 'TH-10-01', 'เขตพระนคร'),
(100101, 4, 'TH-10-01-01', 'แขวงพระบรมมหาราชวัง');

CREATE TABLE customer_addresses (
    address_id   SERIAL PRIMARY KEY,
    customer_id  INTEGER REFERENCES customers,
    street       VARCHAR(200),
    subdistrict_id INTEGER REFERENCES geo_locations,  -- Level 4
    is_default   BOOLEAN DEFAULT FALSE
);

-- ดู full address
SELECT
    ca.street,
    sub.name_th AS subdistrict,
    dis.name_th AS district,
    pro.name_th AS province,
    sub.postal_code
FROM customer_addresses ca
JOIN geo_locations sub ON ca.subdistrict_id = sub.location_id
JOIN geo_locations dis ON sub.parent_id = dis.location_id
JOIN geo_locations pro ON dis.parent_id = pro.location_id
WHERE ca.customer_id = 1001;
```

---

## Pattern 2: Audit/History Tables

### ปัญหา
```
ต้องการทราบ:
- ใครเปลี่ยนอะไร เมื่อไหร่
- ค่าเดิมก่อนเปลี่ยนคืออะไร
- สามารถ Restore ข้อมูลได้
```

### Solution A: Audit Columns (Basic)

```sql
CREATE TABLE products (
    product_id    SERIAL PRIMARY KEY,
    name          VARCHAR(200) NOT NULL,
    price         DECIMAL(10,2) NOT NULL,
    -- Audit Columns
    created_at    TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    created_by    INTEGER REFERENCES users,
    updated_at    TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_by    INTEGER REFERENCES users
);

-- Auto-update updated_at
CREATE OR REPLACE FUNCTION set_updated_at()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = CURRENT_TIMESTAMP;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_products_updated_at
    BEFORE UPDATE ON products
    FOR EACH ROW
    EXECUTE FUNCTION set_updated_at();
```

### Solution B: Audit Log Table (Detailed)

```sql
-- Generic Audit Log ที่ใช้ได้กับทุกตาราง
CREATE TABLE audit_logs (
    log_id       BIGSERIAL PRIMARY KEY,
    table_name   VARCHAR(50) NOT NULL,
    record_id    BIGINT NOT NULL,
    action       VARCHAR(10) NOT NULL CHECK (action IN ('INSERT','UPDATE','DELETE')),
    old_values   JSONB,    -- ค่าเดิม (NULL สำหรับ INSERT)
    new_values   JSONB,    -- ค่าใหม่ (NULL สำหรับ DELETE)
    changed_by   INTEGER REFERENCES users,
    changed_at   TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    ip_address   INET,
    user_agent   TEXT
);

CREATE INDEX idx_audit_table_record ON audit_logs(table_name, record_id);
CREATE INDEX idx_audit_changed_at ON audit_logs(changed_at DESC);
CREATE INDEX idx_audit_changed_by ON audit_logs(changed_by);

-- Generic Audit Trigger Function
CREATE OR REPLACE FUNCTION audit_trigger_function()
RETURNS TRIGGER AS $$
DECLARE
    v_old JSONB;
    v_new JSONB;
    v_action VARCHAR(10);
    v_record_id BIGINT;
BEGIN
    v_action := TG_OP;

    IF TG_OP = 'INSERT' THEN
        v_new := TO_JSONB(NEW);
        v_old := NULL;
        v_record_id := (v_new->>'id')::BIGINT;
    ELSIF TG_OP = 'UPDATE' THEN
        v_old := TO_JSONB(OLD);
        v_new := TO_JSONB(NEW);
        v_record_id := (v_new->>'id')::BIGINT;
        -- เก็บเฉพาะ columns ที่เปลี่ยน
        SELECT
            jsonb_object_agg(key, value)
        INTO v_old
        FROM jsonb_each(TO_JSONB(OLD)) e(key, value)
        WHERE value IS DISTINCT FROM (TO_JSONB(NEW))->key;
    ELSIF TG_OP = 'DELETE' THEN
        v_old := TO_JSONB(OLD);
        v_new := NULL;
        v_record_id := (v_old->>'id')::BIGINT;
    END IF;

    INSERT INTO audit_logs (table_name, record_id, action, old_values, new_values)
    VALUES (TG_TABLE_NAME, v_record_id, v_action, v_old, v_new);

    RETURN COALESCE(NEW, OLD);
END;
$$ LANGUAGE plpgsql;

-- Apply Audit Trigger บน Products
CREATE TRIGGER trg_audit_products
    AFTER INSERT OR UPDATE OR DELETE ON products
    FOR EACH ROW
    EXECUTE FUNCTION audit_trigger_function();
```

### Solution C: History Table Pattern

```sql
-- เก็บ History ของทุก version
CREATE TABLE products (
    product_id  SERIAL PRIMARY KEY,
    name        VARCHAR(200) NOT NULL,
    price       DECIMAL(10,2) NOT NULL,
    is_active   BOOLEAN DEFAULT TRUE,
    version     INTEGER NOT NULL DEFAULT 1,
    updated_at  TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE products_history (
    history_id   BIGSERIAL PRIMARY KEY,
    product_id   INTEGER NOT NULL,
    version      INTEGER NOT NULL,
    -- All product columns
    name         VARCHAR(200),
    price        DECIMAL(10,2),
    is_active    BOOLEAN,
    -- History metadata
    changed_by   INTEGER,
    changed_at   TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    change_type  VARCHAR(10),  -- 'INSERT', 'UPDATE', 'DELETE'
    UNIQUE (product_id, version)
);

-- Trigger เพื่อ Archive ก่อน Update
CREATE OR REPLACE FUNCTION archive_product_history()
RETURNS TRIGGER AS $$
BEGIN
    IF TG_OP = 'UPDATE' OR TG_OP = 'DELETE' THEN
        INSERT INTO products_history
            (product_id, version, name, price, is_active, change_type)
        VALUES
            (OLD.product_id, OLD.version, OLD.name, OLD.price, OLD.is_active, TG_OP);
    END IF;

    IF TG_OP = 'UPDATE' THEN
        NEW.version := OLD.version + 1;
    END IF;

    RETURN COALESCE(NEW, OLD);
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_product_history
    BEFORE UPDATE OR DELETE ON products
    FOR EACH ROW
    EXECUTE FUNCTION archive_product_history();

-- Restore ข้อมูลเดิม
UPDATE products
SET name = h.name, price = h.price, version = h.version
FROM products_history h
WHERE products.product_id = h.product_id
AND h.version = 5;  -- Restore เป็น version 5
```

---

## Pattern 3: Soft Delete

### ปัญหา
```
Hard Delete: DELETE ข้อมูลออกไปเลย
- ข้อมูลหาย ไม่สามารถ recover ได้
- References อาจขาด
- ไม่มี Audit Trail

Solution: Soft Delete
- เพิ่ม deleted_at column
- ไม่ DELETE จริง แค่ Mark ว่าลบแล้ว
```

### Implementation

```sql
-- Basic Soft Delete
CREATE TABLE customers (
    customer_id  SERIAL PRIMARY KEY,
    name         VARCHAR(100) NOT NULL,
    email        VARCHAR(255) UNIQUE NOT NULL,
    created_at   TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    deleted_at   TIMESTAMP,         -- NULL = active, NOT NULL = deleted
    deleted_by   INTEGER REFERENCES users
);

-- ต้อง Filter ทุกครั้ง
SELECT * FROM customers WHERE deleted_at IS NULL;

-- ✅ ใช้ Row-Level Security หรือ View เพื่อ Default Filter
CREATE VIEW active_customers AS
SELECT * FROM customers WHERE deleted_at IS NULL;

-- Soft Delete Function
CREATE OR REPLACE FUNCTION soft_delete_customer(p_customer_id INTEGER, p_deleted_by INTEGER)
RETURNS VOID AS $$
BEGIN
    UPDATE customers
    SET
        deleted_at = CURRENT_TIMESTAMP,
        deleted_by = p_deleted_by
    WHERE customer_id = p_customer_id
    AND deleted_at IS NULL;

    IF NOT FOUND THEN
        RAISE EXCEPTION 'Customer % not found or already deleted', p_customer_id;
    END IF;
END;
$$ LANGUAGE plpgsql;

-- Restore
CREATE OR REPLACE FUNCTION restore_customer(p_customer_id INTEGER)
RETURNS VOID AS $$
BEGIN
    UPDATE customers
    SET deleted_at = NULL, deleted_by = NULL
    WHERE customer_id = p_customer_id;
END;
$$ LANGUAGE plpgsql;
```

### Soft Delete กับ UNIQUE Constraints

```sql
-- ปัญหา: ลบ email แล้วสมัครใหม่ด้วย email เดิมไม่ได้ (เพราะ UNIQUE)
CREATE TABLE users (
    user_id    SERIAL PRIMARY KEY,
    email      VARCHAR(255),
    deleted_at TIMESTAMP,
    -- ❌ นี้จะ Error ถ้า email ซ้ำกับ record ที่ soft deleted
    UNIQUE (email)
);

-- ✅ Partial Unique Index (PostgreSQL)
CREATE UNIQUE INDEX idx_users_email_active
ON users(email)
WHERE deleted_at IS NULL;
-- UNIQUE เฉพาะ active records
-- ลบแล้ว email นั้น Available ใหม่

-- หรือใช้ deleted_at ใน Unique
CREATE UNIQUE INDEX idx_users_email_undeleted
ON users(email, COALESCE(deleted_at, '1970-01-01'));
-- ถ้า deleted_at NULL → ใช้ 1970 เป็น stand-in
```

### Cascading Soft Delete

```sql
-- Soft Delete Cascade: ลบ Order → ลบ Order Items ด้วย
CREATE OR REPLACE FUNCTION soft_delete_order(p_order_id INTEGER)
RETURNS VOID AS $$
BEGIN
    UPDATE orders
    SET deleted_at = CURRENT_TIMESTAMP
    WHERE order_id = p_order_id;

    UPDATE order_items
    SET deleted_at = CURRENT_TIMESTAMP
    WHERE order_id = p_order_id;
END;
$$ LANGUAGE plpgsql;
```

---

## Pattern 4: Multi-tenant Schema Patterns

### แนวทางที่ 1: Separate Databases (Database per Tenant)

```
ข้อดี: Strong isolation, ง่ายต่อ Backup/Restore per tenant
ข้อเสีย: ใช้ Resources มาก, ยากต่อ Cross-tenant Analytics
เหมาะกับ: ลูกค้าองค์กรใหญ่, ข้อมูล Sensitive
```

### แนวทางที่ 2: Separate Schemas (Schema per Tenant)

```sql
-- สร้าง Schema สำหรับแต่ละ Tenant
CREATE SCHEMA tenant_abc;
CREATE SCHEMA tenant_xyz;

-- Shared Tables (อยู่ใน public schema)
CREATE TABLE public.tenants (
    tenant_id    VARCHAR(50) PRIMARY KEY,
    name         VARCHAR(200) NOT NULL,
    plan         VARCHAR(20),
    created_at   TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Tenant-specific Tables (อยู่ใน tenant schema)
CREATE TABLE tenant_abc.users (
    user_id   SERIAL PRIMARY KEY,
    email     VARCHAR(255) UNIQUE NOT NULL,
    name      VARCHAR(100)
);

CREATE TABLE tenant_abc.products (
    product_id SERIAL PRIMARY KEY,
    name       VARCHAR(200) NOT NULL,
    price      DECIMAL(10,2)
);

-- ข้อดี: Isolation ดี, ง่ายต่อ per-tenant customization
-- ข้อเสีย: ยาก Cross-tenant, Schema changes ต้องทำทุก Schema
```

### แนวทางที่ 3: Shared Tables (tenant_id Column)

```sql
-- ✅ ส่วนใหญ่ใช้แนวทางนี้
CREATE TABLE customers (
    customer_id INTEGER,
    tenant_id   INTEGER NOT NULL REFERENCES tenants,
    name        VARCHAR(100) NOT NULL,
    email       VARCHAR(255) NOT NULL,
    created_at  TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (customer_id, tenant_id),
    UNIQUE (tenant_id, email)  -- Unique per tenant
);

CREATE TABLE orders (
    order_id    INTEGER,
    tenant_id   INTEGER NOT NULL REFERENCES tenants,
    customer_id INTEGER NOT NULL,
    total       DECIMAL(10,2),
    PRIMARY KEY (order_id, tenant_id),
    FOREIGN KEY (customer_id, tenant_id) REFERENCES customers(customer_id, tenant_id)
);

-- ต้อง Filter ด้วย tenant_id ทุกครั้ง!
SELECT * FROM customers WHERE tenant_id = 1001;

-- ✅ ใช้ Row-Level Security (RLS) เพื่อ Auto-filter
ALTER TABLE customers ENABLE ROW LEVEL SECURITY;

CREATE POLICY tenant_isolation ON customers
    USING (tenant_id = current_setting('app.current_tenant')::INTEGER);

-- Application ต้อง set tenant context:
-- SET app.current_tenant = '1001';
-- ทุก Query จะ auto-filter ตาม tenant แล้ว

-- Indexes ต้องรวม tenant_id เสมอ
CREATE INDEX idx_customers_tenant ON customers(tenant_id, email);
CREATE INDEX idx_orders_tenant ON orders(tenant_id, customer_id);
```

---

## Pattern 5: EAV (Entity-Attribute-Value)

### EAV Pattern คืออะไร?

```sql
-- EAV: เก็บ attributes เป็น rows แทน columns
CREATE TABLE product_attributes (
    product_id  INTEGER REFERENCES products,
    attr_name   VARCHAR(100) NOT NULL,
    attr_value  TEXT,
    PRIMARY KEY (product_id, attr_name)
);

-- ข้อมูล:
-- (1, 'color', 'red')
-- (1, 'size', 'M')
-- (1, 'material', 'cotton')
-- (2, 'brand', 'Apple')
-- (2, 'storage', '128GB')
-- (2, 'color', 'black')
```

### ทำไม EAV ถึงไม่ดี?

```sql
-- ❌ ปัญหาของ EAV:

-- 1. ไม่มี Type Safety
-- attr_value เป็น TEXT ทั้งหมด
-- ไม่รู้ว่าเป็น number, date, หรือ string
UPDATE product_attributes SET attr_value = 'not-a-number'
WHERE attr_name = 'price';  -- ไม่มีอะไรป้องกัน!

-- 2. Query ยากมาก
-- "หาสินค้าที่มี color='red' และ size='M'"
SELECT DISTINCT ea1.product_id
FROM product_attributes ea1
JOIN product_attributes ea2 ON ea1.product_id = ea2.product_id
WHERE ea1.attr_name = 'color' AND ea1.attr_value = 'red'
AND ea2.attr_name = 'size' AND ea2.attr_value = 'M';
-- ต้อง Self-JOIN สำหรับแต่ละ attribute!

-- 3. ไม่สามารถ FOREIGN KEY หรือ CHECK Constraint ได้

-- 4. Performance แย่มาก
```

### เมื่อใดที่ EAV อาจพอยอมรับได้?

```sql
-- ยอมรับ EAV เมื่อ:
-- 1. Attributes ไม่แน่นอน มีได้หลากหลายมาก (ไม่รู้ล่วงหน้า)
-- 2. แต่ละ Entity มี Attributes ต่างกันมากๆ
-- 3. Schema เปลี่ยนบ่อยมาก

-- ✅ ทางเลือกที่ดีกว่า EAV:

-- ทางเลือก 1: JSONB (PostgreSQL)
CREATE TABLE products (
    product_id  SERIAL PRIMARY KEY,
    name        VARCHAR(200) NOT NULL,
    category_id INTEGER,
    price       DECIMAL(10,2),
    specs       JSONB DEFAULT '{}'  -- Flexible attributes
);

-- ใช้งานได้ดีกว่า EAV:
SELECT * FROM products
WHERE specs->>'color' = 'red'
AND specs->>'size' = 'M';

-- สร้าง Index บน JSONB fields:
CREATE INDEX idx_products_color ON products((specs->>'color'));
CREATE INDEX idx_products_specs_gin ON products USING gin(specs);

-- ทางเลือก 2: Typed Tables สำหรับแต่ละ Category
CREATE TABLE clothing_specs (
    product_id INTEGER PRIMARY KEY REFERENCES products,
    color      VARCHAR(50),
    size       VARCHAR(10),
    material   VARCHAR(100),
    care       VARCHAR(200)
);

CREATE TABLE electronics_specs (
    product_id  INTEGER PRIMARY KEY REFERENCES products,
    brand       VARCHAR(100),
    storage_gb  INTEGER,
    ram_gb      INTEGER,
    warranty_months SMALLINT
);
```

---

## Pattern 6: Polymorphic Associations

### ปัญหา: Comments บน หลายประเภท Content

```sql
-- ❌ Anti-pattern: Nullable FK columns
CREATE TABLE comments_bad (
    comment_id  SERIAL PRIMARY KEY,
    content     TEXT NOT NULL,
    -- หนึ่งในนี้จะเป็น NOT NULL อีกเป็น NULL
    post_id     INTEGER REFERENCES posts,
    product_id  INTEGER REFERENCES products,
    photo_id    INTEGER REFERENCES photos,
    -- ปัญหา: ไม่สามารถ Enforce ว่าต้องมีอย่างน้อย 1 FK
    -- ปัญหา: Sparse ข้อมูล (ส่วนใหญ่ NULL)
);
```

### Solution A: Separate Tables

```sql
-- ✅ ง่ายที่สุด: แยก Comment Table ต่อ Content Type
CREATE TABLE post_comments (
    comment_id  SERIAL PRIMARY KEY,
    post_id     INTEGER NOT NULL REFERENCES posts,
    user_id     INTEGER NOT NULL REFERENCES users,
    content     TEXT NOT NULL,
    created_at  TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE product_comments (
    comment_id  SERIAL PRIMARY KEY,
    product_id  INTEGER NOT NULL REFERENCES products,
    user_id     INTEGER NOT NULL REFERENCES users,
    content     TEXT NOT NULL,
    rating      SMALLINT CHECK (rating BETWEEN 1 AND 5),
    created_at  TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- ข้อดี: Clean Schema, Referential Integrity
-- ข้อเสีย: ถ้ามี Content Types เยอะ ต้องหลายตาราง
```

### Solution B: Generic Entity Table

```sql
-- ✅ Generic Polymorphic Pattern
CREATE TABLE commentable_entities (
    entity_id   SERIAL PRIMARY KEY,
    entity_type VARCHAR(30) NOT NULL CHECK (entity_type IN ('post','product','photo')),
    external_id INTEGER NOT NULL,
    UNIQUE (entity_type, external_id)
);

CREATE TABLE comments (
    comment_id  SERIAL PRIMARY KEY,
    entity_id   INTEGER NOT NULL REFERENCES commentable_entities,
    user_id     INTEGER NOT NULL REFERENCES users,
    content     TEXT NOT NULL,
    created_at  TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Usage:
-- ลงทะเบียน entity ก่อน:
INSERT INTO commentable_entities (entity_type, external_id) VALUES ('post', 123);
-- แล้ว comment:
INSERT INTO comments (entity_id, user_id, content) VALUES (1, 456, 'Great post!');

-- Query comments ของ post:
SELECT c.*
FROM comments c
JOIN commentable_entities ce ON c.entity_id = ce.entity_id
WHERE ce.entity_type = 'post' AND ce.external_id = 123;
```

### Solution C: Type Discriminator Pattern

```sql
-- ✅ Flexible แต่ยังมี Structure
CREATE TABLE comments (
    comment_id      SERIAL PRIMARY KEY,
    commentable_type VARCHAR(30) NOT NULL,  -- 'post', 'product', 'photo'
    commentable_id   INTEGER NOT NULL,
    user_id         INTEGER NOT NULL REFERENCES users,
    content         TEXT NOT NULL,
    created_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_comments_type_id ON comments(commentable_type, commentable_id);

-- Partial Indexes สำหรับแต่ละ Type
CREATE INDEX idx_comments_post ON comments(commentable_id) WHERE commentable_type = 'post';
CREATE INDEX idx_comments_product ON comments(commentable_id) WHERE commentable_type = 'product';

-- Query
SELECT * FROM comments
WHERE commentable_type = 'post' AND commentable_id = 123
ORDER BY created_at DESC;

-- ข้อเสีย: ไม่สามารถ FOREIGN KEY ได้ (เพราะ commentable_id ไม่รู้ว่า reference ตารางไหน)
-- ต้องจัดการ Referential Integrity ด้วย Application หรือ Trigger
```

---

## Pattern 7: Category/Subcategory Trees

### Tree Structures ใน SQL

#### Adjacency List (Self-referencing)

```sql
-- Simplest approach
CREATE TABLE categories (
    category_id INTEGER PRIMARY KEY,
    parent_id   INTEGER REFERENCES categories,
    name        VARCHAR(100) NOT NULL,
    slug        VARCHAR(100) UNIQUE NOT NULL,
    sort_order  SMALLINT DEFAULT 0
);

-- ข้อดี: Simple, ง่ายต่อ Insert/Update
-- ข้อเสีย: Recursive Query ซับซ้อนสำหรับ Descendants

-- ดูลูกหลาน (Recursive CTE)
WITH RECURSIVE descendants AS (
    SELECT category_id, name, 0 AS level
    FROM categories WHERE category_id = 5  -- Electronics

    UNION ALL

    SELECT c.category_id, c.name, d.level + 1
    FROM categories c
    JOIN descendants d ON c.parent_id = d.category_id
)
SELECT * FROM descendants ORDER BY level, name;
```

#### Closure Table (Best for Read Performance)

```sql
-- เก็บทุก ancestor-descendant relationship
CREATE TABLE categories (
    category_id INTEGER PRIMARY KEY,
    name        VARCHAR(100) NOT NULL
);

CREATE TABLE category_closure (
    ancestor_id   INTEGER REFERENCES categories,
    descendant_id INTEGER REFERENCES categories,
    depth         INTEGER NOT NULL,  -- 0 = self
    PRIMARY KEY (ancestor_id, descendant_id)
);

-- Populate: เมื่อเพิ่ม node ใหม่ (parent_id = 5)
CREATE OR REPLACE FUNCTION add_category_to_tree(
    p_category_id INTEGER,
    p_parent_id   INTEGER
) RETURNS VOID AS $$
BEGIN
    -- Self reference (depth=0)
    INSERT INTO category_closure (ancestor_id, descendant_id, depth)
    VALUES (p_category_id, p_category_id, 0);

    -- Ancestors from parent
    INSERT INTO category_closure (ancestor_id, descendant_id, depth)
    SELECT cl.ancestor_id, p_category_id, cl.depth + 1
    FROM category_closure cl
    WHERE cl.descendant_id = p_parent_id;
END;
$$ LANGUAGE plpgsql;

-- Query descendants of 'Electronics' (category_id=5): FAST!
SELECT c.*
FROM categories c
JOIN category_closure cc ON c.category_id = cc.descendant_id
WHERE cc.ancestor_id = 5 AND cc.depth > 0;

-- Query ancestors of 'iPhone 15' (category_id=150): FAST!
SELECT c.*, cc.depth
FROM categories c
JOIN category_closure cc ON c.category_id = cc.ancestor_id
WHERE cc.descendant_id = 150
ORDER BY cc.depth DESC;
```

#### Materialized Path

```sql
-- เก็บ Path ไว้ใน Column
CREATE TABLE categories (
    category_id INTEGER PRIMARY KEY,
    parent_id   INTEGER REFERENCES categories,
    name        VARCHAR(100) NOT NULL,
    path        TEXT NOT NULL,   -- '/1/5/12/150/'
    level       SMALLINT NOT NULL DEFAULT 0
);

-- Query descendants: ใช้ LIKE
SELECT * FROM categories WHERE path LIKE '/1/5/%';

-- Query ancestors: Parse path
SELECT * FROM categories
WHERE category_id = ANY(
    string_to_array(trim('/' FROM '/1/5/12/150/'), '/')::INTEGER[]
);

-- ข้อดี: Fast reads, ง่าย query
-- ข้อเสีย: Update path ยากเมื่อย้าย node
```

---

## Pattern 8: Tag/Label Systems

### Simple Tag System

```sql
-- Tags Table
CREATE TABLE tags (
    tag_id   SERIAL PRIMARY KEY,
    name     VARCHAR(50) UNIQUE NOT NULL,
    slug     VARCHAR(50) UNIQUE NOT NULL,
    color    VARCHAR(7),  -- hex color '#FF5733'
    usage_count INTEGER DEFAULT 0  -- Denormalized counter
);

-- Polymorphic Tagging (Tags บน หลาย Entity Types)
CREATE TABLE taggings (
    tagging_id    SERIAL PRIMARY KEY,
    tag_id        INTEGER NOT NULL REFERENCES tags ON DELETE CASCADE,
    taggable_type VARCHAR(30) NOT NULL,  -- 'post', 'product', 'article'
    taggable_id   INTEGER NOT NULL,
    created_at    TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UNIQUE (tag_id, taggable_type, taggable_id)
);

CREATE INDEX idx_taggings_tag ON taggings(tag_id);
CREATE INDEX idx_taggings_entity ON taggings(taggable_type, taggable_id);

-- Maintain tag usage_count
CREATE OR REPLACE FUNCTION update_tag_count()
RETURNS TRIGGER AS $$
BEGIN
    IF TG_OP = 'INSERT' THEN
        UPDATE tags SET usage_count = usage_count + 1 WHERE tag_id = NEW.tag_id;
    ELSIF TG_OP = 'DELETE' THEN
        UPDATE tags SET usage_count = GREATEST(usage_count - 1, 0) WHERE tag_id = OLD.tag_id;
    END IF;
    RETURN COALESCE(NEW, OLD);
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_tag_count
    AFTER INSERT OR DELETE ON taggings
    FOR EACH ROW EXECUTE FUNCTION update_tag_count();

-- Query: หา posts ที่มี tag = 'tutorial'
SELECT p.*
FROM posts p
JOIN taggings t ON t.taggable_id = p.post_id AND t.taggable_type = 'post'
JOIN tags tag ON t.tag_id = tag.tag_id
WHERE tag.slug = 'tutorial';

-- Query: หา posts ที่มีทุก tags ที่ระบุ
SELECT p.post_id, p.title
FROM posts p
WHERE (
    SELECT COUNT(DISTINCT t.tag_id)
    FROM taggings t
    JOIN tags tag ON t.tag_id = tag.tag_id
    WHERE t.taggable_id = p.post_id
    AND t.taggable_type = 'post'
    AND tag.slug = ANY(ARRAY['sql', 'database', 'tutorial'])
) = 3;  -- ต้องมีครบทั้ง 3 tags
```

### PostgreSQL Array Tags (Simpler แต่ Less Flexible)

```sql
-- สำหรับ Simple use case
CREATE TABLE articles (
    article_id  SERIAL PRIMARY KEY,
    title       VARCHAR(300) NOT NULL,
    content     TEXT NOT NULL,
    tags        TEXT[] NOT NULL DEFAULT '{}',
    published_at TIMESTAMP
);

-- GIN Index สำหรับ Array Search
CREATE INDEX idx_articles_tags ON articles USING gin(tags);

-- Query
SELECT * FROM articles WHERE tags @> ARRAY['sql'];        -- มี 'sql'
SELECT * FROM articles WHERE tags && ARRAY['sql', 'database'];  -- มีอย่างน้อย 1 จาก list

-- Popular Tags
SELECT
    UNNEST(tags) AS tag,
    COUNT(*) AS article_count
FROM articles
WHERE published_at IS NOT NULL
GROUP BY 1
ORDER BY 2 DESC
LIMIT 20;
```

---

## สรุป Design Patterns

```
Pattern 1: Lookup Tables
- ใช้สำหรับ: Fixed values, Enums, Reference data
- Key: FK + metadata columns (label, sort_order, is_active)

Pattern 2: Audit Tables
- ใช้สำหรับ: Change tracking, Compliance, Debugging
- Key: Trigger + JSONB old/new values

Pattern 3: Soft Delete
- ใช้สำหรับ: Data recovery, Referential integrity
- Key: deleted_at + Partial Unique Index

Pattern 4: Multi-tenant
- ใช้สำหรับ: SaaS applications
- Key: tenant_id column + Row-Level Security

Pattern 5: EAV (Avoid หรือใช้ด้วยความระมัดระวัง)
- ใช้สำหรับ: Dynamic attributes (ถ้าจำเป็นจริงๆ)
- Key: ใช้ JSONB แทน EAV ถ้าทำได้

Pattern 6: Polymorphic Associations
- ใช้สำหรับ: Comments, Likes, Tags บน multiple entity types
- Key: Type discriminator + Partial Index

Pattern 7: Category Trees
- ใช้สำหรับ: Hierarchical data
- Key: Adjacency List (simple) หรือ Closure Table (fast reads)

Pattern 8: Tag System
- ใช้สำหรับ: Flexible categorization
- Key: Tags table + Taggings junction table + GIN Index
```

---

## แบบฝึกหัด (10 ข้อ)

### ข้อ 1
ออกแบบ Lookup Table สำหรับ "ระดับความสำคัญ" (Priority) ของ Tickets

**เฉลย**:
```sql
CREATE TABLE priorities (
    code        VARCHAR(10) PRIMARY KEY,  -- 'low', 'medium', 'high', 'critical'
    label       VARCHAR(50) NOT NULL,
    description TEXT,
    color       VARCHAR(7),   -- #28A745 (green), #FFC107 (yellow), etc.
    sla_hours   INTEGER,      -- SLA response time
    sort_order  SMALLINT DEFAULT 0
);

INSERT INTO priorities (code, label, color, sla_hours, sort_order) VALUES
('low',      'ต่ำ',         '#6C757D', 72, 1),
('medium',   'ปานกลาง',     '#FFC107', 24, 2),
('high',     'สูง',         '#FD7E14', 8,  3),
('critical', 'วิกฤต',       '#DC3545', 2,  4);

CREATE TABLE tickets (
    ticket_id   SERIAL PRIMARY KEY,
    subject     VARCHAR(300) NOT NULL,
    priority    VARCHAR(10) NOT NULL DEFAULT 'medium' REFERENCES priorities(code),
    created_at  TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

### ข้อ 2
เพิ่ม Audit Log ให้กับตาราง employees โดยใช้ Trigger

**เฉลย**:
```sql
CREATE TABLE employee_audit_log (
    log_id      BIGSERIAL PRIMARY KEY,
    emp_id      INTEGER NOT NULL,
    action      VARCHAR(10) NOT NULL,
    old_values  JSONB,
    new_values  JSONB,
    changed_by  INTEGER,
    changed_at  TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE OR REPLACE FUNCTION log_employee_changes()
RETURNS TRIGGER AS $$
BEGIN
    INSERT INTO employee_audit_log (emp_id, action, old_values, new_values)
    VALUES (
        COALESCE(NEW.emp_id, OLD.emp_id),
        TG_OP,
        CASE WHEN TG_OP = 'INSERT' THEN NULL ELSE TO_JSONB(OLD) END,
        CASE WHEN TG_OP = 'DELETE' THEN NULL ELSE TO_JSONB(NEW) END
    );
    RETURN COALESCE(NEW, OLD);
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_employee_audit
    AFTER INSERT OR UPDATE OR DELETE ON employees
    FOR EACH ROW EXECUTE FUNCTION log_employee_changes();
```

### ข้อ 3
Implement Soft Delete สำหรับตาราง articles พร้อม Partial Unique Index

**เฉลย**:
```sql
ALTER TABLE articles
ADD COLUMN deleted_at TIMESTAMP,
ADD COLUMN deleted_by INTEGER REFERENCES users;

-- Partial Unique Index: slug unique เฉพาะ active records
CREATE UNIQUE INDEX idx_articles_slug_active
ON articles(slug) WHERE deleted_at IS NULL;

-- View สำหรับ Active articles
CREATE VIEW active_articles AS
SELECT * FROM articles WHERE deleted_at IS NULL;

-- Soft delete function
CREATE OR REPLACE FUNCTION delete_article(
    p_article_id INTEGER,
    p_user_id INTEGER
) RETURNS VOID AS $$
BEGIN
    UPDATE articles
    SET deleted_at = CURRENT_TIMESTAMP, deleted_by = p_user_id
    WHERE article_id = p_article_id AND deleted_at IS NULL;
    IF NOT FOUND THEN
        RAISE EXCEPTION 'Article not found or already deleted';
    END IF;
END;
$$ LANGUAGE plpgsql;
```

### ข้อ 4
ออกแบบ Multi-tenant Schema สำหรับ Project Management App

**เฉลย**:
```sql
CREATE TABLE tenants (
    tenant_id   SERIAL PRIMARY KEY,
    name        VARCHAR(200) NOT NULL,
    subdomain   VARCHAR(50) UNIQUE NOT NULL,
    plan        VARCHAR(20) DEFAULT 'free',
    created_at  TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    is_active   BOOLEAN DEFAULT TRUE
);

CREATE TABLE projects (
    project_id  SERIAL PRIMARY KEY,
    tenant_id   INTEGER NOT NULL REFERENCES tenants,
    name        VARCHAR(200) NOT NULL,
    created_at  TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UNIQUE (tenant_id, name)
);

CREATE TABLE tasks (
    task_id     SERIAL PRIMARY KEY,
    tenant_id   INTEGER NOT NULL REFERENCES tenants,
    project_id  INTEGER NOT NULL,
    title       VARCHAR(300) NOT NULL,
    status      VARCHAR(20) DEFAULT 'todo',
    FOREIGN KEY (tenant_id, project_id) REFERENCES projects(tenant_id, project_id) -- แต่ projects pk เป็น simple key
);

-- Row-Level Security
ALTER TABLE projects ENABLE ROW LEVEL SECURITY;
CREATE POLICY tenant_projects ON projects
    USING (tenant_id = current_setting('app.tenant_id')::INTEGER);

CREATE INDEX idx_projects_tenant ON projects(tenant_id);
CREATE INDEX idx_tasks_tenant ON tasks(tenant_id, project_id);
```

### ข้อ 5
อธิบายว่าทำไม EAV Pattern ถึงไม่ดีและแนะนำทางเลือกที่ดีกว่า

**เฉลย**:
```
ทำไม EAV ไม่ดี:
1. ไม่มี Type Safety: attr_value = TEXT ทุก attribute
2. Query ยาก: ต้อง Self-JOIN หลายครั้ง
3. ไม่มี Constraints: CHECK, FOREIGN KEY ไม่สามารถทำได้
4. Performance แย่: ต้องใช้ Index หลายตัว
5. ยากต่อการ Query หลาย attributes พร้อมกัน
6. SQL Aggregation ซับซ้อนมาก

ทางเลือกที่ดีกว่า:
1. JSONB (ถ้า attributes หลากหลายมาก):
   ALTER TABLE products ADD COLUMN specs JSONB;
   
2. Typed Attribute Tables (ถ้า attributes แบ่งตาม Category):
   CREATE TABLE electronics_specs (...);
   CREATE TABLE clothing_specs (...);
   
3. Class Table Inheritance (ถ้า Entity มี Subtypes):
   CREATE TABLE products (...);
   CREATE TABLE electronics (...) INHERITS (products);
   
4. Concrete Table per Type:
   CREATE TABLE phones (...);
   CREATE TABLE shirts (...);
```

### ข้อ 6
Implement Tag System สำหรับ Blog ที่รองรับหลาย Content Types

**เฉลย**:
```sql
CREATE TABLE tags (
    tag_id    SERIAL PRIMARY KEY,
    name      VARCHAR(50) UNIQUE NOT NULL,
    slug      VARCHAR(50) UNIQUE NOT NULL,
    usage_count INTEGER DEFAULT 0
);

CREATE TABLE taggings (
    tagging_id    SERIAL PRIMARY KEY,
    tag_id        INTEGER NOT NULL REFERENCES tags ON DELETE CASCADE,
    taggable_type VARCHAR(20) NOT NULL CHECK (taggable_type IN ('post','video','podcast')),
    taggable_id   INTEGER NOT NULL,
    created_at    TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UNIQUE (tag_id, taggable_type, taggable_id)
);

CREATE INDEX idx_taggings_entity ON taggings(taggable_type, taggable_id);

-- Maintain counts
CREATE OR REPLACE FUNCTION maintain_tag_count()
RETURNS TRIGGER AS $$
BEGIN
    IF TG_OP = 'INSERT' THEN
        UPDATE tags SET usage_count = usage_count + 1 WHERE tag_id = NEW.tag_id;
    ELSE
        UPDATE tags SET usage_count = GREATEST(0, usage_count - 1) WHERE tag_id = OLD.tag_id;
    END IF;
    RETURN COALESCE(NEW, OLD);
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_tag_count AFTER INSERT OR DELETE ON taggings
FOR EACH ROW EXECUTE FUNCTION maintain_tag_count();
```

### ข้อ 7
สร้าง Category Tree ด้วย Closure Table สำหรับ E-Commerce

**เฉลย**:
```sql
CREATE TABLE categories (
    category_id  SERIAL PRIMARY KEY,
    name         VARCHAR(100) NOT NULL,
    slug         VARCHAR(100) UNIQUE NOT NULL,
    is_active    BOOLEAN DEFAULT TRUE
);

CREATE TABLE category_paths (
    ancestor_id   INTEGER NOT NULL REFERENCES categories,
    descendant_id INTEGER NOT NULL REFERENCES categories,
    path_length   INTEGER NOT NULL,
    PRIMARY KEY (ancestor_id, descendant_id)
);

-- Insert root category
CREATE OR REPLACE FUNCTION insert_category(
    p_name      VARCHAR,
    p_slug      VARCHAR,
    p_parent_id INTEGER DEFAULT NULL
) RETURNS INTEGER AS $$
DECLARE
    v_new_id INTEGER;
BEGIN
    INSERT INTO categories (name, slug) VALUES (p_name, p_slug)
    RETURNING category_id INTO v_new_id;

    -- Self reference
    INSERT INTO category_paths (ancestor_id, descendant_id, path_length)
    VALUES (v_new_id, v_new_id, 0);

    -- Inherit parent's ancestors
    IF p_parent_id IS NOT NULL THEN
        INSERT INTO category_paths (ancestor_id, descendant_id, path_length)
        SELECT ancestor_id, v_new_id, path_length + 1
        FROM category_paths
        WHERE descendant_id = p_parent_id;
    END IF;

    RETURN v_new_id;
END;
$$ LANGUAGE plpgsql;

-- Get all subcategories
SELECT c.*, cp.path_length AS depth
FROM categories c
JOIN category_paths cp ON c.category_id = cp.descendant_id
WHERE cp.ancestor_id = 5  -- Electronics
AND cp.path_length > 0;
```

### ข้อ 8
ออกแบบ Schema สำหรับ Comment System ที่รองรับ Posts, Products, Videos

**เฉลย**:
```sql
-- Solution: Type Discriminator + Partial Indexes
CREATE TABLE comments (
    comment_id      SERIAL PRIMARY KEY,
    parent_id       INTEGER REFERENCES comments,  -- nested comments
    commentable_type VARCHAR(20) NOT NULL CHECK (commentable_type IN ('post','product','video')),
    commentable_id   INTEGER NOT NULL,
    user_id         INTEGER NOT NULL REFERENCES users,
    content         TEXT NOT NULL,
    is_approved     BOOLEAN DEFAULT TRUE,
    created_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    deleted_at      TIMESTAMP
);

-- Indexes per type
CREATE INDEX idx_comments_post ON comments(commentable_id)
    WHERE commentable_type = 'post' AND deleted_at IS NULL;
CREATE INDEX idx_comments_product ON comments(commentable_id)
    WHERE commentable_type = 'product' AND deleted_at IS NULL;

-- Query comments for post #123
SELECT c.*, u.username, u.avatar_url
FROM comments c
JOIN users u ON c.user_id = u.user_id
WHERE c.commentable_type = 'post'
AND c.commentable_id = 123
AND c.deleted_at IS NULL
AND c.parent_id IS NULL
ORDER BY c.created_at;
```

### ข้อ 9
Implement Versioning สำหรับ Documents (เก็บ History ทุก Version)

**เฉลย**:
```sql
CREATE TABLE documents (
    doc_id      SERIAL PRIMARY KEY,
    title       VARCHAR(300) NOT NULL,
    current_version INTEGER NOT NULL DEFAULT 1,
    created_by  INTEGER REFERENCES users,
    created_at  TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at  TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE document_versions (
    version_id  SERIAL PRIMARY KEY,
    doc_id      INTEGER NOT NULL REFERENCES documents,
    version_no  INTEGER NOT NULL,
    title       VARCHAR(300) NOT NULL,
    content     TEXT NOT NULL,
    change_notes TEXT,
    created_by  INTEGER REFERENCES users,
    created_at  TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UNIQUE (doc_id, version_no)
);

-- Save new version function
CREATE OR REPLACE FUNCTION save_document_version(
    p_doc_id    INTEGER,
    p_title     VARCHAR,
    p_content   TEXT,
    p_notes     TEXT,
    p_user_id   INTEGER
) RETURNS INTEGER AS $$
DECLARE
    v_version INTEGER;
BEGIN
    SELECT current_version + 1 INTO v_version
    FROM documents WHERE doc_id = p_doc_id;

    INSERT INTO document_versions (doc_id, version_no, title, content, change_notes, created_by)
    VALUES (p_doc_id, v_version, p_title, p_content, p_notes, p_user_id);

    UPDATE documents
    SET title = p_title, current_version = v_version, updated_at = NOW()
    WHERE doc_id = p_doc_id;

    RETURN v_version;
END;
$$ LANGUAGE plpgsql;
```

### ข้อ 10
สร้าง Complete E-Commerce Tag System พร้อม Popular Tags และ Auto-complete

**เฉลย**:
```sql
CREATE TABLE tags (
    tag_id      SERIAL PRIMARY KEY,
    name        VARCHAR(50) NOT NULL,
    slug        VARCHAR(50) UNIQUE NOT NULL,
    usage_count INTEGER DEFAULT 0,
    created_at  TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Full-text search on tags
CREATE INDEX idx_tags_name_trgm ON tags USING gin(name gin_trgm_ops);
-- ต้องติดตั้ง extension: CREATE EXTENSION pg_trgm;

CREATE TABLE product_tags (
    product_id INTEGER NOT NULL REFERENCES products ON DELETE CASCADE,
    tag_id     INTEGER NOT NULL REFERENCES tags ON DELETE CASCADE,
    PRIMARY KEY (product_id, tag_id)
);

-- Popular Tags (Materialized View)
CREATE MATERIALIZED VIEW popular_tags AS
SELECT
    t.tag_id,
    t.name,
    t.slug,
    t.usage_count,
    COUNT(DISTINCT pt.product_id) AS product_count
FROM tags t
LEFT JOIN product_tags pt ON t.tag_id = pt.tag_id
GROUP BY t.tag_id, t.name, t.slug, t.usage_count
ORDER BY product_count DESC
WITH DATA;

REFRESH MATERIALIZED VIEW popular_tags;  -- รายวัน

-- Auto-complete Query
SELECT name, slug, product_count
FROM popular_tags
WHERE name ILIKE 'sale%'  -- หรือใช้ trgm: name % 'sale'
ORDER BY product_count DESC
LIMIT 10;

-- Products ที่มี Tag
SELECT p.*
FROM products p
JOIN product_tags pt ON p.product_id = pt.product_id
JOIN tags t ON pt.tag_id = t.tag_id
WHERE t.slug = 'on-sale'
ORDER BY p.name;
```

---

*จบ Part 058: Database Schema Design Patterns*

**ในส่วนต่อไป (Part 059)**: เราจะเรียนรู้เรื่อง Advanced Schema Topics - Inheritance ซึ่งเป็นเทคนิคสำหรับจัดการ Entity Hierarchies ที่ซับซ้อน
