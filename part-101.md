# ตอนที่ 101: PostgreSQL Advanced Features - คุณสมบัติขั้นสูงของ PostgreSQL

## บทนำ

PostgreSQL เป็นระบบฐานข้อมูลเชิงสัมพันธ์ที่มีประสิทธิภาพสูงและมีคุณสมบัติขั้นสูงมากมายที่ทำให้แตกต่างจากระบบฐานข้อมูลอื่น ๆ ในบทนี้เราจะศึกษาคุณสมบัติเหล่านี้อย่างละเอียด ตั้งแต่ Array data types, HSTORE extension, Range types ไปจนถึง Row Level Security และคุณสมบัติใหม่ใน PostgreSQL 14/15/16

---

## 1. Array Data Type และ Array Functions

### 1.1 การสร้างและใช้งาน Array

PostgreSQL รองรับ Array ของทุก data type ซึ่งช่วยให้เก็บข้อมูลหลายค่าในคอลัมน์เดียวได้

```sql
-- สร้างตารางที่มี Array columns
CREATE TABLE students (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100),
    scores INTEGER[],           -- array ของ integer
    tags TEXT[],                -- array ของ text
    schedule TIMESTAMP[]        -- array ของ timestamp
);

-- แทรกข้อมูลลงใน array
INSERT INTO students (name, scores, tags) VALUES
('สมชาย', ARRAY[85, 92, 78, 90], ARRAY['math', 'science', 'honor']),
('สมหญิง', ARRAY[95, 88, 91, 96], ARRAY['arts', 'music']),
('สมศรี', '{70, 75, 80}', '{history, literature}');

-- query ข้อมูลจาก array
SELECT name, scores[1] AS first_score, scores[2] AS second_score
FROM students;

-- Array index ใน PostgreSQL เริ่มที่ 1 (ไม่ใช่ 0)
SELECT name, scores[1:3] AS first_three_scores  -- slice
FROM students;
```

### 1.2 Array Functions ที่สำคัญ

```sql
-- array_length: ความยาวของ array
SELECT name, array_length(scores, 1) AS num_scores
FROM students;

-- array_append: เพิ่มค่าท้าย array
UPDATE students
SET tags = array_append(tags, 'new_student')
WHERE name = 'สมชาย';

-- array_prepend: เพิ่มค่าหน้า array
SELECT array_prepend(0, ARRAY[1, 2, 3]);  -- {0,1,2,3}

-- array_cat: รวม arrays
SELECT array_cat(ARRAY[1, 2], ARRAY[3, 4]);  -- {1,2,3,4}

-- array_remove: ลบค่าออกจาก array
SELECT array_remove(ARRAY[1, 2, 3, 2, 1], 2);  -- {1,3,1}

-- unnest: แตก array เป็นแถว
SELECT name, unnest(scores) AS score
FROM students;

-- array_agg: รวมค่าหลายแถวเป็น array
SELECT array_agg(name ORDER BY name) AS all_names
FROM students;

-- ANY และ ALL กับ arrays
SELECT name FROM students
WHERE 95 = ANY(scores);  -- หานักเรียนที่มีคะแนน 95

SELECT name FROM students
WHERE 70 < ALL(scores);  -- หานักเรียนที่ทุกคะแนนมากกว่า 70

-- @> (contains) และ <@ (contained by)
SELECT name FROM students
WHERE tags @> ARRAY['math'];  -- มี tag 'math'

SELECT name FROM students
WHERE ARRAY['math', 'science'] <@ tags;  -- tags มี math และ science ทั้งคู่

-- && (overlap): มีค่าร่วมกัน
SELECT name FROM students
WHERE tags && ARRAY['math', 'arts'];

-- cardinality: จำนวนสมาชิกใน array (เทียบเท่า array_length)
SELECT name, cardinality(scores) AS count
FROM students;
```

### 1.3 Multi-dimensional Arrays

```sql
-- สร้าง 2D array
SELECT ARRAY[[1,2,3],[4,5,6]] AS matrix;

-- เข้าถึงค่าใน 2D array
SELECT (ARRAY[[1,2,3],[4,5,6]])[1][2];  -- ได้ 2

-- array_dims: ดู dimensions
SELECT array_dims(ARRAY[[1,2],[3,4]]);  -- [1:2][1:2]
```

### 1.4 GIN Index สำหรับ Arrays

```sql
-- สร้าง GIN index เพื่อค้นหา array ได้เร็วขึ้น
CREATE INDEX idx_students_tags ON students USING GIN(tags);
CREATE INDEX idx_students_scores ON students USING GIN(scores);

-- query จะใช้ index อัตโนมัติ
SELECT name FROM students WHERE tags @> ARRAY['math'];
EXPLAIN ANALYZE SELECT name FROM students WHERE tags @> ARRAY['math'];
```

---

## 2. HSTORE Extension

HSTORE เป็น extension ที่ช่วยให้เก็บ key-value pairs ในคอลัมน์เดียวได้ เหมาะสำหรับข้อมูลที่มีโครงสร้างไม่แน่นอน

### 2.1 การติดตั้งและใช้งาน HSTORE

```sql
-- ติดตั้ง extension
CREATE EXTENSION IF NOT EXISTS hstore;

-- สร้างตารางที่ใช้ hstore
CREATE TABLE products (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100),
    attributes HSTORE
);

-- แทรกข้อมูล
INSERT INTO products (name, attributes) VALUES
('iPhone 15', 'color => "blue", storage => "128GB", ram => "6GB"'),
('Samsung S24', hstore(ARRAY['color', 'black', 'storage', '256GB'])),
('MacBook Pro', 'cpu => "M3 Pro", ram => "18GB", ssd => "512GB", os => "macOS"');

-- query ค่าจาก hstore
SELECT name, attributes->'color' AS color
FROM products;

-- ตรวจสอบว่า key มีอยู่
SELECT name FROM products
WHERE attributes ? 'ram';  -- มี key 'ram'

-- ตรวจสอบหลาย keys
SELECT name FROM products
WHERE attributes ?& ARRAY['color', 'storage'];  -- มีทั้ง 'color' และ 'storage'

SELECT name FROM products
WHERE attributes ?| ARRAY['ram', 'cpu'];  -- มี 'ram' หรือ 'cpu'

-- ตรวจสอบ key-value pair
SELECT name FROM products
WHERE attributes @> 'color => "blue"';

-- แก้ไข hstore
UPDATE products
SET attributes = attributes || 'discount => "10%"'
WHERE name = 'iPhone 15';

-- ลบ key ออกจาก hstore
UPDATE products
SET attributes = delete(attributes, 'discount')
WHERE name = 'iPhone 15';

-- แปลง hstore เป็น JSON
SELECT name, hstore_to_json(attributes) AS json_attrs
FROM products;

-- แปลง hstore เป็น array
SELECT name, akeys(attributes) AS keys, avals(attributes) AS values
FROM products;

-- each: แตก hstore เป็นแถว key-value
SELECT name, (each(attributes)).*
FROM products;
```

### 2.2 GIN/GiST Index สำหรับ HSTORE

```sql
-- GIN index ดีสำหรับ @>, ?, ?&, ?|
CREATE INDEX idx_products_hstore_gin ON products USING GIN(attributes);

-- GiST index ดีสำหรับ @>, ?, ?&, ?|
CREATE INDEX idx_products_hstore_gist ON products USING GIST(attributes);

-- query ที่ใช้ index
SELECT name FROM products WHERE attributes @> 'color => "blue"';
```

---

## 3. Range Types

PostgreSQL มี range types ที่ช่วยแทน interval หรือช่วงของค่าต่าง ๆ

### 3.1 Built-in Range Types

```sql
-- int4range: ช่วงของ integer
-- int8range: ช่วงของ bigint
-- numrange: ช่วงของ numeric
-- tsrange: ช่วงของ timestamp (ไม่มี timezone)
-- tstzrange: ช่วงของ timestamp (มี timezone)
-- daterange: ช่วงของ date

-- ตัวอย่างการสร้าง range
SELECT int4range(1, 10);         -- [1,10)  (ซ้ายปิด, ขวาเปิด)
SELECT int4range(1, 10, '[]');   -- [1,10]  (ปิดทั้งสองด้าน)
SELECT int4range(1, 10, '()');   -- (1,10)  (เปิดทั้งสองด้าน)
SELECT int4range(1, 10, '[)');   -- [1,10)  (default)
```

### 3.2 การใช้งาน Range Types

```sql
-- สร้างตารางสำหรับห้องพัก
CREATE TABLE hotel_reservations (
    id SERIAL PRIMARY KEY,
    room_number INTEGER,
    guest_name VARCHAR(100),
    stay_period DATERANGE,
    EXCLUDE USING GIST (room_number WITH =, stay_period WITH &&)
    -- ป้องกัน double booking
);

-- แทรกการจอง
INSERT INTO hotel_reservations (room_number, guest_name, stay_period) VALUES
(101, 'คุณสมชาย', '[2024-01-01, 2024-01-05)'),
(101, 'คุณสมหญิง', '[2024-01-05, 2024-01-10)'),
(102, 'คุณสมศรี', '[2024-01-01, 2024-01-15)');

-- นี้จะ error เพราะห้อง 101 ถูกจองแล้วในช่วงนั้น
-- INSERT INTO hotel_reservations VALUES (4, 101, 'ทดสอบ', '[2024-01-03, 2024-01-07)');

-- ค้นหาการจองที่ตรงกับวันที่
SELECT * FROM hotel_reservations
WHERE stay_period @> DATE '2024-01-03';  -- รวม 2024-01-03

-- ค้นหาการจองที่ทับซ้อน
SELECT * FROM hotel_reservations
WHERE stay_period && daterange('2024-01-04', '2024-01-08');

-- Range operators
SELECT 
    int4range(1,10) @> 5 AS contains_5,          -- ค่า 5 อยู่ใน range
    int4range(1,10) @> int4range(2,8) AS contains_range,  -- range อยู่ใน range
    int4range(1,5) && int4range(3,8) AS overlap,  -- ทับซ้อน
    int4range(1,5) << int4range(8,10) AS left_of,  -- อยู่ทางซ้าย
    int4range(8,10) >> int4range(1,5) AS right_of, -- อยู่ทางขวา
    int4range(1,5) -|- int4range(5,10) AS adjacent; -- ติดกัน

-- Range functions
SELECT 
    lower(int4range(1,10)) AS lower_bound,
    upper(int4range(1,10)) AS upper_bound,
    isempty(int4range(5,5)) AS is_empty,     -- empty range
    lower_inf(int4range(NULL, 10)) AS lower_is_infinite,
    upper_inf(int4range(1, NULL)) AS upper_is_infinite;

-- range_merge: รวม ranges
SELECT range_merge(int4range(1,5), int4range(3,8));  -- [1,8)
```

### 3.3 Custom Range Types

```sql
-- สร้าง range type ของตัวเอง
CREATE TYPE float8range AS RANGE (
    subtype = float8,
    subtype_diff = float8mi
);

-- ใช้งาน custom range type
SELECT float8range(1.5, 3.7);
SELECT float8range(1.5, 3.7) @> 2.5;
```

---

## 4. UUID Functions

### 4.1 gen_random_uuid() และ UUID Operations

```sql
-- ต้องติดตั้ง pgcrypto สำหรับ gen_random_uuid() ใน PostgreSQL < 13
CREATE EXTENSION IF NOT EXISTS pgcrypto;

-- ใน PostgreSQL 13+ gen_random_uuid() built-in
SELECT gen_random_uuid();  -- สร้าง UUID v4 แบบ random

-- สร้างตารางที่ใช้ UUID เป็น primary key
CREATE TABLE api_keys (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id INTEGER,
    key_name VARCHAR(100),
    created_at TIMESTAMP DEFAULT NOW()
);

-- แทรกข้อมูล (id จะถูกสร้างอัตโนมัติ)
INSERT INTO api_keys (user_id, key_name) VALUES (1, 'Production API Key');
INSERT INTO api_keys (user_id, key_name) VALUES (1, 'Development API Key');

-- ดูข้อมูล
SELECT * FROM api_keys;

-- ใช้ uuid-ossp extension สำหรับฟังก์ชันเพิ่มเติม
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";

SELECT uuid_generate_v1();    -- UUID v1 (based on MAC address and time)
SELECT uuid_generate_v4();    -- UUID v4 (random)
SELECT uuid_generate_v5(uuid_ns_url(), 'https://example.com');  -- UUID v5 (hash-based)

-- ตรวจสอบว่าเป็น UUID ที่ถูกต้อง
SELECT 'a0eebc99-9c0b-4ef8-bb6d-6bb9bd380a11'::uuid;

-- เปรียบเทียบ UUID
SELECT uuid_generate_v4() = uuid_generate_v4();  -- false เสมอ
```

---

## 5. LISTEN/NOTIFY - Event-Driven Applications

LISTEN/NOTIFY เป็น mechanism สำหรับ real-time communication ระหว่าง database sessions

### 5.1 การใช้งาน LISTEN/NOTIFY

```sql
-- Session 1: รับฟัง event
LISTEN order_created;
LISTEN user_registered;

-- Session 2: ส่ง notification
NOTIFY order_created, 'order_id:12345';
NOTIFY user_registered, 'user_id:678';

-- ใช้กับ triggers เพื่อ auto-notify
CREATE OR REPLACE FUNCTION notify_order_created()
RETURNS TRIGGER AS $$
BEGIN
    PERFORM pg_notify('order_created', 
        json_build_object(
            'order_id', NEW.id,
            'customer_id', NEW.customer_id,
            'total', NEW.total_amount
        )::text
    );
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TABLE orders (
    id SERIAL PRIMARY KEY,
    customer_id INTEGER,
    total_amount DECIMAL(10,2),
    created_at TIMESTAMP DEFAULT NOW()
);

CREATE TRIGGER trigger_order_created
AFTER INSERT ON orders
FOR EACH ROW EXECUTE FUNCTION notify_order_created();

-- ทดสอบ trigger
INSERT INTO orders (customer_id, total_amount) VALUES (101, 1500.00);
-- Session ที่ LISTEN 'order_created' จะได้รับ notification

-- pg_listening_channels: ดู channels ที่กำลัง listen
SELECT pg_listening_channels();

-- ยกเลิก listen
UNLISTEN order_created;
UNLISTEN *;  -- ยกเลิกทั้งหมด
```

### 5.2 ตัวอย่าง Python ที่ใช้ LISTEN/NOTIFY

```python
# Python code สำหรับ listen to notifications
import psycopg2
import select
import json

conn = psycopg2.connect("dbname=mydb user=postgres")
conn.set_isolation_level(0)  # autocommit

cur = conn.cursor()
cur.execute("LISTEN order_created")

print("Listening for order_created events...")
while True:
    if select.select([conn], [], [], 5) == ([], [], []):
        print("Timeout, still listening...")
    else:
        conn.poll()
        while conn.notifies:
            notify = conn.notifies.pop(0)
            data = json.loads(notify.payload)
            print(f"New order: {data}")
```

---

## 6. Advisory Locks

Advisory Locks เป็น application-level locks ที่ developer สามารถควบคุมได้เอง

### 6.1 Session-level Advisory Locks

```sql
-- pg_advisory_lock: lock จนกว่าจะ unlock หรือ session จะปิด
SELECT pg_advisory_lock(12345);

-- ทำงานที่ต้องการ exclusive access
UPDATE critical_resource SET value = value + 1 WHERE id = 1;

-- unlock
SELECT pg_advisory_unlock(12345);

-- pg_try_advisory_lock: ลอง lock แต่ไม่รอ (non-blocking)
SELECT pg_try_advisory_lock(12345) AS acquired;

-- ถ้า return false แสดงว่า lock ถูกถืออยู่แล้ว
DO $$
BEGIN
    IF pg_try_advisory_lock(99999) THEN
        RAISE NOTICE 'Lock acquired, doing work...';
        PERFORM pg_sleep(2);
        PERFORM pg_advisory_unlock(99999);
    ELSE
        RAISE NOTICE 'Could not acquire lock, another process is working';
    END IF;
END $$;

-- ดู locks ที่กำลัง active
SELECT pid, granted, locktype, mode, relation::regclass
FROM pg_locks
WHERE locktype = 'advisory';
```

### 6.2 Transaction-level Advisory Locks

```sql
-- Transaction-level locks จะ release อัตโนมัติเมื่อ transaction จบ
BEGIN;
SELECT pg_advisory_xact_lock(55555);
-- ทำงาน...
COMMIT;  -- lock จะถูก release อัตโนมัติ

-- ตัวอย่างการใช้งานจริง: Job queue
CREATE TABLE job_queue (
    id SERIAL PRIMARY KEY,
    job_type VARCHAR(50),
    status VARCHAR(20) DEFAULT 'pending',
    created_at TIMESTAMP DEFAULT NOW()
);

-- Function สำหรับดึงงานแบบ safe (ป้องกัน race condition)
CREATE OR REPLACE FUNCTION claim_next_job()
RETURNS job_queue AS $$
DECLARE
    job job_queue;
BEGIN
    SELECT * INTO job
    FROM job_queue
    WHERE status = 'pending'
    AND pg_try_advisory_xact_lock(id)
    ORDER BY created_at
    LIMIT 1
    FOR UPDATE SKIP LOCKED;
    
    IF FOUND THEN
        UPDATE job_queue SET status = 'processing' WHERE id = job.id;
    END IF;
    
    RETURN job;
END;
$$ LANGUAGE plpgsql;
```

---

## 7. Table Inheritance

PostgreSQL รองรับ table inheritance ที่ช่วยให้สร้าง hierarchy ของตารางได้

### 7.1 Basic Table Inheritance

```sql
-- Parent table
CREATE TABLE vehicles (
    id SERIAL PRIMARY KEY,
    make VARCHAR(50),
    model VARCHAR(50),
    year INTEGER,
    color VARCHAR(30),
    price DECIMAL(10,2)
);

-- Child tables (inherit from vehicles)
CREATE TABLE cars (
    num_doors INTEGER,
    body_style VARCHAR(30)  -- sedan, suv, coupe, etc.
) INHERITS (vehicles);

CREATE TABLE motorcycles (
    engine_cc INTEGER,
    has_sidecar BOOLEAN DEFAULT FALSE
) INHERITS (vehicles);

CREATE TABLE trucks (
    payload_tons DECIMAL(5,2),
    has_trailer BOOLEAN DEFAULT FALSE
) INHERITS (vehicles);

-- แทรกข้อมูล
INSERT INTO cars (make, model, year, color, price, num_doors, body_style)
VALUES ('Toyota', 'Camry', 2024, 'White', 1500000, 4, 'Sedan');

INSERT INTO motorcycles (make, model, year, color, price, engine_cc)
VALUES ('Honda', 'CB650R', 2024, 'Black', 350000, 650);

-- Query ทุกยานพาหนะ (รวม child tables)
SELECT * FROM vehicles;  -- รวมข้อมูลจาก cars, motorcycles, trucks ด้วย

-- Query เฉพาะ parent (ไม่รวม children)
SELECT * FROM ONLY vehicles;

-- ตรวจสอบว่าแถวมาจาก table ไหน
SELECT tableoid::regclass AS source_table, make, model
FROM vehicles;

-- ลบข้อมูลจาก child tables ทั้งหมด
DELETE FROM vehicles;  -- ลบทั้ง parent และ children
```

### 7.2 Partitioned Tables (Table Partitioning)

```sql
-- สร้างตาราง partitioned
CREATE TABLE sales_data (
    id BIGSERIAL,
    sale_date DATE NOT NULL,
    product_id INTEGER,
    amount DECIMAL(10,2),
    region VARCHAR(50)
) PARTITION BY RANGE (sale_date);

-- สร้าง partitions
CREATE TABLE sales_data_2022 PARTITION OF sales_data
FOR VALUES FROM ('2022-01-01') TO ('2023-01-01');

CREATE TABLE sales_data_2023 PARTITION OF sales_data
FOR VALUES FROM ('2023-01-01') TO ('2024-01-01');

CREATE TABLE sales_data_2024 PARTITION OF sales_data
FOR VALUES FROM ('2024-01-01') TO ('2025-01-01');

-- แทรกข้อมูล (จะไปอยู่ใน partition ที่ถูกต้องอัตโนมัติ)
INSERT INTO sales_data (sale_date, product_id, amount, region)
VALUES 
('2023-06-15', 1, 1500.00, 'Bangkok'),
('2024-01-10', 2, 2300.00, 'Chiang Mai');

-- Query ใน partition ที่ถูกต้องโดยอัตโนมัติ
SELECT * FROM sales_data WHERE sale_date = '2023-06-15';

-- ดู partition
SELECT 
    parent.relname AS parent_table,
    child.relname AS partition_name,
    pg_get_expr(child.relpartbound, child.oid) AS partition_bound
FROM pg_class parent
JOIN pg_inherits ON pg_inherits.inhparent = parent.oid
JOIN pg_class child ON child.oid = pg_inherits.inhrelid
WHERE parent.relname = 'sales_data';
```

---

## 8. Logical Replication Basics

Logical replication ให้ความยืดหยุ่นมากกว่า physical replication เพราะสามารถเลือก tables ที่จะ replicate ได้

### 8.1 การตั้งค่า Logical Replication

```sql
-- บน Publisher (Primary server)
-- แก้ไข postgresql.conf:
-- wal_level = logical

-- สร้าง publication
CREATE PUBLICATION my_publication FOR ALL TABLES;
-- หรือระบุ tables
CREATE PUBLICATION product_pub FOR TABLE products, categories;

-- ดู publications
SELECT * FROM pg_publication;
SELECT * FROM pg_publication_tables;

-- บน Subscriber (Replica server)
-- สร้าง subscription
CREATE SUBSCRIPTION my_subscription
CONNECTION 'host=primary_host dbname=mydb user=replication_user password=secret'
PUBLICATION my_publication;

-- ดู subscriptions
SELECT * FROM pg_subscription;
SELECT * FROM pg_stat_subscription;

-- ยกเลิก subscription
DROP SUBSCRIPTION my_subscription;

-- บน Publisher: ดู replication slots
SELECT * FROM pg_replication_slots;

-- ลบ replication slot
SELECT pg_drop_replication_slot('my_subscription');
```

---

## 9. pg_trgm - Trigram Similarity

pg_trgm extension ช่วยในการค้นหา text ที่คล้ายกัน (fuzzy search)

### 9.1 การใช้งาน pg_trgm

```sql
-- ติดตั้ง extension
CREATE EXTENSION IF NOT EXISTS pg_trgm;

-- ความคล้ายกันระหว่าง strings
SELECT similarity('Bangkok', 'Bangkoh');     -- ~0.6
SELECT similarity('PostgreSQL', 'Postgres'); -- ~0.7
SELECT similarity('Hello', 'World');          -- ~0.0

-- ค้นหาข้อมูลแบบ fuzzy
CREATE TABLE cities (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100),
    country VARCHAR(50)
);

INSERT INTO cities (name, country) VALUES
('Bangkok', 'Thailand'),
('Chiang Mai', 'Thailand'),
('Phuket', 'Thailand'),
('Singapore', 'Singapore'),
('Kuala Lumpur', 'Malaysia');

-- ค้นหาชื่อเมืองที่คล้ายกัน
SELECT name, similarity(name, 'Bangok') AS sim
FROM cities
WHERE similarity(name, 'Bangok') > 0.3
ORDER BY sim DESC;

-- สร้าง GIN/GiST index สำหรับ trigram
CREATE INDEX idx_cities_name_trgm ON cities USING GIN(name gin_trgm_ops);
-- หรือ GiST (ดีกว่าสำหรับ LIKE patterns)
CREATE INDEX idx_cities_name_gist ON cities USING GIST(name gist_trgm_ops);

-- LIKE/ILIKE ที่ใช้ trigram index
SELECT name FROM cities WHERE name ILIKE '%angkok%';

-- word_similarity: ความคล้ายกันแบบ word-level
SELECT word_similarity('Chiang', 'Chiang Mai');  -- ~0.5

-- strict_word_similarity
SELECT strict_word_similarity('Mai', 'Chiang Mai');

-- show_trgm: แสดง trigrams ของ string
SELECT show_trgm('Bangkok');
-- {" b","ba","an","ng","gk","ko","ok","k "}

-- set_limit: ตั้งค่า minimum similarity threshold
SELECT set_limit(0.3);  -- ค่าเริ่มต้นคือ 0.3
```

---

## 10. Custom Types (DOMAIN)

DOMAIN เป็น constraint ที่เพิ่มเติมบน existing data type

### 10.1 การสร้างและใช้ DOMAIN

```sql
-- สร้าง domain สำหรับ email
CREATE DOMAIN email_address AS VARCHAR(255)
CHECK (VALUE ~* '^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Z]{2,}$');

-- สร้าง domain สำหรับ positive number
CREATE DOMAIN positive_int AS INTEGER
CHECK (VALUE > 0);

-- สร้าง domain สำหรับ Thai phone number
CREATE DOMAIN thai_phone AS VARCHAR(15)
CHECK (VALUE ~ '^0[0-9]{8,9}$');

-- สร้าง domain สำหรับ percentage
CREATE DOMAIN percentage AS DECIMAL(5,2)
CHECK (VALUE >= 0 AND VALUE <= 100);

-- ใช้งาน domain ในตาราง
CREATE TABLE user_profiles (
    id SERIAL PRIMARY KEY,
    email email_address NOT NULL,
    phone thai_phone,
    age positive_int,
    discount_rate percentage DEFAULT 0
);

-- ลอง insert ข้อมูลที่ไม่ถูกต้อง
INSERT INTO user_profiles (email, phone, age) 
VALUES ('invalid-email', '0812345678', 25);  -- ERROR: invalid email format

INSERT INTO user_profiles (email, phone, age) 
VALUES ('user@example.com', '0812345678', 25);  -- OK

-- แก้ไข domain
ALTER DOMAIN email_address
ADD CONSTRAINT no_temp_email
CHECK (VALUE NOT LIKE '%@tempmail.%');

-- ลบ constraint จาก domain
ALTER DOMAIN email_address
DROP CONSTRAINT no_temp_email;

-- ดู domains ที่มีอยู่
SELECT 
    typname AS domain_name,
    pg_get_constraintdef(oid) AS constraint
FROM pg_constraint
WHERE contypid = (SELECT oid FROM pg_type WHERE typname = 'email_address');
```

### 10.2 Composite Types

```sql
-- สร้าง composite type
CREATE TYPE address_type AS (
    street VARCHAR(100),
    city VARCHAR(50),
    province VARCHAR(50),
    postal_code VARCHAR(10),
    country VARCHAR(50)
);

-- ใช้งาน composite type
CREATE TABLE companies (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100),
    headquarters address_type,
    billing_address address_type
);

-- แทรกข้อมูล
INSERT INTO companies (name, headquarters)
VALUES (
    'บริษัท ตัวอย่าง จำกัด',
    ROW('123 ถนนสุขุมวิท', 'กรุงเทพฯ', 'กรุงเทพมหานคร', '10110', 'Thailand')::address_type
);

-- Query ค่าจาก composite type
SELECT name, (headquarters).city, (headquarters).province
FROM companies;

-- Update composite field
UPDATE companies
SET headquarters.postal_code = '10120'
WHERE id = 1;
```

---

## 11. Row Security Policies (RLS)

Row Level Security ช่วยให้กำหนดว่า user แต่ละคนเห็นข้อมูลแถวไหนได้บ้าง

### 11.1 การตั้งค่า RLS

```sql
-- สร้างตารางและเปิด RLS
CREATE TABLE employee_data (
    id SERIAL PRIMARY KEY,
    employee_id INTEGER,
    department VARCHAR(50),
    salary DECIMAL(10,2),
    review_notes TEXT,
    manager_id INTEGER
);

-- เปิด RLS สำหรับตาราง
ALTER TABLE employee_data ENABLE ROW LEVEL SECURITY;

-- สร้าง roles
CREATE ROLE employee;
CREATE ROLE manager;
CREATE ROLE hr_staff;

-- Policy: พนักงานเห็นเฉพาะข้อมูลของตัวเอง
CREATE POLICY employee_own_data ON employee_data
FOR SELECT TO employee
USING (employee_id = current_setting('app.current_user_id')::INTEGER);

-- Policy: ผู้จัดการเห็นข้อมูลของลูกน้อง
CREATE POLICY manager_team_data ON employee_data
FOR SELECT TO manager
USING (
    manager_id = current_setting('app.current_user_id')::INTEGER
    OR employee_id = current_setting('app.current_user_id')::INTEGER
);

-- Policy: HR เห็นข้อมูลทั้งหมด
CREATE POLICY hr_all_data ON employee_data
FOR ALL TO hr_staff
USING (TRUE);

-- ทดสอบ: ตั้งค่า user context
SET app.current_user_id = '5';
SELECT * FROM employee_data;  -- เห็นเฉพาะแถวที่ employee_id = 5

-- Policy สำหรับ INSERT
CREATE POLICY employee_insert ON employee_data
FOR INSERT TO manager
WITH CHECK (
    manager_id = current_setting('app.current_user_id')::INTEGER
);

-- ดู policies ที่มีอยู่
SELECT schemaname, tablename, policyname, permissive, roles, cmd, qual
FROM pg_policies
WHERE tablename = 'employee_data';

-- Bypass RLS
ALTER TABLE employee_data FORCE ROW LEVEL SECURITY;
-- superuser สามารถ bypass ได้
SET row_security = off;  -- หรือใช้ SET LOCAL ใน function
```

### 11.2 Multi-tenant RLS Pattern

```sql
-- Pattern สำหรับ multi-tenant application
CREATE TABLE tenant_data (
    id SERIAL PRIMARY KEY,
    tenant_id INTEGER NOT NULL,
    data JSONB,
    created_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_tenant_data_tenant ON tenant_data(tenant_id);

ALTER TABLE tenant_data ENABLE ROW LEVEL SECURITY;

-- Policy ที่ใช้ tenant_id
CREATE POLICY tenant_isolation ON tenant_data
USING (tenant_id = current_setting('app.tenant_id')::INTEGER)
WITH CHECK (tenant_id = current_setting('app.tenant_id')::INTEGER);

-- ใช้งาน
SET app.tenant_id = '42';
SELECT * FROM tenant_data;  -- เห็นเฉพาะ tenant 42
INSERT INTO tenant_data (tenant_id, data) VALUES (42, '{"key": "value"}');
```

---

## 12. PostgreSQL 14/15/16 New Features

### 12.1 PostgreSQL 14 Features

```sql
-- SEARCH clause ใน recursive CTEs
WITH RECURSIVE departments (id, name, parent_id, path) AS (
    SELECT id, name, parent_id, ARRAY[id]
    FROM org_chart WHERE parent_id IS NULL
    UNION ALL
    SELECT d.id, d.name, d.parent_id, path || d.id
    FROM org_chart d
    JOIN departments ON d.parent_id = departments.id
)
SEARCH BREADTH FIRST BY id SET ordercol
SELECT * FROM departments ORDER BY ordercol;

-- CYCLE detection ใน recursive CTEs
WITH RECURSIVE cycle_test (id, parent_id) AS (
    SELECT id, parent_id FROM nodes WHERE id = 1
    UNION ALL
    SELECT n.id, n.parent_id
    FROM nodes n
    JOIN cycle_test ON n.id = cycle_test.parent_id
)
CYCLE id SET is_cycle USING cycle_path
SELECT * FROM cycle_test WHERE NOT is_cycle;

-- Subscripting สำหรับ JSON
-- PostgreSQL 14+: ใช้ [] กับ jsonb
CREATE TABLE json_test (data JSONB);
INSERT INTO json_test VALUES ('{"name": "Alice", "scores": [10, 20, 30]}');

SELECT data['name'] FROM json_test;           -- "Alice"
SELECT data['scores'][0] FROM json_test;      -- 10 (0-indexed สำหรับ json!)
```

### 12.2 PostgreSQL 15 Features

```sql
-- MERGE statement (SQL standard)
CREATE TABLE inventory (product_id INT, quantity INT);
CREATE TABLE new_arrivals (product_id INT, quantity INT);

INSERT INTO inventory VALUES (1, 100), (2, 50);
INSERT INTO new_arrivals VALUES (1, 25), (3, 75);

MERGE INTO inventory AS target
USING new_arrivals AS source
ON target.product_id = source.product_id
WHEN MATCHED THEN
    UPDATE SET quantity = target.quantity + source.quantity
WHEN NOT MATCHED THEN
    INSERT (product_id, quantity) VALUES (source.product_id, source.quantity)
WHEN NOT MATCHED BY SOURCE THEN
    DELETE;

-- SELECT DISTINCT ON improvements
SELECT DISTINCT ON (department) 
    department, 
    employee_name, 
    salary
FROM employees
ORDER BY department, salary DESC;

-- Improved JSON functions (pg15)
SELECT json_object(
    'name' VALUE 'Alice',
    'age' VALUE 30,
    'active' VALUE true
);

SELECT json_array(1, 2, 3, null, 'text');
```

### 12.3 PostgreSQL 16 Features

```sql
-- Parallel query improvements
SET max_parallel_workers_per_gather = 4;

-- EXPLAIN ANALYZE กับ parallel
EXPLAIN (ANALYZE, VERBOSE, BUFFERS)
SELECT department, COUNT(*), AVG(salary)
FROM large_employees_table
GROUP BY department;

-- pg_stat_io view (ใหม่ใน pg16)
SELECT * FROM pg_stat_io;

-- Logical replication improvements
-- Allow logical replication from standby
-- Bidirectional logical replication

-- New functions ใน pg16
SELECT string_to_table('a,b,c,d', ',');  -- แปลง string เป็น table rows
SELECT array_sample(ARRAY[1,2,3,4,5], 3);  -- สุ่ม 3 ค่าจาก array

-- any_value aggregate function
SELECT department, any_value(manager_name) AS sample_manager
FROM employees
GROUP BY department;
```

---

## 13. PostGIS Overview

PostGIS เป็น extension สำหรับข้อมูล geographic/spatial

### 13.1 Basic PostGIS Operations

```sql
-- ติดตั้ง PostGIS
CREATE EXTENSION IF NOT EXISTS postgis;
CREATE EXTENSION IF NOT EXISTS postgis_topology;

-- สร้างตารางที่มีข้อมูล geographic
CREATE TABLE locations (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100),
    geom GEOMETRY(POINT, 4326)  -- WGS 84 coordinate system
);

-- แทรกข้อมูล (longitude, latitude)
INSERT INTO locations (name, geom) VALUES
('Central World', ST_GeomFromText('POINT(100.5394 13.7467)', 4326)),
('Suvarnabhumi Airport', ST_SetSRID(ST_Point(100.7501, 13.6900), 4326)),
('Chatuchak Market', ST_GeomFromEWKT('SRID=4326;POINT(100.5499 13.7997)'));

-- คำนวณระยะทาง
SELECT 
    a.name AS from_location,
    b.name AS to_location,
    ST_Distance(
        ST_Transform(a.geom, 32647),  -- แปลงเป็น UTM Zone 47N (meters)
        ST_Transform(b.geom, 32647)
    ) / 1000 AS distance_km
FROM locations a, locations b
WHERE a.id < b.id;

-- ค้นหาสถานที่ในรัศมี 5 กม.
SELECT name
FROM locations
WHERE ST_DWithin(
    ST_Transform(geom, 32647),
    ST_Transform(ST_SetSRID(ST_Point(100.5394, 13.7467), 4326), 32647),
    5000  -- 5 km in meters
);

-- สร้าง GIST index สำหรับ geometry
CREATE INDEX idx_locations_geom ON locations USING GIST(geom);
```

---

## 14. ตัวอย่างเพิ่มเติม (รวม)

```sql
-- Example 1: Array aggregation with filtering
SELECT 
    department,
    array_agg(name ORDER BY salary DESC) FILTER (WHERE salary > 50000) AS high_earners
FROM employees
GROUP BY department;

-- Example 2: Range type ใน financial application
CREATE TABLE price_ranges (
    product_id INTEGER,
    tier VARCHAR(20),
    price_range numrange,
    discount_pct percentage
);

INSERT INTO price_ranges VALUES
(1, 'basic', '[0, 1000)', 0),
(1, 'standard', '[1000, 10000)', 5),
(1, 'premium', '[10000, NULL)', 10);

SELECT tier, discount_pct
FROM price_ranges
WHERE price_range @> 5000::NUMERIC;

-- Example 3: HSTORE ใน product catalog
CREATE TABLE product_specs (
    product_id INTEGER PRIMARY KEY,
    category VARCHAR(50),
    specs HSTORE
);

-- Full-text search + trigram ผสมกัน
SELECT 
    p.name,
    ts_rank(to_tsvector('english', p.description), query) AS rank,
    similarity(p.name, 'iphone') AS name_sim
FROM products p,
     to_tsquery('english', 'smartphone') query
WHERE 
    to_tsvector('english', p.description) @@ query
    OR similarity(p.name, 'iphone') > 0.3
ORDER BY rank DESC, name_sim DESC;

-- Example 4: UUID + RLS pattern
CREATE TABLE documents (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    owner_id UUID NOT NULL,
    title VARCHAR(200),
    content TEXT,
    is_public BOOLEAN DEFAULT FALSE
);

ALTER TABLE documents ENABLE ROW LEVEL SECURITY;

CREATE POLICY docs_policy ON documents
USING (
    is_public = TRUE
    OR owner_id = current_setting('app.user_id')::UUID
);

-- Example 5: Partitioned table + logical replication
-- สำหรับ high-volume data ที่ต้อง replicate ไป analytics server
CREATE TABLE events (
    id BIGSERIAL,
    event_time TIMESTAMP NOT NULL,
    event_type VARCHAR(50),
    user_id INTEGER,
    data JSONB
) PARTITION BY RANGE (event_time);

-- สร้าง partitions ทุกเดือน
CREATE TABLE events_2024_01 PARTITION OF events
FOR VALUES FROM ('2024-01-01') TO ('2024-02-01');

CREATE TABLE events_2024_02 PARTITION OF events
FOR VALUES FROM ('2024-02-01') TO ('2024-03-01');

-- Index แต่ละ partition อัตโนมัติ
CREATE INDEX ON events (event_time, event_type);
CREATE INDEX ON events USING GIN(data);
```

---

## 15. Performance Tips สำหรับ PostgreSQL Advanced Features

```sql
-- 1. VACUUM และ ANALYZE สำหรับ arrays และ JSONB
VACUUM ANALYZE students;

-- 2. ตั้งค่า work_mem สำหรับ complex queries
SET work_mem = '256MB';

-- 3. ใช้ EXPLAIN ANALYZE เพื่อดู query plan
EXPLAIN (ANALYZE, BUFFERS, FORMAT JSON)
SELECT * FROM students WHERE tags @> ARRAY['math'];

-- 4. pg_stat_user_tables: ดูสถิติการใช้งาน
SELECT 
    relname AS table_name,
    n_live_tup AS live_tuples,
    n_dead_tup AS dead_tuples,
    last_vacuum,
    last_analyze
FROM pg_stat_user_tables
ORDER BY n_dead_tup DESC;

-- 5. สำหรับ RLS: ระวัง performance ถ้า policy ซับซ้อน
-- ใช้ security_barrier view เพื่อป้องกัน function inlining
CREATE VIEW secure_employees WITH (security_barrier) AS
SELECT * FROM employees
WHERE department = current_setting('app.user_dept');
```

---

## แบบฝึกหัด

### ข้อที่ 1: Array Operations
สร้างตาราง `course_enrollments` ที่มีคอลัมน์ `student_id`, `courses` (TEXT[]), `grades` (DECIMAL[]) แล้วเขียน query เพื่อหาค่าเฉลี่ยของเกรดและนักเรียนที่ลงทะเบียนวิชา 'Database'

**เฉลย:**
```sql
CREATE TABLE course_enrollments (
    student_id INTEGER PRIMARY KEY,
    student_name VARCHAR(100),
    courses TEXT[],
    grades DECIMAL[]
);

INSERT INTO course_enrollments VALUES
(1, 'สมชาย', ARRAY['Database', 'Algorithms', 'Web Dev'], ARRAY[85.5, 78.0, 92.0]),
(2, 'สมหญิง', ARRAY['Database', 'AI', 'Networks'], ARRAY[92.0, 88.0, 76.0]),
(3, 'สมศรี', ARRAY['Web Dev', 'Mobile Dev'], ARRAY[95.0, 88.0]);

-- หาค่าเฉลี่ยเกรด
SELECT 
    student_name,
    (SELECT AVG(g) FROM unnest(grades) AS g) AS avg_grade
FROM course_enrollments;

-- หานักเรียนที่ลงทะเบียน Database
SELECT student_name
FROM course_enrollments
WHERE courses @> ARRAY['Database'];
```

### ข้อที่ 2: Range Types
สร้างระบบจองห้องประชุมที่ป้องกัน double-booking

**เฉลย:**
```sql
CREATE TABLE meeting_rooms (
    id INTEGER PRIMARY KEY,
    name VARCHAR(50),
    capacity INTEGER
);

CREATE TABLE room_bookings (
    id SERIAL PRIMARY KEY,
    room_id INTEGER REFERENCES meeting_rooms(id),
    booked_by VARCHAR(100),
    booking_time TSTZRANGE,
    EXCLUDE USING GIST (
        room_id WITH =,
        booking_time WITH &&
    )
);

INSERT INTO meeting_rooms VALUES (1, 'Conference Room A', 10);

-- จองห้องประชุม
INSERT INTO room_bookings (room_id, booked_by, booking_time)
VALUES (1, 'ทีม Marketing', '[2024-01-15 09:00, 2024-01-15 11:00)');

-- นี้จะ fail เพราะ overlap
-- INSERT INTO room_bookings (room_id, booked_by, booking_time)
-- VALUES (1, 'ทีม Dev', '[2024-01-15 10:00, 2024-01-15 12:00)');
```

### ข้อที่ 3: HSTORE Product Attributes
สร้างระบบ product catalog ที่ใช้ HSTORE

**เฉลย:**
```sql
CREATE EXTENSION IF NOT EXISTS hstore;

CREATE TABLE product_catalog (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100),
    category VARCHAR(50),
    attributes HSTORE,
    price DECIMAL(10,2)
);

INSERT INTO product_catalog (name, category, attributes, price) VALUES
('Samsung TV 55"', 'Electronics', 'size => "55 inch", resolution => "4K", smart => "yes", refresh_rate => "120Hz"', 35000),
('iPhone 15 Pro', 'Mobile', 'storage => "256GB", color => "titanium", chip => "A17 Pro", camera => "48MP"', 45000),
('Nike Air Max', 'Shoes', 'size => "42", color => "white", sport => "running", material => "mesh"', 4500);

-- ค้นหาสินค้าอิเล็กทรอนิกส์ที่เป็น smart device
SELECT name, price
FROM product_catalog
WHERE category = 'Electronics'
AND attributes @> 'smart => "yes"';

-- ค้นหาสินค้าที่มีข้อมูล storage
SELECT name, attributes->'storage' AS storage
FROM product_catalog
WHERE attributes ? 'storage';

-- สรุปข้อมูลตาม key
SELECT 
    skeys(attributes) AS attribute_key,
    COUNT(*) AS product_count
FROM product_catalog
GROUP BY attribute_key
ORDER BY product_count DESC;
```

### ข้อที่ 4: RLS สำหรับ Multi-tenant Application
ตั้งค่า RLS สำหรับระบบ CRM ที่มีหลาย company

**เฉลย:**
```sql
CREATE TABLE crm_contacts (
    id SERIAL PRIMARY KEY,
    company_id INTEGER NOT NULL,
    contact_name VARCHAR(100),
    email VARCHAR(255),
    phone VARCHAR(20),
    notes TEXT,
    created_by INTEGER
);

ALTER TABLE crm_contacts ENABLE ROW LEVEL SECURITY;

-- Policy: แต่ละบริษัทเห็นเฉพาะ contacts ของตัวเอง
CREATE POLICY company_isolation ON crm_contacts
FOR ALL
USING (company_id = current_setting('app.company_id', true)::INTEGER)
WITH CHECK (company_id = current_setting('app.company_id', true)::INTEGER);

-- Test
SET app.company_id = '1';
INSERT INTO crm_contacts (company_id, contact_name, email)
VALUES (1, 'ลูกค้า A', 'customera@example.com');

-- จะไม่เห็นข้อมูล company อื่น
SELECT * FROM crm_contacts;

-- Admin policy
CREATE POLICY admin_all ON crm_contacts
FOR ALL TO admin_role
USING (TRUE)
WITH CHECK (TRUE);
```

### ข้อที่ 5: UUID Primary Keys
เปรียบเทียบ performance ระหว่าง UUID และ SERIAL primary key

**เฉลย:**
```sql
-- ตารางที่ใช้ SERIAL (integer auto-increment)
CREATE TABLE serial_test (
    id SERIAL PRIMARY KEY,
    data VARCHAR(100)
);

-- ตารางที่ใช้ UUID
CREATE TABLE uuid_test (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    data VARCHAR(100)
);

-- Insert 10,000 rows
INSERT INTO serial_test (data)
SELECT 'data_' || i
FROM generate_series(1, 10000) AS i;

INSERT INTO uuid_test (data)
SELECT 'data_' || i
FROM generate_series(1, 10000) AS i;

-- เปรียบเทียบขนาด
SELECT 
    relname AS table_name,
    pg_size_pretty(pg_total_relation_size(oid)) AS total_size
FROM pg_class
WHERE relname IN ('serial_test', 'uuid_test');

-- ทดสอบ query performance
EXPLAIN ANALYZE SELECT * FROM serial_test WHERE id = 5000;
EXPLAIN ANALYZE SELECT * FROM uuid_test WHERE id = (SELECT id FROM uuid_test LIMIT 1 OFFSET 5000);
```

### ข้อที่ 6: LISTEN/NOTIFY สำหรับ Cache Invalidation

**เฉลย:**
```sql
-- Function สำหรับ notify เมื่อ product เปลี่ยนแปลง
CREATE OR REPLACE FUNCTION notify_product_change()
RETURNS TRIGGER AS $$
BEGIN
    PERFORM pg_notify(
        'product_changed',
        json_build_object(
            'operation', TG_OP,
            'product_id', COALESCE(NEW.id, OLD.id),
            'table', TG_TABLE_NAME
        )::text
    );
    RETURN COALESCE(NEW, OLD);
END;
$$ LANGUAGE plpgsql;

-- Attach trigger
CREATE TRIGGER product_change_trigger
AFTER INSERT OR UPDATE OR DELETE ON products
FOR EACH ROW EXECUTE FUNCTION notify_product_change();

-- Application code จะ LISTEN 'product_changed' และ invalidate cache
```

### ข้อที่ 7: Table Inheritance สำหรับ Audit Log

**เฉลย:**
```sql
-- Parent: ข้อมูลพื้นฐาน audit
CREATE TABLE audit_logs (
    id BIGSERIAL PRIMARY KEY,
    action_time TIMESTAMP DEFAULT NOW(),
    user_id INTEGER,
    ip_address INET,
    action_type VARCHAR(50)
);

-- Child tables แยกตาม module
CREATE TABLE auth_audit_logs (
    username VARCHAR(100),
    success BOOLEAN
) INHERITS (audit_logs);

CREATE TABLE data_audit_logs (
    table_name VARCHAR(50),
    record_id INTEGER,
    old_data JSONB,
    new_data JSONB
) INHERITS (audit_logs);

-- Function สำหรับ log data changes
CREATE OR REPLACE FUNCTION log_data_change()
RETURNS TRIGGER AS $$
BEGIN
    INSERT INTO data_audit_logs (user_id, action_type, table_name, record_id, old_data, new_data)
    VALUES (
        current_setting('app.user_id', true)::INTEGER,
        TG_OP,
        TG_TABLE_NAME,
        COALESCE(NEW.id, OLD.id),
        CASE WHEN TG_OP != 'INSERT' THEN row_to_json(OLD)::jsonb END,
        CASE WHEN TG_OP != 'DELETE' THEN row_to_json(NEW)::jsonb END
    );
    RETURN COALESCE(NEW, OLD);
END;
$$ LANGUAGE plpgsql;
```

### ข้อที่ 8: Advisory Locks สำหรับ Job Processing

**เฉลย:**
```sql
CREATE TABLE background_jobs (
    id SERIAL PRIMARY KEY,
    job_type VARCHAR(50),
    payload JSONB,
    status VARCHAR(20) DEFAULT 'queued',
    priority INTEGER DEFAULT 5,
    attempts INTEGER DEFAULT 0,
    max_attempts INTEGER DEFAULT 3,
    scheduled_at TIMESTAMP DEFAULT NOW(),
    started_at TIMESTAMP,
    completed_at TIMESTAMP,
    error_message TEXT
);

-- Function ที่ปลอดภัยสำหรับ concurrent workers
CREATE OR REPLACE FUNCTION pick_next_job(worker_id TEXT)
RETURNS background_jobs AS $$
DECLARE
    job background_jobs;
BEGIN
    SELECT * INTO job
    FROM background_jobs
    WHERE status = 'queued'
    AND scheduled_at <= NOW()
    AND attempts < max_attempts
    AND pg_try_advisory_xact_lock(id)
    ORDER BY priority DESC, scheduled_at
    LIMIT 1
    FOR UPDATE SKIP LOCKED;
    
    IF FOUND THEN
        UPDATE background_jobs
        SET 
            status = 'running',
            started_at = NOW(),
            attempts = attempts + 1
        WHERE id = job.id;
        
        job.status = 'running';
    END IF;
    
    RETURN job;
END;
$$ LANGUAGE plpgsql;
```

### ข้อที่ 9: pg_trgm สำหรับ Search Engine

**เฉลย:**
```sql
CREATE EXTENSION IF NOT EXISTS pg_trgm;

CREATE TABLE articles (
    id SERIAL PRIMARY KEY,
    title VARCHAR(200),
    content TEXT,
    author VARCHAR(100),
    tags TEXT[]
);

-- Index สำหรับ fuzzy search
CREATE INDEX idx_articles_title_trgm ON articles USING GIN(title gin_trgm_ops);
CREATE INDEX idx_articles_content_trgm ON articles USING GIN(content gin_trgm_ops);

-- Full-text search index
CREATE INDEX idx_articles_fts ON articles 
USING GIN(to_tsvector('english', title || ' ' || content));

-- Hybrid search function
CREATE OR REPLACE FUNCTION search_articles(search_term TEXT)
RETURNS TABLE (
    id INTEGER,
    title VARCHAR,
    author VARCHAR,
    relevance FLOAT
) AS $$
BEGIN
    RETURN QUERY
    SELECT 
        a.id,
        a.title,
        a.author,
        GREATEST(
            similarity(a.title, search_term),
            ts_rank(
                to_tsvector('english', a.title || ' ' || a.content),
                plainto_tsquery('english', search_term)
            )
        ) AS relevance
    FROM articles a
    WHERE 
        similarity(a.title, search_term) > 0.2
        OR to_tsvector('english', a.title || ' ' || a.content) @@ 
           plainto_tsquery('english', search_term)
    ORDER BY relevance DESC
    LIMIT 20;
END;
$$ LANGUAGE plpgsql;
```

### ข้อที่ 10: PostgreSQL 15 MERGE Statement
ใช้ MERGE เพื่อ sync ข้อมูลระหว่าง staging และ production tables

**เฉลย:**
```sql
CREATE TABLE product_prices (
    product_id INTEGER PRIMARY KEY,
    current_price DECIMAL(10,2),
    last_updated TIMESTAMP DEFAULT NOW()
);

CREATE TABLE price_updates_staging (
    product_id INTEGER,
    new_price DECIMAL(10,2),
    effective_date TIMESTAMP
);

-- MERGE: sync staging ไปยัง production
MERGE INTO product_prices AS target
USING (
    SELECT product_id, new_price, effective_date
    FROM price_updates_staging
    WHERE effective_date <= NOW()
) AS source
ON target.product_id = source.product_id
WHEN MATCHED AND source.new_price != target.current_price THEN
    UPDATE SET 
        current_price = source.new_price,
        last_updated = NOW()
WHEN NOT MATCHED THEN
    INSERT (product_id, current_price)
    VALUES (source.product_id, source.new_price)
WHEN NOT MATCHED BY SOURCE THEN
    -- สินค้าที่ไม่มีใน staging อาจจะถูก discontinue
    UPDATE SET current_price = current_price * 0.7;  -- ลดราคา 30%

-- ดูผลลัพธ์
SELECT * FROM product_prices ORDER BY product_id;
```

---

## สรุป

ในบทนี้เราได้เรียนรู้คุณสมบัติขั้นสูงของ PostgreSQL ที่สำคัญ:

1. **Array types** - เก็บหลายค่าในคอลัมน์เดียว พร้อม operators ที่หลากหลาย
2. **HSTORE** - key-value storage ที่ยืดหยุ่น
3. **Range types** - แทนช่วงของค่า พร้อม operators เช่น @>, &&, -|-
4. **UUID** - สร้าง globally unique identifiers
5. **LISTEN/NOTIFY** - real-time messaging ระหว่าง connections
6. **Advisory Locks** - application-level locking
7. **Table Inheritance** - table hierarchy
8. **Logical Replication** - selective replication
9. **pg_trgm** - fuzzy text search
10. **PostGIS** - geographic data
11. **Domain types** - custom constrained types
12. **Row Level Security** - fine-grained access control
13. **PostgreSQL 14/15/16 features** - latest improvements

คุณสมบัติเหล่านี้ทำให้ PostgreSQL เป็นมากกว่าแค่ฐานข้อมูลเชิงสัมพันธ์ทั่วไป และสามารถรองรับ use cases ที่หลากหลายได้อย่างมีประสิทธิภาพ
