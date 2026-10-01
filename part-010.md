# Part 010: String Functions Deep Dive

> **หลักสูตร SQL ครบวงจร | Part 10 of 120**

---

## 🎯 สิ่งที่จะได้เรียนรู้ในบทนี้

- UPPER / LOWER - แปลง case ตัวอักษร
- TRIM / LTRIM / RTRIM - ตัด whitespace
- LENGTH / LEN / CHAR_LENGTH - ความยาว string
- SUBSTRING / SUBSTR / MID - ตัด string
- CONCAT / || - ต่อ strings
- REPLACE - แทนที่ข้อความ
- CHARINDEX / INSTR / POSITION / LOCATE - หาตำแหน่ง
- LEFT / RIGHT - ตัดจากซ้าย/ขวา
- LPAD / RPAD - เติม padding
- LIKE vs ILIKE - case-sensitive vs insensitive
- REGEXP / SIMILAR TO - regular expressions
- FORMAT สำหรับ strings
- Cross-database comparison table

**เวลาที่ใช้เรียน**: ประมาณ 2.5 ชั่วโมง

---

## ฐานข้อมูลที่ใช้ในบทนี้

```sql
-- ตรวจสอบข้อมูล
SELECT first_name, last_name, email, phone FROM employees LIMIT 5;
SELECT product_name, category FROM products LIMIT 5;
SELECT full_name, email, city FROM customers LIMIT 5;
```

---

## 10.1 UPPER และ LOWER

```sql
-- ============================================================
-- EXAMPLE 1: UPPER - แปลงเป็นตัวพิมพ์ใหญ่
-- ============================================================

SELECT 
    first_name,
    UPPER(first_name) AS upper_name
FROM employees
LIMIT 5;

-- ผลลัพธ์:
-- first_name | upper_name
-- -----------|----------
-- สมชาย      | สมชาย      ← ภาษาไทยไม่มี case
-- Somchai    | SOMCHAI

-- ============================================================
-- EXAMPLE 2: LOWER - แปลงเป็นตัวพิมพ์เล็ก
-- ============================================================

SELECT 
    email,
    LOWER(email) AS lower_email
FROM employees
WHERE employee_id = 1;

-- ผลลัพธ์:
-- email                    | lower_email
-- -------------------------|-------------------------
-- Somchai.J@company.co.th  | somchai.j@company.co.th

-- ============================================================
-- EXAMPLE 3: ใช้ใน WHERE - case insensitive search
-- ============================================================

-- ค้นหา email โดยไม่สนใจ case
SELECT first_name, last_name, email
FROM employees
WHERE LOWER(email) = 'somchai.j@company.co.th';

-- ============================================================
-- EXAMPLE 4: Normalize ข้อมูลก่อนเปรียบเทียบ
-- ============================================================

-- หา categories แบบ case insensitive
SELECT DISTINCT LOWER(category) AS category
FROM products
ORDER BY 1;

-- ผลลัพธ์:
-- category
-- -----------
-- books
-- electronics
-- furniture
-- stationery

-- ============================================================
-- EXAMPLE 5: INITCAP - capitalize first letter (PostgreSQL)
-- ============================================================

-- PostgreSQL เท่านั้น:
-- SELECT INITCAP('hello world') AS result;  -- 'Hello World'

-- MySQL ใช้ combination:
-- CONCAT(UPPER(LEFT(name,1)), LOWER(SUBSTRING(name,2)))

-- ============================================================
-- EXAMPLE 6: ใช้ UPPER/LOWER ใน formatting
-- ============================================================

SELECT 
    employee_id,
    UPPER(first_name) || ' ' || UPPER(last_name) AS FULL_NAME_CAPS,
    LOWER(email) AS email_normalized
FROM employees
ORDER BY last_name;
```

---

## 10.2 TRIM, LTRIM, RTRIM

```sql
-- ============================================================
-- EXAMPLE 7: TRIM - ตัด whitespace รอบข้าง
-- ============================================================

SELECT 
    '  hello  ' AS original,
    TRIM('  hello  ') AS trimmed,
    LENGTH('  hello  ') AS orig_len,
    LENGTH(TRIM('  hello  ')) AS trim_len;

-- ผลลัพธ์:
-- original | trimmed | orig_len | trim_len
-- ---------|---------|----------|----------
--   hello  | hello   | 9        | 5

-- ============================================================
-- EXAMPLE 8: LTRIM - ตัดเฉพาะด้านซ้าย
-- ============================================================

SELECT 
    LTRIM('   hello   ') AS left_trimmed;
-- ผลลัพธ์: 'hello   ' (ยังมี space ด้านขวา)

-- ============================================================
-- EXAMPLE 9: RTRIM - ตัดเฉพาะด้านขวา
-- ============================================================

SELECT 
    RTRIM('   hello   ') AS right_trimmed;
-- ผลลัพธ์: '   hello' (ยังมี space ด้านซ้าย)

-- ============================================================
-- EXAMPLE 10: TRIM ตัด characters อื่น (PostgreSQL)
-- ============================================================

-- PostgreSQL: TRIM(LEADING/TRAILING/BOTH 'char' FROM string)
-- SELECT TRIM(LEADING '0' FROM '000123') AS result;  -- '123'
-- SELECT TRIM(BOTH '*' FROM '***hello***') AS result; -- 'hello'
-- SELECT TRIM(TRAILING '/' FROM 'https://example.com/') AS result;

-- SQLite/MySQL: ใช้ REPLACE หรือ REGEXP

-- ============================================================
-- EXAMPLE 11: Data Cleanup - ตัด whitespace จาก real data
-- ============================================================

-- สมมุติ users กรอก name มี space เกิน
SELECT 
    full_name,
    TRIM(full_name) AS clean_name,
    LENGTH(full_name) AS raw_len,
    LENGTH(TRIM(full_name)) AS clean_len
FROM customers
WHERE LENGTH(full_name) != LENGTH(TRIM(full_name));
-- หา rows ที่มี leading/trailing spaces

-- ============================================================
-- EXAMPLE 12: UPDATE ด้วย TRIM (Data Cleansing)
-- ============================================================

-- ทำความสะอาดข้อมูล (ระวัง: ใช้ WHERE เสมอ!)
-- UPDATE customers 
-- SET full_name = TRIM(full_name)
-- WHERE full_name != TRIM(full_name);
```

---

## 10.3 LENGTH / LEN / CHAR_LENGTH

```sql
-- ============================================================
-- EXAMPLE 13: LENGTH พื้นฐาน
-- ============================================================

SELECT 
    product_name,
    LENGTH(product_name) AS name_length
FROM products
ORDER BY name_length DESC
LIMIT 5;

-- ผลลัพธ์:
-- product_name                    | name_length
-- --------------------------------|-------------
-- MacBook Pro 14-inch M3 Pro      | 26
-- Samsung Galaxy S24 Ultra 256GB  | 30
-- ...

-- ============================================================
-- EXAMPLE 14: Cross-database LENGTH
-- ============================================================

-- SQLite:    LENGTH(str)        → ความยาวในหน่วย characters
-- PostgreSQL: LENGTH(str)        → characters (unicode-aware)
--             CHAR_LENGTH(str)   → same
--             OCTET_LENGTH(str)  → bytes
-- MySQL:     LENGTH(str)        → bytes (ระวัง UTF-8!)
--            CHAR_LENGTH(str)   → characters ✓ (ใช้อันนี้สำหรับ unicode)

-- ตัวอย่างปัญหา MySQL:
-- สมมุติ "สวัสดี" (6 Thai characters)
-- LENGTH('สวัสดี')      → 18 (bytes, UTF-8 = 3 bytes/char)
-- CHAR_LENGTH('สวัสดี') → 6  (characters) ← ถูกต้อง

-- ============================================================
-- EXAMPLE 15: กรอง string ตาม length
-- ============================================================

-- หา products ที่ชื่อยาวเกิน 20 characters
SELECT product_name, LENGTH(product_name) AS len
FROM products
WHERE LENGTH(product_name) > 20
ORDER BY len DESC;

-- หา employees ที่ email สั้นที่สุด
SELECT first_name, email, LENGTH(email) AS email_len
FROM employees
ORDER BY email_len ASC
LIMIT 3;

-- ============================================================
-- EXAMPLE 16: CHAR_LENGTH vs LENGTH ใน MySQL
-- ============================================================

-- สำหรับ portability ควรใช้ CHAR_LENGTH แทน LENGTH ใน MySQL
-- PostgreSQL และ SQLite: LENGTH ทำงานถูกต้องกับ Unicode
-- SQL Server: LEN() (ไม่มี CHAR_LENGTH)
```

---

## 10.4 SUBSTRING / SUBSTR

```sql
-- ============================================================
-- EXAMPLE 17: SUBSTRING พื้นฐาน
-- ============================================================

-- SUBSTRING(string, start_position, length)
-- start_position เริ่มที่ 1 (ไม่ใช่ 0!)

SELECT 
    'Hello World' AS original,
    SUBSTRING('Hello World', 1, 5) AS first_5,    -- 'Hello'
    SUBSTRING('Hello World', 7, 5) AS last_5,      -- 'World'
    SUBSTRING('Hello World', 7)    AS from_pos_7;  -- 'World' (ไม่ระบุ length = ถึงสุด)

-- ============================================================
-- EXAMPLE 18: ดึง domain จาก email
-- ============================================================

-- ดึง domain part ของ email (ส่วนหลัง @)
SELECT 
    email,
    SUBSTRING(email, INSTR(email, '@') + 1) AS domain
FROM employees;

-- SQLite: INSTR หาตำแหน่ง @
-- PostgreSQL: POSITION('@' IN email)
-- MySQL: LOCATE('@', email)

-- ============================================================
-- EXAMPLE 19: ดึง prefix จาก employee code
-- ============================================================

-- สมมุติ phone เก็บแบบ '+66-XX-XXX-XXXX'
SELECT 
    phone,
    SUBSTRING(phone, 1, 3) AS country_code  -- '+66'
FROM employees
WHERE phone IS NOT NULL
LIMIT 3;

-- ============================================================
-- EXAMPLE 20: SUBSTR ใน SQLite/PostgreSQL/MySQL
-- ============================================================

-- SUBSTR เป็น alias ของ SUBSTRING
-- ทำงานเหมือนกัน:
SELECT SUBSTR('SQL Course', 1, 3);    -- 'SQL'
SELECT SUBSTRING('SQL Course', 1, 3); -- 'SQL' (เหมือนกัน)

-- SQL Server ใช้ SUBSTRING เท่านั้น (ไม่มี SUBSTR)

-- ============================================================
-- EXAMPLE 21: ดึง Year จาก date string
-- ============================================================

-- ถ้า date เก็บเป็น text format 'YYYY-MM-DD'
SELECT 
    hire_date,
    SUBSTR(hire_date, 1, 4) AS year,
    SUBSTR(hire_date, 6, 2) AS month,
    SUBSTR(hire_date, 9, 2) AS day
FROM employees
LIMIT 5;
```

---

## 10.5 CONCAT / ||

```sql
-- ============================================================
-- EXAMPLE 22: CONCAT พื้นฐาน
-- ============================================================

-- MySQL / SQL Server:
-- SELECT CONCAT(first_name, ' ', last_name) AS full_name FROM employees;

-- PostgreSQL / SQLite: ใช้ || operator
SELECT first_name || ' ' || last_name AS full_name
FROM employees
LIMIT 5;

-- ============================================================
-- EXAMPLE 23: CONCAT กับ NULL
-- ============================================================

-- MySQL CONCAT: ถ้า argument ใดเป็น NULL → ผลเป็น NULL!
-- CONCAT('Hello', NULL, 'World') → NULL

-- PostgreSQL ||: NULL concatenation = NULL ด้วย
-- 'Hello' || NULL → NULL

-- แก้ด้วย COALESCE:
SELECT 
    first_name || ' ' || COALESCE(last_name, '') AS safe_name
FROM employees;

-- ============================================================
-- EXAMPLE 24: CONCAT_WS (MySQL/PostgreSQL)
-- ============================================================

-- CONCAT_WS = CONCAT With Separator
-- ฉลาดกว่า: ข้าม NULL values

-- MySQL:
-- SELECT CONCAT_WS(', ', city, country) FROM customers;
-- ถ้า city เป็น NULL → 'Thailand' (ไม่มี leading comma)

-- PostgreSQL: ใช้ CONCAT_WS เหมือนกัน
-- SELECT CONCAT_WS(' ', first_name, middle_name, last_name);
-- ถ้า middle_name = NULL → 'First Last' (ไม่มี double space)

-- ============================================================
-- EXAMPLE 25: สร้าง formatted strings
-- ============================================================

-- สร้าง full address
SELECT 
    full_name,
    address || ', ' || city || ', ' || country AS full_address
FROM customers
WHERE address IS NOT NULL
LIMIT 5;

-- สร้าง email format
SELECT 
    employee_id,
    LOWER(first_name) || '.' || LOWER(SUBSTR(last_name, 1, 1)) || 
        '@company.co.th' AS suggested_email
FROM employees
LIMIT 5;
```

---

## 10.6 REPLACE

```sql
-- ============================================================
-- EXAMPLE 26: REPLACE พื้นฐาน
-- ============================================================

-- REPLACE(string, search, replacement)
SELECT REPLACE('Hello World', 'World', 'SQL');
-- ผลลัพธ์: 'Hello SQL'

-- ============================================================
-- EXAMPLE 27: ลบ characters ออก
-- ============================================================

-- ลบ dashes จาก phone number
SELECT 
    phone,
    REPLACE(phone, '-', '') AS clean_phone
FROM employees
WHERE phone IS NOT NULL;

-- ผลลัพธ์:
-- phone          | clean_phone
-- ---------------|-------------
-- 081-234-5678   | 0812345678

-- ============================================================
-- EXAMPLE 28: Replace หลายครั้ง (nested)
-- ============================================================

-- แทนที่หลาย characters:
SELECT 
    REPLACE(
        REPLACE(
            REPLACE(phone, '-', ''),
            '+66', '0'),
        ' ', '')
    AS normalized_phone
FROM employees
WHERE phone IS NOT NULL;

-- ============================================================
-- EXAMPLE 29: REPLACE ใน UPDATE
-- ============================================================

-- Masking email domain สำหรับ test environment
-- SELECT REPLACE(email, '@company.co.th', '@test.com') AS test_email
-- FROM employees;

-- ============================================================
-- EXAMPLE 30: REPLACE ไม่ supports regex
-- ============================================================

-- REPLACE ทำงานแบบ literal match เท่านั้น (ไม่ใช่ regex)
-- สำหรับ pattern replacement ต้องใช้:
-- PostgreSQL: REGEXP_REPLACE(string, pattern, replacement)
-- MySQL 8+:   REGEXP_REPLACE(string, pattern, replacement)
-- SQLite:     ไม่มี built-in regex replace (ต้องใช้ extension)

-- PostgreSQL:
-- SELECT REGEXP_REPLACE(phone, '[^0-9]', '', 'g') AS digits_only
-- FROM employees WHERE phone IS NOT NULL;
```

---

## 10.7 CHARINDEX / INSTR / POSITION / LOCATE

```sql
-- ============================================================
-- EXAMPLE 31: หาตำแหน่งของ substring
-- ============================================================

-- Function ต่างกันตาม database:
-- SQLite:     INSTR(string, substring)
-- PostgreSQL: POSITION(substring IN string)   หรือ STRPOS(string, substring)
-- MySQL:      LOCATE(substring, string)        หรือ INSTR(string, substring)
-- SQL Server: CHARINDEX(substring, string)

-- SQLite:
SELECT INSTR('hello world', 'world') AS pos;  -- 7

-- PostgreSQL:
-- SELECT POSITION('world' IN 'hello world');  -- 7

-- MySQL:
-- SELECT LOCATE('world', 'hello world');      -- 7
-- SELECT INSTR('hello world', 'world');       -- 7

-- SQL Server:
-- SELECT CHARINDEX('world', 'hello world');   -- 7

-- ============================================================
-- EXAMPLE 32: ใช้ INSTR หา @ ใน email (SQLite)
-- ============================================================

SELECT 
    email,
    INSTR(email, '@') AS at_position,
    SUBSTR(email, 1, INSTR(email, '@') - 1) AS username,
    SUBSTR(email, INSTR(email, '@') + 1) AS domain
FROM employees
LIMIT 5;

-- ผลลัพธ์:
-- email                   | at_pos | username   | domain
-- ------------------------|--------|------------|----------------
-- somchai.j@company.co.th | 10     | somchai.j  | company.co.th

-- ============================================================
-- EXAMPLE 33: ตรวจสอบว่า string มี substring หรือไม่
-- ============================================================

-- SQLite: INSTR คืน 0 ถ้าไม่พบ
SELECT 
    email,
    CASE WHEN INSTR(email, '@company') > 0 
         THEN 'Internal' 
         ELSE 'External' 
    END AS email_type
FROM employees;

-- PostgreSQL ใช้:
-- CASE WHEN POSITION('@company' IN email) > 0 THEN...

-- ============================================================
-- EXAMPLE 34: CHARINDEX กับ start position (SQL Server)
-- ============================================================

-- SQL Server: CHARINDEX(search, string, start_pos)
-- SELECT CHARINDEX('o', 'hello world', 6) → 8 (ค้นหาจากตำแหน่ง 6)
```

---

## 10.8 LEFT และ RIGHT

```sql
-- ============================================================
-- EXAMPLE 35: LEFT - ตัดจากซ้าย
-- ============================================================

-- SQLite ไม่มี LEFT โดยตรง → ใช้ SUBSTR
-- SUBSTR(string, 1, n) = LEFT(string, n)

-- PostgreSQL/MySQL/SQL Server:
-- SELECT LEFT(product_name, 10) FROM products;

-- SQLite:
SELECT SUBSTR(product_name, 1, 10) AS short_name
FROM products;

-- ============================================================
-- EXAMPLE 36: RIGHT - ตัดจากขวา
-- ============================================================

-- SQLite ไม่มี RIGHT โดยตรง → ใช้ SUBSTR + LENGTH
-- RIGHT(string, n) = SUBSTR(string, -n) ใน SQLite
-- หรือ SUBSTR(string, LENGTH(string) - n + 1, n)

-- SQLite:
SELECT SUBSTR(product_name, -5) AS last_5_chars
FROM products;

-- PostgreSQL/MySQL:
-- SELECT RIGHT(product_name, 5) FROM products;

-- ============================================================
-- EXAMPLE 37: ดึง file extension
-- ============================================================

-- สมมุติ product มี image_file column
-- หา extension ของไฟล์ (3 chars หลังสุด)
SELECT 
    'product_image.jpg' AS filename,
    SUBSTR('product_image.jpg', -3) AS extension;  -- 'jpg'

-- ============================================================
-- EXAMPLE 38: สร้าง ID prefix
-- ============================================================

-- สร้าง display ID จาก category + number
SELECT 
    product_id,
    UPPER(SUBSTR(category, 1, 3)) || 
        printf('%04d', product_id) AS display_code
FROM products;

-- ผลลัพธ์:
-- product_id | display_code
-- -----------|-------------
-- 1          | ELE0001
-- 2          | ELE0002
-- 7          | FUR0007
-- 12         | BOO0012
```

---

## 10.9 LPAD และ RPAD

```sql
-- ============================================================
-- EXAMPLE 39: LPAD - เติม padding ด้านซ้าย (MySQL/PostgreSQL)
-- ============================================================

-- MySQL/PostgreSQL:
-- SELECT LPAD('42', 6, '0');     -- '000042'
-- SELECT LPAD('hello', 10, '-'); -- '-----hello'

-- SQLite ไม่มี LPAD → ใช้ printf:
SELECT printf('%06d', employee_id) AS padded_id
FROM employees
LIMIT 3;
-- ผลลัพธ์: 000001, 000002, 000003

-- ============================================================
-- EXAMPLE 40: RPAD - เติม padding ด้านขวา
-- ============================================================

-- MySQL/PostgreSQL:
-- SELECT RPAD('hello', 10, ' ');    -- 'hello     '
-- SELECT RPAD('SQL', 10, '-');      -- 'SQL-------'

-- SQLite: ใช้ printf หรือ SUBSTR + REPLACE
SELECT 
    printf('%-20s', product_name) AS padded_name
FROM products
LIMIT 3;
```

---

## 10.10 LIKE vs ILIKE

```sql
-- ============================================================
-- EXAMPLE 41: LIKE - case sensitive
-- ============================================================

-- LIKE ทั่วไปเป็น case-sensitive (ขึ้นกับ collation)
SELECT * FROM products WHERE product_name LIKE '%Pro%';
-- จะหา: 'MacBook Pro', 'iPhone 15 Pro' ฯลฯ
-- ไม่พบ: 'macbook pro' (lowercase)

-- ============================================================
-- EXAMPLE 42: ILIKE - case insensitive (PostgreSQL เท่านั้น)
-- ============================================================

-- PostgreSQL:
-- SELECT * FROM products WHERE product_name ILIKE '%pro%';
-- จะหา: 'MacBook Pro', 'macbook pro', 'MACBOOK PRO' ทั้งหมด

-- ============================================================
-- EXAMPLE 43: เลียนแบบ ILIKE ใน MySQL/SQLite
-- ============================================================

-- วิธีที่ 1: ใช้ LOWER + LIKE
SELECT * FROM products
WHERE LOWER(product_name) LIKE '%pro%';

-- วิธีที่ 2: MySQL ใช้ LIKE with case-insensitive collation
-- SELECT * FROM products 
-- WHERE product_name LIKE '%pro%' COLLATE utf8mb4_general_ci;

-- ============================================================
-- EXAMPLE 44: LIKE patterns
-- ============================================================

-- % = wildcard หลายตัวอักษร
-- _ = wildcard 1 ตัวอักษร

-- สินค้าที่ขึ้นต้นด้วย 'Mac'
SELECT product_name FROM products WHERE product_name LIKE 'Mac%';

-- สินค้าที่ลงท้ายด้วย 'Pro'
SELECT product_name FROM products WHERE product_name LIKE '%Pro';

-- สินค้าที่มี 'phone' อยู่ที่ไหนก็ได้
SELECT product_name FROM products WHERE LOWER(product_name) LIKE '%phone%';

-- Email ที่มี 'j' เป็นตัวที่ 8
SELECT email FROM employees WHERE email LIKE '_______j%';

-- Phone ที่ขึ้นต้นด้วย '08' (format: 08X-XXX-XXXX)
SELECT phone FROM employees WHERE phone LIKE '08_-%';

-- ============================================================
-- EXAMPLE 45: Escape special characters ใน LIKE
-- ============================================================

-- ถ้าต้องการค้นหา literal % หรือ _ ต้องใช้ ESCAPE
-- SQLite/PostgreSQL:
SELECT * FROM products 
WHERE product_name LIKE '%50\%%' ESCAPE '\';
-- ค้นหา product names ที่มี '50%' อยู่ใน string

-- SQL Server ใช้ []
-- WHERE name LIKE '%50[%]%'
```

---

## 10.11 Regular Expressions

```sql
-- ============================================================
-- EXAMPLE 46: REGEXP ใน MySQL
-- ============================================================

-- MySQL: REGEXP หรือ RLIKE
-- SELECT * FROM employees WHERE phone REGEXP '^08[0-9]-';
-- ค้นหา phone ที่ขึ้นต้นด้วย 08X-

-- SELECT * FROM employees WHERE email REGEXP '[A-Z]';
-- หา emails ที่มีตัวพิมพ์ใหญ่ (data quality check)

-- ============================================================
-- EXAMPLE 47: REGEXP ใน PostgreSQL
-- ============================================================

-- PostgreSQL: ~ (match), !~ (not match), ~* (case insensitive)
-- SELECT * FROM employees WHERE phone ~ '^08[0-9]-';

-- SELECT * FROM employees WHERE email ~* '@COMPANY';
-- ~* = case insensitive

-- ============================================================
-- EXAMPLE 48: GLOB ใน SQLite
-- ============================================================

-- SQLite มี GLOB (case sensitive, UNIX-style wildcards)
-- * = หลายตัวอักษร (เหมือน % ใน LIKE)
-- ? = 1 ตัวอักษร (เหมือน _ ใน LIKE)
-- [abc] = character class

-- หา phone ที่ขึ้นต้นด้วย 08:
SELECT phone FROM employees WHERE phone GLOB '08*';

-- หา email ที่ลงท้ายด้วย .th:
SELECT email FROM employees WHERE email GLOB '*.th';

-- ============================================================
-- EXAMPLE 49: REGEXP_LIKE ใน Oracle/MySQL 8+
-- ============================================================

-- MySQL 8+ / Oracle:
-- SELECT * FROM employees WHERE REGEXP_LIKE(phone, '^0[689]');
-- หา phones ที่ขึ้นต้นด้วย 06, 08, หรือ 09

-- ============================================================
-- EXAMPLE 50: REGEXP_REPLACE (PostgreSQL/MySQL 8+)
-- ============================================================

-- PostgreSQL: ลบ non-digit characters จาก phone
-- SELECT REGEXP_REPLACE(phone, '[^0-9]', '', 'g') AS digits
-- FROM employees WHERE phone IS NOT NULL;

-- ผลลัพธ์:
-- phone        | digits
-- -------------|--------
-- 081-234-5678 | 0812345678
```

---

## 10.12 Cross-Database Comparison Table

```
===================================================================================
FUNCTION         | SQLite          | PostgreSQL      | MySQL           | SQL Server
===================================================================================
UPPER            | UPPER()         | UPPER()         | UPPER()         | UPPER()
LOWER            | LOWER()         | LOWER()         | LOWER()         | LOWER()
INITCAP          | ไม่มี           | INITCAP()       | ไม่มี           | ไม่มี
----------------------------------------------------------------------------------
TRIM             | TRIM()          | TRIM()          | TRIM()          | LTRIM(RTRIM())
LTRIM            | LTRIM()         | LTRIM()         | LTRIM()         | LTRIM()
RTRIM            | RTRIM()         | RTRIM()         | RTRIM()         | RTRIM()
TRIM chars       | TRIM(x,str)     | TRIM(BOTH x FROM| TRIM(str)       | ไม่มี built-in
----------------------------------------------------------------------------------
LENGTH           | LENGTH()        | LENGTH() *      | CHAR_LENGTH()** | LEN()
OCTET_LENGTH     | LENGTH()        | OCTET_LENGTH()  | LENGTH()        | DATALENGTH()
----------------------------------------------------------------------------------
SUBSTRING        | SUBSTR()        | SUBSTRING()     | SUBSTRING()     | SUBSTRING()
SUBSTR           | SUBSTR()        | SUBSTR()        | SUBSTR()        | ไม่มี
LEFT             | SUBSTR(s,1,n)   | LEFT()          | LEFT()          | LEFT()
RIGHT            | SUBSTR(s,-n)    | RIGHT()         | RIGHT()         | RIGHT()
----------------------------------------------------------------------------------
CONCAT           | ||              | CONCAT()/ ||    | CONCAT()        | CONCAT()/ +
CONCAT_WS        | ไม่มี           | CONCAT_WS()     | CONCAT_WS()     | ไม่มี
----------------------------------------------------------------------------------
REPLACE          | REPLACE()       | REPLACE()       | REPLACE()       | REPLACE()
REGEXP_REPLACE   | ไม่มี           | REGEXP_REPLACE()| REGEXP_REPLACE()| ไม่มี
----------------------------------------------------------------------------------
FIND_IN_STRING   | INSTR(s,sub)    | POSITION/STRPOS | LOCATE/INSTR    | CHARINDEX()
----------------------------------------------------------------------------------
LPAD             | printf('%0Nd',x)| LPAD()          | LPAD()          | ไม่มี
RPAD             | ไม่มี           | RPAD()          | RPAD()          | ไม่มี
----------------------------------------------------------------------------------
CASE SENSITIVE   | LIKE (ci*)      | LIKE (cs)       | LIKE (ci*)      | LIKE (ci*)
INSENSITIVE      | LIKE (default)  | ILIKE           | LIKE (default)  | LIKE (default)
REGEX            | GLOB            | ~ / ~* / !~     | REGEXP/RLIKE    | LIKE '[pattern]'
===================================================================================

* PostgreSQL: LENGTH = characters, OCTET_LENGTH = bytes
** MySQL: LENGTH = bytes, CHAR_LENGTH = characters (ใช้ CHAR_LENGTH สำหรับ Unicode)
* ci = case insensitive (by default), cs = case sensitive
```

---

## 10.13 String Functions ใน Real World Scenarios

```sql
-- ============================================================
-- SCENARIO 1: Normalize ข้อมูลก่อน import
-- ============================================================

-- ตรวจสอบความสะอาดของข้อมูล
SELECT 
    employee_id,
    first_name,
    LENGTH(first_name) AS name_len,
    LENGTH(TRIM(first_name)) AS trim_len,
    CASE WHEN LENGTH(first_name) != LENGTH(TRIM(first_name)) 
         THEN 'HAS SPACES' ELSE 'OK' END AS space_check,
    CASE WHEN first_name != LOWER(first_name) AND 
              first_name != UPPER(first_name) AND
              first_name != first_name  -- custom check
         THEN 'MIXED CASE' ELSE 'OK' END AS case_check
FROM employees;

-- ============================================================
-- SCENARIO 2: สร้าง Username จาก Name
-- ============================================================

SELECT 
    first_name,
    last_name,
    LOWER(SUBSTR(first_name, 1, 1) || last_name) AS username_style1,
    LOWER(first_name || '_' || last_name) AS username_style2
FROM employees
LIMIT 5;

-- ============================================================
-- SCENARIO 3: Mask ข้อมูล Sensitive (PII)
-- ============================================================

-- Mask เบอร์โทร: แสดงเฉพาะ 4 ตัวท้าย
SELECT 
    employee_id,
    phone,
    REPLACE(SUBSTR(phone, 1, LENGTH(phone) - 4), 
            SUBSTR(phone, 1, LENGTH(phone) - 4),
            REPLACE(SUBSTR(phone, 1, LENGTH(phone) - 4), 
                    SUBSTR(phone, 1, LENGTH(phone) - 4),
                    '****-***')) AS masked_phone
FROM employees
WHERE phone IS NOT NULL;

-- วิธีง่ายกว่า:
SELECT 
    phone,
    '****-***-' || SUBSTR(phone, -4) AS masked_phone
FROM employees
WHERE phone IS NOT NULL;

-- ============================================================
-- SCENARIO 4: Parse CSV string (SQLite)
-- ============================================================

-- สมมุติมี tags เก็บเป็น 'tag1,tag2,tag3'
-- หาว่ามี 'Electronics' ใน tags หรือไม่
SELECT 
    product_name,
    category,
    CASE WHEN INSTR(',' || category || ',', ',Electronics,') > 0 
         THEN 'Is Electronics' 
         ELSE 'Not Electronics' 
    END AS check_result
FROM products;

-- ============================================================
-- SCENARIO 5: Email Validation (Basic)
-- ============================================================

-- ตรวจสอบ email format อย่างง่าย
SELECT 
    email,
    CASE 
        WHEN email IS NULL THEN 'NULL'
        WHEN INSTR(email, '@') = 0 THEN 'No @ sign'
        WHEN INSTR(email, '.') = 0 THEN 'No dot'
        WHEN INSTR(email, '@') = 1 THEN 'Starts with @'
        WHEN INSTR(email, ' ') > 0 THEN 'Has spaces'
        ELSE 'Valid format'
    END AS email_check
FROM employees;

-- ============================================================
-- SCENARIO 6: Full-Text Search Simulation
-- ============================================================

-- ค้นหา products ที่ชื่อหรือ category มีคำที่ต้องการ
-- SQLite:
SELECT product_name, category, unit_price
FROM products
WHERE LOWER(product_name) LIKE '%apple%'
   OR LOWER(category) LIKE '%electronics%'
ORDER BY unit_price DESC;

-- ============================================================
-- SCENARIO 7: สร้าง Report Header
-- ============================================================

SELECT 
    'รายงาน: ' || 
    UPPER(category) || 
    ' (จำนวน ' || CAST(COUNT(*) AS TEXT) || ' รายการ)' AS report_header,
    MIN(unit_price) AS min_price,
    MAX(unit_price) AS max_price
FROM products
WHERE discontinued = FALSE
GROUP BY category;
```

---

## 10.14 50+ ตัวอย่างสรุป

```sql
-- UPPER / LOWER
SELECT UPPER('hello') AS u, LOWER('HELLO') AS l;
SELECT UPPER(first_name), LOWER(email) FROM employees LIMIT 3;
SELECT * FROM employees WHERE LOWER(email) LIKE '%@company%';
SELECT UPPER(first_name || ' ' || last_name) AS name FROM employees;
SELECT DISTINCT LOWER(category) FROM products ORDER BY 1;

-- TRIM
SELECT TRIM('  hello  ') AS t;
SELECT LTRIM('  hello  ') AS lt;
SELECT RTRIM('  hello  ') AS rt;
SELECT * FROM customers WHERE full_name != TRIM(full_name);
SELECT TRIM(LOWER(email)) FROM employees;

-- LENGTH
SELECT LENGTH('SQL') AS len;
SELECT product_name, LENGTH(product_name) AS len FROM products ORDER BY 2 DESC;
SELECT * FROM products WHERE LENGTH(product_name) > 25;
SELECT AVG(LENGTH(first_name)) FROM employees;

-- SUBSTR/SUBSTRING
SELECT SUBSTR('Hello World', 1, 5);
SELECT SUBSTR(email, 1, INSTR(email,'@')-1) AS username FROM employees;
SELECT SUBSTR(phone, -4) AS last4 FROM employees WHERE phone IS NOT NULL;
SELECT SUBSTR(hire_date, 1, 4) AS year FROM employees;
SELECT SUBSTR(product_name, 1, 15) || '...' AS short_name FROM products WHERE LENGTH(product_name) > 15;

-- CONCAT / ||
SELECT 'Hello' || ' ' || 'World';
SELECT first_name || ' ' || last_name FROM employees;
SELECT full_name || ' (' || city || ')' FROM customers;
SELECT 'Order #' || CAST(order_id AS TEXT) FROM orders;
SELECT LOWER(first_name) || '@company.co.th' AS email FROM employees;

-- REPLACE
SELECT REPLACE('Hello World', 'World', 'SQL');
SELECT REPLACE(phone, '-', '') FROM employees WHERE phone IS NOT NULL;
SELECT REPLACE(REPLACE(phone, '+66', '0'), '-', '') FROM employees WHERE phone IS NOT NULL;
SELECT REPLACE(email, '@company.co.th', '@test.com') FROM employees;

-- INSTR / POSITION
SELECT INSTR('hello world', 'world');
SELECT INSTR(email, '@') AS at_pos FROM employees;
SELECT CASE WHEN INSTR(email,'@company') > 0 THEN 'Internal' ELSE 'External' END FROM employees;

-- LIKE / GLOB
SELECT * FROM products WHERE product_name LIKE 'Mac%';
SELECT * FROM products WHERE product_name LIKE '%Pro';
SELECT * FROM products WHERE LOWER(product_name) LIKE '%phone%';
SELECT * FROM employees WHERE phone LIKE '08_%';
SELECT * FROM employees WHERE email GLOB '*.th';

-- Combinations
SELECT UPPER(SUBSTR(category,1,3)) || printf('%04d', product_id) AS code FROM products;
SELECT LENGTH(TRIM(first_name)) AS clean_len FROM employees;
SELECT INSTR(email,'@') > 0 AS has_at FROM employees;
SELECT REPLACE(TRIM(LOWER(email)),' ','') AS clean_email FROM employees;

-- String analysis
SELECT product_name, LENGTH(product_name) - LENGTH(REPLACE(product_name,' ','')) AS word_spaces FROM products;
SELECT COUNT(*) FROM employees WHERE INSTR(email, '.') > 0;
SELECT DISTINCT SUBSTR(phone, 1, 2) AS prefix FROM employees WHERE phone IS NOT NULL;
```

---

## 10.15 สรุป Cross-Database Quick Reference

```sql
-- ============================================================
-- PORTABLE CODE (ทำงานได้ทุก database)
-- ============================================================

-- UPPER / LOWER ใช้ได้ทุกที่
SELECT UPPER(name), LOWER(email) FROM users;

-- TRIM ใช้ได้ทุกที่
SELECT TRIM(name) FROM users;

-- SUBSTRING ใช้ได้ใน PostgreSQL, MySQL, SQL Server
-- SUBSTR ใช้ได้ใน SQLite, PostgreSQL, MySQL
SELECT SUBSTRING(name, 1, 10) FROM users;  -- safe
SELECT SUBSTR(name, 1, 10) FROM users;     -- safe (ยกเว้น SQL Server)

-- REPLACE ใช้ได้ทุกที่
SELECT REPLACE(phone, '-', '') FROM users;

-- ============================================================
-- NON-PORTABLE (ต้องเช็ค database)
-- ============================================================

-- LEFT / RIGHT: ไม่มีใน SQLite
-- LPAD / RPAD: ไม่มีใน SQLite, SQL Server
-- INITCAP: PostgreSQL เท่านั้น
-- ILIKE: PostgreSQL เท่านั้น
-- ||: ไม่ใช้ใน MySQL (ใช้ CONCAT แทน)
-- CONCAT: ทำงานต่างกันกับ NULL ใน MySQL vs PostgreSQL

-- ============================================================
-- RECOMMENDATION สำหรับ Portable Code
-- ============================================================

-- ใช้ SUBSTR แทน LEFT/RIGHT
-- ใช้ LOWER(x) LIKE pattern แทน ILIKE
-- ใช้ COALESCE(x, '') ก่อน concatenate
-- ใช้ CHAR_LENGTH แทน LENGTH (safe สำหรับ Unicode)
-- ทดสอบบน target database เสมอ
```

---

## 📝 แบบฝึกหัดท้ายบท

**คำถาม 1:** แสดงชื่อ-นามสกุลพนักงานทั้งหมดเป็นตัวพิมพ์ใหญ่ในรูปแบบ "FIRST LAST"

**คำถาม 2:** ดึง username จาก email (ส่วนก่อน @)

**คำถาม 3:** หาพนักงานที่ email มีตัวพิมพ์ใหญ่ (ใช้ LOWER เปรียบเทียบ)

**คำถาม 4:** แสดง 10 ตัวอักษรแรกของ product_name ตามด้วย '...' (ถ้ายาวกว่า 10 ตัว)

**คำถาม 5:** ลบ '-' ออกจาก phone numbers ทั้งหมด

**คำถาย 6:** หา product names ที่มี space มากกว่า 2 ตัว (นับโดยเปรียบเทียบ LENGTH)

**คำถาม 7:** สร้าง full address ในรูปแบบ "address, city, country"

**คำถาม 8:** หา customers ที่อยู่ในเมืองที่มีตัวอักษรน้อยกว่า 10 ตัว

**คำถาม 9:** แสดง employees ที่ phone ขึ้นต้นด้วย '08'

**คำถาม 10:** สร้าง employee code ในรูปแบบ: 3 ตัวแรกของ last_name (uppercase) + padded employee_id (4 digits)

---

## ✅ เฉลยแบบฝึกหัด

### เฉลยที่ 1:
```sql
SELECT 
    UPPER(first_name) || ' ' || UPPER(last_name) AS full_name_caps
FROM employees
ORDER BY last_name;
```

### เฉลยที่ 2:
```sql
-- SQLite:
SELECT 
    email,
    SUBSTR(email, 1, INSTR(email, '@') - 1) AS username
FROM employees
WHERE email IS NOT NULL;
```

### เฉลยที่ 3:
```sql
-- หา emails ที่มีตัวพิมพ์ใหญ่ (email ควรเป็น lowercase ทั้งหมด)
SELECT employee_id, first_name, email
FROM employees
WHERE email != LOWER(email);
```

### เฉลยที่ 4:
```sql
SELECT 
    product_name,
    CASE 
        WHEN LENGTH(product_name) > 10 
        THEN SUBSTR(product_name, 1, 10) || '...'
        ELSE product_name 
    END AS short_name
FROM products;
```

### เฉลยที่ 5:
```sql
SELECT 
    employee_id,
    phone AS original_phone,
    REPLACE(phone, '-', '') AS clean_phone
FROM employees
WHERE phone IS NOT NULL;
```

### เฉลยที่ 6:
```sql
SELECT 
    product_name,
    LENGTH(product_name) - LENGTH(REPLACE(product_name, ' ', '')) AS space_count
FROM products
WHERE LENGTH(product_name) - LENGTH(REPLACE(product_name, ' ', '')) > 2
ORDER BY space_count DESC;
```

### เฉลยที่ 7:
```sql
SELECT 
    full_name,
    COALESCE(address, '') || 
    CASE WHEN address IS NOT NULL THEN ', ' ELSE '' END ||
    city || ', ' || country AS full_address
FROM customers
ORDER BY full_name;
```

### เฉลยที่ 8:
```sql
SELECT full_name, city, LENGTH(city) AS city_len
FROM customers
WHERE LENGTH(city) < 10
ORDER BY city_len;
```

### เฉลยที่ 9:
```sql
SELECT employee_id, first_name, phone
FROM employees
WHERE phone LIKE '08%'
ORDER BY phone;
```

### เฉลยที่ 10:
```sql
SELECT 
    employee_id,
    first_name,
    last_name,
    UPPER(SUBSTR(last_name, 1, 3)) || printf('%04d', employee_id) AS employee_code
FROM employees
ORDER BY employee_code;
```

---

## ❓ Quiz 10 ข้อท้ายบท

**ข้อ 1:** ฟังก์ชันใดที่ใช้ตัด whitespace ทั้งสองด้านของ string?
- A) STRIP()
- B) CLEAN()
- C) **TRIM()** ✓
- D) LTRIM()

**ข้อ 2:** ใน MySQL ควรใช้ฟังก์ชันใดวัดความยาวของ Unicode string?
- A) LENGTH()
- B) **CHAR_LENGTH()** ✓
- C) SIZE()
- D) STRLEN()

**ข้อ 3:** `INSTR('hello world', 'world')` คืนค่าอะไร?
- A) 5
- B) 6
- C) **7** ✓
- D) 8

**ข้อ 4:** ใน PostgreSQL ฟังก์ชันใดทำ case-insensitive LIKE?
- A) CLIKE()
- B) **ILIKE** ✓
- C) ILIKE()
- D) NOCASE LIKE

**ข้อ 5:** `REPLACE('ABC-123', '-', '')` คืนค่าอะไร?
- A) 'ABC 123'
- B) 'ABC_123'
- C) **'ABC123'** ✓
- D) 'abc123'

**ข้อ 6:** ใน SQLite ถ้าต้องการ functionality เหมือน LEFT(name, 5) ควรใช้อะไร?
- A) FIRST(name, 5)
- B) LEFT_STR(name, 5)
- C) **SUBSTR(name, 1, 5)** ✓
- D) SUBSTRING(name FROM 1 TO 5)

**ข้อ 7:** `CONCAT('Hello', NULL, 'World')` ใน MySQL คืนค่าอะไร?
- A) 'HelloWorld'
- B) 'Hello World'
- C) 'Hello null World'
- D) **NULL** ✓

**ข้อ 8:** GLOB ใน SQLite ต่างจาก LIKE ตรงไหน?
- A) GLOB เร็วกว่า
- B) **GLOB เป็น case-sensitive, LIKE เป็น case-insensitive** ✓
- C) GLOB รองรับ regex
- D) GLOB ไม่มี wildcard

**ข้อ 9:** `UPPER(SUBSTR(last_name, 1, 1)) || LOWER(SUBSTR(last_name, 2))` ทำอะไร?
- A) แปลงเป็นตัวพิมพ์ใหญ่ทั้งหมด
- B) แปลงเป็นตัวพิมพ์เล็กทั้งหมด
- C) **Capitalize ตัวแรก, เล็กที่เหลือ (เหมือน INITCAP)** ✓
- D) กลับลำดับตัวอักษร

**ข้อ 10:** ฟังก์ชันที่ Portable ที่สุด (ทำงานได้ทุก database) คือ?
- A) ILIKE
- B) LEFT(), RIGHT()
- C) INITCAP()
- D) **UPPER(), LOWER(), TRIM(), REPLACE()** ✓

---

## ➡️ บทถัดไป

**[Part 011: Numeric Functions](part-011.md)**

บทถัดไปเราจะเรียน Numeric Functions: ROUND, FLOOR, CEILING, ABS, MOD, POWER, SQRT และอื่นๆ!

---

*Part 010 of 120 | หลักสูตร SQL ครบวงจร*
