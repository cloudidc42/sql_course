# ตอนที่ 103: SQLite - Lightweight Powerhouse

## บทนำ

SQLite เป็นฐานข้อมูลที่เบาที่สุด แต่ทรงพลังมาก โดดเด่นตรงที่ไม่ต้องการ server - ข้อมูลทั้งหมดเก็บอยู่ในไฟล์เดียว SQLite ถูกใช้งานอยู่ในทุกที่ตั้งแต่ smartphones, browsers, IoT devices ไปจนถึง aircraft systems บทนี้จะศึกษา SQLite อย่างละเอียดทั้งการใช้งาน, ข้อจำกัด, และการ integration กับ Python

---

## 1. SQLite Use Cases

### 1.1 เมื่อไรควรใช้ SQLite

SQLite เหมาะสมสำหรับ:

- **Mobile applications** - Android, iOS ใช้ SQLite เป็น default
- **Embedded systems** - ฝังในแอปพลิเคชัน
- **Testing** - ทดสอบ queries ก่อน deploy ไปยัง production database
- **Local data storage** - เก็บข้อมูลบนเครื่องของ user
- **Prototyping** - พัฒนาและทดสอบ schema เร็ว ๆ
- **Single-user applications** - desktop apps, CLI tools
- **Small to medium websites** - traffic ไม่สูงมาก
- **Configuration storage** - แทน config files
- **Cache** - เก็บ cached data

```sql
-- SQLite ไม่ต้อง setup - แค่เปิดไฟล์
-- $ sqlite3 myapp.db

-- สร้างตาราง
CREATE TABLE IF NOT EXISTS users (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    username TEXT NOT NULL UNIQUE,
    email TEXT NOT NULL,
    created_at TEXT DEFAULT (datetime('now'))
);

-- SQLite มีอยู่ทุกที่
-- Python: import sqlite3 (built-in)
-- Node.js: npm install better-sqlite3
-- Browser: sql.js
-- Electron apps
-- Flutter: sqflite
-- Android: Room (based on SQLite)
```

### 1.2 SQLite ไม่เหมาะสำหรับ

```sql
-- 1. High concurrency writes (multiple writers พร้อมกัน)
-- SQLite ใช้ file-level locking - writer lock ทั้งไฟล์
-- ถ้า concurrent writes สูง -> ใช้ PostgreSQL/MySQL แทน

-- 2. Large-scale distributed systems
-- ไม่มี native replication/clustering

-- 3. Fine-grained access control
-- ไม่มี user management แบบ full-featured

-- 4. High availability requirements
-- ไม่มี built-in failover

-- 5. Very large databases (> 100GB)
-- เหมาะกับ databases ขนาดเล็กถึงกลาง
-- (แต่ทางเทคนิค SQLite รองรับได้ถึง 281 terabytes)
```

---

## 2. SQLite Data Types - Type Affinity System

SQLite มีระบบ types ที่ต่างจาก databases อื่นอย่างสิ้นเชิง

### 2.1 Storage Classes

```sql
-- SQLite มี 5 storage classes:
-- NULL, INTEGER, REAL, TEXT, BLOB

-- Type Affinity rules:
-- ถ้า column type มีคำว่า INT -> INTEGER affinity
-- ถ้ามี CHAR, CLOB, TEXT -> TEXT affinity
-- ถ้าไม่มี type ระบุ หรือมี BLOB -> BLOB/NONE affinity
-- ถ้ามี REAL, FLOA, DOUB -> REAL affinity
-- อื่น ๆ -> NUMERIC affinity

CREATE TABLE type_affinity_demo (
    id INTEGER,           -- INTEGER affinity
    name TEXT,            -- TEXT affinity
    balance REAL,         -- REAL affinity
    data BLOB,            -- BLOB/NONE affinity
    value NUMERIC,        -- NUMERIC affinity
    
    -- SQLite ยอมรับ type names ทั้งหมดนี้:
    int_col INT,
    integer_col INTEGER,
    tinyint_col TINYINT,
    bigint_col BIGINT,
    varchar_col VARCHAR(100),  -- ความยาวไม่มีผล!
    char_col CHAR(10),
    decimal_col DECIMAL(10,2),
    float_col FLOAT,
    double_col DOUBLE PRECISION,
    boolean_col BOOLEAN,  -- เก็บเป็น 0/1
    date_col DATE,        -- เก็บเป็น TEXT!
    datetime_col DATETIME -- เก็บเป็น TEXT!
);

-- Type flexibility
INSERT INTO type_affinity_demo (id, name, balance) VALUES
(1, 'Alice', 1000.50),
('2', 42, '500.75'),  -- SQLite จะแปลง type ให้
(3, NULL, NULL);

-- ตรวจสอบ actual storage type
SELECT id, typeof(id), name, typeof(name), balance, typeof(balance)
FROM type_affinity_demo;

-- SQLite type casting
SELECT CAST('123' AS INTEGER);    -- 123
SELECT CAST(3.14 AS INTEGER);     -- 3
SELECT CAST(100 AS REAL);         -- 100.0
SELECT CAST('hello' AS INTEGER);  -- 0
```

### 2.2 Boolean ใน SQLite

```sql
-- SQLite ไม่มี Boolean type จริง ๆ
-- ใช้ 0/1 หรือ TRUE/FALSE (aliases สำหรับ 1/0)

CREATE TABLE settings (
    key TEXT PRIMARY KEY,
    value TEXT,
    is_active INTEGER DEFAULT 1  -- 0 หรือ 1
);

INSERT INTO settings VALUES ('notifications', 'email', 1);
INSERT INTO settings VALUES ('dark_mode', NULL, TRUE);  -- TRUE = 1

SELECT key, CASE WHEN is_active THEN 'Yes' ELSE 'No' END AS active
FROM settings;

-- Boolean checks
SELECT * FROM settings WHERE is_active;       -- is_active != 0
SELECT * FROM settings WHERE NOT is_active;   -- is_active = 0
```

### 2.3 Date/Time ใน SQLite

```sql
-- SQLite ไม่มี Date/Time type ของตัวเอง
-- เก็บได้ 3 แบบ: TEXT, INTEGER (unix timestamp), REAL (Julian day)

CREATE TABLE events (
    id INTEGER PRIMARY KEY,
    event_name TEXT,
    -- เก็บเป็น ISO 8601 text (แนะนำ)
    event_date TEXT,
    -- เก็บเป็น Unix timestamp (integer)
    event_timestamp INTEGER,
    -- เก็บเป็น Julian day number (real)
    event_julian REAL
);

INSERT INTO events (event_name, event_date, event_timestamp, event_julian) VALUES
('Conference', '2024-03-15 09:00:00', 1710489600, julianday('2024-03-15'));

-- Date functions
SELECT 
    event_name,
    date('now') AS today,
    datetime('now') AS current_datetime,
    datetime('now', 'localtime') AS local_time,
    date(event_date) AS event_date_only,
    strftime('%Y', event_date) AS year,
    strftime('%m', event_date) AS month,
    strftime('%d', event_date) AS day,
    strftime('%H:%M', event_date) AS time
FROM events;

-- Date arithmetic
SELECT 
    date('now', '+7 days') AS next_week,
    date('now', '-1 month') AS last_month,
    date('now', '+1 year') AS next_year,
    date('now', 'start of month') AS month_start,
    date('now', 'start of year') AS year_start;

-- Unix timestamp
SELECT datetime(1710489600, 'unixepoch');
SELECT strftime('%s', '2024-03-15 09:00:00');  -- to unix timestamp
```

---

## 3. SQLite-Specific Functions

### 3.1 String Functions

```sql
-- SQLite มี string functions มาตรฐาน + บางอย่างพิเศษ

-- SUBSTR (หรือ SUBSTRING)
SELECT SUBSTR('Hello World', 7);        -- World
SELECT SUBSTR('Hello World', 7, 5);     -- World

-- INSTR
SELECT INSTR('Hello World', 'World');   -- 7 (1-indexed)
SELECT INSTR('Hello World', 'xyz');     -- 0 (not found)

-- REPLACE
SELECT REPLACE('Hello World', 'World', 'SQLite');  -- Hello SQLite

-- TRIM, LTRIM, RTRIM
SELECT TRIM('  hello  ');
SELECT LTRIM('  hello  ');
SELECT RTRIM('  hello  ');
SELECT TRIM('xxxhelloxxx', 'x');  -- ลบ 'x' จากทั้งสองด้าน

-- UPPER, LOWER (สำหรับ ASCII เท่านั้น - ไม่รองรับ unicode อักขระไทย)
SELECT UPPER('hello');   -- HELLO
SELECT LOWER('HELLO');   -- hello

-- HEX: แปลงเป็น hexadecimal
SELECT HEX('ABC');    -- 414243
SELECT HEX(255);      -- FF

-- QUOTE: escape string สำหรับใช้ใน SQL
SELECT QUOTE('it''s fine');  -- 'it''s fine'
SELECT QUOTE(42);             -- 42
SELECT QUOTE(NULL);           -- NULL

-- PRINTF / FORMAT: string formatting (SQLite 3.38+)
SELECT PRINTF('Hello %s, you are %d years old', 'Alice', 30);
SELECT FORMAT('Price: %.2f THB', 1234.5);  -- SQLite 3.38+
```

### 3.2 Math Functions

```sql
-- Basic math
SELECT ABS(-5);        -- 5
SELECT ROUND(3.14159, 2);  -- 3.14
SELECT CEIL(3.1);     -- 4 (ต้องใช้ math extension หรือ SQLite 3.35+)
SELECT FLOOR(3.9);    -- 3
SELECT MAX(1, 2, 3);  -- 3 (scalar MAX ไม่ใช่ aggregate)
SELECT MIN(1, 2, 3);  -- 1

-- SQLite 3.35+: built-in math functions
SELECT SQRT(16);      -- 4.0
SELECT POW(2, 8);     -- 256.0
SELECT EXP(1);        -- 2.71828...
SELECT LOG(100);      -- 4.60517... (natural log)
SELECT LOG2(8);       -- 3.0
SELECT LOG10(1000);   -- 3.0
SELECT SIN(3.14159/2);  -- ~1.0
SELECT COS(0);          -- 1.0
SELECT TAN(3.14159/4);  -- ~1.0
SELECT CEIL(3.1);    -- 4.0
SELECT FLOOR(3.9);   -- 3.0
SELECT TRUNC(3.9);   -- 3.0
SELECT PI();         -- 3.14159...
SELECT SIGN(-5);     -- -1
SELECT SIGN(0);      -- 0
SELECT SIGN(5);      -- 1
```

### 3.3 Aggregate Functions พิเศษ

```sql
-- GROUP_CONCAT: รวม values เป็น string
CREATE TABLE tags (item_id INTEGER, tag TEXT);
INSERT INTO tags VALUES (1, 'python'), (1, 'sql'), (1, 'database');
INSERT INTO tags VALUES (2, 'javascript'), (2, 'web');

SELECT 
    item_id,
    GROUP_CONCAT(tag) AS tags_comma,
    GROUP_CONCAT(tag, ' | ') AS tags_pipe,
    GROUP_CONCAT(tag ORDER BY tag) AS tags_sorted
FROM tags
GROUP BY item_id;

-- Total ด้วย GROUP_CONCAT
SELECT GROUP_CONCAT(DISTINCT tag ORDER BY tag) AS all_unique_tags FROM tags;

-- TOTAL vs SUM: TOTAL ไม่ return NULL เมื่อทุกค่าเป็น NULL
SELECT SUM(NULL), TOTAL(NULL);  -- NULL, 0.0
```

---

## 4. WAL Mode (Write-Ahead Logging)

WAL mode ปรับปรุง performance ของ concurrent reads และ writes อย่างมาก

### 4.1 การเปิดใช้งาน WAL Mode

```sql
-- เปิด WAL mode (ต้องทำทุกครั้งที่เปิด database)
PRAGMA journal_mode = WAL;

-- ตรวจสอบ mode ปัจจุบัน
PRAGMA journal_mode;  -- ค่าได้: delete, truncate, persist, memory, wal, off

-- WAL mode ให้ประโยชน์อะไร:
-- 1. Readers ไม่ block writers, writers ไม่ block readers
-- 2. Performance ดีขึ้นมากสำหรับ write-heavy workloads
-- 3. Crash recovery เร็วขึ้น

-- WAL checkpointing
PRAGMA wal_checkpoint;          -- passive checkpoint
PRAGMA wal_checkpoint(FULL);    -- full checkpoint
PRAGMA wal_checkpoint(RESTART); -- restart checkpoint

-- Synchronous mode
PRAGMA synchronous = NORMAL;   -- ดีกว่า FULL สำหรับ WAL
PRAGMA synchronous = FULL;     -- ปลอดภัยที่สุด (default)
PRAGMA synchronous = OFF;      -- เร็วที่สุดแต่เสี่ยง
```

### 4.2 Performance Settings

```sql
-- Cache size (จำนวน pages ใน memory)
PRAGMA cache_size = 10000;        -- 10000 pages (~ 40MB)
PRAGMA cache_size = -102400;      -- 100MB ในหน่วย kibibytes (negative = kibibytes)

-- Page size (ตั้งก่อนสร้าง database)
PRAGMA page_size = 4096;          -- 4KB (default)
PRAGMA page_size = 8192;          -- 8KB (ดีกว่าสำหรับ read-heavy)

-- Memory-mapped I/O
PRAGMA mmap_size = 268435456;     -- 256MB memory-mapped

-- Temp store
PRAGMA temp_store = MEMORY;       -- เก็บ temp tables ใน memory

-- สรุปการตั้งค่าที่แนะนำสำหรับ production
PRAGMA journal_mode = WAL;
PRAGMA synchronous = NORMAL;
PRAGMA cache_size = -64000;       -- 64MB
PRAGMA temp_store = MEMORY;
PRAGMA mmap_size = 134217728;     -- 128MB

-- ดู database info
PRAGMA database_list;
PRAGMA table_info(users);
PRAGMA index_list(users);
PRAGMA foreign_key_list(orders);
```

---

## 5. Virtual Tables

### 5.1 FTS5 (Full-Text Search)

```sql
-- FTS5 virtual table สำหรับ full-text search
CREATE VIRTUAL TABLE articles_fts USING fts5(
    title,
    content,
    author,
    tokenize = 'unicode61'  -- รองรับ Unicode
);

-- หรือ link กับตารางจริง (content table)
CREATE TABLE articles_real (
    id INTEGER PRIMARY KEY,
    title TEXT,
    content TEXT,
    author TEXT
);

CREATE VIRTUAL TABLE articles_fts_linked USING fts5(
    title, content, author,
    content = 'articles_real',
    content_rowid = 'id'
);

-- แทรกข้อมูล
INSERT INTO articles_fts (title, content, author) VALUES
('Introduction to SQLite', 'SQLite is a lightweight database engine that does not require a separate server process...', 'John Doe'),
('Database Performance Tips', 'Optimize your database performance by using proper indexes and query optimization...', 'Jane Smith'),
('SQL Best Practices', 'Writing clean and efficient SQL queries is essential for good database applications...', 'Bob Wilson');

-- Basic full-text search
SELECT * FROM articles_fts WHERE articles_fts MATCH 'SQLite database';

-- Phrase search
SELECT * FROM articles_fts WHERE articles_fts MATCH '"lightweight database"';

-- Column-specific search
SELECT * FROM articles_fts WHERE articles_fts MATCH 'author:John';

-- Prefix search
SELECT * FROM articles_fts WHERE articles_fts MATCH 'data*';

-- Boolean operators (FTS5 ใช้ AND, OR, NOT)
SELECT * FROM articles_fts WHERE articles_fts MATCH 'SQLite AND performance';
SELECT * FROM articles_fts WHERE articles_fts MATCH 'database OR SQL';
SELECT * FROM articles_fts WHERE articles_fts MATCH 'database NOT NoSQL';

-- Ranking results
SELECT 
    title,
    author,
    bm25(articles_fts) AS rank
FROM articles_fts
WHERE articles_fts MATCH 'database'
ORDER BY rank;  -- bm25 score (negative, smaller = better match)

-- Snippet function
SELECT 
    title,
    snippet(articles_fts, 1, '<b>', '</b>', '...', 15) AS excerpt
FROM articles_fts
WHERE articles_fts MATCH 'SQLite';

-- Rebuild FTS index
INSERT INTO articles_fts(articles_fts) VALUES('rebuild');
```

### 5.2 R-Tree (Spatial Index)

```sql
-- R-Tree virtual table สำหรับ spatial data
CREATE VIRTUAL TABLE locations_rtree USING rtree(
    id,
    min_lat, max_lat,   -- latitude range
    min_lon, max_lon    -- longitude range
);

CREATE TABLE locations (
    id INTEGER PRIMARY KEY,
    name TEXT,
    latitude REAL,
    longitude REAL
);

-- แทรกข้อมูล
INSERT INTO locations VALUES (1, 'Central World', 13.7467, 100.5394);
INSERT INTO locations VALUES (2, 'Siam Paragon', 13.7464, 100.5332);
INSERT INTO locations VALUES (3, 'MBK Center', 13.7449, 100.5297);

INSERT INTO locations_rtree VALUES (1, 13.7467, 13.7467, 100.5394, 100.5394);
INSERT INTO locations_rtree VALUES (2, 13.7464, 13.7464, 100.5332, 100.5332);
INSERT INTO locations_rtree VALUES (3, 13.7449, 13.7449, 100.5297, 100.5297);

-- ค้นหาสถานที่ในกรอบพื้นที่
SELECT l.name, l.latitude, l.longitude
FROM locations l
JOIN locations_rtree r ON l.id = r.id
WHERE r.min_lat >= 13.74 AND r.max_lat <= 13.75
AND r.min_lon >= 100.52 AND r.max_lon <= 100.55;
```

---

## 6. SQLite Extensions

### 6.1 JSON1 Extension (Built-in)

```sql
-- JSON1 extension เป็น built-in ใน SQLite 3.38+
-- สำหรับเวอร์ชันเก่า: SELECT load_extension('json1');

CREATE TABLE json_data (
    id INTEGER PRIMARY KEY,
    data TEXT  -- เก็บ JSON เป็น TEXT
);

INSERT INTO json_data (data) VALUES
('{"name": "Alice", "age": 30, "hobbies": ["reading", "coding"]}'),
('{"name": "Bob", "age": 25, "city": "Bangkok", "scores": [85, 92, 78]}');

-- json_extract: ดึงค่าจาก JSON
SELECT 
    json_extract(data, '$.name') AS name,
    json_extract(data, '$.age') AS age
FROM json_data;

-- -> operator (SQLite 3.38+)
SELECT data->>'$.name' AS name FROM json_data;

-- json_object: สร้าง JSON object
SELECT json_object('key1', 'value1', 'key2', 42);

-- json_array: สร้าง JSON array
SELECT json_array(1, 2, 'three', NULL);

-- json_insert, json_replace, json_set, json_remove
SELECT json_insert('{"a": 1}', '$.b', 2);    -- {"a":1,"b":2}
SELECT json_set('{"a": 1}', '$.a', 99);      -- {"a":99}
SELECT json_remove('{"a": 1, "b": 2}', '$.b');  -- {"a":1}

-- json_each: iterate JSON array
SELECT value FROM json_each('["apple", "banana", "cherry"]');

-- json_tree: iterate ทั้ง JSON document
SELECT key, value, type, path
FROM json_tree('{"name": "Alice", "hobbies": ["reading", "coding"]}');

-- json_valid: ตรวจสอบ JSON
SELECT json_valid('{"key": "value"}');  -- 1
SELECT json_valid('{invalid}');          -- 0

-- json_quote, json_type
SELECT json_type('{"key": "value"}');   -- object
SELECT json_type('[1,2,3]');             -- array
SELECT json_type('"hello"');             -- text
```

### 6.2 Math Extension (Built-in SQLite 3.35+)

```sql
-- Math functions (ต้อง compile ด้วย -DSQLITE_ENABLE_MATH_FUNCTIONS)
SELECT SQRT(16);     -- 4.0
SELECT POW(2, 10);   -- 1024.0
SELECT EXP(1);       -- 2.718...
SELECT LOG(100);     -- natural log
SELECT LOG10(1000);  -- 3.0
SELECT SIN(0);       -- 0.0
SELECT COS(0);       -- 1.0
SELECT PI();         -- 3.14159...
SELECT CEIL(3.2);    -- 4.0
SELECT FLOOR(3.9);   -- 3.0
```

---

## 7. SQLite ใน Python - sqlite3 Module

### 7.1 Basic Operations

```python
import sqlite3
from contextlib import contextmanager
import json
from datetime import datetime

# เชื่อมต่อกับ database (สร้างใหม่ถ้าไม่มี)
conn = sqlite3.connect('example.db')
# หรือ in-memory database:
# conn = sqlite3.connect(':memory:')

# ตั้งค่า WAL mode และ performance settings
conn.execute("PRAGMA journal_mode = WAL")
conn.execute("PRAGMA synchronous = NORMAL")
conn.execute("PRAGMA cache_size = -64000")
conn.execute("PRAGMA foreign_keys = ON")  # เปิด foreign key enforcement

# สร้าง cursor
cursor = conn.cursor()

# สร้างตาราง
cursor.execute('''
CREATE TABLE IF NOT EXISTS users (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    username TEXT NOT NULL UNIQUE,
    email TEXT NOT NULL,
    profile TEXT,  -- JSON
    created_at TEXT DEFAULT (datetime('now'))
)
''')

# Insert ข้อมูล
cursor.execute(
    "INSERT INTO users (username, email, profile) VALUES (?, ?, ?)",
    ('somchai', 'somchai@example.com', json.dumps({'city': 'Bangkok', 'age': 28}))
)

# executemany: insert หลายแถว
users = [
    ('somying', 'somying@example.com', json.dumps({'city': 'Chiang Mai', 'age': 25})),
    ('somrit', 'somrit@example.com', json.dumps({'city': 'Phuket', 'age': 32})),
]
cursor.executemany(
    "INSERT INTO users (username, email, profile) VALUES (?, ?, ?)",
    users
)

conn.commit()

# Query ข้อมูล
cursor.execute("SELECT * FROM users")
rows = cursor.fetchall()
for row in rows:
    print(row)

# fetchone: ดึงทีละแถว
cursor.execute("SELECT * FROM users WHERE id = ?", (1,))
user = cursor.fetchone()
print(user)

# ใช้ row_factory เพื่อให้ผลลัพธ์เป็น dict
conn.row_factory = sqlite3.Row
cursor = conn.cursor()
cursor.execute("SELECT * FROM users")
for row in cursor.fetchall():
    print(dict(row))  # แปลงเป็น dict
    print(row['username'])  # เข้าถึงด้วย column name

conn.close()
```

### 7.2 Context Manager Pattern (แนะนำ)

```python
import sqlite3
from contextlib import contextmanager

DATABASE_PATH = 'myapp.db'

@contextmanager
def get_db_connection(db_path=DATABASE_PATH):
    """Context manager สำหรับ database connections"""
    conn = sqlite3.connect(db_path)
    conn.row_factory = sqlite3.Row
    conn.execute("PRAGMA journal_mode = WAL")
    conn.execute("PRAGMA foreign_keys = ON")
    try:
        yield conn
        conn.commit()
    except Exception:
        conn.rollback()
        raise
    finally:
        conn.close()

# ใช้งาน
with get_db_connection() as conn:
    # Transaction จะ commit อัตโนมัติ หรือ rollback ถ้า exception
    conn.execute(
        "INSERT INTO users (username, email) VALUES (?, ?)",
        ('newuser', 'newuser@example.com')
    )
    
    cursor = conn.execute("SELECT COUNT(*) FROM users")
    count = cursor.fetchone()[0]
    print(f"Total users: {count}")
```

### 7.3 Advanced Python Integration

```python
import sqlite3
import json
from typing import List, Dict, Any, Optional
from dataclasses import dataclass, asdict
from datetime import datetime

# Custom types: register adapters สำหรับ Python objects
def adapt_dict(d: dict) -> str:
    return json.dumps(d)

def convert_dict(b: bytes) -> dict:
    return json.loads(b.decode())

sqlite3.register_adapter(dict, adapt_dict)
sqlite3.register_converter("JSON", convert_dict)

# ใช้งาน custom converter
conn = sqlite3.connect(':memory:', detect_types=sqlite3.PARSE_DECLTYPES)
conn.execute('''
CREATE TABLE config (
    key TEXT PRIMARY KEY,
    value JSON  -- ใช้ JSON type ที่ register ไว้
)
''')

config = {'theme': 'dark', 'language': 'th', 'notifications': True}
conn.execute("INSERT INTO config VALUES (?, ?)", ('app_settings', config))
conn.commit()

cursor = conn.execute("SELECT value FROM config WHERE key = ?", ('app_settings',))
result = cursor.fetchone()[0]
print(type(result), result)  # <class 'dict'> {'theme': 'dark', ...}

# Dataclass integration
@dataclass
class Product:
    id: Optional[int]
    name: str
    price: float
    category: str
    specs: dict
    
class ProductRepository:
    def __init__(self, db_path: str):
        self.db_path = db_path
        self._setup_db()
    
    def _get_conn(self):
        conn = sqlite3.connect(self.db_path)
        conn.row_factory = sqlite3.Row
        conn.execute("PRAGMA journal_mode = WAL")
        conn.execute("PRAGMA foreign_keys = ON")
        return conn
    
    def _setup_db(self):
        with self._get_conn() as conn:
            conn.execute('''
            CREATE TABLE IF NOT EXISTS products (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                name TEXT NOT NULL,
                price REAL NOT NULL,
                category TEXT,
                specs TEXT  -- JSON stored as text
            )
            ''')
            conn.commit()
    
    def create(self, product: Product) -> Product:
        with self._get_conn() as conn:
            cursor = conn.execute(
                "INSERT INTO products (name, price, category, specs) VALUES (?, ?, ?, ?)",
                (product.name, product.price, product.category, 
                 json.dumps(product.specs))
            )
            conn.commit()
            product.id = cursor.lastrowid
            return product
    
    def get_by_id(self, product_id: int) -> Optional[Product]:
        with self._get_conn() as conn:
            cursor = conn.execute(
                "SELECT * FROM products WHERE id = ?",
                (product_id,)
            )
            row = cursor.fetchone()
            if row:
                return Product(
                    id=row['id'],
                    name=row['name'],
                    price=row['price'],
                    category=row['category'],
                    specs=json.loads(row['specs']) if row['specs'] else {}
                )
            return None
    
    def search(self, keyword: str, max_price: float = None) -> List[Product]:
        query = "SELECT * FROM products WHERE name LIKE ?"
        params = [f'%{keyword}%']
        
        if max_price:
            query += " AND price <= ?"
            params.append(max_price)
        
        with self._get_conn() as conn:
            cursor = conn.execute(query, params)
            return [
                Product(
                    id=row['id'],
                    name=row['name'],
                    price=row['price'],
                    category=row['category'],
                    specs=json.loads(row['specs']) if row['specs'] else {}
                )
                for row in cursor.fetchall()
            ]

# ใช้งาน
repo = ProductRepository(':memory:')
p1 = repo.create(Product(None, 'iPhone 15', 45000, 'Mobile', {'color': 'blue', 'storage': '128GB'}))
p2 = repo.create(Product(None, 'MacBook Pro', 89000, 'Laptop', {'cpu': 'M3 Pro', 'ram': '18GB'}))

product = repo.get_by_id(1)
print(product)

results = repo.search('iPhone')
for p in results:
    print(f"{p.name}: {p.price} THB")
```

### 7.4 Thread Safety และ Connection Pooling

```python
import sqlite3
import threading
from queue import Queue
from typing import Optional

class SQLitePool:
    """Thread-safe SQLite connection pool"""
    
    def __init__(self, db_path: str, pool_size: int = 5):
        self.db_path = db_path
        self.pool_size = pool_size
        self._pool = Queue(maxsize=pool_size)
        self._lock = threading.Lock()
        
        # Pre-create connections
        for _ in range(pool_size):
            conn = self._create_connection()
            self._pool.put(conn)
    
    def _create_connection(self):
        # check_same_thread=False สำหรับ multi-threaded
        conn = sqlite3.connect(
            self.db_path,
            check_same_thread=False
        )
        conn.row_factory = sqlite3.Row
        conn.execute("PRAGMA journal_mode = WAL")
        conn.execute("PRAGMA foreign_keys = ON")
        return conn
    
    def get_connection(self, timeout: float = 30) -> sqlite3.Connection:
        return self._pool.get(timeout=timeout)
    
    def return_connection(self, conn: sqlite3.Connection):
        self._pool.put(conn)
    
    def execute(self, query: str, params=None):
        conn = self.get_connection()
        try:
            cursor = conn.execute(query, params or [])
            conn.commit()
            return cursor
        except Exception:
            conn.rollback()
            raise
        finally:
            self.return_connection(conn)

# ใช้งาน
pool = SQLitePool('myapp.db', pool_size=5)

def worker(pool, thread_id):
    result = pool.execute(
        "INSERT INTO users (username, email) VALUES (?, ?)",
        (f'user_{thread_id}', f'user_{thread_id}@example.com')
    )
    print(f"Thread {thread_id}: inserted row {result.lastrowid}")

# Run concurrent workers
threads = []
for i in range(10):
    t = threading.Thread(target=worker, args=(pool, i))
    threads.append(t)
    t.start()

for t in threads:
    t.join()
```

### 7.5 Migration System

```python
import sqlite3
import hashlib
from datetime import datetime

class SQLiteMigration:
    """Simple database migration system"""
    
    MIGRATIONS = [
        {
            'version': 1,
            'description': 'Initial schema',
            'up': '''
                CREATE TABLE users (
                    id INTEGER PRIMARY KEY AUTOINCREMENT,
                    username TEXT NOT NULL UNIQUE,
                    email TEXT NOT NULL,
                    created_at TEXT DEFAULT (datetime('now'))
                );
                CREATE TABLE products (
                    id INTEGER PRIMARY KEY AUTOINCREMENT,
                    name TEXT NOT NULL,
                    price REAL,
                    created_at TEXT DEFAULT (datetime('now'))
                );
            ''',
            'down': 'DROP TABLE IF EXISTS users; DROP TABLE IF EXISTS products;'
        },
        {
            'version': 2,
            'description': 'Add user roles',
            'up': '''
                ALTER TABLE users ADD COLUMN role TEXT DEFAULT 'user';
                CREATE INDEX idx_users_role ON users(role);
            ''',
            'down': "-- SQLite ไม่รองรับ DROP COLUMN ก่อน 3.35"
        },
        {
            'version': 3,
            'description': 'Add product categories',
            'up': '''
                CREATE TABLE categories (
                    id INTEGER PRIMARY KEY AUTOINCREMENT,
                    name TEXT NOT NULL UNIQUE
                );
                ALTER TABLE products ADD COLUMN category_id INTEGER 
                    REFERENCES categories(id);
            ''',
            'down': '''
                DROP TABLE IF EXISTS categories;
                -- ไม่สามารถลบ column ง่าย ๆ ใน SQLite เก่า
            '''
        }
    ]
    
    def __init__(self, db_path: str):
        self.db_path = db_path
        self._ensure_migrations_table()
    
    def _get_conn(self):
        conn = sqlite3.connect(self.db_path)
        conn.execute("PRAGMA foreign_keys = ON")
        return conn
    
    def _ensure_migrations_table(self):
        with self._get_conn() as conn:
            conn.execute('''
            CREATE TABLE IF NOT EXISTS schema_migrations (
                version INTEGER PRIMARY KEY,
                description TEXT,
                applied_at TEXT DEFAULT (datetime('now'))
            )
            ''')
            conn.commit()
    
    def current_version(self) -> int:
        with self._get_conn() as conn:
            cursor = conn.execute(
                "SELECT MAX(version) FROM schema_migrations"
            )
            result = cursor.fetchone()[0]
            return result or 0
    
    def migrate(self, target_version: int = None):
        current = self.current_version()
        
        if target_version is None:
            target_version = max(m['version'] for m in self.MIGRATIONS)
        
        for migration in sorted(self.MIGRATIONS, key=lambda m: m['version']):
            if migration['version'] > current and migration['version'] <= target_version:
                print(f"Applying migration {migration['version']}: {migration['description']}")
                with self._get_conn() as conn:
                    conn.executescript(migration['up'])
                    conn.execute(
                        "INSERT INTO schema_migrations (version, description) VALUES (?, ?)",
                        (migration['version'], migration['description'])
                    )
                    conn.commit()
                print(f"  Done!")

# ใช้งาน
migration = SQLiteMigration(':memory:')
migration.migrate()
print(f"Current version: {migration.current_version()}")
```

---

## 8. SQLite ใน Browser - sql.js

### 8.1 การใช้งาน sql.js

```html
<!DOCTYPE html>
<html>
<head>
    <script src="https://cdnjs.cloudflare.com/ajax/libs/sql.js/1.10.2/sql-wasm.js"></script>
</head>
<body>
<script>
// โหลด sql.js
initSqlJs({ locateFile: file => `https://cdnjs.cloudflare.com/ajax/libs/sql.js/1.10.2/${file}` })
.then(function(SQL) {
    // สร้าง in-memory database
    const db = new SQL.Database();
    
    // สร้างตาราง
    db.run(`
        CREATE TABLE users (id INTEGER PRIMARY KEY, name TEXT, age INTEGER);
        INSERT INTO users VALUES (1, 'Alice', 25);
        INSERT INTO users VALUES (2, 'Bob', 30);
    `);
    
    // Query ข้อมูล
    const result = db.exec("SELECT * FROM users WHERE age > 20");
    console.log(result);
    // [{columns: ['id', 'name', 'age'], values: [[1, 'Alice', 25], [2, 'Bob', 30]]}]
    
    // Parameterized query
    const stmt = db.prepare("SELECT * FROM users WHERE age > $minAge");
    stmt.bind({$minAge: 26});
    while (stmt.step()) {
        const row = stmt.getAsObject();
        console.log(row);  // {id: 2, name: 'Bob', age: 30}
    }
    stmt.free();
    
    // Export database เป็น Uint8Array (สำหรับ download)
    const data = db.export();
    const blob = new Blob([data], {type: 'application/octet-stream'});
    const url = URL.createObjectURL(blob);
    
    // Download
    const a = document.createElement('a');
    a.href = url;
    a.download = 'database.sqlite';
    a.click();
    
    // Load database จากไฟล์
    // const fileReader = new FileReader();
    // fileReader.onload = function() {
    //     const uInt8 = new Uint8Array(this.result);
    //     const db = new SQL.Database(uInt8);
    // };
    // fileReader.readAsArrayBuffer(file);
});
</script>
</body>
</html>
```

---

## 9. SQLite Performance Tuning

### 9.1 Index Optimization

```sql
-- สร้าง indexes อย่างถูกต้อง
CREATE TABLE orders (
    id INTEGER PRIMARY KEY,
    customer_id INTEGER NOT NULL,
    status TEXT NOT NULL,
    total REAL,
    created_at TEXT
);

-- Composite index สำหรับ common queries
CREATE INDEX idx_orders_customer_status ON orders(customer_id, status);
CREATE INDEX idx_orders_status_date ON orders(status, created_at);

-- Partial index: index เฉพาะ rows ที่ตรงเงื่อนไข
CREATE INDEX idx_orders_pending ON orders(created_at) 
WHERE status = 'pending';

-- Expression index
CREATE INDEX idx_orders_year ON orders(strftime('%Y', created_at));

-- ใช้ EXPLAIN QUERY PLAN เพื่อ debug
EXPLAIN QUERY PLAN
SELECT * FROM orders 
WHERE customer_id = 101 AND status = 'pending';

-- ANALYZE: อัพเดท statistics สำหรับ query planner
ANALYZE orders;
ANALYZE;  -- analyze ทุก table

-- ดู query plan อย่างละเอียด
.eqp on  -- ใน sqlite3 CLI
SELECT * FROM orders WHERE customer_id = 101;
```

### 9.2 Batch Operations

```sql
-- INSERT ทีละหลายแถวด้วย transaction
-- ช้า: ทุก INSERT เป็น transaction ของตัวเอง
INSERT INTO data VALUES (1, 'a');
INSERT INTO data VALUES (2, 'b');
-- ...

-- เร็ว: รวมใน transaction เดียว
BEGIN;
INSERT INTO data VALUES (1, 'a');
INSERT INTO data VALUES (2, 'b');
-- ...หลายพัน rows...
COMMIT;

-- เร็วมาก: ใช้ executemany ใน Python
conn.executemany("INSERT INTO data VALUES (?, ?)", [(i, f'val_{i}') for i in range(10000)])

-- ใช้ WITHOUT ROWID สำหรับตารางที่ใช้ TEXT primary key
CREATE TABLE sessions (
    session_token TEXT PRIMARY KEY,
    user_id INTEGER,
    expires_at TEXT
) WITHOUT ROWID;

-- STRICT mode (SQLite 3.37+): บังคับ type checking
CREATE TABLE strict_table (
    id INTEGER PRIMARY KEY,
    name TEXT NOT NULL,
    price REAL,
    count INTEGER
) STRICT;

-- จะ error ถ้าใส่ type ผิด
-- INSERT INTO strict_table VALUES (1, 'test', 'not-a-number', 5);  -- ERROR
```

---

## 10. SQLite Cloud

SQLite Cloud เป็น service ที่ให้ SQLite แบบ cloud-hosted

```python
# ใช้งาน sqlitecloud (pip install sqlitecloud)
import sqlitecloud

# เชื่อมต่อกับ SQLite Cloud
conn = sqlitecloud.connect("sqlitecloud://your-node.sqlite.cloud:8860/mydb?apikey=YOUR_API_KEY")

# ใช้งานเหมือน sqlite3 ปกติ
conn.execute("CREATE TABLE IF NOT EXISTS users (id INTEGER PRIMARY KEY, name TEXT)")
conn.execute("INSERT INTO users VALUES (1, 'Alice')")

cursor = conn.execute("SELECT * FROM users")
print(cursor.fetchall())

conn.close()
```

---

## แบบฝึกหัด

### ข้อที่ 1: SQLite WAL Mode Setup
ตั้งค่า SQLite database ด้วย optimal settings

**เฉลย:**
```python
import sqlite3

def create_optimized_db(db_path):
    conn = sqlite3.connect(db_path)
    
    # Performance settings
    pragmas = [
        "PRAGMA journal_mode = WAL",
        "PRAGMA synchronous = NORMAL",
        "PRAGMA cache_size = -64000",  # 64MB
        "PRAGMA temp_store = MEMORY",
        "PRAGMA mmap_size = 134217728",  # 128MB
        "PRAGMA foreign_keys = ON",
        "PRAGMA page_size = 4096",
    ]
    
    for pragma in pragmas:
        conn.execute(pragma)
    
    return conn

conn = create_optimized_db(':memory:')
cursor = conn.execute("PRAGMA journal_mode")
print(f"Journal mode: {cursor.fetchone()[0]}")
```

### ข้อที่ 2: Full-Text Search สำหรับ Product Search

**เฉลย:**
```python
import sqlite3

conn = sqlite3.connect(':memory:')
conn.executescript('''
    CREATE TABLE products (
        id INTEGER PRIMARY KEY,
        name TEXT NOT NULL,
        description TEXT,
        price REAL,
        category TEXT
    );
    
    CREATE VIRTUAL TABLE products_fts USING fts5(
        name, description, category,
        content=products,
        content_rowid=id
    );
    
    -- Trigger เพื่อ sync FTS กับ products table
    CREATE TRIGGER products_ai AFTER INSERT ON products BEGIN
        INSERT INTO products_fts(rowid, name, description, category)
        VALUES (new.id, new.name, new.description, new.category);
    END;
    
    CREATE TRIGGER products_au AFTER UPDATE ON products BEGIN
        INSERT INTO products_fts(products_fts, rowid, name, description, category)
        VALUES ('delete', old.id, old.name, old.description, old.category);
        INSERT INTO products_fts(rowid, name, description, category)
        VALUES (new.id, new.name, new.description, new.category);
    END;
''')

# Insert test data
products = [
    ('iPhone 15 Pro', 'Latest Apple smartphone with A17 Pro chip', 45000, 'Mobile'),
    ('Samsung Galaxy S24', 'Android flagship with AI features', 32000, 'Mobile'),
    ('MacBook Pro M3', 'Professional laptop for developers', 89000, 'Laptop'),
    ('Dell XPS 15', 'Windows laptop with OLED display', 65000, 'Laptop'),
]

conn.executemany(
    "INSERT INTO products (name, description, price, category) VALUES (?, ?, ?, ?)",
    products
)
conn.commit()

# Search
def search_products(conn, keyword):
    cursor = conn.execute('''
        SELECT p.id, p.name, p.price, p.category,
               bm25(products_fts) AS score
        FROM products p
        JOIN products_fts ON p.id = products_fts.rowid
        WHERE products_fts MATCH ?
        ORDER BY score
    ''', (keyword,))
    return cursor.fetchall()

results = search_products(conn, 'smartphone OR mobile')
for r in results:
    print(f"{r[1]}: {r[2]:,.0f} THB (score: {r[4]:.4f})")
```

### ข้อที่ 3: JSON Data ใน SQLite

**เฉลย:**
```python
import sqlite3
import json

conn = sqlite3.connect(':memory:')
conn.execute('''
    CREATE TABLE user_events (
        id INTEGER PRIMARY KEY,
        user_id INTEGER,
        event_data TEXT,  -- JSON
        created_at TEXT DEFAULT (datetime('now'))
    )
''')

events = [
    (1, json.dumps({'type': 'login', 'ip': '192.168.1.1', 'device': 'mobile'})),
    (1, json.dumps({'type': 'purchase', 'amount': 1500, 'product': 'iPhone'})),
    (2, json.dumps({'type': 'login', 'ip': '10.0.0.1', 'device': 'desktop'})),
    (2, json.dumps({'type': 'view', 'page': '/products', 'duration': 120})),
]

conn.executemany("INSERT INTO user_events (user_id, event_data) VALUES (?, ?)", events)
conn.commit()

# Query JSON data
cursor = conn.execute('''
    SELECT 
        user_id,
        json_extract(event_data, '$.type') AS event_type,
        json_extract(event_data, '$.amount') AS amount
    FROM user_events
    WHERE json_extract(event_data, '$.type') = 'purchase'
''')

for row in cursor:
    print(f"User {row[0]}: {row[1]}, Amount: {row[2]}")
```

### ข้อที่ 4: SQLite R-Tree สำหรับ Location Search

**เฉลย:**
```python
import sqlite3
import math

conn = sqlite3.connect(':memory:')
conn.executescript('''
    CREATE TABLE places (
        id INTEGER PRIMARY KEY,
        name TEXT,
        latitude REAL,
        longitude REAL,
        category TEXT
    );
    
    CREATE VIRTUAL TABLE places_rtree USING rtree(
        id, min_lat, max_lat, min_lon, max_lon
    );
    
    CREATE TRIGGER places_ai AFTER INSERT ON places BEGIN
        INSERT INTO places_rtree VALUES (
            new.id, new.latitude, new.latitude,
            new.longitude, new.longitude
        );
    END;
''')

# เพิ่มสถานที่ในกรุงเทพ
places = [
    ('Central World', 13.7467, 100.5394, 'mall'),
    ('Siam Paragon', 13.7464, 100.5332, 'mall'),
    ('MBK Center', 13.7449, 100.5297, 'mall'),
    ('Chatuchak Market', 13.7997, 100.5499, 'market'),
    ('Asiatique', 13.7000, 100.5106, 'entertainment'),
]

conn.executemany(
    "INSERT INTO places (name, latitude, longitude, category) VALUES (?, ?, ?, ?)",
    places
)
conn.commit()

def find_nearby(conn, lat, lon, radius_km, category=None):
    # Convert km to degrees (approximate)
    lat_deg = radius_km / 111.0
    lon_deg = radius_km / (111.0 * math.cos(math.radians(lat)))
    
    query = '''
        SELECT p.name, p.latitude, p.longitude, p.category
        FROM places p
        JOIN places_rtree r ON p.id = r.id
        WHERE r.min_lat >= ? AND r.max_lat <= ?
        AND r.min_lon >= ? AND r.max_lon <= ?
    '''
    params = [lat - lat_deg, lat + lat_deg, lon - lon_deg, lon + lon_deg]
    
    if category:
        query += " AND p.category = ?"
        params.append(category)
    
    return conn.execute(query, params).fetchall()

# ค้นหาห้างใกล้ Siam (13.7467, 100.5394)
nearby = find_nearby(conn, 13.7467, 100.5394, radius_km=1, category='mall')
print("Malls within 1km of Siam:")
for place in nearby:
    print(f"  {place[0]}")
```

### ข้อที่ 5: SQLite Backup System

**เฉลย:**
```python
import sqlite3
import shutil
import os
from datetime import datetime

def backup_sqlite_db(source_path: str, backup_dir: str) -> str:
    """Backup SQLite database with online backup API"""
    os.makedirs(backup_dir, exist_ok=True)
    
    timestamp = datetime.now().strftime('%Y%m%d_%H%M%S')
    backup_path = os.path.join(backup_dir, f'backup_{timestamp}.db')
    
    # ใช้ backup API ที่ safe กว่า file copy
    source_conn = sqlite3.connect(source_path)
    backup_conn = sqlite3.connect(backup_path)
    
    source_conn.backup(backup_conn, pages=100, progress=lambda status, remaining, total: 
        print(f"Backed up {total-remaining}/{total} pages"))
    
    backup_conn.close()
    source_conn.close()
    
    size = os.path.getsize(backup_path)
    print(f"Backup created: {backup_path} ({size:,} bytes)")
    return backup_path

# ทดสอบ backup
backup_path = backup_sqlite_db('production.db', '/tmp/backups')
```

### ข้อที่ 6: SQLite ใช้กับ Pandas

**เฉลย:**
```python
import sqlite3
import pandas as pd
from datetime import datetime, timedelta
import random

# สร้าง test database
conn = sqlite3.connect(':memory:')
conn.execute('''
    CREATE TABLE sales (
        id INTEGER PRIMARY KEY,
        date TEXT,
        product TEXT,
        category TEXT,
        quantity INTEGER,
        revenue REAL
    )
''')

# สร้าง sample data
products = ['iPhone', 'iPad', 'MacBook', 'AirPods', 'Apple Watch']
categories = ['Mobile', 'Tablet', 'Laptop', 'Accessory', 'Wearable']

data = []
base_date = datetime(2024, 1, 1)
for i in range(1000):
    idx = random.randint(0, 4)
    data.append((
        (base_date + timedelta(days=random.randint(0, 365))).strftime('%Y-%m-%d'),
        products[idx],
        categories[idx],
        random.randint(1, 10),
        random.uniform(1000, 100000)
    ))

conn.executemany(
    "INSERT INTO sales (date, product, category, quantity, revenue) VALUES (?, ?, ?, ?, ?)",
    data
)
conn.commit()

# ใช้ Pandas กับ SQLite
df = pd.read_sql_query('''
    SELECT 
        strftime('%Y-%m', date) AS month,
        category,
        SUM(quantity) AS total_qty,
        SUM(revenue) AS total_revenue,
        AVG(revenue) AS avg_revenue
    FROM sales
    GROUP BY month, category
    ORDER BY month, total_revenue DESC
''', conn)

print(df.head(20))
print(f"\nTotal revenue: {df['total_revenue'].sum():,.0f} THB")

# Export ไป SQLite
summary_df = df.groupby('category').agg({
    'total_revenue': 'sum',
    'total_qty': 'sum'
}).reset_index()

summary_df.to_sql('sales_summary', conn, if_exists='replace', index=False)
print("\nSummary saved to database")
```

### ข้อที่ 7: SQLite สำหรับ Testing

**เฉลย:**
```python
import sqlite3
import unittest
from contextlib import contextmanager

# Database setup functions
def create_schema(conn):
    conn.executescript('''
        CREATE TABLE IF NOT EXISTS products (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            name TEXT NOT NULL,
            price REAL NOT NULL CHECK (price > 0),
            stock INTEGER NOT NULL DEFAULT 0
        );
        
        CREATE TABLE IF NOT EXISTS orders (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            product_id INTEGER NOT NULL REFERENCES products(id),
            quantity INTEGER NOT NULL CHECK (quantity > 0),
            total_price REAL,
            created_at TEXT DEFAULT (datetime('now'))
        );
        
        CREATE TRIGGER calculate_total BEFORE INSERT ON orders
        BEGIN
            UPDATE orders SET total_price = 
                (SELECT price FROM products WHERE id = NEW.product_id) * NEW.quantity
            WHERE id = NEW.id;
        END;
    ''')

class TestDatabaseOperations(unittest.TestCase):
    
    def setUp(self):
        """สร้าง in-memory database สำหรับทุก test"""
        self.conn = sqlite3.connect(':memory:')
        self.conn.execute("PRAGMA foreign_keys = ON")
        self.conn.row_factory = sqlite3.Row
        create_schema(self.conn)
        
        # Insert test data
        self.conn.execute("INSERT INTO products (name, price, stock) VALUES ('Test Product', 100.0, 10)")
        self.conn.commit()
    
    def tearDown(self):
        self.conn.close()
    
    def test_insert_product(self):
        self.conn.execute("INSERT INTO products (name, price) VALUES ('New Product', 200.0)")
        self.conn.commit()
        
        cursor = self.conn.execute("SELECT COUNT(*) FROM products")
        self.assertEqual(cursor.fetchone()[0], 2)
    
    def test_negative_price_fails(self):
        with self.assertRaises(sqlite3.IntegrityError):
            self.conn.execute("INSERT INTO products (name, price) VALUES ('Bad Product', -10.0)")
    
    def test_foreign_key_constraint(self):
        with self.assertRaises(sqlite3.IntegrityError):
            self.conn.execute("INSERT INTO orders (product_id, quantity) VALUES (999, 1)")
    
    def test_order_creation(self):
        cursor = self.conn.execute(
            "INSERT INTO orders (product_id, quantity) VALUES (1, 3)"
        )
        self.conn.commit()
        
        cursor = self.conn.execute("SELECT * FROM orders WHERE id = ?", (cursor.lastrowid,))
        order = cursor.fetchone()
        self.assertIsNotNone(order)

if __name__ == '__main__':
    unittest.main()
```

### ข้อที่ 8: SQLite Recursive CTE

**เฉลย:**
```sql
-- สร้าง bill of materials (BOM) ใน SQLite
CREATE TABLE components (
    id INTEGER PRIMARY KEY,
    name TEXT NOT NULL,
    parent_id INTEGER REFERENCES components(id),
    quantity INTEGER DEFAULT 1,
    cost REAL DEFAULT 0
);

INSERT INTO components (id, name, parent_id, cost) VALUES
(1, 'Laptop', NULL, 0),
(2, 'CPU', 1, 15000),
(3, 'Motherboard', 1, 8000),
(4, 'RAM Slots', 3, 0),
(5, 'RAM 8GB', 4, 1500),
(6, 'RAM 8GB', 4, 1500),
(7, 'Storage', 1, 0),
(8, 'SSD 512GB', 7, 4000),
(9, 'Display', 1, 12000);

-- Recursive CTE แสดง BOM tree
WITH RECURSIVE bom (id, name, parent_id, level, path, total_cost) AS (
    SELECT id, name, parent_id, 0, name, cost
    FROM components WHERE parent_id IS NULL
    
    UNION ALL
    
    SELECT c.id, c.name, c.parent_id, b.level + 1,
           b.path || ' > ' || c.name,
           c.cost
    FROM components c
    JOIN bom b ON c.parent_id = b.id
)
SELECT 
    substr('          ', 1, level * 2) || name AS component,
    level,
    total_cost,
    path
FROM bom
ORDER BY path;

-- คำนวณต้นทุนรวม
WITH RECURSIVE bom AS (
    SELECT id, name, parent_id, cost
    FROM components WHERE parent_id IS NULL
    UNION ALL
    SELECT c.id, c.name, c.parent_id, c.cost
    FROM components c JOIN bom b ON c.parent_id = b.id
)
SELECT SUM(cost) AS total_laptop_cost FROM bom;
```

### ข้อที่ 9: SQLite FTS5 Advanced Search

**เฉลย:**
```python
import sqlite3

conn = sqlite3.connect(':memory:')
conn.executescript('''
    CREATE VIRTUAL TABLE docs USING fts5(
        title, 
        body,
        tokenize = 'unicode61 remove_diacritics 1'
    );
    
    INSERT INTO docs VALUES 
        ('Python Tutorial', 'Learn Python programming language basics and advanced topics'),
        ('SQL Fundamentals', 'Database management with SQL including SELECT UPDATE DELETE'),
        ('Web Development', 'Build websites with HTML CSS JavaScript and Python'),
        ('Machine Learning', 'AI and ML with Python scikit-learn tensorflow'),
        ('Data Analysis', 'Analyze data using Python pandas numpy matplotlib');
''')

def advanced_search(conn, query, highlight=True):
    if highlight:
        cursor = conn.execute('''
            SELECT 
                highlight(docs, 0, '<mark>', '</mark>') AS title,
                snippet(docs, 1, '<mark>', '</mark>', '...', 20) AS excerpt,
                rank
            FROM docs
            WHERE docs MATCH ?
            ORDER BY rank
        ''', (query,))
    else:
        cursor = conn.execute(
            "SELECT title, body, rank FROM docs WHERE docs MATCH ? ORDER BY rank",
            (query,)
        )
    return cursor.fetchall()

# ทดสอบ
results = advanced_search(conn, 'Python AND (tutorial OR learning)')
for title, excerpt, rank in results:
    print(f"Title: {title}")
    print(f"Excerpt: {excerpt}")
    print(f"Rank: {rank:.4f}")
    print("---")
```

### ข้อที่ 10: SQLite Configuration Profile

**เฉลย:**
```python
import sqlite3
import os

def setup_production_sqlite(db_path: str, read_only: bool = False) -> sqlite3.Connection:
    """Setup SQLite ด้วย production-ready configuration"""
    
    if read_only:
        uri = f'file:{db_path}?mode=ro'
        conn = sqlite3.connect(f'file:{db_path}?mode=ro', uri=True)
    else:
        conn = sqlite3.connect(db_path)
    
    # Performance optimizations
    conn.execute("PRAGMA journal_mode = WAL")
    conn.execute("PRAGMA synchronous = NORMAL")
    conn.execute("PRAGMA cache_size = -131072")  # 128MB
    conn.execute("PRAGMA temp_store = MEMORY")
    conn.execute("PRAGMA mmap_size = 268435456")  # 256MB
    conn.execute("PRAGMA page_size = 4096")
    
    # Safety settings
    conn.execute("PRAGMA foreign_keys = ON")
    conn.execute("PRAGMA recursive_triggers = ON")
    
    if not read_only:
        conn.execute("PRAGMA wal_autocheckpoint = 1000")
    
    conn.row_factory = sqlite3.Row
    
    return conn

def get_db_stats(conn: sqlite3.Connection) -> dict:
    """ดูสถิติ database"""
    stats = {}
    
    pragmas = ['page_count', 'freelist_count', 'page_size', 'wal_checkpoint']
    for pragma in pragmas:
        try:
            cursor = conn.execute(f"PRAGMA {pragma}")
            stats[pragma] = cursor.fetchone()[0]
        except:
            pass
    
    stats['db_size_mb'] = stats.get('page_count', 0) * stats.get('page_size', 4096) / 1024 / 1024
    
    # Table sizes
    cursor = conn.execute("""
        SELECT name, 
               (SELECT COUNT(*) FROM sqlite_master WHERE type='index' AND tbl_name=m.name) as index_count
        FROM sqlite_master m 
        WHERE type='table' AND name NOT LIKE 'sqlite_%'
    """)
    stats['tables'] = [{'name': row['name'], 'indexes': row['index_count']} for row in cursor]
    
    return stats

# ใช้งาน
conn = setup_production_sqlite(':memory:')
conn.execute("CREATE TABLE test (id INTEGER PRIMARY KEY, data TEXT)")
conn.execute("INSERT INTO test VALUES (1, 'hello')")
conn.commit()

stats = get_db_stats(conn)
print(f"Database size: {stats['db_size_mb']:.2f} MB")
print(f"Tables: {stats['tables']}")
```

---

## สรุป

ในบทนี้เราได้เรียนรู้ SQLite อย่างครบถ้วน:

1. **Use Cases** - เมื่อไรควรใช้ SQLite และเมื่อไรไม่ควร
2. **Type Affinity** - ระบบ types ที่ยืดหยุ่นของ SQLite
3. **Built-in Functions** - string, math, date/time functions
4. **WAL Mode** - เพิ่ม performance concurrent access
5. **FTS5** - full-text search engine ที่ทรงพลัง
6. **R-Tree** - spatial indexing สำหรับ location data
7. **JSON1 Extension** - จัดการ JSON data
8. **Python Integration** - sqlite3 module พร้อม patterns
9. **Migration System** - จัดการ schema changes
10. **Performance Tuning** - indexes, batch operations, PRAGMA settings

SQLite เป็น database ที่ "just works" ไม่ต้อง setup server ไม่ต้อง maintain และมีประสิทธิภาพสูงสำหรับ use cases ที่เหมาะสม
