# Part 063: Index Types Deep Dive

## เจาะลึกประเภทของ Index

---

## บทนำ

ฐานข้อมูลสมัยใหม่มี Index หลายประเภทให้เลือกใช้ตามลักษณะข้อมูลและ query pattern แต่ละประเภทมีจุดเด่นที่แตกต่างกัน การเลือกใช้ index ประเภทที่เหมาะสมสามารถเพิ่มประสิทธิภาพได้มหาศาล

---

## 1. B-tree Index (Default)

B-tree (Balanced Tree) คือ index ประเภท default ที่ใช้ได้กับงานทั่วไปส่วนใหญ่

### เมื่อใดควรใช้ B-tree:
- Equality comparisons: `=`, `!=`
- Range comparisons: `<`, `>`, `<=`, `>=`, `BETWEEN`
- Pattern matching ที่ขึ้นต้นด้วย constant: `LIKE 'prefix%'`
- `IS NULL`, `IS NOT NULL`
- `ORDER BY` และ `GROUP BY`
- `IN` กับค่าไม่มาก

```sql
-- B-tree สำหรับ equality
CREATE INDEX idx_emp_dept ON employees(department_id);
SELECT * FROM employees WHERE department_id = 5;  -- ใช้ index

-- B-tree สำหรับ range
CREATE INDEX idx_orders_date ON orders(order_date);
SELECT * FROM orders WHERE order_date BETWEEN '2024-01-01' AND '2024-12-31';

-- B-tree สำหรับ prefix LIKE
CREATE INDEX idx_products_name ON products(name);
SELECT * FROM products WHERE name LIKE 'iPhone%';  -- ใช้ index ✓
SELECT * FROM products WHERE name LIKE '%Phone%';   -- ไม่ใช้ index ✗

-- B-tree สำหรับ ORDER BY
CREATE INDEX idx_emp_salary_desc ON employees(salary DESC);
SELECT * FROM employees ORDER BY salary DESC LIMIT 10;  -- Index scan!
```

### B-tree: ตัวอย่างเชิงลึก

```sql
-- ตรวจสอบว่า query ใช้ B-tree index
EXPLAIN SELECT * FROM employees WHERE salary > 50000;
/*
Index Scan using idx_emp_salary_desc on employees
  (cost=0.43..1234.56 rows=5000 width=150)
  Index Cond: (salary > 50000)
*/
```

---

## 2. Hash Index

Hash Index ใช้ hash function เพื่อ map ค่าไปยัง bucket ทำให้ค้นหาด้วย equality ได้เร็วมาก แต่ไม่รองรับ range queries

```
Hash Index Structure:
ค่า "สมชาย" → hash("สมชาย") = 7291 → bucket 7291 → row pointer
ค่า "สมหญิง" → hash("สมหญิง") = 3847 → bucket 3847 → row pointer

เร็วมากสำหรับ = แต่ไม่รองรับ <, >, BETWEEN, LIKE
```

### เมื่อใดควรใช้ Hash Index:
- **เฉพาะ equality comparisons เท่านั้น** (`=`)
- คอลัมน์ที่มี high cardinality
- ไม่ต้องการ range queries

```sql
-- PostgreSQL: สร้าง Hash Index
CREATE INDEX idx_sessions_token ON sessions USING HASH(session_token);

-- ใช้ได้:
SELECT * FROM sessions WHERE session_token = 'abc123xyz';  -- ✓

-- ใช้ไม่ได้:
SELECT * FROM sessions WHERE session_token > 'abc';  -- ✗ Hash ไม่รองรับ range
SELECT * FROM sessions WHERE session_token LIKE 'abc%';  -- ✗
ORDER BY session_token  -- ✗
```

### ข้อจำกัด Hash Index

```sql
-- PostgreSQL: Hash index ก่อน version 10 ไม่ crash-safe!
-- ตั้งแต่ PostgreSQL 10+ Hash indexes เป็น WAL-logged
-- MySQL/InnoDB: ไม่รองรับ explicit hash indexes (มีแค่ adaptive hash index ภายใน)

-- เปรียบเทียบ B-tree vs Hash (PostgreSQL)
-- B-tree:
CREATE INDEX idx_b ON sessions(session_token);
-- Hash:
CREATE INDEX idx_h ON sessions USING HASH(session_token);

-- Hash อาจเร็วกว่า B-tree เล็กน้อยสำหรับ pure equality
-- แต่ B-tree ยืดหยุ่นกว่ามาก
```

---

## 3. GIN Index (PostgreSQL) - สำหรับ JSON/Arrays/Full-text

GIN (Generalized Inverted Index) เหมาะสำหรับข้อมูลที่มีหลาย "elements" ต่อค่า เช่น arrays, JSON, tsvector (full-text)

```
GIN สร้าง "inverted index":
Document 1: ["sql", "database", "performance"]
Document 2: ["sql", "index", "btree"]
Document 3: ["database", "optimization"]

GIN Index:
"sql"         → [doc1, doc2]
"database"    → [doc1, doc3]
"performance" → [doc1]
"index"       → [doc2]
"btree"       → [doc2]
"optimization"→ [doc3]

Query: ค้นหา documents ที่มีคำว่า "sql"
→ GIN ค้นหาเฉพาะ entry "sql" → ได้ [doc1, doc2] ทันที!
```

### GIN สำหรับ Arrays

```sql
-- ตาราง articles พร้อม tags array
CREATE TABLE articles (
    article_id SERIAL PRIMARY KEY,
    title TEXT,
    tags TEXT[],  -- array ของ tags
    published_at TIMESTAMP
);

-- สร้าง GIN index บน tags array
CREATE INDEX idx_articles_tags ON articles USING GIN(tags);

-- Query ที่ใช้ GIN index:
-- ค้นหาบทความที่มี tag 'postgresql'
SELECT * FROM articles WHERE tags @> ARRAY['postgresql'];

-- ค้นหาบทความที่มี tag 'sql' หรือ 'database'
SELECT * FROM articles WHERE tags && ARRAY['sql', 'database'];

-- ตรวจสอบว่าใช้ index:
EXPLAIN SELECT * FROM articles WHERE tags @> ARRAY['postgresql'];
/*
Bitmap Heap Scan on articles
  Recheck Cond: (tags @> '{postgresql}'::text[])
  -> Bitmap Index Scan on idx_articles_tags
       Index Cond: (tags @> '{postgresql}'::text[])
*/
```

### GIN สำหรับ JSONB

```sql
-- ตาราง events พร้อม JSONB data
CREATE TABLE events (
    event_id SERIAL PRIMARY KEY,
    event_type VARCHAR(50),
    event_data JSONB,
    created_at TIMESTAMP
);

-- สร้าง GIN index บน JSONB
CREATE INDEX idx_events_data ON events USING GIN(event_data);

-- หรือ GIN index บน path เฉพาะ (ประหยัดพื้นที่กว่า)
CREATE INDEX idx_events_user_id ON events USING GIN((event_data->'user_id'));

-- Queries ที่ใช้ GIN:
-- ค้นหา events ที่มี key 'user_id' ใน JSON
SELECT * FROM events WHERE event_data ? 'user_id';

-- ค้นหาด้วยค่าใน JSON
SELECT * FROM events WHERE event_data @> '{"status": "active"}';

-- ค้นหา nested value
SELECT * FROM events WHERE event_data -> 'user' ->> 'country' = 'TH';
```

### GIN สำหรับ Full-text Search

```sql
-- ตาราง products พร้อม full-text search
CREATE TABLE products (
    product_id SERIAL PRIMARY KEY,
    name TEXT,
    description TEXT,
    search_vector TSVECTOR  -- precomputed search vector
);

-- สร้าง search vector
UPDATE products 
SET search_vector = to_tsvector('english', name || ' ' || COALESCE(description, ''));

-- สร้าง GIN index บน tsvector
CREATE INDEX idx_products_search ON products USING GIN(search_vector);

-- หรือ สร้าง index โดยตรงบน expression
CREATE INDEX idx_products_fts ON products 
USING GIN(to_tsvector('english', name || ' ' || COALESCE(description, '')));

-- Full-text query
SELECT * FROM products 
WHERE search_vector @@ to_tsquery('english', 'laptop & gaming');

-- หรือ
SELECT * FROM products 
WHERE to_tsvector('english', name) @@ plainto_tsquery('english', 'gaming laptop');
```

---

## 4. GiST Index (PostgreSQL) - สำหรับ Geometric/Range Data

GiST (Generalized Search Tree) เป็น framework สำหรับ index ที่รองรับข้อมูลหลายมิติ เช่น geometric data, range types

```sql
-- GiST สำหรับ Range Types
CREATE TABLE reservations (
    reservation_id SERIAL PRIMARY KEY,
    room_id INT,
    guest_name TEXT,
    stay_period DATERANGE  -- range type
);

-- สร้าง GiST index บน daterange
CREATE INDEX idx_reservations_period ON reservations USING GIST(stay_period);

-- Query ที่ใช้ GiST:
-- ห้องที่มีการจองในช่วงวันที่นี้
SELECT * FROM reservations 
WHERE stay_period && daterange('2024-12-20', '2024-12-25');

-- ห้องที่ว่างช่วง Christmas
SELECT room_id FROM rooms
WHERE room_id NOT IN (
    SELECT room_id FROM reservations
    WHERE stay_period @> '2024-12-25'::date
);
```

```sql
-- GiST สำหรับ Geometric Data
CREATE TABLE stores (
    store_id SERIAL PRIMARY KEY,
    store_name TEXT,
    location POINT  -- geometric point (x, y)
);

-- สร้าง GiST index บน location
CREATE INDEX idx_stores_location ON stores USING GIST(location);

-- Query: ร้านค้าที่อยู่ใกล้จุด (13.7563, 100.5018) ในรัศมี 5km
SELECT store_id, store_name, location
FROM stores
WHERE location <-> point(13.7563, 100.5018) < 0.045;  -- ~5km in degrees
-- หรือใช้ PostGIS สำหรับ geography ที่แม่นยำกว่า
```

```sql
-- GiST สำหรับ IP Address Ranges (inet)
CREATE TABLE ip_blocks (
    block_id SERIAL PRIMARY KEY,
    ip_range CIDR,
    country_code CHAR(2)
);

CREATE INDEX idx_ip_blocks_range ON ip_blocks USING GIST(ip_range inet_ops);

-- ค้นหาประเทศของ IP
SELECT country_code FROM ip_blocks
WHERE '192.168.1.50' << ip_range;
```

### GiST vs GIN เปรียบเทียบ

```
┌─────────────────────┬─────────────────────────┬─────────────────────────┐
│ Feature             │ GIN                     │ GiST                    │
├─────────────────────┼─────────────────────────┼─────────────────────────┤
│ สร้าง index         │ ช้า (ต้อง scan ทุก item)│ เร็วกว่า                │
│ ขนาด index          │ ใหญ่กว่า                │ เล็กกว่า                │
│ Query speed         │ เร็วกว่า (exact match)  │ ช้ากว่าเล็กน้อย         │
│ Update performance  │ ช้ากว่า                 │ เร็วกว่า                │
│ Use cases           │ Full-text, arrays, JSONB│ Geometry, ranges, IP    │
│ False positives     │ ไม่มี                   │ มีได้ (recheck needed)  │
└─────────────────────┴─────────────────────────┴─────────────────────────┘

แนะนำ:
- Full-text search → GIN
- Geometric/range → GiST
- JSONB ค้นหา @> operator → GIN
- JSONB ค้นหา distance → GiST
```

---

## 5. BRIN Index (PostgreSQL) - สำหรับ Sequential Data

BRIN (Block Range Index) เก็บเพียงค่า min/max สำหรับแต่ละ block range ของตาราง ประหยัดพื้นที่มากแต่เหมาะเฉพาะกับข้อมูลที่ sequential

```
BRIN Structure:
ตาราง (1M rows, เรียงตาม created_at):

Block 1-128 (rows 1-10240):   min_date=2020-01-01, max_date=2020-02-15
Block 129-256 (rows 10241-20480): min_date=2020-02-15, max_date=2020-03-30
...
Block 8000-8128: min_date=2024-11-01, max_date=2024-12-31

Query: WHERE created_at = '2024-12-15'
→ BRIN ตรวจสอบว่า block ไหนที่ min_date ≤ 2024-12-15 ≤ max_date
→ ข้าม blocks ที่แน่นอนว่าไม่มีข้อมูล
→ ลดการอ่าน disk อย่างมาก!
```

```sql
-- BRIN Index สำหรับ time-series
CREATE TABLE sensor_readings (
    reading_id BIGSERIAL PRIMARY KEY,
    sensor_id INT,
    temperature FLOAT,
    recorded_at TIMESTAMP NOT NULL  -- sequential, monotonically increasing
);

-- BRIN ประหยัดพื้นที่มากเมื่อเทียบกับ B-tree
CREATE INDEX idx_sensor_brin ON sensor_readings USING BRIN(recorded_at);
-- ขนาด: ~คงที่ ไม่ขึ้นกับจำนวนแถว!
-- เทียบกับ B-tree: ขนาดเพิ่มตามจำนวนแถว

-- B-tree ที่เทียบกัน:
CREATE INDEX idx_sensor_btree ON sensor_readings(recorded_at);

-- ขนาดเปรียบเทียบ (1M rows):
-- BRIN:  ~48 KB
-- B-tree: ~25 MB
-- ประหยัดได้: ~500x!
```

```sql
-- ปรับ pages_per_range (default = 128)
-- ยิ่งเล็ก = แม่นยำกว่า แต่ index ใหญ่ขึ้น
CREATE INDEX idx_sensor_brin_fine ON sensor_readings 
USING BRIN(recorded_at) WITH (pages_per_range = 32);

-- เมื่อใดควรใช้ BRIN:
-- ✓ ข้อมูล time-series (logs, events, readings)
-- ✓ ข้อมูลที่ insert ตามลำดับเวลา
-- ✓ ตารางใหญ่มาก (> 100M rows)
-- ✗ ข้อมูลที่ random distribution
-- ✗ ต้องการ high precision lookups
```

---

## 6. Filtered/Partial Index

Partial Index สร้าง index เฉพาะ rows ที่ตรงเงื่อนไข ทำให้ index เล็กลงมากและค้นหาเร็วขึ้น

```sql
-- ตัวอย่าง: ตาราง orders มี 50 ล้านแถว
-- 99% มี status = 'completed' หรือ 'cancelled'
-- 1% มี status = 'pending' (แต่ query บ่อยมาก!)

-- Full index: ครอบคลุมทั้ง 50M แถว (ใหญ่)
CREATE INDEX idx_orders_status_full ON orders(status, created_at);

-- Partial index: ครอบคลุมเฉพาะ pending (500K แถว)
CREATE INDEX idx_orders_pending ON orders(created_at)
WHERE status = 'pending';
-- เล็กกว่า 100x! และค้นหาเร็วกว่ามาก

-- Query ที่ใช้ partial index:
SELECT * FROM orders WHERE status = 'pending' ORDER BY created_at;
-- PostgreSQL จะใช้ idx_orders_pending อัตโนมัติ!
```

```sql
-- Partial Unique Index
-- อนุญาตให้มี NULL แต่ non-NULL ต้องไม่ซ้ำ
CREATE UNIQUE INDEX idx_employees_unique_tax_id 
ON employees(tax_id) 
WHERE tax_id IS NOT NULL;

-- Partial Index สำหรับ soft delete pattern
CREATE INDEX idx_users_active_email ON users(email)
WHERE deleted_at IS NULL;

CREATE INDEX idx_products_active_category ON products(category_id, name)
WHERE is_active = TRUE;

-- Partial Index สำหรับ high-value records
CREATE INDEX idx_orders_large ON orders(customer_id, order_date)
WHERE total_amount > 100000;
```

### เปรียบเทียบ Full vs Partial Index

```sql
-- สถิติ: orders table (50M rows, 1% pending)
-- Full index:    ~2.5GB, query time ~50ms
-- Partial index: ~25MB,  query time ~5ms

-- ประหยัดพื้นที่และเพิ่มความเร็ว 10x!

-- ดู partial index definition
SELECT indexname, indexdef 
FROM pg_indexes 
WHERE tablename = 'orders'
AND indexdef LIKE '%WHERE%';
```

---

## 7. Full-text Indexes

### PostgreSQL Full-text Search

```sql
-- วิธีที่ 1: GIN index บน computed tsvector column
ALTER TABLE articles ADD COLUMN search_vector TSVECTOR;

-- สร้าง trigger เพื่ออัพเดต search_vector อัตโนมัติ
CREATE OR REPLACE FUNCTION update_search_vector()
RETURNS TRIGGER AS $$
BEGIN
    NEW.search_vector := 
        setweight(to_tsvector('english', COALESCE(NEW.title, '')), 'A') ||
        setweight(to_tsvector('english', COALESCE(NEW.body, '')), 'B');
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trigger_update_search_vector
BEFORE INSERT OR UPDATE ON articles
FOR EACH ROW EXECUTE FUNCTION update_search_vector();

-- สร้าง GIN index
CREATE INDEX idx_articles_fts ON articles USING GIN(search_vector);

-- Full-text queries:
-- ค้นหาบทความที่มีคำว่า "database performance"
SELECT title, ts_rank(search_vector, query) AS rank
FROM articles, to_tsquery('english', 'database & performance') query
WHERE search_vector @@ query
ORDER BY rank DESC;
```

```sql
-- วิธีที่ 2: Index บน expression (ไม่ต้องมี column พิเศษ)
CREATE INDEX idx_articles_fts_expr ON articles
USING GIN(to_tsvector('english', title || ' ' || COALESCE(body, '')));

-- ใช้งาน:
SELECT title FROM articles
WHERE to_tsvector('english', title || ' ' || COALESCE(body, ''))
    @@ to_tsquery('english', 'machine & learning');
```

### MySQL Full-text Indexes

```sql
-- MySQL Full-text index
CREATE TABLE articles (
    id INT AUTO_INCREMENT PRIMARY KEY,
    title VARCHAR(500),
    body TEXT,
    FULLTEXT INDEX ft_title_body (title, body)
);

-- หรือ ALTER TABLE
ALTER TABLE articles ADD FULLTEXT INDEX ft_title_body (title, body);

-- Full-text query ใน MySQL
-- Natural language mode (default)
SELECT title, MATCH(title, body) AGAINST('database performance') AS score
FROM articles
WHERE MATCH(title, body) AGAINST('database performance')
ORDER BY score DESC;

-- Boolean mode
SELECT title FROM articles
WHERE MATCH(title, body) AGAINST('+database +performance -slow' IN BOOLEAN MODE);

-- หมายเหตุ: MySQL Full-text ไม่ทำงานกับ InnoDB ก่อน MySQL 5.6
-- MySQL 5.6+ รองรับ Full-text ใน InnoDB
```

---

## 8. Spatial Indexes

### PostGIS (PostgreSQL + Geography Extension)

```sql
-- ติดตั้ง PostGIS
CREATE EXTENSION postgis;

-- ตาราง locations พร้อม geography column
CREATE TABLE restaurants (
    restaurant_id SERIAL PRIMARY KEY,
    name TEXT,
    location GEOGRAPHY(POINT, 4326),  -- WGS84 coordinate system
    cuisine_type TEXT
);

-- สร้าง index บน geography
CREATE INDEX idx_restaurants_location ON restaurants USING GIST(location);

-- Query: ร้านอาหารในรัศมี 1km จากสยามสแควร์ (13.7466, 100.5340)
SELECT name, cuisine_type,
    ST_Distance(location, ST_MakePoint(100.5340, 13.7466)::geography) AS distance_m
FROM restaurants
WHERE ST_DWithin(
    location, 
    ST_MakePoint(100.5340, 13.7466)::geography,
    1000  -- 1000 meters
)
ORDER BY distance_m;
```

---

## 9. MySQL-specific Index Types

### MySQL Covering Index

```sql
-- MySQL Covering Index (index มีข้อมูลทุกคอลัมน์ที่ query ต้องการ)
CREATE TABLE orders (
    order_id INT PRIMARY KEY,
    customer_id INT,
    order_date DATE,
    total_amount DECIMAL(10,2),
    status VARCHAR(20)
);

-- Covering index สำหรับ query นี้:
SELECT customer_id, order_date, total_amount 
FROM orders 
WHERE customer_id = 1001 AND order_date >= '2024-01-01';

-- สร้าง covering index ที่มีทุก columns ที่ query ต้องการ:
CREATE INDEX idx_orders_cover ON orders(customer_id, order_date, total_amount);
-- คอลัมน์ใน SELECT: customer_id, order_date, total_amount → ทั้งหมดอยู่ใน index!
-- MySQL จะใช้ "Using index" แทน "Using index; Using MRR"
```

### MySQL SPATIAL Index

```sql
-- MySQL Spatial index
CREATE TABLE locations (
    id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(100),
    coords POINT NOT NULL,
    SPATIAL INDEX(coords)
);

-- Insert
INSERT INTO locations (name, coords) VALUES 
('Bangkok', ST_GeomFromText('POINT(100.5018 13.7563)'));

-- Query nearby
SELECT name,
    ST_Distance(coords, ST_GeomFromText('POINT(100.5340 13.7466)')) AS distance
FROM locations
WHERE ST_Within(
    coords, 
    ST_Buffer(ST_GeomFromText('POINT(100.5340 13.7466)'), 0.01)
)
ORDER BY distance;
```

### MySQL Invisible Indexes (MySQL 8.0+)

```sql
-- สร้าง index แบบ "invisible" เพื่อทดสอบว่าถ้าไม่มี index จะช้าแค่ไหน
CREATE INDEX idx_orders_date ON orders(order_date);

-- ทดสอบโดยซ่อน index
ALTER TABLE orders ALTER INDEX idx_orders_date INVISIBLE;

-- ตรวจสอบ execution plan (index ถูก ignore)
EXPLAIN SELECT * FROM orders WHERE order_date > '2024-01-01';

-- ทำให้ index กลับมา visible
ALTER TABLE orders ALTER INDEX idx_orders_date VISIBLE;
```

---

## 10. Index Type Comparison Table

```
┌───────────────┬────────────┬──────────────┬─────────────┬────────────────────────────┐
│ Index Type    │ Equality   │ Range        │ Full-text   │ Best For                   │
├───────────────┼────────────┼──────────────┼─────────────┼────────────────────────────┤
│ B-tree        │ ✓ ดีมาก   │ ✓ ดีมาก     │ ✗           │ งานทั่วไป, numbers, dates  │
│ Hash          │ ✓ เร็วสุด │ ✗            │ ✗           │ Exact match เท่านั้น       │
│ GIN           │ ✓ ดี      │ ✗            │ ✓ ดีมาก    │ Arrays, JSONB, tsvector    │
│ GiST          │ ✓ ปานกลาง │ ✓ (ranges)  │ ✓ ปานกลาง  │ Geometry, IP ranges        │
│ BRIN          │ ✗ (approx)│ ✓ (coarse)  │ ✗           │ Sequential/time-series     │
│ Partial       │ ✓          │ ✓            │ ✓           │ Subset ของข้อมูล           │
│ Spatial/GIST  │ ✓          │ ✓            │ ✗           │ Geographic data            │
│ Full-text(MY) │ ✗          │ ✗            │ ✓ ดีมาก    │ Text search ใน MySQL       │
└───────────────┴────────────┴──────────────┴─────────────┴────────────────────────────┘
```

---

## 11. การเลือก Index Type ที่เหมาะสม

### Decision Tree

```
ข้อมูลเป็นอะไร?
│
├── Text ทั่วไป (name, email, code) → B-tree
├── Number/Date → B-tree  
├── ต้องการค้นหาคำใน text (full-text) → GIN (PostgreSQL) / FULLTEXT (MySQL)
├── Arrays หรือ JSONB → GIN
├── Range types (daterange, numrange) → GiST
├── Geographic/Geometric data → GiST + PostGIS
├── Time-series, sequential data → BRIN (หรือ B-tree ถ้าต้องการ precision)
├── เฉพาะ equality บน high-cardinality column → Hash (PostgreSQL)
└── เฉพาะ subset ของข้อมูล → Partial/Filtered index
```

---

## 12. ตัวอย่างการใช้งานจริง: Multi-Index Strategy

```sql
-- ระบบ e-commerce ที่ต้องการ index หลายประเภท

-- Table: products
CREATE TABLE products (
    product_id SERIAL PRIMARY KEY,
    name TEXT NOT NULL,
    description TEXT,
    tags TEXT[],                    -- array ของ tags
    attributes JSONB,               -- flexible attributes
    price DECIMAL(10,2),
    category_id INT,
    search_vector TSVECTOR,         -- full-text search
    location GEOGRAPHY(POINT, 4326),-- สำหรับ local products
    created_at TIMESTAMP DEFAULT NOW()
);

-- 1. B-tree สำหรับ common filters
CREATE INDEX idx_products_category ON products(category_id);
CREATE INDEX idx_products_price ON products(price);
CREATE INDEX idx_products_category_price ON products(category_id, price);

-- 2. GIN สำหรับ tags array
CREATE INDEX idx_products_tags ON products USING GIN(tags);
-- ใช้: WHERE tags @> ARRAY['electronics', 'gaming']

-- 3. GIN สำหรับ JSONB attributes
CREATE INDEX idx_products_attrs ON products USING GIN(attributes);
-- ใช้: WHERE attributes @> '{"brand": "Apple"}'

-- 4. GIN สำหรับ full-text search
CREATE INDEX idx_products_search ON products USING GIN(search_vector);
-- ใช้: WHERE search_vector @@ to_tsquery('english', 'wireless headphone')

-- 5. GiST สำหรับ geographic search
CREATE INDEX idx_products_location ON products USING GIST(location);
-- ใช้: WHERE ST_DWithin(location, $user_location, 5000)

-- 6. Partial index สำหรับ active products
CREATE INDEX idx_products_active_price ON products(category_id, price)
WHERE is_active = TRUE AND stock_quantity > 0;

-- 7. BRIN สำหรับ created_at (time-series)
CREATE INDEX idx_products_created_brin ON products USING BRIN(created_at);
```

---

## แบบฝึกหัด (10 ข้อ)

**ข้อ 1:** ตาราง `recipes` มี columns: recipe_id, name, ingredients TEXT[], cuisine TEXT, rating FLOAT จงเลือก index type ที่เหมาะสมสำหรับแต่ละ query:
- (a) `WHERE ingredients @> ARRAY['chicken', 'garlic']`
- (b) `WHERE cuisine = 'Thai' AND rating > 4.0`
- (c) `WHERE name LIKE 'Tom%'`

**เฉลยข้อ 1:**
```sql
-- (a) GIN สำหรับ array containment
CREATE INDEX idx_recipes_ingredients ON recipes USING GIN(ingredients);

-- (b) B-tree composite
CREATE INDEX idx_recipes_cuisine_rating ON recipes(cuisine, rating);

-- (c) B-tree (prefix LIKE)
CREATE INDEX idx_recipes_name ON recipes(name);
```

---

**ข้อ 2:** ทำไม BRIN จึงเหมาะกับ time-series data มากกว่า B-tree แต่ไม่เหมาะกับ randomly distributed data?

**เฉลยข้อ 2:**
BRIN เก็บ min/max ต่อ block range ถ้าข้อมูลเรียงตามลำดับ (sequential) เช่น timestamp ที่เพิ่มขึ้นเรื่อยๆ แต่ละ block จะมี min/max ที่ไม่ทับซ้อนกัน ทำให้ BRIN สามารถข้าม blocks ที่ไม่เกี่ยวข้องได้มาก แต่ถ้าข้อมูล random แต่ละ block จะมี min=ต่ำสุด max=สูงสุดทุก block → BRIN บอกได้แค่ "อาจมีในทุก block" → ต้องอ่านทุก block → ไม่ต่างจาก full scan!

---

**ข้อ 3:** เขียน SQL เพื่อสร้าง full-text search สำหรับตาราง `news` ที่มี title, summary, body ใน PostgreSQL โดย title มีน้ำหนักสูงกว่า body

**เฉลยข้อ 3:**
```sql
-- เพิ่ม search_vector column
ALTER TABLE news ADD COLUMN search_vector TSVECTOR;

-- สร้าง trigger
CREATE OR REPLACE FUNCTION update_news_search()
RETURNS TRIGGER AS $$
BEGIN
    NEW.search_vector := 
        setweight(to_tsvector('english', COALESCE(NEW.title, '')), 'A') ||
        setweight(to_tsvector('english', COALESCE(NEW.summary, '')), 'B') ||
        setweight(to_tsvector('english', COALESCE(NEW.body, '')), 'C');
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trig_news_search
BEFORE INSERT OR UPDATE ON news
FOR EACH ROW EXECUTE FUNCTION update_news_search();

-- สร้าง GIN index
CREATE INDEX idx_news_search ON news USING GIN(search_vector);
```

---

**ข้อ 4:** อธิบาย use case ที่ Hash Index เหมาะกว่า B-tree พร้อมตัวอย่าง

**เฉลยข้อ 4:**
Hash Index เหมาะกว่าสำหรับ:
- Session tokens: `WHERE session_token = 'abc123'` - always exact match
- API keys: `WHERE api_key = 'key_xyz'`
- MD5/SHA hashes: `WHERE file_hash = '...'`

```sql
CREATE INDEX idx_sessions_hash ON user_sessions USING HASH(session_token);
-- เร็วกว่า B-tree สำหรับ pure equality ประมาณ 20-30%
-- แต่ไม่รองรับ ORDER BY session_token หรือ LIKE 'abc%'
```

---

**ข้อ 5:** สร้าง GiST index สำหรับตาราง bookings ที่มี tstzrange column ชื่อ booking_period

**เฉลยข้อ 5:**
```sql
CREATE TABLE bookings (
    booking_id SERIAL PRIMARY KEY,
    resource_id INT,
    user_id INT,
    booking_period TSTZRANGE NOT NULL,
    EXCLUDE USING GIST (resource_id WITH =, booking_period WITH &&)
    -- ป้องกัน double-booking! GiST exclusion constraint
);

CREATE INDEX idx_bookings_period ON bookings USING GIST(booking_period);

-- Query: bookings ที่ทับซ้อนกับช่วงเวลาที่ต้องการ
SELECT * FROM bookings
WHERE booking_period && tstzrange('2024-12-20 14:00', '2024-12-20 16:00');
```

---

**ข้อ 6:** เปรียบเทียบขนาดและประสิทธิภาพของ B-tree vs BRIN index สำหรับตาราง logs ที่มี 100 ล้านแถว

**เฉลยข้อ 6:**
```sql
-- สร้างทั้งสองอย่าง
CREATE INDEX idx_logs_btree ON logs(created_at);
CREATE INDEX idx_logs_brin ON logs USING BRIN(created_at) WITH (pages_per_range=128);

-- ตรวจสอบขนาด
SELECT 
    indexname,
    pg_size_pretty(pg_relation_size(indexrelid)) AS size
FROM pg_stat_user_indexes
WHERE tablename = 'logs';

-- ผลลัพธ์โดยประมาณ:
-- B-tree:  ~2.5 GB สำหรับ 100M rows
-- BRIN:    ~256 KB (เล็กกว่า ~10,000x!)

-- Performance:
-- B-tree: range query ใน ~10ms
-- BRIN:   range query ใน ~50ms (แต่ยังเร็วกว่า full scan ที่ ~60s)
```

---

**ข้อ 7:** Partial index เหมาะกับ use case แบบไหน? ยกตัวอย่าง 3 กรณีในระบบจริง

**เฉลยข้อ 7:**
1. **Soft delete pattern**: `WHERE deleted_at IS NULL` - index เฉพาะ active records
2. **Pending transactions**: `WHERE status = 'pending'` - มีแค่ 1% แต่ query บ่อย
3. **High-value orders**: `WHERE amount > 10000` - ต้องการ fast access, มีน้อย

```sql
CREATE INDEX idx_users_active ON users(email) WHERE deleted_at IS NULL;
CREATE INDEX idx_orders_pending ON orders(created_at DESC) WHERE status = 'pending';
CREATE INDEX idx_orders_high_value ON orders(customer_id) WHERE amount > 10000;
```

---

**ข้อ 8:** สร้าง GIN index สำหรับ JSONB column ที่ต้องการค้นหาทั้ง key existence และ value equality

**เฉลยข้อ 8:**
```sql
-- ตาราง user_profiles พร้อม JSONB
CREATE TABLE user_profiles (
    user_id INT PRIMARY KEY,
    profile JSONB
);

-- GIN index แบบ default (รองรับ @>, ?, ?|, ?&)
CREATE INDEX idx_profiles_gin ON user_profiles USING GIN(profile);

-- ค้นหา users ที่มี key 'premium'
SELECT * FROM user_profiles WHERE profile ? 'premium';

-- ค้นหา users ที่เป็น premium member
SELECT * FROM user_profiles WHERE profile @> '{"tier": "premium"}';

-- ค้นหา users ที่มี skill 'python' ใน skills array
SELECT * FROM user_profiles WHERE profile @> '{"skills": ["python"]}';
```

---

**ข้อ 9:** ทำไม MySQL Fulltext Index ถึงต้องมีขีดจำกัดความยาวของคำ (minimum word length)?

**เฉลยข้อ 9:**
MySQL Full-text index มีการกำหนด `ft_min_word_len` (default = 4 สำหรับ MyISAM) และ `innodb_ft_min_token_size` (default = 3 สำหรับ InnoDB) เพื่อ:
1. ลดขนาด index (คำสั้นๆ เช่น "a", "an", "the" มีเยอะมาก)
2. ประสิทธิภาพ: คำสั้นๆ มักไม่มีประโยชน์ในการค้นหา
3. หลีกเลี่ยง stop words ที่มีทั้งหมด

แก้ไขได้ด้วย:
```sql
SET GLOBAL innodb_ft_min_token_size = 2;
-- ต้อง restart MySQL และ rebuild fulltext indexes
```

---

**ข้อ 10:** ออกแบบ index strategy สำหรับตาราง `geo_events` ที่มี event_id, location GEOGRAPHY, event_type, created_at, severity และต้องรองรับ queries: (1) events ในรัศมี 10km (2) events ระหว่างวันที่ (3) critical events (severity='critical') ในพื้นที่

**เฉลยข้อ 10:**
```sql
-- 1. GiST สำหรับ geographic queries
CREATE INDEX idx_geo_events_location ON geo_events USING GIST(location);

-- 2. BRIN สำหรับ time-series (sequential timestamps)
CREATE INDEX idx_geo_events_time ON geo_events USING BRIN(created_at);

-- 3. Partial GiST สำหรับ critical events (ถ้า critical น้อย)
CREATE INDEX idx_geo_events_critical_location ON geo_events USING GIST(location)
WHERE severity = 'critical';

-- 4. B-tree สำหรับ type+severity filtering
CREATE INDEX idx_geo_events_type_severity ON geo_events(event_type, severity);

-- Query ตัวอย่าง:
SELECT * FROM geo_events
WHERE ST_DWithin(location, ST_MakePoint(100.5018, 13.7563)::geography, 10000)
AND created_at >= NOW() - INTERVAL '24 hours'
AND severity = 'critical';
-- ใช้: idx_geo_events_critical_location + idx_geo_events_time
```

---

## สรุป

ใน Part 063 เราได้เรียนรู้:

1. **B-tree** - index ทั่วไป รองรับทุก comparison operators
2. **Hash** - เร็วสุดสำหรับ equality เท่านั้น
3. **GIN** - สำหรับ arrays, JSONB, full-text (PostgreSQL)
4. **GiST** - สำหรับ geometry, ranges, IP (PostgreSQL)
5. **BRIN** - ประหยัดพื้นที่มาก สำหรับ sequential data
6. **Partial/Filtered** - index เฉพาะ subset ของข้อมูล
7. **Spatial** - สำหรับ geographic data (PostGIS/MySQL Spatial)
8. **Full-text** - ค้นหาข้อความ (GIN/tsvector ใน PostgreSQL, FULLTEXT ใน MySQL)
9. **MySQL-specific** - Covering, Invisible, Spatial indexes

ใน Part 064 เราจะเรียนรู้เกี่ยวกับ Composite Indexes และการเรียงลำดับคอลัมน์ที่สำคัญมาก
