# ตอนที่ 102: MySQL/MariaDB Advanced Features - คุณสมบัติขั้นสูงของ MySQL และ MariaDB

## บทนำ

MySQL และ MariaDB เป็นระบบฐานข้อมูลที่ได้รับความนิยมสูงมากในโลก โดยเฉพาะสำหรับ web applications MySQL 8.0 นำเสนอคุณสมบัติขั้นสูงมากมายที่เทียบเคียงกับ PostgreSQL ได้ ในบทนี้เราจะศึกษาคุณสมบัติเหล่านี้อย่างละเอียด

---

## 1. MySQL JSON Functions Deep Dive

MySQL 5.7+ มี native JSON support ที่ช่วยให้จัดการข้อมูล JSON ได้อย่างมีประสิทธิภาพ

### 1.1 JSON Data Type และ Basic Operations

```sql
-- สร้างตารางที่ใช้ JSON
CREATE TABLE user_profiles (
    id INT AUTO_INCREMENT PRIMARY KEY,
    username VARCHAR(50) NOT NULL,
    profile JSON,
    settings JSON,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- แทรกข้อมูล JSON
INSERT INTO user_profiles (username, profile, settings) VALUES
('somchai', 
 '{"name": "สมชาย ใจดี", "age": 28, "city": "Bangkok", "hobbies": ["reading", "coding", "gaming"]}',
 '{"theme": "dark", "language": "th", "notifications": {"email": true, "sms": false}}'),
('somying',
 '{"name": "สมหญิง สวยงาม", "age": 25, "city": "Chiang Mai", "hobbies": ["photography", "travel"]}',
 '{"theme": "light", "language": "en", "notifications": {"email": false, "sms": true}}');

-- ดึงค่าจาก JSON
SELECT 
    username,
    profile->>'$.name' AS full_name,
    profile->>'$.city' AS city,
    JSON_EXTRACT(profile, '$.age') AS age
FROM user_profiles;

-- JSON_EXTRACT หลายค่า
SELECT 
    username,
    JSON_EXTRACT(profile, '$.name', '$.city') AS extracted_data
FROM user_profiles;

-- แก้ไขค่าใน JSON
UPDATE user_profiles
SET profile = JSON_SET(profile, '$.age', 29, '$.verified', true)
WHERE username = 'somchai';

-- JSON_SET vs JSON_INSERT vs JSON_REPLACE
-- JSON_SET: แทนที่ถ้ามีอยู่, เพิ่มถ้าไม่มี
-- JSON_INSERT: เพิ่มถ้าไม่มีอยู่, ไม่แตะถ้ามีอยู่แล้ว
-- JSON_REPLACE: แทนที่ถ้ามีอยู่, ไม่เพิ่มถ้าไม่มี

UPDATE user_profiles
SET profile = JSON_REMOVE(profile, '$.verified')
WHERE username = 'somchai';

-- JSON_MERGE_PATCH: merge JSON objects
SELECT JSON_MERGE_PATCH(
    '{"name": "Alice", "age": 30}',
    '{"age": 31, "city": "Bangkok"}'
);
-- Result: {"age": 31, "city": "Bangkok", "name": "Alice"}

-- JSON_MERGE_PRESERVE: merge พร้อมเก็บค่า duplicate
SELECT JSON_MERGE_PRESERVE(
    '{"a": 1}',
    '{"a": 2}'
);
-- Result: {"a": [1, 2]}
```

### 1.2 JSON Array Operations

```sql
-- JSON_ARRAY: สร้าง JSON array
SELECT JSON_ARRAY(1, 'two', NULL, TRUE, '{"key": "value"}');

-- JSON_ARRAYAGG: aggregate เป็น JSON array
SELECT JSON_ARRAYAGG(username) AS all_users FROM user_profiles;

-- เข้าถึง array elements
SELECT 
    username,
    profile->>'$.hobbies[0]' AS first_hobby,
    profile->>'$.hobbies[1]' AS second_hobby
FROM user_profiles;

-- JSON_CONTAINS: ตรวจสอบว่ามีค่าใน JSON
SELECT username
FROM user_profiles
WHERE JSON_CONTAINS(profile->>'$.hobbies', '"coding"');

-- JSON_CONTAINS_PATH: ตรวจสอบว่า path มีอยู่
SELECT username
FROM user_profiles
WHERE JSON_CONTAINS_PATH(profile, 'one', '$.hobbies');

-- JSON_SEARCH: ค้นหาค่าใน JSON
SELECT 
    username,
    JSON_SEARCH(profile, 'all', 'coding') AS coding_path
FROM user_profiles;

-- JSON_OVERLAPS (MySQL 8.0.17+): ตรวจสอบการ overlap
SELECT JSON_OVERLAPS('[1,2,3]', '[3,4,5]');  -- 1 (true)

-- JSON_TABLE: แปลง JSON เป็น table
SELECT jt.*
FROM user_profiles,
JSON_TABLE(
    profile->'$.hobbies',
    '$[*]' COLUMNS (
        hobby VARCHAR(50) PATH '$'
    )
) AS jt
WHERE username = 'somchai';
```

### 1.3 JSON Path และ Advanced Functions

```sql
-- JSON_KEYS: ดู keys ทั้งหมด
SELECT JSON_KEYS(profile) AS profile_keys FROM user_profiles LIMIT 1;

-- JSON_LENGTH: ความยาวของ JSON array หรือ object
SELECT 
    username,
    JSON_LENGTH(profile) AS profile_fields,
    JSON_LENGTH(profile, '$.hobbies') AS num_hobbies
FROM user_profiles;

-- JSON_TYPE: ดู type ของ JSON value
SELECT JSON_TYPE('{"key": "value"}');  -- OBJECT
SELECT JSON_TYPE('[1,2,3]');            -- ARRAY
SELECT JSON_TYPE('"hello"');            -- STRING
SELECT JSON_TYPE('42');                 -- INTEGER

-- JSON_QUOTE: escape สำหรับ JSON string
SELECT JSON_QUOTE('Hello "World"');

-- JSON_UNQUOTE: ถอด quotes
SELECT JSON_UNQUOTE('"Hello World"');

-- JSON_VALID: ตรวจสอบว่าเป็น valid JSON
SELECT JSON_VALID('{"key": "value"}');  -- 1
SELECT JSON_VALID('{invalid}');          -- 0

-- JSON_SCHEMA_VALID (MySQL 8.0.17+): validate ต่อ JSON schema
SELECT JSON_SCHEMA_VALID(
    '{"type": "object", "properties": {"name": {"type": "string"}, "age": {"type": "number"}}, "required": ["name"]}',
    '{"name": "Alice", "age": 30}'
);  -- 1

-- Functional index บน JSON column
CREATE INDEX idx_user_city 
ON user_profiles ((profile->>'$.city'));

-- ใช้งาน index
SELECT username FROM user_profiles
WHERE profile->>'$.city' = 'Bangkok';
EXPLAIN SELECT username FROM user_profiles WHERE profile->>'$.city' = 'Bangkok';
```

---

## 2. Generated Columns

Generated columns ช่วยให้คำนวณค่าอัตโนมัติจากคอลัมน์อื่น

### 2.1 Virtual และ Stored Generated Columns

```sql
-- สร้างตารางที่มี generated columns
CREATE TABLE order_items (
    id INT AUTO_INCREMENT PRIMARY KEY,
    product_name VARCHAR(100),
    quantity INT,
    unit_price DECIMAL(10,2),
    discount_percent DECIMAL(5,2) DEFAULT 0,
    
    -- Virtual column: คำนวณทุกครั้งที่ query
    subtotal DECIMAL(12,2) GENERATED ALWAYS AS (quantity * unit_price) VIRTUAL,
    
    -- Stored column: คำนวณและเก็บไว้ใน disk
    total_after_discount DECIMAL(12,2) GENERATED ALWAYS AS 
        (quantity * unit_price * (1 - discount_percent/100)) STORED
);

INSERT INTO order_items (product_name, quantity, unit_price, discount_percent)
VALUES 
('iPhone 15', 2, 45000, 10),
('MacBook Pro', 1, 89000, 5),
('iPad Air', 3, 22000, 0);

SELECT product_name, quantity, unit_price, discount_percent, subtotal, total_after_discount
FROM order_items;

-- Index บน generated column
CREATE INDEX idx_total ON order_items (total_after_discount);

-- ค้นหาโดยใช้ generated column
SELECT * FROM order_items WHERE total_after_discount > 50000;
```

### 2.2 JSON + Generated Columns

```sql
-- ใช้ generated column ดึงค่าจาก JSON เพื่อ index
CREATE TABLE events (
    id INT AUTO_INCREMENT PRIMARY KEY,
    event_data JSON,
    
    -- ดึง event_type จาก JSON เพื่อ index
    event_type VARCHAR(50) GENERATED ALWAYS AS (event_data->>'$.type') STORED,
    -- ดึง user_id จาก JSON เพื่อ index
    user_id INT GENERATED ALWAYS AS (event_data->>'$.user_id') STORED,
    
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    
    INDEX idx_event_type (event_type),
    INDEX idx_user_id (user_id)
);

INSERT INTO events (event_data) VALUES
('{"type": "login", "user_id": 101, "ip": "192.168.1.1"}'),
('{"type": "purchase", "user_id": 101, "amount": 1500}'),
('{"type": "login", "user_id": 202, "ip": "10.0.0.1"}');

-- Query ที่ใช้ generated column index
SELECT * FROM events WHERE event_type = 'login' AND user_id = 101;
EXPLAIN SELECT * FROM events WHERE event_type = 'login';
```

---

## 3. Window Functions ใน MySQL 8+

### 3.1 Ranking Functions

```sql
CREATE TABLE sales (
    id INT AUTO_INCREMENT PRIMARY KEY,
    salesperson VARCHAR(50),
    region VARCHAR(30),
    sale_date DATE,
    amount DECIMAL(10,2)
);

INSERT INTO sales (salesperson, region, sale_date, amount) VALUES
('Alice', 'North', '2024-01-15', 25000),
('Bob', 'North', '2024-01-20', 18000),
('Charlie', 'South', '2024-01-10', 32000),
('Diana', 'South', '2024-01-25', 28000),
('Eve', 'North', '2024-02-01', 22000),
('Frank', 'South', '2024-02-10', 35000);

-- ROW_NUMBER
SELECT 
    salesperson,
    region,
    amount,
    ROW_NUMBER() OVER (PARTITION BY region ORDER BY amount DESC) AS rn
FROM sales;

-- RANK และ DENSE_RANK
SELECT 
    salesperson,
    amount,
    RANK() OVER (ORDER BY amount DESC) AS rank_pos,
    DENSE_RANK() OVER (ORDER BY amount DESC) AS dense_rank_pos
FROM sales;

-- NTILE: แบ่งเป็น N กลุ่ม
SELECT 
    salesperson,
    amount,
    NTILE(3) OVER (ORDER BY amount DESC) AS quartile
FROM sales;

-- PERCENT_RANK และ CUME_DIST
SELECT 
    salesperson,
    amount,
    PERCENT_RANK() OVER (ORDER BY amount) AS pct_rank,
    CUME_DIST() OVER (ORDER BY amount) AS cum_dist
FROM sales;
```

### 3.2 Value Functions

```sql
-- LAG และ LEAD
SELECT 
    salesperson,
    sale_date,
    amount,
    LAG(amount) OVER (PARTITION BY salesperson ORDER BY sale_date) AS prev_amount,
    LEAD(amount) OVER (PARTITION BY salesperson ORDER BY sale_date) AS next_amount,
    amount - LAG(amount) OVER (PARTITION BY salesperson ORDER BY sale_date) AS change
FROM sales;

-- FIRST_VALUE และ LAST_VALUE
SELECT 
    region,
    salesperson,
    amount,
    FIRST_VALUE(salesperson) OVER (
        PARTITION BY region ORDER BY amount DESC
        ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
    ) AS top_seller,
    LAST_VALUE(amount) OVER (
        PARTITION BY region ORDER BY amount DESC
        ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
    ) AS min_sale
FROM sales;

-- NTH_VALUE
SELECT 
    region,
    salesperson,
    amount,
    NTH_VALUE(salesperson, 2) OVER (
        PARTITION BY region ORDER BY amount DESC
        ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
    ) AS second_best
FROM sales;
```

### 3.3 Aggregate Window Functions

```sql
-- Running total
SELECT 
    sale_date,
    salesperson,
    amount,
    SUM(amount) OVER (ORDER BY sale_date) AS running_total,
    AVG(amount) OVER (ORDER BY sale_date ROWS BETWEEN 2 PRECEDING AND CURRENT ROW) AS moving_avg_3
FROM sales;

-- Partition aggregate
SELECT 
    region,
    salesperson,
    amount,
    SUM(amount) OVER (PARTITION BY region) AS region_total,
    amount / SUM(amount) OVER (PARTITION BY region) * 100 AS pct_of_region
FROM sales;
```

---

## 4. MySQL 8 CTEs และ Recursive Queries

### 4.1 Common Table Expressions (CTEs)

```sql
-- Simple CTE
WITH monthly_sales AS (
    SELECT 
        DATE_FORMAT(sale_date, '%Y-%m') AS month,
        SUM(amount) AS total
    FROM sales
    GROUP BY DATE_FORMAT(sale_date, '%Y-%m')
)
SELECT month, total,
    total - LAG(total) OVER (ORDER BY month) AS growth
FROM monthly_sales;

-- Multiple CTEs
WITH 
top_salespeople AS (
    SELECT salesperson, SUM(amount) AS total_sales
    FROM sales
    GROUP BY salesperson
    HAVING total_sales > 40000
),
region_summary AS (
    SELECT region, AVG(amount) AS avg_sale
    FROM sales
    GROUP BY region
)
SELECT ts.salesperson, ts.total_sales, rs.region, rs.avg_sale
FROM top_salespeople ts
JOIN sales s ON ts.salesperson = s.salesperson
JOIN region_summary rs ON s.region = rs.region;
```

### 4.2 Recursive CTEs

```sql
-- Employee hierarchy
CREATE TABLE employees_hier (
    emp_id INT PRIMARY KEY,
    emp_name VARCHAR(100),
    manager_id INT,
    department VARCHAR(50),
    salary DECIMAL(10,2)
);

INSERT INTO employees_hier VALUES
(1, 'CEO John', NULL, 'Executive', 500000),
(2, 'VP Alice', 1, 'Sales', 300000),
(3, 'VP Bob', 1, 'Tech', 320000),
(4, 'Manager Charlie', 2, 'Sales', 150000),
(5, 'Manager Diana', 3, 'Tech', 160000),
(6, 'Staff Eve', 4, 'Sales', 80000),
(7, 'Staff Frank', 4, 'Sales', 75000),
(8, 'Developer Grace', 5, 'Tech', 120000),
(9, 'Developer Henry', 5, 'Tech', 115000);

-- Recursive CTE สำหรับ hierarchy
WITH RECURSIVE emp_hierarchy AS (
    -- Anchor: CEO
    SELECT 
        emp_id, 
        emp_name, 
        manager_id,
        salary,
        0 AS level,
        emp_name AS path
    FROM employees_hier
    WHERE manager_id IS NULL
    
    UNION ALL
    
    -- Recursive: ลูกน้องของแต่ละคน
    SELECT 
        e.emp_id,
        e.emp_name,
        e.manager_id,
        e.salary,
        eh.level + 1,
        CONCAT(eh.path, ' -> ', e.emp_name)
    FROM employees_hier e
    JOIN emp_hierarchy eh ON e.manager_id = eh.emp_id
)
SELECT 
    REPEAT('  ', level) || emp_name AS hierarchy_view,
    level,
    salary,
    path
FROM emp_hierarchy
ORDER BY path;

-- Fibonacci sequence
WITH RECURSIVE fibonacci (n, fib_n, fib_next) AS (
    SELECT 1, 0, 1
    UNION ALL
    SELECT n + 1, fib_next, fib_n + fib_next
    FROM fibonacci
    WHERE n < 20
)
SELECT n, fib_n AS fibonacci_value FROM fibonacci;

-- Date series generation
WITH RECURSIVE date_series (d) AS (
    SELECT '2024-01-01'::DATE
    UNION ALL
    SELECT DATE_ADD(d, INTERVAL 1 DAY)
    FROM date_series
    WHERE d < '2024-01-31'
)
SELECT d, DAYNAME(d) AS day_name FROM date_series;
```

---

## 5. InnoDB Specifics

### 5.1 InnoDB Architecture และ Configuration

```sql
-- ดูสถิติ InnoDB
SHOW ENGINE INNODB STATUS\G

-- Buffer pool statistics
SELECT 
    POOL_ID,
    POOL_SIZE * 16384 / 1024 / 1024 AS pool_size_mb,
    FREE_BUFFERS,
    DATABASE_PAGES,
    OLD_DATABASE_PAGES,
    MODIFIED_DATABASE_PAGES
FROM INFORMATION_SCHEMA.INNODB_BUFFER_POOL_STATS;

-- Transaction information
SELECT 
    trx_id,
    trx_state,
    trx_started,
    trx_query,
    trx_rows_locked,
    trx_rows_modified
FROM INFORMATION_SCHEMA.INNODB_TRX;

-- Lock information
SELECT 
    r.trx_id AS waiting_trx,
    r.trx_query AS waiting_query,
    b.trx_id AS blocking_trx,
    b.trx_query AS blocking_query
FROM INFORMATION_SCHEMA.INNODB_LOCK_WAITS w
JOIN INFORMATION_SCHEMA.INNODB_TRX b ON b.trx_id = w.blocking_trx_id
JOIN INFORMATION_SCHEMA.INNODB_TRX r ON r.trx_id = w.requesting_trx_id;

-- Clustered index
-- InnoDB เก็บข้อมูลตาม PRIMARY KEY (clustered)
CREATE TABLE innodb_example (
    id INT AUTO_INCREMENT,
    email VARCHAR(100) UNIQUE,
    name VARCHAR(100),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (id)  -- clustered index
) ENGINE=InnoDB;

-- Secondary index ใน InnoDB เก็บ primary key ด้วย
EXPLAIN SELECT * FROM innodb_example WHERE email = 'test@example.com';
```

### 5.2 InnoDB Locking

```sql
-- Shared lock (S lock)
SELECT * FROM orders WHERE id = 1 LOCK IN SHARE MODE;

-- Exclusive lock (X lock) 
SELECT * FROM orders WHERE id = 1 FOR UPDATE;

-- SKIP LOCKED: ข้าม rows ที่ถูก lock อยู่ (MySQL 8.0+)
SELECT * FROM job_queue
WHERE status = 'pending'
ORDER BY created_at
LIMIT 10
FOR UPDATE SKIP LOCKED;

-- NOWAIT: ไม่รอถ้า row ถูก lock (MySQL 8.0+)
SELECT * FROM orders WHERE id = 1 FOR UPDATE NOWAIT;

-- Gap locks และ Next-key locks
-- ป้องกัน phantom reads ใน REPEATABLE READ
BEGIN;
SELECT * FROM products WHERE price BETWEEN 100 AND 200 FOR UPDATE;
-- Gap lock ป้องกันไม่ให้ insert ข้อมูลใน range นี้
COMMIT;
```

---

## 6. MySQL Full-Text Search

### 6.1 FULLTEXT Index และ MATCH AGAINST

```sql
CREATE TABLE articles (
    id INT AUTO_INCREMENT PRIMARY KEY,
    title VARCHAR(200),
    content TEXT,
    author VARCHAR(100),
    FULLTEXT INDEX ft_title_content (title, content),
    FULLTEXT INDEX ft_title (title)
);

INSERT INTO articles (title, content, author) VALUES
('Introduction to MySQL', 'MySQL is a popular database management system...', 'John'),
('Advanced SQL Techniques', 'This article covers advanced SQL features including CTEs, window functions...', 'Alice'),
('Database Performance Tips', 'Optimize your MySQL queries with proper indexing strategies...', 'Bob'),
('MySQL vs PostgreSQL', 'Comparing two popular open source databases for modern applications...', 'Charlie');

-- Natural language search
SELECT title, author,
    MATCH(title, content) AGAINST('MySQL database') AS relevance
FROM articles
WHERE MATCH(title, content) AGAINST('MySQL database')
ORDER BY relevance DESC;

-- Boolean mode search
SELECT title, MATCH(title, content) AGAINST('+MySQL -PostgreSQL' IN BOOLEAN MODE) AS rel
FROM articles
WHERE MATCH(title, content) AGAINST('+MySQL -PostgreSQL' IN BOOLEAN MODE);

-- Boolean operators:
-- +word: word ต้องมี
-- -word: word ต้องไม่มี
-- *: wildcard เช่น MySQL* matches MySQL, MySQLite, etc.
-- "": phrase search
-- >word: เพิ่ม relevance
-- <word: ลด relevance
-- ~word: ใส่ negative weight
-- (group): grouping

SELECT title
FROM articles
WHERE MATCH(title, content) AGAINST('+"MySQL" +"advanced" -"basic"' IN BOOLEAN MODE);

-- Query expansion mode
SELECT title
FROM articles
WHERE MATCH(title, content) AGAINST('database' WITH QUERY EXPANSION);

-- ดูสถิติ fulltext index
SELECT * FROM INFORMATION_SCHEMA.INNODB_FT_INDEX_TABLE LIMIT 10;
```

### 6.2 ngram Full-Text Parser (สำหรับภาษา CJK)

```sql
-- สำหรับภาษาที่ไม่มี word boundary (จีน, ญี่ปุ่น, เกาหลี)
CREATE TABLE thai_articles (
    id INT AUTO_INCREMENT PRIMARY KEY,
    title VARCHAR(200),
    content TEXT,
    FULLTEXT INDEX ft_content (title, content) WITH PARSER ngram
);

-- ตั้งค่า ngram_token_size (default = 2)
-- SET GLOBAL ngram_token_size = 2;
```

---

## 7. MySQL Event Scheduler

Event Scheduler ช่วยให้รัน SQL statements ตามเวลาที่กำหนด

### 7.1 การสร้างและจัดการ Events

```sql
-- เปิด event scheduler
SET GLOBAL event_scheduler = ON;

-- ตรวจสอบสถานะ
SHOW VARIABLES LIKE 'event_scheduler';

-- สร้าง event ที่รันครั้งเดียว
CREATE EVENT one_time_cleanup
ON SCHEDULE AT CURRENT_TIMESTAMP + INTERVAL 1 HOUR
DO
    DELETE FROM temp_logs WHERE created_at < NOW() - INTERVAL 7 DAY;

-- สร้าง recurring event
CREATE EVENT daily_stats_update
ON SCHEDULE EVERY 1 DAY
STARTS '2024-01-01 00:00:00'
DO
BEGIN
    -- อัพเดทสถิติรายวัน
    INSERT INTO daily_stats (stat_date, total_users, total_orders, total_revenue)
    SELECT 
        CURDATE() - INTERVAL 1 DAY,
        COUNT(DISTINCT u.id),
        COUNT(DISTINCT o.id),
        COALESCE(SUM(o.total), 0)
    FROM users u
    LEFT JOIN orders o ON u.id = o.user_id 
        AND DATE(o.created_at) = CURDATE() - INTERVAL 1 DAY
    ON DUPLICATE KEY UPDATE
        total_users = VALUES(total_users),
        total_orders = VALUES(total_orders),
        total_revenue = VALUES(total_revenue);
END;

-- สร้าง event สำหรับ cleanup sessions
CREATE EVENT cleanup_expired_sessions
ON SCHEDULE EVERY 15 MINUTE
DO
    DELETE FROM user_sessions 
    WHERE expires_at < NOW() 
    AND created_at < NOW() - INTERVAL 24 HOUR;

-- ดู events
SHOW EVENTS;
SELECT * FROM INFORMATION_SCHEMA.EVENTS\G

-- แก้ไข event
ALTER EVENT daily_stats_update
ON SCHEDULE EVERY 6 HOUR;

-- หยุด/เริ่ม event
ALTER EVENT daily_stats_update DISABLE;
ALTER EVENT daily_stats_update ENABLE;

-- ลบ event
DROP EVENT IF EXISTS one_time_cleanup;
```

---

## 8. MySQL Stored Routines Specifics

### 8.1 Stored Procedures ใน MySQL

```sql
DELIMITER //

-- Procedure พื้นฐาน
CREATE PROCEDURE get_customer_orders(
    IN p_customer_id INT,
    IN p_from_date DATE,
    IN p_to_date DATE,
    OUT p_total_amount DECIMAL(12,2),
    OUT p_order_count INT
)
BEGIN
    -- Handler สำหรับ errors
    DECLARE CONTINUE HANDLER FOR SQLEXCEPTION
    BEGIN
        ROLLBACK;
        RESIGNAL;
    END;
    
    SELECT 
        COUNT(*),
        COALESCE(SUM(total_amount), 0)
    INTO p_order_count, p_total_amount
    FROM orders
    WHERE customer_id = p_customer_id
    AND order_date BETWEEN p_from_date AND p_to_date;
END //

-- เรียกใช้ procedure
CALL get_customer_orders(101, '2024-01-01', '2024-12-31', @total, @count);
SELECT @total AS total_amount, @count AS order_count;

-- Procedure ที่ return result set
CREATE PROCEDURE search_products(
    IN p_keyword VARCHAR(100),
    IN p_min_price DECIMAL(10,2),
    IN p_max_price DECIMAL(10,2)
)
BEGIN
    SELECT 
        id,
        name,
        price,
        category,
        MATCH(name, description) AGAINST(p_keyword) AS relevance
    FROM products
    WHERE 
        (p_keyword IS NULL OR MATCH(name, description) AGAINST(p_keyword IN BOOLEAN MODE))
        AND price BETWEEN COALESCE(p_min_price, 0) AND COALESCE(p_max_price, 999999)
    ORDER BY relevance DESC, price ASC;
END //

DELIMITER ;

-- เรียกใช้
CALL search_products('laptop', 10000, 50000);
```

### 8.2 Functions ใน MySQL

```sql
DELIMITER //

-- Scalar function
CREATE FUNCTION calculate_age(birth_date DATE)
RETURNS INT
DETERMINISTIC
BEGIN
    RETURN TIMESTAMPDIFF(YEAR, birth_date, CURDATE());
END //

-- Function ที่ซับซ้อนกว่า
CREATE FUNCTION get_discount_rate(
    p_customer_id INT,
    p_amount DECIMAL(10,2)
) 
RETURNS DECIMAL(5,2)
READS SQL DATA
BEGIN
    DECLARE v_total_purchases DECIMAL(12,2);
    DECLARE v_membership_years INT;
    DECLARE v_discount DECIMAL(5,2);
    
    -- ดูยอดซื้อทั้งหมด
    SELECT COALESCE(SUM(total_amount), 0)
    INTO v_total_purchases
    FROM orders
    WHERE customer_id = p_customer_id
    AND order_date >= DATE_SUB(CURDATE(), INTERVAL 1 YEAR);
    
    -- ดูอายุการเป็นสมาชิก
    SELECT TIMESTAMPDIFF(YEAR, created_at, NOW())
    INTO v_membership_years
    FROM customers
    WHERE id = p_customer_id;
    
    -- คำนวณส่วนลด
    SET v_discount = CASE
        WHEN v_total_purchases >= 100000 AND v_membership_years >= 3 THEN 15.00
        WHEN v_total_purchases >= 50000 OR v_membership_years >= 2 THEN 10.00
        WHEN p_amount >= 10000 THEN 5.00
        ELSE 0.00
    END;
    
    RETURN v_discount;
END //

DELIMITER ;

-- ใช้งาน function
SELECT 
    customer_id,
    total_amount,
    get_discount_rate(customer_id, total_amount) AS discount_pct,
    total_amount * (1 - get_discount_rate(customer_id, total_amount)/100) AS final_amount
FROM orders
WHERE order_date = CURDATE();
```

### 8.3 Triggers ใน MySQL

```sql
DELIMITER //

-- BEFORE INSERT trigger
CREATE TRIGGER before_order_insert
BEFORE INSERT ON orders
FOR EACH ROW
BEGIN
    -- ตรวจสอบ stock
    DECLARE v_available INT;
    
    SELECT quantity_available INTO v_available
    FROM inventory
    WHERE product_id = NEW.product_id;
    
    IF v_available < NEW.quantity THEN
        SIGNAL SQLSTATE '45000'
        SET MESSAGE_TEXT = 'Insufficient inventory';
    END IF;
END //

-- AFTER INSERT trigger
CREATE TRIGGER after_order_insert
AFTER INSERT ON orders
FOR EACH ROW
BEGIN
    -- ลด stock
    UPDATE inventory
    SET quantity_available = quantity_available - NEW.quantity
    WHERE product_id = NEW.product_id;
    
    -- บันทึก audit log
    INSERT INTO order_audit_log (
        order_id, action, performed_at, user_info
    ) VALUES (
        NEW.id, 'INSERT', NOW(), USER()
    );
END //

-- AFTER UPDATE trigger
CREATE TRIGGER after_order_update
AFTER UPDATE ON orders
FOR EACH ROW
BEGIN
    IF OLD.status != NEW.status THEN
        INSERT INTO order_status_history (
            order_id, old_status, new_status, changed_at
        ) VALUES (
            NEW.id, OLD.status, NEW.status, NOW()
        );
    END IF;
END //

DELIMITER ;
```

---

## 9. Binary Logging และ GTID

### 9.1 Binary Log Basics

```sql
-- ตรวจสอบ binary log status
SHOW VARIABLES LIKE 'log_bin%';
SHOW VARIABLES LIKE 'binlog_format';  -- ROW, STATEMENT, MIXED

-- ดู binary log files
SHOW BINARY LOGS;
SHOW MASTER STATUS;

-- ดูเนื้อหา binary log
-- mysqlbinlog /var/lib/mysql/mysql-bin.000001

-- Flush binary logs
FLUSH BINARY LOGS;

-- ลบ binary logs เก่า
PURGE BINARY LOGS TO 'mysql-bin.000010';
PURGE BINARY LOGS BEFORE '2024-01-01 00:00:00';
```

### 9.2 GTID (Global Transaction Identifier)

```sql
-- ตรวจสอบ GTID settings
SHOW VARIABLES LIKE 'gtid_mode';
SHOW VARIABLES LIKE 'enforce_gtid_consistency';

-- ดู GTID state
SHOW VARIABLES LIKE 'gtid_executed';
SHOW VARIABLES LIKE 'gtid_purged';

-- Transaction ที่มี GTID จะแสดงใน binary log เช่น:
-- SET @@SESSION.GTID_NEXT = 'xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx:1';
-- BEGIN;
-- INSERT INTO ...;
-- COMMIT;

-- GTID function
SELECT GTID_SUBSET('3E11FA47-71CA-11E1-9E33-C80AA9429562:1-5',
                   '3E11FA47-71CA-11E1-9E33-C80AA9429562:1-10');  -- 1 (true)

SELECT GTID_SUBTRACT('3E11FA47-71CA-11E1-9E33-C80AA9429562:1-10',
                     '3E11FA47-71CA-11E1-9E33-C80AA9429562:1-5');
-- Result: '3E11FA47-71CA-11E1-9E33-C80AA9429562:6-10'
```

---

## 10. MySQL Replication Overview

### 10.1 การตั้งค่า Replication

```sql
-- บน Primary (Master):
-- my.cnf:
-- [mysqld]
-- server-id = 1
-- log_bin = mysql-bin
-- binlog_format = ROW

-- สร้าง replication user
CREATE USER 'repl'@'%' IDENTIFIED BY 'strong_password';
GRANT REPLICATION SLAVE ON *.* TO 'repl'@'%';
FLUSH PRIVILEGES;

-- ดู master status
SHOW MASTER STATUS;

-- บน Replica (Slave):
-- my.cnf:
-- [mysqld]
-- server-id = 2
-- relay_log = relay-bin
-- read_only = 1

-- ตั้งค่า replication
CHANGE REPLICATION SOURCE TO
    SOURCE_HOST = 'primary-server',
    SOURCE_USER = 'repl',
    SOURCE_PASSWORD = 'strong_password',
    SOURCE_LOG_FILE = 'mysql-bin.000001',
    SOURCE_LOG_POS = 157,
    GET_SOURCE_PUBLIC_KEY = 1;

-- เริ่ม replication
START REPLICA;

-- ตรวจสอบ replication status
SHOW REPLICA STATUS\G

-- Monitor replication lag
SELECT 
    CHANNEL_NAME,
    SERVICE_STATE,
    RECEIVED_TRANSACTION_SET,
    LAST_ERROR_MESSAGE,
    LAST_HEARTBEAT_TIMESTAMP
FROM performance_schema.replication_connection_status;
```

---

## 11. MariaDB vs MySQL Differences

### 11.1 คุณสมบัติที่แตกต่างกัน

```sql
-- MariaDB: Sequence objects (ไม่มีใน MySQL)
-- MariaDB
CREATE SEQUENCE seq_order_number
START WITH 1000
INCREMENT BY 1
MINVALUE 1000
MAXVALUE 9999999
CYCLE;

SELECT NEXTVAL(seq_order_number);
SELECT CURRVAL(seq_order_number);

-- MariaDB: Temporal Tables (System-versioned)
CREATE TABLE prices (
    product_id INT,
    price DECIMAL(10,2),
    PERIOD FOR SYSTEM_TIME(valid_from, valid_to)
) WITH SYSTEM VERSIONING;

-- ดูประวัติราคา
SELECT * FROM prices
FOR SYSTEM_TIME AS OF '2024-01-01 00:00:00';

SELECT * FROM prices
FOR SYSTEM_TIME BETWEEN '2024-01-01' AND '2024-06-30';

-- MySQL: Clone Plugin (ไม่มีใน MariaDB)
INSTALL PLUGIN clone SONAME 'mysql_clone.so';
CLONE LOCAL DATA DIRECTORY = '/backup/mysql_clone';

-- MariaDB: Spider Storage Engine (สำหรับ distributed tables)
-- MariaDB: Galera Cluster support (wsrep)
-- MySQL: Group Replication

-- MariaDB: CONNECT Storage Engine
-- ช่วย query ข้อมูลจาก external sources (CSV, ODBC, etc.)

-- MariaDB-specific functions
-- compat functions ที่แตกต่าง
SELECT DECODE('encoded_string', 'key');  -- MariaDB
-- MySQL ใช้ AES_DECRYPT แทน

-- MariaDB: window functions available earlier (10.2+)
-- MySQL: window functions ใน 8.0+

-- MariaDB: CHECK constraints enforced (ตั้งแต่ 10.2.1)
CREATE TABLE salary_data (
    id INT,
    salary DECIMAL(10,2) CHECK (salary > 0),
    age INT CHECK (age BETWEEN 18 AND 65)
);
-- MySQL ก็มีแล้วตั้งแต่ 8.0.16 (enforced)

-- MariaDB: JSON_TABLE ผ่าน Dynamic columns
-- MySQL 8+: JSON_TABLE เป็น native

-- ตรวจสอบว่าเป็น MySQL หรือ MariaDB
SELECT VERSION();
SELECT @@version_comment;  -- ถ้ามี "MariaDB" = MariaDB
```

---

## 12. MySQL 8 Roles

### 12.1 การสร้างและจัดการ Roles

```sql
-- สร้าง roles
CREATE ROLE 'read_only';
CREATE ROLE 'developer';
CREATE ROLE 'admin_db';

-- Grant privileges ให้ roles
GRANT SELECT ON mydb.* TO 'read_only';

GRANT SELECT, INSERT, UPDATE, DELETE ON mydb.* TO 'developer';
GRANT CREATE, DROP ON mydb.* TO 'developer';

GRANT ALL PRIVILEGES ON mydb.* TO 'admin_db';

-- สร้าง users
CREATE USER 'alice'@'%' IDENTIFIED BY 'password123';
CREATE USER 'bob'@'%' IDENTIFIED BY 'password456';
CREATE USER 'charlie'@'%' IDENTIFIED BY 'password789';

-- Grant roles ให้ users
GRANT 'read_only' TO 'alice'@'%';
GRANT 'developer' TO 'bob'@'%';
GRANT 'developer', 'admin_db' TO 'charlie'@'%';

-- ตั้ง default role
SET DEFAULT ROLE 'developer' TO 'bob'@'%';
SET DEFAULT ROLE ALL TO 'charlie'@'%';

-- User ต้อง activate role
SET ROLE 'developer';  -- activate role ใน session
SET ROLE ALL;           -- activate all granted roles
SET ROLE NONE;          -- ปิด roles ทั้งหมด

-- ดู roles ที่ active
SELECT CURRENT_ROLE();

-- ดู role assignments
SELECT * FROM INFORMATION_SCHEMA.APPLICABLE_ROLES;
SELECT * FROM mysql.role_edges;

-- Revoke role
REVOKE 'read_only' FROM 'alice'@'%';

-- ลบ role
DROP ROLE 'read_only';
```

---

## 13. ตัวอย่างเพิ่มเติม

```sql
-- Example 1: JSON aggregation สำหรับ reporting
SELECT 
    category,
    JSON_ARRAYAGG(
        JSON_OBJECT(
            'id', id,
            'name', name,
            'price', price
        )
    ) AS products
FROM products
GROUP BY category;

-- Example 2: Recursive CTE สำหรับ path finding
WITH RECURSIVE paths (from_city, to_city, path, cost, hops) AS (
    SELECT from_city, to_city, 
           CONCAT(from_city, ' -> ', to_city),
           distance, 1
    FROM routes
    WHERE from_city = 'Bangkok'
    
    UNION ALL
    
    SELECT p.from_city, r.to_city,
           CONCAT(p.path, ' -> ', r.to_city),
           p.cost + r.distance, p.hops + 1
    FROM paths p
    JOIN routes r ON p.to_city = r.from_city
    WHERE p.hops < 5
    AND FIND_IN_SET(r.to_city, REPLACE(p.path, ' -> ', ',')) = 0
)
SELECT path, cost, hops
FROM paths
WHERE to_city = 'Chiang Mai'
ORDER BY cost;

-- Example 3: Window function สำหรับ top-N per category
WITH ranked_products AS (
    SELECT 
        *,
        ROW_NUMBER() OVER (PARTITION BY category ORDER BY revenue DESC) AS rn
    FROM products
)
SELECT * FROM ranked_products WHERE rn <= 3;

-- Example 4: Generated column + JSON index
CREATE TABLE log_entries (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    log_data JSON NOT NULL,
    log_level VARCHAR(10) GENERATED ALWAYS AS (log_data->>'$.level') STORED,
    service_name VARCHAR(50) GENERATED ALWAYS AS (log_data->>'$.service') STORED,
    log_time TIMESTAMP GENERATED ALWAYS AS 
        (STR_TO_DATE(log_data->>'$.timestamp', '%Y-%m-%dT%H:%i:%s')) STORED,
    INDEX idx_level (log_level),
    INDEX idx_service (service_name),
    INDEX idx_time (log_time)
);

INSERT INTO log_entries (log_data) VALUES
('{"level": "ERROR", "service": "auth", "message": "Login failed", "timestamp": "2024-01-15T10:30:00", "user_id": 123}'),
('{"level": "INFO", "service": "order", "message": "Order created", "timestamp": "2024-01-15T10:31:00", "order_id": 456}');

-- Query ใช้ generated column index
SELECT log_data->>'$.message', log_time
FROM log_entries
WHERE log_level = 'ERROR'
AND log_time >= NOW() - INTERVAL 1 HOUR;

-- Example 5: Full-text search + relevance ranking
SELECT 
    id,
    title,
    author,
    MATCH(title, content) AGAINST('advanced SQL performance' IN NATURAL LANGUAGE MODE) AS relevance,
    LEFT(content, 200) AS excerpt
FROM articles
WHERE MATCH(title, content) AGAINST('advanced SQL performance' IN NATURAL LANGUAGE MODE)
ORDER BY relevance DESC
LIMIT 10;
```

---

## แบบฝึกหัด

### ข้อที่ 1: JSON Product Catalog
สร้างระบบ product catalog ที่ใช้ JSON สำหรับ product specifications

**เฉลย:**
```sql
CREATE TABLE products_json (
    id INT AUTO_INCREMENT PRIMARY KEY,
    sku VARCHAR(50) UNIQUE NOT NULL,
    name VARCHAR(200) NOT NULL,
    category VARCHAR(50),
    specs JSON,
    price DECIMAL(10,2),
    -- Generated columns สำหรับ index
    brand VARCHAR(50) GENERATED ALWAYS AS (specs->>'$.brand') STORED,
    weight_kg DECIMAL(6,2) GENERATED ALWAYS AS (specs->>'$.weight_kg') STORED,
    INDEX idx_brand (brand),
    INDEX idx_price (price),
    FULLTEXT INDEX ft_name (name)
);

INSERT INTO products_json (sku, name, category, specs, price) VALUES
('LAPTOP-001', 'Dell XPS 15', 'Laptop', 
 '{"brand": "Dell", "cpu": "Intel i7-13700H", "ram": "16GB", "storage": "512GB SSD", "display": "15.6 OLED", "weight_kg": 1.86}',
 65000),
('PHONE-001', 'Samsung Galaxy S24', 'Smartphone',
 '{"brand": "Samsung", "cpu": "Snapdragon 8 Gen 3", "ram": "8GB", "storage": "256GB", "display": "6.2 AMOLED", "weight_kg": 0.167}',
 32000);

-- ค้นหาสินค้าตาม spec
SELECT name, price, specs->>'$.ram' AS ram
FROM products_json
WHERE brand = 'Dell'
AND weight_kg < 2.0;
```

### ข้อที่ 2: Employee Hierarchy ด้วย Recursive CTE

**เฉลย:**
```sql
WITH RECURSIVE hierarchy AS (
    SELECT 
        emp_id, emp_name, manager_id, salary, department,
        1 AS level,
        CAST(emp_name AS CHAR(1000)) AS chain
    FROM employees_hier WHERE manager_id IS NULL
    
    UNION ALL
    
    SELECT e.emp_id, e.emp_name, e.manager_id, e.salary, e.department,
           h.level + 1,
           CONCAT(h.chain, ' / ', e.emp_name)
    FROM employees_hier e
    JOIN hierarchy h ON e.manager_id = h.emp_id
)
SELECT 
    CONCAT(REPEAT('  ', level - 1), emp_name) AS name,
    department,
    FORMAT(salary, 0) AS salary,
    level
FROM hierarchy
ORDER BY chain;
```

### ข้อที่ 3: Sales Performance Dashboard ด้วย Window Functions

**เฉลย:**
```sql
WITH monthly_sales AS (
    SELECT 
        salesperson,
        region,
        DATE_FORMAT(sale_date, '%Y-%m') AS month,
        SUM(amount) AS monthly_total
    FROM sales
    GROUP BY salesperson, region, DATE_FORMAT(sale_date, '%Y-%m')
),
ranked AS (
    SELECT *,
        ROW_NUMBER() OVER (PARTITION BY month ORDER BY monthly_total DESC) AS rank_in_month,
        SUM(monthly_total) OVER (PARTITION BY region, month) AS region_month_total,
        SUM(monthly_total) OVER (PARTITION BY salesperson ORDER BY month) AS ytd_total
    FROM monthly_sales
)
SELECT 
    month,
    salesperson,
    region,
    FORMAT(monthly_total, 0) AS monthly_sales,
    rank_in_month AS monthly_rank,
    ROUND(monthly_total / region_month_total * 100, 1) AS pct_of_region,
    FORMAT(ytd_total, 0) AS ytd_sales
FROM ranked
ORDER BY month DESC, rank_in_month;
```

### ข้อที่ 4: Event Scheduler สำหรับ Database Maintenance

**เฉลย:**
```sql
-- Event สำหรับทำ maintenance ทุกคืน
DELIMITER //

CREATE EVENT nightly_maintenance
ON SCHEDULE EVERY 1 DAY
STARTS TIMESTAMP(CURDATE(), '02:00:00')
DO
BEGIN
    DECLARE EXIT HANDLER FOR SQLEXCEPTION
    BEGIN
        INSERT INTO maintenance_log (event_name, status, error_time)
        VALUES ('nightly_maintenance', 'FAILED', NOW());
    END;
    
    START TRANSACTION;
    
    -- ลบ sessions ที่หมดอายุ
    DELETE FROM user_sessions WHERE expires_at < NOW();
    
    -- Archive old logs
    INSERT INTO archived_logs 
    SELECT * FROM activity_logs 
    WHERE created_at < NOW() - INTERVAL 90 DAY;
    
    DELETE FROM activity_logs 
    WHERE created_at < NOW() - INTERVAL 90 DAY;
    
    -- Update statistics table
    REPLACE INTO table_statistics
    SELECT 
        table_name,
        table_rows,
        data_length,
        index_length,
        NOW() AS collected_at
    FROM information_schema.tables
    WHERE table_schema = DATABASE();
    
    COMMIT;
    
    INSERT INTO maintenance_log (event_name, status, error_time)
    VALUES ('nightly_maintenance', 'SUCCESS', NOW());
END //

DELIMITER ;
```

### ข้อที่ 5: Role-based Access Control

**เฉลย:**
```sql
-- สร้าง roles สำหรับ e-commerce system
CREATE ROLE 'customer_service';
CREATE ROLE 'warehouse_staff';
CREATE ROLE 'finance_team';
CREATE ROLE 'system_admin';

-- Customer service: อ่านออเดอร์, แก้ไขได้บางส่วน
GRANT SELECT ON ecommerce.orders TO 'customer_service';
GRANT SELECT ON ecommerce.customers TO 'customer_service';
GRANT UPDATE (status, notes) ON ecommerce.orders TO 'customer_service';

-- Warehouse: จัดการ inventory
GRANT SELECT, UPDATE ON ecommerce.inventory TO 'warehouse_staff';
GRANT SELECT ON ecommerce.orders TO 'warehouse_staff';
GRANT INSERT ON ecommerce.shipments TO 'warehouse_staff';

-- Finance: ดูรายงานทางการเงิน
GRANT SELECT ON ecommerce.orders TO 'finance_team';
GRANT SELECT ON ecommerce.payments TO 'finance_team';
GRANT SELECT ON ecommerce.refunds TO 'finance_team';

-- สร้าง users และ assign roles
CREATE USER 'cs_user1'@'%' IDENTIFIED BY 'cs_pass';
GRANT 'customer_service' TO 'cs_user1'@'%';
SET DEFAULT ROLE 'customer_service' TO 'cs_user1'@'%';
```

### ข้อที่ 6: Full-Text Search Engine

**เฉลย:**
```sql
CREATE TABLE knowledge_base (
    id INT AUTO_INCREMENT PRIMARY KEY,
    category VARCHAR(50),
    question TEXT,
    answer TEXT,
    tags VARCHAR(200),
    view_count INT DEFAULT 0,
    helpful_votes INT DEFAULT 0,
    FULLTEXT INDEX ft_qa (question, answer)
);

INSERT INTO knowledge_base (category, question, answer, tags) VALUES
('Billing', 'How do I cancel my subscription?', 
 'You can cancel your subscription by going to Settings > Subscription > Cancel. Your service will continue until the end of the billing period.',
 'cancel,subscription,billing'),
('Technical', 'Why is my login not working?',
 'Please try clearing your browser cache and cookies. If the problem persists, reset your password using the Forgot Password link.',
 'login,password,troubleshoot');

-- Advanced search with highlighting
SELECT 
    category,
    question,
    LEFT(answer, 300) AS answer_excerpt,
    MATCH(question, answer) AGAINST('cancel subscription billing' IN NATURAL LANGUAGE MODE) AS relevance
FROM knowledge_base
WHERE MATCH(question, answer) AGAINST('cancel subscription billing' IN NATURAL LANGUAGE MODE)
ORDER BY relevance DESC;
```

### ข้อที่ 7: InnoDB Locking Pattern สำหรับ Inventory

**เฉลย:**
```sql
DELIMITER //

CREATE PROCEDURE reserve_inventory(
    IN p_product_id INT,
    IN p_quantity INT,
    IN p_order_id INT
)
BEGIN
    DECLARE v_available INT;
    DECLARE EXIT HANDLER FOR SQLEXCEPTION
    BEGIN
        ROLLBACK;
        RESIGNAL;
    END;
    
    START TRANSACTION;
    
    -- Lock the inventory row
    SELECT quantity_available INTO v_available
    FROM inventory
    WHERE product_id = p_product_id
    FOR UPDATE;
    
    IF v_available < p_quantity THEN
        SIGNAL SQLSTATE '45000'
        SET MESSAGE_TEXT = 'Insufficient inventory';
    END IF;
    
    -- Reduce available quantity
    UPDATE inventory
    SET 
        quantity_available = quantity_available - p_quantity,
        quantity_reserved = quantity_reserved + p_quantity
    WHERE product_id = p_product_id;
    
    -- Record the reservation
    INSERT INTO inventory_reservations 
    (product_id, order_id, quantity, reserved_at)
    VALUES (p_product_id, p_order_id, p_quantity, NOW());
    
    COMMIT;
END //

DELIMITER ;
```

### ข้อที่ 8: Binary Log-based Audit Trail

**เฉลย:**
```sql
-- ตั้งค่าสำหรับ audit
SET GLOBAL binlog_format = 'ROW';
SET GLOBAL binlog_row_image = 'FULL';

-- Audit table
CREATE TABLE audit_trail (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    table_name VARCHAR(64),
    operation ENUM('INSERT', 'UPDATE', 'DELETE'),
    primary_key_value VARCHAR(255),
    old_values JSON,
    new_values JSON,
    changed_by VARCHAR(100) DEFAULT USER(),
    changed_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_table_op (table_name, operation),
    INDEX idx_changed_at (changed_at)
);

-- Generic audit trigger
DELIMITER //

CREATE TRIGGER audit_customers_insert
AFTER INSERT ON customers
FOR EACH ROW
INSERT INTO audit_trail (table_name, operation, primary_key_value, new_values)
VALUES ('customers', 'INSERT', NEW.id,
    JSON_OBJECT('id', NEW.id, 'name', NEW.name, 'email', NEW.email)) //

CREATE TRIGGER audit_customers_update
AFTER UPDATE ON customers
FOR EACH ROW
INSERT INTO audit_trail (table_name, operation, primary_key_value, old_values, new_values)
VALUES ('customers', 'UPDATE', NEW.id,
    JSON_OBJECT('name', OLD.name, 'email', OLD.email),
    JSON_OBJECT('name', NEW.name, 'email', NEW.email)) //

DELIMITER ;
```

### ข้อที่ 9: Window Functions สำหรับ Cohort Analysis

**เฉลย:**
```sql
-- Cohort analysis: ดูว่า users ที่ลงทะเบียนในเดือนไหน ยังคง active อยู่หรือไม่
WITH user_cohorts AS (
    SELECT 
        id AS user_id,
        DATE_FORMAT(created_at, '%Y-%m') AS cohort_month
    FROM users
),
user_activities AS (
    SELECT 
        user_id,
        DATE_FORMAT(activity_date, '%Y-%m') AS activity_month
    FROM user_activity_log
    GROUP BY user_id, DATE_FORMAT(activity_date, '%Y-%m')
),
cohort_data AS (
    SELECT 
        uc.cohort_month,
        ua.activity_month,
        COUNT(DISTINCT uc.user_id) AS active_users,
        PERIOD_DIFF(
            DATE_FORMAT(ua.activity_month, '%Y%m')::SIGNED,  -- MySQL doesn't support this directly
            DATE_FORMAT(uc.cohort_month, '%Y%m')::SIGNED
        ) AS months_since_signup
    FROM user_cohorts uc
    JOIN user_activities ua ON uc.user_id = ua.user_id
    GROUP BY uc.cohort_month, ua.activity_month
)
SELECT 
    cohort_month,
    months_since_signup,
    active_users,
    MAX(active_users) OVER (PARTITION BY cohort_month) AS cohort_size,
    ROUND(active_users / MAX(active_users) OVER (PARTITION BY cohort_month) * 100, 1) AS retention_rate
FROM cohort_data
ORDER BY cohort_month, months_since_signup;
```

### ข้อที่ 10: JSON Schema Validation

**เฉลย:**
```sql
-- MySQL 8.0.17+ JSON schema validation
CREATE TABLE api_requests (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    endpoint VARCHAR(100),
    request_body JSON,
    is_valid TINYINT GENERATED ALWAYS AS (
        JSON_SCHEMA_VALID(
            '{
                "type": "object",
                "properties": {
                    "user_id": {"type": "integer"},
                    "amount": {"type": "number", "minimum": 0},
                    "currency": {"type": "string", "enum": ["THB", "USD", "EUR"]}
                },
                "required": ["user_id", "amount", "currency"]
            }',
            request_body
        )
    ) STORED,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_endpoint (endpoint),
    INDEX idx_valid (is_valid)
);

INSERT INTO api_requests (endpoint, request_body) VALUES
('/api/payment', '{"user_id": 123, "amount": 1500.00, "currency": "THB"}'),
('/api/payment', '{"user_id": 456, "amount": -100, "currency": "THB"}'),  -- invalid amount
('/api/payment', '{"user_id": 789, "amount": 500}');  -- missing currency

-- ดู invalid requests
SELECT id, endpoint, request_body
FROM api_requests
WHERE is_valid = 0;
```

---

## สรุป

ในบทนี้เราได้เรียนรู้คุณสมบัติขั้นสูงของ MySQL 8 และ MariaDB:

1. **JSON Functions** - จัดการข้อมูล JSON อย่างครบวงจร
2. **Generated Columns** - คำนวณค่าอัตโนมัติจาก expressions
3. **Window Functions** - วิเคราะห์ข้อมูลแบบ analytical
4. **CTEs และ Recursive Queries** - โครงสร้าง queries ที่ซับซ้อน
5. **InnoDB Locking** - จัดการ concurrency
6. **Full-Text Search** - ค้นหา text อย่างมีประสิทธิภาพ
7. **Event Scheduler** - รัน tasks อัตโนมัติ
8. **Stored Routines** - business logic ใน database
9. **Binary Logging/GTID** - replication และ recovery
10. **Roles** - จัดการ permissions อย่างมีระบบ
11. **MariaDB-specific features** - Sequences, System-versioned tables

MySQL 8 และ MariaDB มีคุณสมบัติที่แข็งแกร่งพอสำหรับ enterprise applications สมัยใหม่ ทั้งสองระบบมีจุดเด่นของตัวเองและเลือกใช้ตาม use case ได้
