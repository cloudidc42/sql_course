# ส่วนที่ 97: JSON ใน SQL

## บทนำ

JSON (JavaScript Object Notation) กลายเป็นรูปแบบข้อมูลที่สำคัญมากในโลกสมัยใหม่ ฐานข้อมูลหลักทุกตัวรองรับ JSON ในรูปแบบที่แตกต่างกัน บทนี้จะครอบคลุมการใช้งาน JSON ใน PostgreSQL, MySQL, SQLite, และ SQL Server

---

## 97.1 JSON Data Types

### PostgreSQL: json vs jsonb

```sql
-- json: เก็บข้อมูลเป็น text ตัวอักษรต่อตัวอักษร (ไม่ parse)
-- jsonb: เก็บเป็น binary (parse แล้ว), รองรับ index, เร็วกว่าในการ query

CREATE TABLE user_profiles_json (
    user_id     INT PRIMARY KEY,
    profile     JSON          -- เก็บเป็น text ดั้งเดิม
);

CREATE TABLE user_profiles_jsonb (
    user_id     INT PRIMARY KEY,
    profile     JSONB         -- เก็บเป็น binary, แนะนำสำหรับ production
);

-- ความแตกต่าง:
-- json: รักษา whitespace, key order, duplicate keys
-- jsonb: ลบ whitespace, sort keys, dedup keys, รองรับ GIN index
```

### ตัวอย่างที่ 1: สร้างตารางพร้อม JSONB

```sql
-- ตารางสินค้าที่มี attributes หลากหลาย
CREATE TABLE products (
    product_id      INT PRIMARY KEY,
    name            VARCHAR(200),
    category        VARCHAR(50),
    price           DECIMAL(10,2),
    attributes      JSONB,          -- คุณสมบัติที่แตกต่างกันตามประเภท
    metadata        JSONB           -- tag, created_by, ฯลฯ
);

INSERT INTO products VALUES
(1, 'MacBook Pro 14', 'Laptop', 89900, 
    '{"ram": "16GB", "storage": "512GB SSD", "cpu": "M3 Pro", "color": "Silver", "weight_kg": 1.6}',
    '{"brand": "Apple", "warranty_years": 1, "tags": ["premium", "ultrabook"]}'),
(2, 'iPhone 15 Pro', 'Phone', 45900,
    '{"storage": "256GB", "color": "Natural Titanium", "5g": true, "camera_mp": 48}',
    '{"brand": "Apple", "warranty_years": 1, "tags": ["flagship", "5g"]}'),
(3, 'Samsung 65" QLED', 'TV', 39900,
    '{"size_inch": 65, "resolution": "4K", "refresh_rate_hz": 120, "hdr": "HDR10+"}',
    '{"brand": "Samsung", "warranty_years": 2, "tags": ["smart-tv", "4k"]}'),
(4, 'Sony WH-1000XM5', 'Headphone', 12900,
    '{"type": "over-ear", "noise_cancelling": true, "battery_hours": 30, "wireless": true}',
    '{"brand": "Sony", "warranty_years": 1, "tags": ["noise-cancelling", "wireless"]}'),
(5, 'Logitech MX Master 3', 'Mouse', 3900,
    '{"dpi": 8000, "wireless": true, "buttons": 7, "battery_type": "rechargeable"}',
    '{"brand": "Logitech", "warranty_years": 2, "tags": ["ergonomic", "wireless"]}');
```

---

## 97.2 JSON Operators ใน PostgreSQL

### ตัวอย่างที่ 2: Operators พื้นฐาน

```sql
-- -> ดึงค่าเป็น JSON
-- ->> ดึงค่าเป็น text
-- #> ดึง path เป็น JSON
-- #>> ดึง path เป็น text

-- ดึง brand เป็น JSON
SELECT name, metadata -> 'brand' AS brand_json
FROM products;
-- ผลลัพธ์: "Apple" (มี quotes)

-- ดึง brand เป็น text
SELECT name, metadata ->> 'brand' AS brand_text  
FROM products;
-- ผลลัพธ์: Apple (ไม่มี quotes)

-- ดึง nested path
SELECT name, metadata #> '{tags, 0}' AS first_tag_json
FROM products;

-- ดึง nested path เป็น text
SELECT name, metadata #>> '{tags, 0}' AS first_tag_text
FROM products;
```

### ตัวอย่างที่ 3: Query ด้วย WHERE clause

```sql
-- Filter ตาม JSON value
SELECT name, price
FROM products
WHERE attributes ->> 'wireless' = 'true';

-- Filter ตาม numeric JSON value
SELECT name, attributes ->> 'ram' AS ram
FROM products
WHERE (attributes ->> 'weight_kg')::DECIMAL < 2.0;

-- Filter ด้วย @> (containment)
SELECT name
FROM products
WHERE attributes @> '{"wireless": true}';

-- ตรวจว่ามี key นั้นหรือไม่ (? operator)
SELECT name
FROM products
WHERE attributes ? '5g';

-- ตรวจว่ามีหนึ่งใน keys (?| operator)
SELECT name
FROM products
WHERE attributes ?| ARRAY['5g', 'wireless'];

-- ตรวจว่ามีทุก keys (?& operator)
SELECT name
FROM products
WHERE attributes ?& ARRAY['wireless', 'noise_cancelling'];
```

### ตัวอย่างที่ 4: JSON Array Operations

```sql
-- ดึง element จาก JSON array
SELECT 
    name,
    metadata -> 'tags' AS all_tags,
    metadata -> 'tags' -> 0 AS first_tag,
    jsonb_array_length(metadata -> 'tags') AS tag_count
FROM products;

-- Filter ด้วย array containment
SELECT name
FROM products
WHERE metadata -> 'tags' @> '["wireless"]';

-- Unnest JSON array
SELECT 
    p.name,
    tag.value AS tag
FROM products p,
     jsonb_array_elements_text(p.metadata -> 'tags') tag
ORDER BY p.name, tag.value;
```

---

## 97.3 สร้าง JSON ด้วย json_build_object

### ตัวอย่างที่ 5: json_build_object พื้นฐาน

```sql
-- สร้าง JSON object จาก columns
SELECT 
    json_build_object(
        'id', product_id,
        'name', name,
        'price', price,
        'category', category
    ) AS product_json
FROM products;

-- jsonb_build_object (JSONB version)
SELECT 
    jsonb_build_object(
        'product', name,
        'price_thb', price,
        'attributes', attributes
    ) AS product_info
FROM products
WHERE category = 'Laptop';
```

### ตัวอย่างที่ 6: สร้าง JSON Array ด้วย json_agg

```sql
-- รวมข้อมูลเป็น JSON array
SELECT 
    category,
    json_agg(
        json_build_object(
            'id', product_id,
            'name', name,
            'price', price
        ) ORDER BY price
    ) AS products
FROM products
GROUP BY category;

-- jsonb_agg (JSONB version)
SELECT 
    metadata ->> 'brand' AS brand,
    jsonb_agg(
        jsonb_build_object('name', name, 'price', price)
        ORDER BY price
    ) AS products,
    COUNT(*) AS product_count,
    MIN(price) AS min_price,
    MAX(price) AS max_price
FROM products
GROUP BY metadata ->> 'brand';
```

### ตัวอย่างที่ 7: json_object_agg

```sql
-- สร้าง JSON object จาก key-value pairs
SELECT 
    json_object_agg(name, price) AS product_prices
FROM products
WHERE category = 'Laptop';
-- ผลลัพธ์: {"MacBook Pro 14": 89900, ...}
```

---

## 97.4 JSON ใน MySQL

### ตัวอย่างที่ 8: MySQL JSON Type

```sql
-- MySQL 5.7.8+: JSON data type
CREATE TABLE orders_mysql (
    order_id    INT PRIMARY KEY AUTO_INCREMENT,
    customer_id INT,
    items       JSON,     -- MySQL JSON type
    shipping    JSON,
    created_at  DATETIME DEFAULT NOW()
);

INSERT INTO orders_mysql (customer_id, items, shipping) VALUES
(101, 
 '[{"product_id": 1, "qty": 2, "price": 1500}, {"product_id": 2, "qty": 1, "price": 800}]',
 '{"address": "123 Main St", "city": "Bangkok", "zipcode": "10110"}'),
(102,
 '[{"product_id": 3, "qty": 1, "price": 2200}]',
 '{"address": "456 Oak Ave", "city": "Chiang Mai", "zipcode": "50000"}');

-- ดึงข้อมูล JSON ใน MySQL
SELECT 
    order_id,
    customer_id,
    JSON_EXTRACT(shipping, '$.city') AS city,        -- ดึงเป็น JSON
    JSON_UNQUOTE(JSON_EXTRACT(shipping, '$.city')) AS city_text,  -- ดึงเป็น text
    shipping->>'$.city' AS city_shorthand,            -- shorthand
    JSON_LENGTH(items) AS item_count
FROM orders_mysql;

-- Filter ด้วย JSON_CONTAINS
SELECT order_id
FROM orders_mysql
WHERE JSON_CONTAINS(items, '{"product_id": 1}', '$');

-- ดึง array elements
SELECT 
    order_id,
    JSON_EXTRACT(items, '$[0].product_id') AS first_item_id,
    JSON_EXTRACT(items, '$[0].qty') AS first_item_qty
FROM orders_mysql;
```

### ตัวอย่างที่ 9: MySQL JSON Functions

```sql
-- JSON_ARRAYAGG (MySQL 5.7.22+)
SELECT 
    city,
    JSON_ARRAYAGG(order_id) AS order_ids
FROM (
    SELECT order_id, JSON_UNQUOTE(shipping->>'$.city') AS city
    FROM orders_mysql
) t
GROUP BY city;

-- JSON_OBJECTAGG (MySQL 5.7.22+)
SELECT 
    JSON_OBJECTAGG(order_id, JSON_UNQUOTE(shipping->>'$.city')) AS order_cities
FROM orders_mysql;

-- JSON_TABLE: แปลง JSON array เป็น rows
SELECT jt.*
FROM orders_mysql,
     JSON_TABLE(
         items,
         '$[*]' COLUMNS (
             product_id INT PATH '$.product_id',
             qty INT PATH '$.qty',
             price DECIMAL(10,2) PATH '$.price'
         )
     ) AS jt;
```

---

## 97.5 JSON ใน SQLite

### ตัวอย่างที่ 10: SQLite JSON1 Extension

```sql
-- SQLite ใช้ json1 extension (built-in ตั้งแต่ SQLite 3.9)
CREATE TABLE config (
    app_name    TEXT PRIMARY KEY,
    settings    TEXT  -- SQLite ใช้ TEXT สำหรับ JSON
);

INSERT INTO config VALUES
('myapp', '{"debug": false, "max_connections": 100, "cache": {"ttl": 3600, "size": "256MB"}}'),
('analytics', '{"batch_size": 1000, "interval_seconds": 60, "destinations": ["warehouse", "dashboard"]}');

-- ดึงค่าจาก JSON
SELECT 
    app_name,
    json_extract(settings, '$.max_connections') AS max_conn,
    json_extract(settings, '$.cache.ttl') AS cache_ttl,
    json_extract(settings, '$.debug') AS debug_mode
FROM config;

-- json_each: iterate over JSON
SELECT 
    app_name,
    e.key,
    e.value
FROM config,
     json_each(settings) e
WHERE app_name = 'myapp';

-- json_array_length
SELECT 
    app_name,
    json_array_length(settings, '$.destinations') AS num_destinations
FROM config
WHERE app_name = 'analytics';
```

---

## 97.6 JSON ใน SQL Server

### ตัวอย่างที่ 11: SQL Server FOR JSON และ JSON_VALUE

```sql
-- SQL Server 2016+

-- ดึงค่าจาก JSON string
DECLARE @json NVARCHAR(MAX) = N'{"name": "สมชาย", "age": 30, "address": {"city": "Bangkok"}}';

SELECT 
    JSON_VALUE(@json, '$.name') AS name,
    JSON_VALUE(@json, '$.age') AS age,
    JSON_VALUE(@json, '$.address.city') AS city;

-- JSON_QUERY: ดึง JSON object/array (ไม่ใช่ scalar)
SELECT 
    JSON_VALUE(@json, '$.name') AS name,
    JSON_QUERY(@json, '$.address') AS address_object;

-- ISJSON: ตรวจสอบว่าเป็น JSON ที่ valid หรือไม่
SELECT ISJSON(@json) AS is_valid_json;  -- Returns 1

-- สร้าง JSON ด้วย FOR JSON
CREATE TABLE employees_ss (
    emp_id      INT PRIMARY KEY,
    name        NVARCHAR(100),
    department  NVARCHAR(50),
    salary      DECIMAL(10,2)
);

INSERT INTO employees_ss VALUES
(1, N'สมชาย', N'IT', 85000),
(2, N'สมหญิง', N'HR', 65000),
(3, N'วิชัย', N'IT', 90000);

-- FOR JSON PATH
SELECT emp_id, name, department, salary
FROM employees_ss
FOR JSON PATH;
-- ผลลัพธ์: [{"emp_id":1,"name":"สมชาย","department":"IT","salary":85000.00},...]

-- FOR JSON AUTO
SELECT emp_id, name, department, salary
FROM employees_ss
FOR JSON AUTO;

-- Nested JSON
SELECT 
    e.emp_id,
    e.name,
    (SELECT emp_id, name FROM employees_ss WHERE department = e.department FOR JSON PATH) AS team
FROM employees_ss e;
```

### ตัวอย่างที่ 12: OPENJSON ใน SQL Server

```sql
-- แปลง JSON array เป็น rows
DECLARE @items NVARCHAR(MAX) = N'[
    {"id": 1, "name": "Laptop", "price": 89900},
    {"id": 2, "name": "Phone", "price": 45900}
]';

SELECT *
FROM OPENJSON(@items)
WITH (
    id    INT          '$.id',
    name  NVARCHAR(100) '$.name',
    price DECIMAL(10,2) '$.price'
);

-- OPENJSON กับตาราง
CREATE TABLE orders_ss (
    order_id INT,
    items    NVARCHAR(MAX)
);

SELECT o.order_id, j.product_id, j.qty, j.price
FROM orders_ss o
CROSS APPLY OPENJSON(o.items)
WITH (
    product_id INT '$.product_id',
    qty        INT '$.qty',
    price      DECIMAL(10,2) '$.price'
) j;
```

---

## 97.7 jsonb_set - อัพเดต JSON

### ตัวอย่างที่ 13: jsonb_set พื้นฐาน

```sql
-- อัพเดตค่าใน JSONB field
-- jsonb_set(target, path, new_value[, create_missing])

-- เปลี่ยน ram
UPDATE products
SET attributes = jsonb_set(attributes, '{ram}', '"32GB"')
WHERE product_id = 1;

-- เพิ่ม field ใหม่
UPDATE products
SET attributes = jsonb_set(attributes, '{display_inch}', '14.2', true)
WHERE product_id = 1;

-- เพิ่ม element ใน array
UPDATE products
SET metadata = jsonb_set(
    metadata,
    '{tags}',
    (metadata -> 'tags') || '["bestseller"]'
)
WHERE product_id = 1;

-- ลบ key
UPDATE products
SET attributes = attributes - 'weight_kg'
WHERE product_id = 1;

-- ลบ nested key
UPDATE products
SET metadata = metadata #- '{tags, 0}'  -- ลบ element แรกของ array
WHERE product_id = 1;
```

### ตัวอย่างที่ 14: Batch JSON Updates

```sql
-- อัพเดต JSON หลาย fields พร้อมกัน
UPDATE products
SET attributes = attributes 
    || '{"refurbished": false, "in_stock": true}'::JSONB  -- merge
WHERE category = 'Laptop';

-- concat operator || สำหรับ JSONB
UPDATE products
SET metadata = metadata || jsonb_build_object(
    'updated_at', NOW()::TEXT,
    'updated_by', 'admin'
)
WHERE product_id = 1;
```

---

## 97.8 JSON Path Queries

### ตัวอย่างที่ 15: jsonpath ใน PostgreSQL 12+

```sql
-- jsonpath: ภาษา query ที่ทรงพลังสำหรับ JSON

-- jsonb_path_query: ดึงค่าตาม path
SELECT 
    name,
    jsonb_path_query(attributes, '$.ram') AS ram
FROM products;

-- jsonb_path_exists: ตรวจว่า path มีอยู่
SELECT name
FROM products
WHERE jsonb_path_exists(attributes, '$.wireless');

-- jsonb_path_query_first: ดึงค่าแรก
SELECT 
    name,
    jsonb_path_query_first(
        metadata, 
        '$.tags[*] ? (@ starts with "wire")'
    ) AS wireless_tag
FROM products;

-- Filter ด้วย condition
SELECT 
    name,
    jsonb_path_query(attributes, '$.* ? (@ > 1000)') AS large_numbers
FROM products;
```

---

## 97.9 Indexing JSON Fields

### ตัวอย่างที่ 16: สร้าง Index สำหรับ JSONB

```sql
-- GIN index สำหรับ @> containment queries
CREATE INDEX idx_products_attributes ON products USING GIN (attributes);
CREATE INDEX idx_products_metadata ON products USING GIN (metadata);

-- ใช้ index ด้วย @>
SELECT name FROM products WHERE attributes @> '{"wireless": true}';
-- ← ใช้ GIN index

-- Functional index สำหรับ specific path
CREATE INDEX idx_products_brand ON products ((metadata ->> 'brand'));

-- ใช้ functional index
SELECT name FROM products WHERE metadata ->> 'brand' = 'Apple';
-- ← ใช้ functional index

-- jsonb_path_ops: index ที่เล็กกว่าแต่รองรับเฉพาะ @> และ @?
CREATE INDEX idx_products_attrs_ops ON products USING GIN (attributes jsonb_path_ops);
```

---

## 97.10 JSON Aggregation Patterns

### ตัวอย่างที่ 17: Nested JSON Report

```sql
-- สร้าง report แบบ nested JSON
SELECT 
    category,
    jsonb_build_object(
        'category', category,
        'product_count', COUNT(*),
        'avg_price', ROUND(AVG(price), 2),
        'min_price', MIN(price),
        'max_price', MAX(price),
        'products', jsonb_agg(
            jsonb_build_object(
                'id', product_id,
                'name', name,
                'price', price,
                'brand', metadata ->> 'brand'
            ) ORDER BY price DESC
        )
    ) AS category_report
FROM products
GROUP BY category;
```

### ตัวอย่างที่ 18: Flatten JSON Array to Rows

```sql
-- Unnest JSON array เป็น rows
WITH product_tags AS (
    SELECT 
        product_id,
        name,
        jsonb_array_elements_text(metadata -> 'tags') AS tag
    FROM products
)
SELECT 
    tag,
    COUNT(*) AS product_count,
    ARRAY_AGG(name ORDER BY name) AS products_with_tag
FROM product_tags
GROUP BY tag
ORDER BY product_count DESC;
```

### ตัวอย่างที่ 19: JSON to Rows (unnest JSON object)

```sql
-- แปลง JSON object เป็น key-value rows
SELECT 
    name,
    attr_key,
    attr_value
FROM products,
     jsonb_each_text(attributes) AS kv(attr_key, attr_value)
WHERE category = 'Laptop';
```

---

## 97.11 เปรียบเทียบ JSON vs Normalized Tables

### ตัวอย่างที่ 20: เมื่อใช้ JSON?

```sql
-- ✅ ใช้ JSON เมื่อ:
-- 1. Attributes แตกต่างกันมากระหว่าง records
-- 2. Schema เปลี่ยนบ่อย (dynamic attributes)
-- 3. ข้อมูล semi-structured จาก external sources
-- 4. ไม่ต้องการ query บน individual fields บ่อย

-- ✅ ใช้ Normalized tables เมื่อ:
-- 1. Query บน fields เฉพาะบ่อยๆ
-- 2. ต้องการ JOIN กับตารางอื่น
-- 3. ต้องการ foreign key constraints
-- 4. Reporting บน fixed schema

-- ตัวอย่าง: เปรียบเทียบ performance
-- JSON approach
SELECT * FROM products WHERE (attributes ->> 'ram') = '16GB';

-- Normalized approach (เร็วกว่าถ้า index ครบ)
-- CREATE TABLE product_attributes (product_id INT, attr_name VARCHAR, attr_value VARCHAR);
-- SELECT p.* FROM products p JOIN product_attributes pa ON p.product_id = pa.product_id
-- WHERE pa.attr_name = 'ram' AND pa.attr_value = '16GB';
```

### ตัวอย่างที่ 21: Hybrid Approach

```sql
-- Best practice: ผสมทั้งสองแบบ
CREATE TABLE products_hybrid (
    product_id      INT PRIMARY KEY,
    name            VARCHAR(200),
    category        VARCHAR(50),
    price           DECIMAL(10,2),
    brand           VARCHAR(100),           -- fixed field ที่ query บ่อย
    is_wireless     BOOLEAN,                -- fixed field ที่ filter บ่อย
    extra_attributes JSONB                  -- dynamic fields
);

-- สร้าง index บน fixed fields
CREATE INDEX idx_ph_brand ON products_hybrid (brand);
CREATE INDEX idx_ph_wireless ON products_hybrid (is_wireless);
-- GIN index บน dynamic attributes
CREATE INDEX idx_ph_extra ON products_hybrid USING GIN (extra_attributes);
```

---

## 97.12 JSON Validation

### ตัวอย่างที่ 22: ตรวจสอบ JSON ก่อน Insert

```sql
-- PostgreSQL: JSONB validation เกิดขึ้นโดยอัตโนมัติ
-- การ insert JSON ที่ไม่ valid จะ error

-- สร้าง CHECK constraint
CREATE TABLE validated_json (
    id      INT PRIMARY KEY,
    data    JSONB,
    CONSTRAINT valid_structure CHECK (
        data ? 'name' AND 
        data ? 'email' AND
        (data ->> 'age')::INT BETWEEN 0 AND 150
    )
);

-- Function สำหรับ validate JSON schema
CREATE OR REPLACE FUNCTION validate_product_json(attrs JSONB)
RETURNS BOOLEAN AS $$
BEGIN
    -- ตรวจสอบว่ามี required fields
    IF NOT (attrs ? 'ram' OR attrs ? 'storage' OR attrs ? 'size_inch') THEN
        RETURN FALSE;
    END IF;
    RETURN TRUE;
END;
$$ LANGUAGE plpgsql;
```

---

## 97.13 Advanced JSON Queries

### ตัวอย่างที่ 23: JSON ใน CTE

```sql
-- ใช้ JSON ใน complex queries
WITH product_details AS (
    SELECT 
        product_id,
        name,
        category,
        price,
        metadata ->> 'brand' AS brand,
        (metadata ->> 'warranty_years')::INT AS warranty_years,
        jsonb_array_elements_text(metadata -> 'tags') AS tag
    FROM products
),
tag_summary AS (
    SELECT 
        brand,
        tag,
        COUNT(DISTINCT product_id) AS products_with_tag
    FROM product_details
    GROUP BY brand, tag
)
SELECT 
    brand,
    jsonb_object_agg(tag, products_with_tag) AS tag_counts
FROM tag_summary
GROUP BY brand;
```

### ตัวอย่างที่ 24: JSON Diff / Change Tracking

```sql
-- ติดตามการเปลี่ยนแปลง JSON
CREATE TABLE product_history (
    history_id  SERIAL PRIMARY KEY,
    product_id  INT,
    changed_at  TIMESTAMP DEFAULT NOW(),
    old_data    JSONB,
    new_data    JSONB,
    changed_by  VARCHAR(100)
);

-- Function สำหรับ calculate JSON diff
CREATE OR REPLACE FUNCTION jsonb_diff(old_json JSONB, new_json JSONB)
RETURNS JSONB AS $$
DECLARE
    result JSONB = '{}';
    key TEXT;
BEGIN
    FOR key IN SELECT jsonb_object_keys(new_json) LOOP
        IF (old_json ->> key) IS DISTINCT FROM (new_json ->> key) THEN
            result = result || jsonb_build_object(key, jsonb_build_object(
                'old', old_json -> key,
                'new', new_json -> key
            ));
        END IF;
    END LOOP;
    RETURN result;
END;
$$ LANGUAGE plpgsql;

-- ใช้ trigger เพื่อ track changes
CREATE OR REPLACE FUNCTION track_product_changes()
RETURNS TRIGGER AS $$
BEGIN
    INSERT INTO product_history (product_id, old_data, new_data, changed_by)
    VALUES (
        NEW.product_id,
        row_to_json(OLD)::JSONB,
        row_to_json(NEW)::JSONB,
        CURRENT_USER
    );
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER product_audit
AFTER UPDATE ON products
FOR EACH ROW EXECUTE FUNCTION track_product_changes();
```

### ตัวอย่างที่ 25: Complex JSON Aggregation

```sql
-- สร้าง API response format
SELECT jsonb_build_object(
    'status', 'success',
    'data', jsonb_build_object(
        'categories', (
            SELECT jsonb_agg(cat_summary)
            FROM (
                SELECT jsonb_build_object(
                    'name', category,
                    'count', COUNT(*),
                    'total_value', SUM(price),
                    'avg_price', ROUND(AVG(price), 2),
                    'brands', (
                        SELECT jsonb_agg(DISTINCT brand)
                        FROM (
                            SELECT metadata ->> 'brand' AS brand
                            FROM products p2
                            WHERE p2.category = p1.category
                        ) brands
                    )
                ) AS cat_summary
                FROM products p1
                GROUP BY category
            ) summaries
        ),
        'total_products', (SELECT COUNT(*) FROM products),
        'generated_at', NOW()::TEXT
    )
) AS api_response;
```

### ตัวอย่างที่ 26: JSON Search

```sql
-- ค้นหาสินค้าที่มี attribute ตรงกับ criteria หลายข้อ
SELECT 
    product_id,
    name,
    price,
    attributes
FROM products
WHERE 
    attributes @> '{"wireless": true}'   -- มี wireless
    AND (attributes ->> 'battery_hours')::INT >= 20  -- battery >= 20 ชั่วโมง
    AND price < 15000;

-- Full text search ใน JSON values
SELECT name
FROM products
WHERE to_tsvector('english', attributes::TEXT) @@ to_tsquery('wireless & battery');
```

### ตัวอย่างที่ 27: JSON Pivot

```sql
-- เปลี่ยน JSON key-value เป็น columns
WITH product_attrs AS (
    SELECT 
        product_id,
        name,
        attr.key,
        attr.value
    FROM products,
         jsonb_each_text(attributes) AS attr(key, value)
)
SELECT 
    product_id,
    name,
    MAX(CASE WHEN key = 'ram' THEN value END) AS ram,
    MAX(CASE WHEN key = 'storage' THEN value END) AS storage,
    MAX(CASE WHEN key = 'wireless' THEN value END) AS wireless,
    MAX(CASE WHEN key = 'battery_hours' THEN value END) AS battery
FROM product_attrs
GROUP BY product_id, name;
```

---

## 97.14 JSON Performance

### ตัวอย่างที่ 28: Partial Index บน JSON

```sql
-- Index เฉพาะ products ที่เป็น wireless
CREATE INDEX idx_wireless_products 
ON products ((attributes ->> 'battery_hours')::INT)
WHERE attributes @> '{"wireless": true}';

-- ใช้ index นี้
SELECT name, price
FROM products
WHERE attributes @> '{"wireless": true}'
  AND (attributes ->> 'battery_hours')::INT > 20;
```

### ตัวอย่างที่ 29: JSON Query Optimization

```sql
-- ไม่ดี: function บน JSON ทุกแถว
SELECT * FROM products
WHERE LOWER(attributes ->> 'color') = 'silver';

-- ดีกว่า: สร้าง functional index
CREATE INDEX idx_color_lower ON products (LOWER(attributes ->> 'color'));
SELECT * FROM products WHERE LOWER(attributes ->> 'color') = 'silver';

-- ดีที่สุด: extracted column (PostgreSQL 12+)
ALTER TABLE products ADD COLUMN color_extracted TEXT 
GENERATED ALWAYS AS (attributes ->> 'color') STORED;
CREATE INDEX idx_color_extracted ON products (color_extracted);
```

### ตัวอย่างที่ 30: JSON ใน Window Functions

```sql
-- ใช้ JSON ร่วมกับ Window Functions
WITH product_ranked AS (
    SELECT 
        product_id,
        name,
        category,
        price,
        metadata ->> 'brand' AS brand,
        RANK() OVER (PARTITION BY category ORDER BY price DESC) AS price_rank
    FROM products
)
SELECT jsonb_agg(
    jsonb_build_object(
        'rank', price_rank,
        'name', name,
        'brand', brand,
        'price', price
    ) ORDER BY price_rank
) AS top_products_by_category
FROM product_ranked
WHERE price_rank <= 2
GROUP BY category;
```

---

## 97.15 ตัวอย่างเพิ่มเติม

### ตัวอย่างที่ 31: Log Analysis ด้วย JSON

```sql
CREATE TABLE application_logs (
    log_id      SERIAL PRIMARY KEY,
    log_time    TIMESTAMP DEFAULT NOW(),
    log_level   VARCHAR(10),
    log_data    JSONB
);

INSERT INTO application_logs (log_level, log_data) VALUES
('ERROR', '{"message": "Connection timeout", "service": "database", "retry_count": 3, "user_id": 1001}'),
('INFO',  '{"message": "User logged in", "user_id": 1002, "ip": "192.168.1.1", "session_id": "abc123"}'),
('WARN',  '{"message": "Slow query detected", "duration_ms": 5234, "query": "SELECT * FROM orders"}'),
('ERROR', '{"message": "Payment failed", "order_id": 5001, "amount": 15000, "gateway": "stripe"}');

-- วิเคราะห์ logs
SELECT 
    log_level,
    COUNT(*) AS count,
    jsonb_agg(log_data ->> 'message') AS messages
FROM application_logs
GROUP BY log_level;

-- หา errors ที่มี retry
SELECT 
    log_time,
    log_data ->> 'message' AS message,
    log_data ->> 'service' AS service,
    (log_data ->> 'retry_count')::INT AS retries
FROM application_logs
WHERE log_level = 'ERROR'
  AND (log_data ->> 'retry_count')::INT > 0;
```

### ตัวอย่างที่ 32: E-commerce Order Processing

```sql
CREATE TABLE ecommerce_orders (
    order_id    SERIAL PRIMARY KEY,
    customer    JSONB,      -- {name, email, phone}
    items       JSONB,      -- [{product_id, name, qty, price}]
    shipping    JSONB,      -- {method, address, estimated_days}
    payment     JSONB,      -- {method, status, transaction_id}
    created_at  TIMESTAMP DEFAULT NOW()
);

INSERT INTO ecommerce_orders (customer, items, shipping, payment) VALUES
(
    '{"name": "สมชาย ใจดี", "email": "somchai@example.com", "phone": "081-234-5678"}',
    '[{"product_id": 1, "name": "MacBook Pro", "qty": 1, "price": 89900}, {"product_id": 5, "name": "Mouse", "qty": 2, "price": 3900}]',
    '{"method": "express", "address": "123 ถ.สุขุมวิท", "city": "กรุงเทพ", "estimated_days": 2}',
    '{"method": "credit_card", "status": "approved", "transaction_id": "TXN001"}'
);

-- คำนวณยอดรวม
SELECT 
    order_id,
    customer ->> 'name' AS customer_name,
    (SELECT SUM((item ->> 'qty')::INT * (item ->> 'price')::DECIMAL)
     FROM jsonb_array_elements(items) AS item) AS total_amount,
    shipping ->> 'city' AS delivery_city,
    payment ->> 'status' AS payment_status
FROM ecommerce_orders;

-- หา orders ที่มีสินค้า product_id = 1
SELECT order_id, customer ->> 'name' AS customer
FROM ecommerce_orders
WHERE items @> '[{"product_id": 1}]';
```

### ตัวอย่างที่ 33: Configuration Management

```sql
-- Feature flags และ configuration
CREATE TABLE feature_flags (
    flag_id     SERIAL PRIMARY KEY,
    flag_name   VARCHAR(100) UNIQUE,
    config      JSONB,
    is_enabled  BOOLEAN DEFAULT false
);

INSERT INTO feature_flags (flag_name, config, is_enabled) VALUES
('new_checkout', 
 '{"rollout_percentage": 50, "excluded_countries": ["KH", "MM"], "min_user_age_days": 30}',
 true),
('dark_mode',
 '{"default": false, "auto_schedule": {"from": "18:00", "to": "06:00"}}',
 true),
('beta_features',
 '{"allowed_users": [1001, 1002, 1003], "expiry": "2024-12-31"}',
 false);

-- ดึง config สำหรับ feature ที่ enabled
SELECT 
    flag_name,
    config ->> 'rollout_percentage' AS rollout_pct,
    jsonb_array_length(config -> 'excluded_countries') AS excluded_count
FROM feature_flags
WHERE is_enabled = true
  AND config ? 'rollout_percentage';
```

### ตัวอย่างที่ 34: Survey Responses

```sql
CREATE TABLE survey_responses (
    response_id SERIAL PRIMARY KEY,
    user_id     INT,
    survey_id   INT,
    answers     JSONB,      -- {"q1": "answer1", "q2": 5, "q3": ["option1", "option3"]}
    submitted_at TIMESTAMP DEFAULT NOW()
);

INSERT INTO survey_responses (user_id, survey_id, answers) VALUES
(101, 1, '{"q1": "Very satisfied", "q2": 5, "q3": ["price", "quality"], "comments": "Great product!"}'),
(102, 1, '{"q1": "Satisfied", "q2": 4, "q3": ["quality"], "comments": "Good but expensive"}'),
(103, 1, '{"q1": "Neutral", "q2": 3, "q3": ["delivery", "price"], "comments": null}');

-- วิเคราะห์คำตอบ
SELECT 
    survey_id,
    COUNT(*) AS responses,
    ROUND(AVG((answers ->> 'q2')::INT), 2) AS avg_score,
    COUNT(*) FILTER (WHERE (answers ->> 'q2')::INT >= 4) AS satisfied,
    jsonb_object_agg(user_id, answers -> 'q3') AS all_selections
FROM survey_responses
GROUP BY survey_id;

-- ดูว่า option ไหนถูกเลือกบ่อยสุด
SELECT 
    option,
    COUNT(*) AS frequency
FROM survey_responses,
     jsonb_array_elements_text(answers -> 'q3') AS option
GROUP BY option
ORDER BY frequency DESC;
```

### ตัวอย่างที่ 35: Multi-language Content

```sql
CREATE TABLE articles (
    article_id  SERIAL PRIMARY KEY,
    slug        VARCHAR(200) UNIQUE,
    content     JSONB,      -- {"th": {...}, "en": {...}, "zh": {...}}
    published_at TIMESTAMP
);

INSERT INTO articles (slug, content, published_at) VALUES
('intro-sql', 
 '{"th": {"title": "บทนำ SQL", "body": "SQL คือภาษาสำหรับ..."},
   "en": {"title": "Introduction to SQL", "body": "SQL is a language for..."},
   "zh": {"title": "SQL简介", "body": "SQL是一种用于..."}}',
 NOW());

-- ดึงเนื้อหาตามภาษา
SELECT 
    article_id,
    slug,
    content -> 'th' ->> 'title' AS title_th,
    content -> 'en' ->> 'title' AS title_en
FROM articles;

-- Function สำหรับดึงตามภาษาที่ต้องการ
CREATE OR REPLACE FUNCTION get_article_content(
    p_article_id INT,
    p_language VARCHAR DEFAULT 'en'
)
RETURNS JSONB AS $$
BEGIN
    RETURN (SELECT content -> p_language FROM articles WHERE article_id = p_article_id);
END;
$$ LANGUAGE plpgsql;
```

### ตัวอย่างที่ 36: API Response Caching

```sql
-- เก็บ API responses ใน database
CREATE TABLE api_cache (
    cache_key   TEXT PRIMARY KEY,
    response    JSONB,
    created_at  TIMESTAMP DEFAULT NOW(),
    expires_at  TIMESTAMP,
    hit_count   INT DEFAULT 0
);

-- Store response
INSERT INTO api_cache (cache_key, response, expires_at)
VALUES (
    'weather:bangkok:2024-01-01',
    '{"temperature": 32, "humidity": 75, "condition": "partly_cloudy", "forecast": [{"day": "tomorrow", "high": 34, "low": 28}]}',
    NOW() + INTERVAL '1 hour'
);

-- Get valid cache
SELECT response
FROM api_cache
WHERE cache_key = 'weather:bangkok:2024-01-01'
  AND expires_at > NOW();

-- Update hit count
UPDATE api_cache
SET hit_count = hit_count + 1
WHERE cache_key = 'weather:bangkok:2024-01-01';

-- Cleanup expired cache
DELETE FROM api_cache WHERE expires_at < NOW();
```

### ตัวอย่างที่ 37: Audit Trail

```sql
-- Audit trail ที่ยืดหยุ่น
CREATE TABLE audit_log (
    audit_id    SERIAL PRIMARY KEY,
    table_name  VARCHAR(100),
    record_id   INT,
    action      VARCHAR(10),   -- INSERT, UPDATE, DELETE
    changes     JSONB,         -- {"field": {"old": x, "new": y}}
    performed_by VARCHAR(100),
    performed_at TIMESTAMP DEFAULT NOW()
);

-- Trigger สำหรับ audit
CREATE OR REPLACE FUNCTION audit_changes()
RETURNS TRIGGER AS $$
DECLARE
    changes JSONB = '{}';
    col_name TEXT;
    old_val TEXT;
    new_val TEXT;
BEGIN
    IF TG_OP = 'UPDATE' THEN
        -- Record แค่ fields ที่เปลี่ยน
        FOR col_name IN SELECT column_name FROM information_schema.columns WHERE table_name = TG_TABLE_NAME LOOP
            EXECUTE format('SELECT ($1).%I::TEXT', col_name) INTO old_val USING OLD;
            EXECUTE format('SELECT ($1).%I::TEXT', col_name) INTO new_val USING NEW;
            IF old_val IS DISTINCT FROM new_val THEN
                changes = changes || jsonb_build_object(
                    col_name, jsonb_build_object('old', old_val, 'new', new_val)
                );
            END IF;
        END LOOP;
    END IF;
    
    INSERT INTO audit_log (table_name, record_id, action, changes, performed_by)
    VALUES (TG_TABLE_NAME, NEW.product_id, TG_OP, changes, CURRENT_USER);
    
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;
```

### ตัวอย่างที่ 38: GeoJSON Storage

```sql
-- เก็บ GeoJSON สำหรับตำแหน่งร้าน
CREATE TABLE store_locations (
    store_id    INT PRIMARY KEY,
    store_name  VARCHAR(200),
    location    JSONB   -- GeoJSON Point
);

INSERT INTO store_locations VALUES
(1, 'สาขาสยาม', 
 '{"type": "Point", "coordinates": [100.5330, 13.7466], "properties": {"district": "Pathum Wan"}}'),
(2, 'สาขาอารีย์',
 '{"type": "Point", "coordinates": [100.5545, 13.7750], "properties": {"district": "Phaya Thai"}}');

-- ดึงข้อมูลตำแหน่ง
SELECT 
    store_name,
    location -> 'coordinates' -> 0 AS longitude,
    location -> 'coordinates' -> 1 AS latitude,
    location -> 'properties' ->> 'district' AS district
FROM store_locations;
```

### ตัวอย่างที่ 39: Dynamic Form Builder

```sql
-- เก็บ form schema และ responses
CREATE TABLE dynamic_forms (
    form_id     SERIAL PRIMARY KEY,
    form_name   VARCHAR(200),
    schema      JSONB   -- form field definitions
);

CREATE TABLE form_submissions (
    submission_id SERIAL PRIMARY KEY,
    form_id     INT REFERENCES dynamic_forms(form_id),
    data        JSONB,   -- user input
    submitted_at TIMESTAMP DEFAULT NOW()
);

INSERT INTO dynamic_forms (form_name, schema) VALUES
('Job Application', 
 '{"fields": [
    {"name": "full_name", "type": "text", "required": true},
    {"name": "experience_years", "type": "number", "min": 0},
    {"name": "skills", "type": "multiselect", "options": ["Python", "SQL", "Java", "React"]}
 ]}');

-- Validate submission against schema
SELECT 
    f.form_name,
    fs.submission_id,
    -- ตรวจ required fields
    (
        SELECT bool_and(
            fs.data ? (field ->> 'name')
        )
        FROM jsonb_array_elements(f.schema -> 'fields') AS field
        WHERE (field ->> 'required')::BOOLEAN = true
    ) AS all_required_filled
FROM dynamic_forms f
JOIN form_submissions fs ON f.form_id = fs.form_id;
```

### ตัวอย่างที่ 40: JSON Performance Benchmark

```sql
-- เปรียบเทียบ json vs jsonb performance
CREATE TABLE perf_test_json  (id SERIAL, data JSON);
CREATE TABLE perf_test_jsonb (id SERIAL, data JSONB);

-- Insert test data
INSERT INTO perf_test_json (data)
SELECT ('{"key' || n || '": "value' || n || '", "nested": {"inner": ' || n || '}}')::JSON
FROM generate_series(1, 10000) n;

INSERT INTO perf_test_jsonb (data)
SELECT ('{"key' || n || '": "value' || n || '", "nested": {"inner": ' || n || '}}')::JSONB
FROM generate_series(1, 10000) n;

-- jsonb เร็วกว่าในการ query (แต่ insert ช้ากว่าเล็กน้อย)
EXPLAIN ANALYZE
SELECT * FROM perf_test_jsonb WHERE data @> '{"nested": {"inner": 5000}}';
```

---

## แบบฝึกหัดท้ายบท

### แบบฝึกหัดที่ 1
เขียน query ดึงสินค้าที่ wireless และ battery >= 20 ชั่วโมง พร้อมแสดง brand

**คำตอบ:**
```sql
SELECT 
    product_id,
    name,
    price,
    metadata ->> 'brand' AS brand,
    attributes ->> 'battery_hours' AS battery_hours
FROM products
WHERE attributes @> '{"wireless": true}'
  AND (attributes ->> 'battery_hours')::INT >= 20
ORDER BY price;
```

### แบบฝึกหัดที่ 2
สร้าง tag cloud: แสดงแต่ละ tag และจำนวนสินค้าที่มี tag นั้น

**คำตอบ:**
```sql
SELECT 
    tag,
    COUNT(*) AS product_count,
    STRING_AGG(name, ', ' ORDER BY name) AS products
FROM products,
     jsonb_array_elements_text(metadata -> 'tags') AS tag
GROUP BY tag
ORDER BY product_count DESC, tag;
```

### แบบฝึกหัดที่ 3
อัพเดต JSON attributes ของสินค้าทุกตัวในหมวด Laptop: เพิ่ม field "os" = "macOS" ถ้าเป็น Apple

**คำตอบ:**
```sql
UPDATE products
SET attributes = jsonb_set(attributes, '{os}', '"macOS"', true)
WHERE category = 'Laptop'
  AND metadata ->> 'brand' = 'Apple';

SELECT name, attributes -> 'os' AS os FROM products WHERE category = 'Laptop';
```

### แบบฝึกหัดที่ 4
สร้าง JSON report สรุปสินค้าแต่ละหมวด พร้อม top 2 สินค้าราคาแพงสุด

**คำตอบ:**
```sql
WITH ranked AS (
    SELECT 
        product_id, name, category, price,
        metadata ->> 'brand' AS brand,
        RANK() OVER (PARTITION BY category ORDER BY price DESC) AS rnk
    FROM products
)
SELECT 
    category,
    jsonb_build_object(
        'category', category,
        'top_products', jsonb_agg(
            jsonb_build_object('name', name, 'brand', brand, 'price', price)
            ORDER BY rnk
        )
    ) AS category_summary
FROM ranked
WHERE rnk <= 2
GROUP BY category;
```

### แบบฝึกหัดที่ 5
เขียน query แสดง pivot: แต่ละ product เป็น row, แต่ละ attribute เป็น column

**คำตอบ:**
```sql
WITH attrs AS (
    SELECT 
        product_id,
        name,
        attr_key,
        attr_value
    FROM products,
         jsonb_each_text(attributes) AS a(attr_key, attr_value)
)
SELECT 
    product_id,
    name,
    MAX(CASE WHEN attr_key = 'ram' THEN attr_value END) AS ram,
    MAX(CASE WHEN attr_key = 'storage' THEN attr_value END) AS storage,
    MAX(CASE WHEN attr_key = 'wireless' THEN attr_value END) AS wireless,
    MAX(CASE WHEN attr_key = 'battery_hours' THEN attr_value END) AS battery_hours,
    MAX(CASE WHEN attr_key = 'color' THEN attr_value END) AS color
FROM attrs
GROUP BY product_id, name
ORDER BY product_id;
```

### แบบฝึกหัดที่ 6
Query ดึง log entries ที่มี user_id และ duration_ms สูงกว่า 3000

**คำตอบ:**
```sql
SELECT 
    log_id,
    log_time,
    log_level,
    log_data ->> 'message' AS message,
    (log_data ->> 'user_id')::INT AS user_id,
    (log_data ->> 'duration_ms')::INT AS duration_ms
FROM application_logs
WHERE (log_data ? 'user_id' OR log_data ? 'duration_ms')
  AND (
    (log_data ->> 'user_id') IS NOT NULL OR
    (log_data ->> 'duration_ms')::INT > 3000
  )
ORDER BY log_time DESC;
```

### แบบฝึกหัดที่ 7
คำนวณยอดรวมของแต่ละ order จาก JSONB items array

**คำตอบ:**
```sql
SELECT 
    order_id,
    customer ->> 'name' AS customer_name,
    (
        SELECT SUM((item ->> 'qty')::INT * (item ->> 'price')::DECIMAL)
        FROM jsonb_array_elements(items) AS item
    ) AS order_total,
    jsonb_array_length(items) AS item_count,
    payment ->> 'status' AS payment_status
FROM ecommerce_orders
ORDER BY order_total DESC;
```

### แบบฝึกหัดที่ 8
สร้าง function ที่รับ language code และ return content ในภาษานั้น

**คำตอบ:**
```sql
CREATE OR REPLACE FUNCTION get_localized_content(
    p_article_id INT,
    p_lang VARCHAR DEFAULT 'en',
    p_fallback_lang VARCHAR DEFAULT 'en'
)
RETURNS TABLE(title TEXT, body TEXT) AS $$
BEGIN
    RETURN QUERY
    SELECT 
        COALESCE(
            content -> p_lang ->> 'title',
            content -> p_fallback_lang ->> 'title'
        ) AS title,
        COALESCE(
            content -> p_lang ->> 'body',
            content -> p_fallback_lang ->> 'body'
        ) AS body
    FROM articles
    WHERE article_id = p_article_id;
END;
$$ LANGUAGE plpgsql;

SELECT * FROM get_localized_content(1, 'th');
```

### แบบฝึกหัดที่ 9
หา feature flags ที่ยังไม่หมดอายุและ rollout >= 50%

**คำตอบ:**
```sql
SELECT 
    flag_name,
    (config ->> 'rollout_percentage')::INT AS rollout_pct,
    config ->> 'expiry' AS expiry,
    is_enabled
FROM feature_flags
WHERE is_enabled = true
  AND config ? 'rollout_percentage'
  AND (config ->> 'rollout_percentage')::INT >= 50
  AND (
    NOT config ? 'expiry' 
    OR (config ->> 'expiry')::DATE >= CURRENT_DATE
  )
ORDER BY rollout_pct DESC;
```

### แบบฝึกหัดที่ 10
สร้าง comprehensive product catalog API response ใน JSON format

**คำตอบ:**
```sql
SELECT jsonb_build_object(
    'catalog', jsonb_build_object(
        'generated_at', NOW(),
        'total_products', COUNT(*),
        'categories', jsonb_agg(
            DISTINCT jsonb_build_object(
                'name', category,
                'product_count', COUNT(*) OVER (PARTITION BY category)
            )
        ),
        'price_range', jsonb_build_object(
            'min', MIN(price),
            'max', MAX(price),
            'avg', ROUND(AVG(price), 2)
        ),
        'brands', jsonb_agg(DISTINCT metadata ->> 'brand'),
        'all_tags', (
            SELECT jsonb_agg(DISTINCT tag)
            FROM products p2,
                 jsonb_array_elements_text(p2.metadata -> 'tags') tag
        )
    )
) AS catalog_summary
FROM products;
```

---

## สรุปบทที่ 97

### JSON ใน SQL: สรุปเปรียบเทียบ

| Database | JSON Type | Key Operator | Path Query |
|----------|-----------|--------------|------------|
| PostgreSQL | json, jsonb | ->, ->>, #>, #>> | jsonpath |
| MySQL | JSON | ->, ->>, JSON_EXTRACT | JSONPath |
| SQLite | TEXT (json1) | json_extract() | json_each() |
| SQL Server | NVARCHAR | JSON_VALUE(), JSON_QUERY() | FOR JSON |

### Best Practices
1. ใช้ **jsonb** (ไม่ใช่ json) ใน PostgreSQL เสมอ ยกเว้นต้องการ preserve exact text
2. สร้าง **GIN index** สำหรับ containment queries (@>)
3. สร้าง **functional index** สำหรับ specific path ที่ query บ่อย
4. ใช้ **extracted column** (Generated Column) สำหรับ field ที่ query บ่อยมาก
5. ไม่ควรใช้ JSON สำหรับ fields ที่ต้อง JOIN หรือ aggregate บ่อยๆ

ในบทถัดไปเราจะเรียนรู้เกี่ยวกับ Full-Text Search ใน SQL ซึ่งช่วยให้ค้นหาข้อความได้อย่างทรงพลังและ semantic
