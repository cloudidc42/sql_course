# Part 001: Introduction to Databases and SQL

> **หลักสูตร SQL ครบวงจร | Part 1 of 120**

---

## 🎯 สิ่งที่จะได้เรียนรู้ในบทนี้

- ประวัติความเป็นมาของฐานข้อมูล
- ความแตกต่างระหว่าง RDBMS และ NoSQL
- SQL คืออะไรและทำไมต้องเรียน
- ประเภทของคำสั่ง SQL (DDL, DML, DCL, TCL)
- ระบบ RDBMS ยอดนิยม
- การติดตั้ง SQLite สำหรับเริ่มต้น
- การรัน SQL Query แรกของคุณ

**เวลาที่ใช้เรียน**: ประมาณ 2-3 ชั่วโมง

---

## 1.1 ฐานข้อมูลคืออะไร?

### ความหมายพื้นฐาน

**ฐานข้อมูล (Database)** คือการรวบรวมข้อมูลที่มีการจัดระเบียบ (organized collection of data) ที่จัดเก็บในระบบคอมพิวเตอร์ โดยข้อมูลเหล่านั้นสามารถเข้าถึง จัดการ และอัปเดตได้อย่างมีประสิทธิภาพ

ก่อนจะมีฐานข้อมูล มนุษย์เก็บข้อมูลอย่างไร?

```
ยุคโบราณ:
├── แผ่นดินเหนียว (Clay tablets) - 3500 ปีก่อนคริสตกาล
├── กระดาษปาปิรัส (Papyrus scrolls)
├── สมุดบัญชี (Ledger books)
└── ระบบ Card catalog ในห้องสมุด

ยุคดิจิทัลแรก:
├── ไฟล์ข้อความธรรมดา (Flat files)
├── Spreadsheets
└── Custom file formats
```

**ปัญหาของการเก็บข้อมูลในไฟล์ธรรมดา:**

1. **Data Redundancy (ข้อมูลซ้ำซ้อน)** - ข้อมูลเดียวกันถูกเก็บในหลายที่
2. **Data Inconsistency (ข้อมูลไม่สอดคล้อง)** - แก้ที่หนึ่งแต่อีกที่ยังเก่า
3. **Difficulty in Accessing Data** - ต้องเขียนโปรแกรมพิเศษทุกครั้ง
4. **Data Isolation** - ข้อมูลกระจัดกระจาย
5. **Concurrent Access Problems** - ถ้าหลายคนใช้พร้อมกัน ข้อมูลจะเสียหาย
6. **Security Problems** - ควบคุมสิทธิ์การเข้าถึงยาก

---

## 1.2 ประวัติความเป็นมาของ Database

### Timeline สำคัญ

```
1950s - เริ่มยุคคอมพิวเตอร์
    └── เก็บข้อมูลบน Magnetic Tape
    └── เข้าถึงข้อมูลแบบ Sequential เท่านั้น

1960s - แรกเริ่มของ Database
    └── 1961: Charles Bachman ออกแบบ IDS (Integrated Data Store)
            - ถือเป็น DBMS ตัวแรกของโลก
    └── 1968: IBM พัฒนา IMS (Information Management System)
            - ใช้ใน Apollo Space Program!
            - ยังใช้งานอยู่ในบางธนาคารวันนี้

1970s - การปฏิวัติด้วย Relational Model
    └── 1970: Edgar F. Codd เผยแพร่บทความ
            "A Relational Model of Data for Large Shared Data Banks"
            - เปลี่ยนโลกของ database ไปตลอดกาล!
    └── 1974: SEQUEL (Structured English Query Language)
            - ต่อมาเปลี่ยนชื่อเป็น SQL
    └── 1979: Oracle v2 - เชิงพาณิชย์ตัวแรก
    └── 1979: IBM System R สาธิต SQL ต่อสาธารณะ

1980s - SQL กลายเป็น Standard
    └── 1986: SQL-86 - Standard แรกโดย ANSI
    └── 1987: SQL-87 - ยืนยันโดย ISO
    └── 1989: SQL-89

1990s - ยุคทองของ RDBMS
    └── 1992: SQL-92 (SQL2) - มาตรฐานสำคัญ
    └── 1995: MySQL (ฟรีและโอเพ่นซอร์ส)
    └── 1995: PostgreSQL (เดิมชื่อ POSTGRES)
    └── 1996: SQLite เริ่มพัฒนา

2000s - NoSQL เกิดขึ้น
    └── 2000: SQL:1999 (SQL3) - รองรับ OO features
    └── 2003: SQL:2003 - XML, Window Functions
    └── 2007: MongoDB
    └── 2008: Cassandra (Facebook)

2010s - Big Data และ Cloud
    └── 2011: SQL:2011 - Temporal Tables
    └── 2012: Google Spanner
    └── 2016: SQL:2016 - JSON, Polymorphic tables
    └── Cloud SQL services (AWS RDS, Google Cloud SQL, Azure SQL)

2020s - ยุคปัจจุบัน
    └── NewSQL databases (CockroachDB, TiDB)
    └── HTAP (Hybrid Transactional/Analytical Processing)
    └── Serverless databases
```

### บุคคลสำคัญในประวัติ SQL

**Edgar F. Codd (1923-2003)**
- นักวิทยาศาสตร์ IBM
- ผู้คิดค้น Relational Model ในปี 1970
- ได้รับ Turing Award ในปี 1981
- หลักการ "12 Rules of Codd" ยังใช้อยู่วันนี้

**Donald D. Chamberlin และ Raymond F. Boyce**
- พัฒนา SEQUEL (ต่อมาเป็น SQL) ที่ IBM ในปี 1974
- ออกแบบให้ใช้งานง่าย ไม่ต้องเป็นโปรแกรมเมอร์

---

## 1.3 RDBMS คืออะไร?

**Relational Database Management System (RDBMS)** คือระบบซอฟต์แวร์ที่จัดการฐานข้อมูลเชิงสัมพันธ์ โดยข้อมูลถูกจัดเก็บในรูปแบบ **ตาราง (Tables)**

### โครงสร้างพื้นฐาน

```
Database
└── Tables (ตาราง)
    ├── Rows (แถว / Records / Tuples)
    └── Columns (คอลัมน์ / Fields / Attributes)

ตัวอย่าง: employees table
┌────┬──────────────┬────────────┬────────┐
│ id │ name         │ department │ salary │
├────┼──────────────┼────────────┼────────┤
│  1 │ สมชาย นักเขียน│ IT         │ 45000  │
│  2 │ สมหญิง ดีงาม  │ HR         │ 38000  │
│  3 │ วิภา รักงาน  │ Finance    │ 52000  │
└────┴──────────────┴────────────┴────────┘
```

### คำศัพท์สำคัญ

| คำศัพท์ SQL | คำศัพท์อื่น | ความหมาย |
|------------|------------|---------|
| Table | Relation | ตารางเก็บข้อมูล |
| Row | Record, Tuple | แถวข้อมูลหนึ่งแถว |
| Column | Field, Attribute | คอลัมน์ข้อมูล |
| Primary Key | PK | ตัวระบุแถวที่ไม่ซ้ำกัน |
| Foreign Key | FK | อ้างอิงไปยัง Primary Key อีกตาราง |
| Index | - | โครงสร้างช่วยค้นหาเร็วขึ้น |

### Codd's 12 Rules (กฎ 12 ข้อของ Codd)

Edgar Codd กำหนด 12 กฎที่ RDBMS "แท้จริง" ควรปฏิบัติตาม:

```
Rule 0:  The Foundation Rule - ต้องใช้ relational capabilities จัดการทุกอย่าง
Rule 1:  The Information Rule - ข้อมูลทุกอย่างต้องอยู่ในตาราง
Rule 2:  Guaranteed Access Rule - ทุก value เข้าถึงได้ด้วย table name + PK + column name
Rule 3:  Systematic Treatment of NULL - NULL ต้องแยกจาก 0 และ empty string
Rule 4:  Dynamic Online Catalog - metadata ต้องอยู่ในตารางด้วย
Rule 5:  Comprehensive Data Sublanguage - ต้องมี language ครบสำหรับ DML, DDL, etc.
Rule 6:  View Updating Rule - Views ที่ updateable ได้ต้องสามารถ update ได้จริง
Rule 7:  High-Level Insert, Update, Delete - ต้องทำได้บน set of rows ไม่ใช่แค่ row เดียว
Rule 8:  Physical Data Independence - app ต้องไม่ได้รับผลกระทบจากการเปลี่ยน physical storage
Rule 9:  Logical Data Independence - app ต้องไม่ได้รับผลกระทบจากการเปลี่ยน schema (บางอย่าง)
Rule 10: Integrity Independence - integrity rules ต้องเก็บใน catalog ไม่ใช่ใน app
Rule 11: Distribution Independence - ไม่ว่าจะ distributed หรือไม่ query ต้องทำงานเหมือนกัน
Rule 12: Non-Subversion Rule - ห้ามข้ามผ่าน relational rules ผ่าน low-level interfaces
```

---

## 1.4 RDBMS vs NoSQL - ความแตกต่างที่สำคัญ

### RDBMS (Relational Database Management System)

**ลักษณะเด่น:**
- ข้อมูลเป็น structured (มีโครงสร้างชัดเจน)
- ใช้ SQL เป็นภาษาหลัก
- รองรับ ACID transactions
- มี Schema แน่นอน (schema-on-write)
- ความสัมพันธ์ระหว่างตารางชัดเจน (Foreign Keys)

**เหมาะสำหรับ:**
- ระบบธนาคาร/การเงิน
- ระบบ ERP
- ระบบจัดการคำสั่งซื้อ
- ข้อมูลที่ต้องการความสม่ำเสมอสูง

### NoSQL (Not Only SQL)

**ประเภทหลัก:**

```
1. Document Databases
   ตัวอย่าง: MongoDB, CouchDB
   เก็บข้อมูลเป็น JSON documents
   {
     "_id": "001",
     "name": "สมชาย",
     "skills": ["Python", "SQL", "JavaScript"],
     "address": {
       "city": "Bangkok",
       "zip": "10110"
     }
   }

2. Key-Value Stores
   ตัวอย่าง: Redis, DynamoDB
   เก็บเป็น key-value pairs
   "user:001" -> "สมชาย นักเขียน"
   "session:abc123" -> {expires: 3600, user_id: 001}

3. Column-Family Databases
   ตัวอย่าง: Cassandra, HBase
   เก็บข้อมูลแบบ column-oriented
   เหมาะสำหรับ write-heavy workloads

4. Graph Databases
   ตัวอย่าง: Neo4j, Amazon Neptune
   เก็บข้อมูลเป็น nodes และ edges
   เหมาะสำหรับ social networks, recommendation engines
```

### การเปรียบเทียบ RDBMS vs NoSQL

| คุณสมบัติ | RDBMS | NoSQL |
|-----------|-------|-------|
| Schema | Fixed (rigid) | Flexible (dynamic) |
| Query Language | SQL (standard) | Varies by product |
| Transactions | Full ACID | Eventually consistent (mostly) |
| Scaling | Vertical (scale up) | Horizontal (scale out) |
| Joins | Easy and efficient | Difficult, avoid |
| Consistency | Strong | Eventual (usually) |
| Learning Curve | Moderate | Varies |
| Maturity | 50+ years | 10-15 years |
| Use Case | Structured data | Unstructured/semi-structured |

### เมื่อไหร่ควรใช้อะไร?

```
ใช้ RDBMS เมื่อ:
✓ ข้อมูลมีโครงสร้างชัดเจน
✓ ต้องการ ACID transactions
✓ มีความสัมพันธ์ซับซ้อนระหว่างข้อมูล
✓ ต้องการ complex queries
✓ ต้องการความถูกต้องสูง (เช่น การเงิน)

ใช้ NoSQL เมื่อ:
✓ ข้อมูล unstructured หรือ semi-structured
✓ ต้องการ scale horizontal อย่างมาก
✓ ข้อมูลเปลี่ยนรูปแบบบ่อย
✓ Write performance สำคัญมาก
✓ ข้อมูล time-series หรือ event logs
✓ Caching layers
```

---

## 1.5 SQL คืออะไร?

**SQL (Structured Query Language)** คือภาษามาตรฐานที่ใช้สื่อสารกับ Relational Database ออกเสียงว่า "S-Q-L" หรือ "sequel"

### ทำไมต้องเรียน SQL?

```
สถิติที่น่าสนใจ:
├── SQL เป็น skill #1 ที่บริษัทเทคโนโลยีต้องการ
├── ใช้ใน Data Science, Data Engineering, Backend Dev
├── ยังคง relevant หลังจาก 50 ปี
├── Stack Overflow Survey 2023: SQL = ภาษาที่ใช้มากที่สุด
└── เงินเดือน SQL Developer: 60,000-150,000+ บาท/เดือน

ใครต้องรู้ SQL?
├── Software Developers
├── Data Scientists / Data Analysts
├── Database Administrators (DBA)
├── Business Analysts
├── Data Engineers
├── System Architects
└── ผู้จัดการที่ต้องการวิเคราะห์ข้อมูลเอง
```

### ลักษณะของภาษา SQL

SQL เป็นภาษาประเภท **Declarative** - คุณบอก "อะไร" แต่ไม่ต้องบอก "อย่างไร"

```sql
-- บอกว่า "อยากได้อะไร" ไม่ต้องบอกวิธี
SELECT name, salary
FROM employees
WHERE department = 'IT'
ORDER BY salary DESC;

-- เปรียบเทียบกับ Python (Imperative) ที่ต้องบอกวิธี:
employees = load_from_file("employees.csv")
it_employees = []
for emp in employees:
    if emp['department'] == 'IT':
        it_employees.append(emp)
it_employees.sort(key=lambda x: x['salary'], reverse=True)
for emp in it_employees:
    print(emp['name'], emp['salary'])
```

SQL ง่ายกว่ามาก!

---

## 1.6 ประเภทของคำสั่ง SQL

SQL แบ่งออกเป็น 5 หมวดหลัก:

### 1. DDL - Data Definition Language

ใช้สำหรับ **สร้างและแก้ไขโครงสร้าง** ของฐานข้อมูล

```sql
-- CREATE: สร้าง object ใหม่
CREATE TABLE employees (
    id INTEGER PRIMARY KEY,
    name VARCHAR(100),
    salary DECIMAL(10,2)
);

-- ALTER: แก้ไข object ที่มีอยู่
ALTER TABLE employees ADD COLUMN department VARCHAR(50);
ALTER TABLE employees MODIFY COLUMN salary DECIMAL(12,2);

-- DROP: ลบ object
DROP TABLE employees;
DROP DATABASE company_db;

-- TRUNCATE: ลบข้อมูลทั้งหมดในตาราง (เก็บโครงสร้างไว้)
TRUNCATE TABLE employees;

-- RENAME: เปลี่ยนชื่อ
RENAME TABLE employees TO staff;
```

### 2. DML - Data Manipulation Language

ใช้สำหรับ **จัดการข้อมูล** ในตาราง

```sql
-- INSERT: เพิ่มข้อมูลใหม่
INSERT INTO employees (id, name, salary)
VALUES (1, 'สมชาย นักเขียน', 45000);

-- SELECT: ดึงข้อมูล
SELECT name, salary
FROM employees
WHERE salary > 40000;

-- UPDATE: แก้ไขข้อมูล
UPDATE employees
SET salary = 50000
WHERE id = 1;

-- DELETE: ลบข้อมูล
DELETE FROM employees
WHERE id = 1;
```

### 3. DCL - Data Control Language

ใช้สำหรับ **ควบคุมสิทธิ์การเข้าถึง**

```sql
-- GRANT: ให้สิทธิ์
GRANT SELECT ON employees TO reporting_user;
GRANT INSERT, UPDATE ON orders TO sales_team;

-- REVOKE: เพิกถอนสิทธิ์
REVOKE SELECT ON employees FROM reporting_user;
REVOKE ALL PRIVILEGES ON orders FROM sales_team;
```

### 4. TCL - Transaction Control Language

ใช้สำหรับ **ควบคุม Transactions**

```sql
-- BEGIN TRANSACTION: เริ่ม transaction
BEGIN TRANSACTION;
-- หรือ
START TRANSACTION;

-- COMMIT: บันทึกการเปลี่ยนแปลง
COMMIT;

-- ROLLBACK: ยกเลิกการเปลี่ยนแปลง
ROLLBACK;

-- SAVEPOINT: จุดบันทึกกลาง
SAVEPOINT before_update;

-- ROLLBACK TO: กลับไปยัง savepoint
ROLLBACK TO before_update;

-- ตัวอย่างการใช้จริง:
BEGIN TRANSACTION;
    UPDATE accounts SET balance = balance - 1000 WHERE id = 1;
    UPDATE accounts SET balance = balance + 1000 WHERE id = 2;
COMMIT;
-- ถ้า error เกิดขึ้นระหว่างกลาง ก็ ROLLBACK ทั้งหมด
```

### 5. DQL - Data Query Language (บางคนแยกออกจาก DML)

```sql
-- SELECT เป็น DQL
SELECT * FROM employees;
```

### สรุปประเภทคำสั่ง

```
SQL Commands
├── DDL (Data Definition Language)
│   ├── CREATE
│   ├── ALTER
│   ├── DROP
│   ├── TRUNCATE
│   └── RENAME
│
├── DML (Data Manipulation Language)
│   ├── SELECT (บางคนแยกเป็น DQL)
│   ├── INSERT
│   ├── UPDATE
│   └── DELETE
│
├── DCL (Data Control Language)
│   ├── GRANT
│   └── REVOKE
│
└── TCL (Transaction Control Language)
    ├── COMMIT
    ├── ROLLBACK
    ├── SAVEPOINT
    └── SET TRANSACTION
```

---

## 1.7 ระบบ RDBMS ยอดนิยม

### 1. PostgreSQL

```
ข้อมูลพื้นฐาน:
├── เริ่มพัฒนา: 1986 ที่ UC Berkeley
├── เวอร์ชันแรก: 1995 (เปลี่ยนชื่อจาก Postgres95)
├── License: PostgreSQL License (เหมือน MIT)
├── เขียนด้วย: C
└── เว็บไซต์: postgresql.org

จุดเด่น:
✓ Full SQL compliance
✓ รองรับ JSON/JSONB
✓ Full-text search built-in
✓ Window functions ครบ
✓ Extensible (สร้าง extension ได้)
✓ PostGIS สำหรับ geospatial data
✓ Row Level Security
✓ Concurrent transactions ดีมาก (MVCC)
✓ Community ใหญ่มาก

ใช้โดย: Instagram, Apple, Spotify, Twitch, Reddit
```

```sql
-- PostgreSQL specific syntax
SELECT version();
-- PostgreSQL 16.1 on x86_64-pc-linux-gnu...

-- JSON support
SELECT '{"name": "สมชาย", "age": 30}'::jsonb ->> 'name';
-- Output: สมชาย
```

### 2. MySQL / MariaDB

```
ข้อมูลพื้นฐาน:
├── MySQL เริ่ม: 1995 โดย MySQL AB (ปัจจุบัน Oracle)
├── MariaDB แยกออกมา: 2009 โดยผู้ก่อตั้ง MySQL เดิม
├── License: GPL v2 / Commercial
└── เขียนด้วย: C, C++

จุดเด่น:
✓ ยอดนิยมมากสำหรับ web applications
✓ LAMP stack (Linux, Apache, MySQL, PHP)
✓ ง่ายต่อการเริ่มต้น
✓ ประสิทธิภาพ read สูง
✓ Replication ง่าย

ใช้โดย: Facebook (แต่ปรับแต่งเยอะมาก), YouTube, Twitter, Airbnb
```

```sql
-- MySQL specific syntax
SELECT VERSION();
-- 8.0.35

-- MySQL's GROUP_CONCAT (ไม่มีใน standard SQL)
SELECT department, GROUP_CONCAT(name SEPARATOR ', ')
FROM employees
GROUP BY department;
```

### 3. SQLite

```
ข้อมูลพื้นฐาน:
├── เริ่มพัฒนา: 2000 โดย D. Richard Hipp
├── License: Public Domain (ไม่มี license!)
├── เขียนด้วย: C
└── เว็บไซต์: sqlite.org

จุดเด่น:
✓ Serverless - ไม่ต้องติดตั้ง server
✓ เก็บข้อมูลในไฟล์เดียว (.db)
✓ Zero configuration
✓ Cross-platform
✓ Built into Android, iOS, Python, etc.
✓ เป็น database ที่ deploy มากที่สุดในโลก!

ใช้โดย: ทุก smartphone, browser (Firefox, Chrome), iTunes,
         Python (built-in), Android, iOS
```

```sql
-- SQLite ใช้ในทุกที่ที่เราไม่รู้ตัว!
-- ใน Python:
-- import sqlite3
-- conn = sqlite3.connect('my_database.db')

-- SQLite specific
SELECT sqlite_version();
-- 3.43.2
```

### 4. Microsoft SQL Server

```
ข้อมูลพื้นฐาน:
├── เริ่มพัฒนา: 1989 โดย Microsoft และ Sybase
├── License: Commercial (มี Express ฟรี)
├── เขียนด้วย: C, C++
└── เว็บไซต์: microsoft.com/sql-server

จุดเด่น:
✓ Integration กับ Windows ecosystem
✓ .NET integration ดีมาก
✓ SQL Server Integration Services (SSIS)
✓ SQL Server Analysis Services (SSAS)
✓ SQL Server Reporting Services (SSRS)
✓ BI features ครบครัน

ใช้โดย: ธนาคาร, ราชการ, องค์กรที่ใช้ Windows
```

```sql
-- SQL Server specific syntax
SELECT @@VERSION;
-- Microsoft SQL Server 2022...

-- T-SQL (Transact-SQL) features
SELECT TOP 10 name, salary
FROM employees
ORDER BY salary DESC;

-- GETDATE() ใน SQL Server
SELECT GETDATE(); -- แทน NOW() ใน standard SQL
```

### 5. Oracle Database

```
ข้อมูลพื้นฐาน:
├── เริ่มพัฒนา: 1979 (เก่าที่สุดในเชิงพาณิชย์)
├── License: Commercial (ราคาสูง)
├── เขียนด้วย: C, C++, Assembly
└── เว็บไซต์: oracle.com

จุดเด่น:
✓ Enterprise-grade ที่สุด
✓ High Availability features ครบ
✓ RAC (Real Application Clusters)
✓ Performance ระดับสูงมาก
✓ Advanced security features

ใช้โดย: ธนาคารขนาดใหญ่, ราชการ, Fortune 500
```

```sql
-- Oracle specific
SELECT * FROM v$version;

-- Oracle PL/SQL (Procedural Language)
BEGIN
    DBMS_OUTPUT.PUT_LINE('Hello World');
END;
/
```

### 6. Cloud Database Services

```
ยุคปัจจุบัน ยังมี Managed Services ด้วย:

Amazon Web Services (AWS):
├── Amazon RDS - MySQL, PostgreSQL, Oracle, SQL Server
├── Amazon Aurora - Compatible กับ MySQL/PostgreSQL แต่เร็วกว่า
└── Amazon Redshift - Data warehouse

Google Cloud Platform (GCP):
├── Cloud SQL - MySQL, PostgreSQL, SQL Server
├── Cloud Spanner - Globally distributed SQL
└── BigQuery - Serverless data warehouse

Microsoft Azure:
├── Azure SQL Database - SQL Server in cloud
├── Azure Database for PostgreSQL
└── Azure Database for MySQL
```

---

## 1.8 การติดตั้ง SQLite สำหรับเริ่มต้น

SQLite เหมาะที่สุดสำหรับการเริ่มเรียน เพราะไม่ต้องติดตั้ง server ใดๆ

### วิธีที่ 1: ดาวน์โหลดโดยตรง

**Windows:**
1. ไปที่ https://www.sqlite.org/download.html
2. ดาวน์โหลด `sqlite-tools-win32-x86-*.zip`
3. แตกไฟล์ไปที่ `C:\sqlite`
4. เพิ่ม `C:\sqlite` ใน PATH

```batch
-- เปิด Command Prompt แล้วรัน:
sqlite3
-- จะเห็น SQLite version และ prompt >
```

**macOS:**
```bash
# SQLite มาพร้อมกับ macOS แล้ว!
sqlite3 --version
# 3.39.5 2022-10-14 20:58:05

# หรือติดตั้งเวอร์ชันล่าสุดผ่าน Homebrew
brew install sqlite
```

**Linux (Ubuntu/Debian):**
```bash
sudo apt-get update
sudo apt-get install sqlite3
sqlite3 --version
```

**Linux (Fedora/RHEL):**
```bash
sudo dnf install sqlite
sqlite3 --version
```

### วิธีที่ 2: ผ่าน Python (ทุก OS)

Python มี SQLite built-in อยู่แล้ว!

```python
import sqlite3

# สร้าง database ใหม่
conn = sqlite3.connect('my_first_db.db')
cursor = conn.cursor()

# รัน SQL
cursor.execute("CREATE TABLE IF NOT EXISTS test (id INTEGER, name TEXT)")
cursor.execute("INSERT INTO test VALUES (1, 'Hello SQL!')")
conn.commit()

# Query
cursor.execute("SELECT * FROM test")
print(cursor.fetchall())
# [(1, 'Hello SQL!')]

conn.close()
```

### วิธีที่ 3: Online Playground

ไม่ต้องติดตั้งอะไรเลย! ใช้เว็บนี้:

- **DB Fiddle**: https://www.db-fiddle.com
- **SQLite Online**: https://sqliteonline.com
- **SQL Playground**: https://sql-playground.wizardsofweb.dev

### การใช้ SQLite Command Line

```bash
# สร้าง/เปิด database file
sqlite3 company.db

# คำสั่งพิเศษใน SQLite (ขึ้นต้นด้วย .)
.help          -- แสดง help
.databases     -- แสดง databases ที่เปิดอยู่
.tables        -- แสดง tables ทั้งหมด
.schema        -- แสดง CREATE statements
.mode column   -- แสดงผลแบบ column
.headers on    -- แสดง column headers
.width 10 20   -- กำหนดความกว้าง
.quit          -- ออกจาก SQLite
```

---

## 1.9 ฐานข้อมูลตัวอย่างที่ใช้ตลอดหลักสูตร

เราจะใช้ฐานข้อมูลนี้ตลอดทั้งหลักสูตร ลองสร้างมันตอนนี้เลย!

```sql
-- ========================================
-- สร้าง Tables พื้นฐาน
-- ========================================

-- 1. ตาราง departments (แผนก)
CREATE TABLE departments (
    department_id   INTEGER PRIMARY KEY,
    department_name VARCHAR(50) NOT NULL,
    location        VARCHAR(100),
    budget          DECIMAL(15, 2),
    manager_id      INTEGER,
    created_at      DATE DEFAULT CURRENT_DATE
);

-- 2. ตาราง employees (พนักงาน)
CREATE TABLE employees (
    employee_id    INTEGER PRIMARY KEY,
    first_name     VARCHAR(50) NOT NULL,
    last_name      VARCHAR(50) NOT NULL,
    email          VARCHAR(100) UNIQUE,
    phone          VARCHAR(20),
    hire_date      DATE NOT NULL,
    job_title      VARCHAR(100),
    salary         DECIMAL(10, 2),
    department_id  INTEGER,
    manager_id     INTEGER,
    is_active      BOOLEAN DEFAULT TRUE,
    FOREIGN KEY (department_id) REFERENCES departments(department_id)
);

-- 3. ตาราง customers (ลูกค้า)
CREATE TABLE customers (
    customer_id   INTEGER PRIMARY KEY,
    company_name  VARCHAR(100),
    first_name    VARCHAR(50),
    last_name     VARCHAR(50),
    email         VARCHAR(100),
    phone         VARCHAR(20),
    address       VARCHAR(200),
    city          VARCHAR(50),
    country       VARCHAR(50) DEFAULT 'Thailand',
    created_date  DATE DEFAULT CURRENT_DATE,
    is_active     BOOLEAN DEFAULT TRUE
);

-- 4. ตาราง products (สินค้า)
CREATE TABLE products (
    product_id    INTEGER PRIMARY KEY,
    product_name  VARCHAR(100) NOT NULL,
    category      VARCHAR(50),
    unit_price    DECIMAL(10, 2) NOT NULL,
    units_in_stock INTEGER DEFAULT 0,
    discontinued  BOOLEAN DEFAULT FALSE,
    description   TEXT
);

-- 5. ตาราง orders (คำสั่งซื้อ)
CREATE TABLE orders (
    order_id      INTEGER PRIMARY KEY,
    customer_id   INTEGER NOT NULL,
    employee_id   INTEGER,
    order_date    DATE NOT NULL DEFAULT CURRENT_DATE,
    required_date DATE,
    shipped_date  DATE,
    status        VARCHAR(20) DEFAULT 'Pending',
    total_amount  DECIMAL(15, 2),
    notes         TEXT,
    FOREIGN KEY (customer_id) REFERENCES customers(customer_id),
    FOREIGN KEY (employee_id) REFERENCES employees(employee_id)
);

-- 6. ตาราง order_items (รายการสินค้าในคำสั่งซื้อ)
CREATE TABLE order_items (
    item_id      INTEGER PRIMARY KEY,
    order_id     INTEGER NOT NULL,
    product_id   INTEGER NOT NULL,
    quantity     INTEGER NOT NULL DEFAULT 1,
    unit_price   DECIMAL(10, 2) NOT NULL,
    discount     DECIMAL(5, 4) DEFAULT 0,
    FOREIGN KEY (order_id) REFERENCES orders(order_id),
    FOREIGN KEY (product_id) REFERENCES products(product_id)
);
```

### ใส่ข้อมูลตัวอย่าง

```sql
-- ========================================
-- ใส่ข้อมูล departments
-- ========================================
INSERT INTO departments (department_id, department_name, location, budget) VALUES
(1, 'Information Technology', 'Building A, Floor 3', 5000000.00),
(2, 'Human Resources',        'Building B, Floor 1', 2000000.00),
(3, 'Finance',                'Building A, Floor 2', 3000000.00),
(4, 'Marketing',              'Building C, Floor 2', 4000000.00),
(5, 'Sales',                  'Building C, Floor 1', 8000000.00),
(6, 'Operations',             'Building D',          6000000.00),
(7, 'Research & Development', 'Building E',          7000000.00);

-- ========================================
-- ใส่ข้อมูล employees
-- ========================================
INSERT INTO employees (employee_id, first_name, last_name, email, phone, hire_date, job_title, salary, department_id, manager_id) VALUES
(1,  'สมชาย',   'นักเขียน',  'somchai.n@company.com',   '081-111-1111', '2018-03-15', 'Senior Developer',    75000.00, 1, NULL),
(2,  'สมหญิง',  'ดีงาม',     'somying.d@company.com',   '081-222-2222', '2019-07-01', 'HR Manager',          65000.00, 2, NULL),
(3,  'วิภา',    'รักงาน',    'wipa.r@company.com',      '081-333-3333', '2017-01-10', 'CFO',                 120000.00, 3, NULL),
(4,  'ประยูร',  'มีสุข',     'prayoon.m@company.com',   '081-444-4444', '2020-05-20', 'Marketing Manager',   70000.00, 4, NULL),
(5,  'กิตติ',   'เก่งมาก',   'kitti.k@company.com',     '081-555-5555', '2016-09-30', 'Sales Director',      95000.00, 5, NULL),
(6,  'มาลี',    'สวยงาม',    'malee.s@company.com',     '081-666-6666', '2021-02-14', 'Junior Developer',    45000.00, 1, 1),
(7,  'อนันต์',  'ใจดี',      'anan.j@company.com',      '081-777-7777', '2020-11-01', 'Developer',           58000.00, 1, 1),
(8,  'รัตนา',   'ขยันมาก',   'rattana.k@company.com',   '081-888-8888', '2019-03-22', 'HR Specialist',       42000.00, 2, 2),
(9,  'ชาญชัย', 'ฉลาดเฉลียว', 'chanchai.c@company.com',  '081-999-9999', '2018-08-15', 'Senior Accountant',   55000.00, 3, 3),
(10, 'นงนุช',   'น่ารักมาก', 'nongnuch.n@company.com',  '082-111-1111', '2022-01-03', 'Content Creator',     38000.00, 4, 4),
(11, 'ธนพล',   'รวยแน่',    'thanaphol.r@company.com', '082-222-2222', '2021-06-15', 'Sales Executive',     48000.00, 5, 5),
(12, 'พิมพ์ใจ', 'หวานใจ',    'pimjai.h@company.com',    '082-333-3333', '2020-09-01', 'Operations Manager',  68000.00, 6, NULL),
(13, 'สุรชัย',  'เด่นมาก',   'surachai.d@company.com',  '082-444-4444', '2019-12-20', 'R&D Lead',            85000.00, 7, NULL),
(14, 'จินตนา',  'คิดเก่ง',   'jintana.k@company.com',   '082-555-5555', '2023-03-01', 'Data Analyst',        52000.00, 1, 1),
(15, 'ไพโรจน์', 'กว้างขวาง', 'pairoj.k@company.com',    '082-666-6666', '2017-07-17', 'Senior Sales',        72000.00, 5, 5);

-- ========================================
-- ใส่ข้อมูล customers
-- ========================================
INSERT INTO customers (customer_id, company_name, first_name, last_name, email, phone, city) VALUES
(1,  'บริษัท เทค สตาร์ จำกัด',       'นภดล',  'วงศ์ใหญ่',  'noppadol@techstar.co.th',   '02-111-1111', 'Bangkok'),
(2,  'ร้านสินค้า ABC',               'อรณิช', 'สว่างใจ',   'oranit@abc.co.th',          '02-222-2222', 'Chiang Mai'),
(3,  'บริษัท Global Trade จำกัด',    'วิรัตน์','พานิชย์',   'wirat@globaltrade.th',      '02-333-3333', 'Bangkok'),
(4,  NULL,                           'สุภา',  'แก้วใส',    'supa.k@gmail.com',          '08-444-4444', 'Phuket'),
(5,  'ห้างหุ้นส่วน ไทยดี',           'ปิยะ',  'รักไทย',    'piya@thaidee.co.th',        '02-555-5555', 'Khon Kaen'),
(6,  'บริษัท Smart Solution จำกัด',  'ธีร์',  'ฉลาดยิ่ง',  'thee@smartsol.co.th',       '02-666-6666', 'Bangkok'),
(7,  NULL,                           'กนกวรรณ','มีมาก',    'kanokwan@gmail.com',         '08-777-7777', 'Nakhon Ratchasima'),
(8,  'บริษัท ใหม่ดี จำกัด',          'ชัยณรงค์','ก้าวหน้า', 'chainaong@maidee.co.th',   '02-888-8888', 'Bangkok'),
(9,  'ร้านค้าออนไลน์ Happy Shop',    'พรรณิภา','สุขใจ',    'pannipa@happyshop.th',      '08-999-9999', 'Chiang Rai'),
(10, 'บริษัท Mega Corp จำกัด',       'อุดม',  'เจริญทรัพย์','udom@megacorp.co.th',      '02-000-0000', 'Bangkok');

-- ========================================
-- ใส่ข้อมูล products
-- ========================================
INSERT INTO products (product_id, product_name, category, unit_price, units_in_stock) VALUES
(1,  'Laptop Pro 15"',        'Electronics',   45000.00,  25),
(2,  'Wireless Mouse',        'Electronics',    850.00,  150),
(3,  'USB-C Hub',             'Electronics',   2500.00,   80),
(4,  'Office Chair Premium',  'Furniture',      8900.00,  30),
(5,  'Standing Desk',         'Furniture',     15000.00,  15),
(6,  'SQL Programming Book',  'Books',           450.00, 200),
(7,  'Python for Data Science','Books',          550.00, 180),
(8,  'Noise Cancelling Headphone', 'Electronics', 7500.00, 45),
(9,  'Monitor 27" 4K',        'Electronics',   18000.00,  20),
(10, 'Keyboard Mechanical',   'Electronics',    3200.00,  60),
(11, 'Webcam HD',             'Electronics',    2800.00,  40),
(12, 'Desk Lamp LED',         'Furniture',       1200.00, 100),
(13, 'Whiteboard A4',         'Stationery',       150.00, 500),
(14, 'Notebook Premium',      'Stationery',       250.00, 300),
(15, 'Pen Set',               'Stationery',       120.00, 400);

-- ========================================
-- ใส่ข้อมูล orders
-- ========================================
INSERT INTO orders (order_id, customer_id, employee_id, order_date, required_date, shipped_date, status, total_amount) VALUES
(1,  1, 11, '2024-01-05', '2024-01-15', '2024-01-10', 'Delivered', 48700.00),
(2,  2,  5, '2024-01-08', '2024-01-20', '2024-01-15', 'Delivered', 22500.00),
(3,  3, 15, '2024-01-12', '2024-01-25', NULL,          'Processing', 9200.00),
(4,  4, 11, '2024-01-15', '2024-01-30', '2024-01-22', 'Delivered', 850.00),
(5,  5,  5, '2024-02-01', '2024-02-15', '2024-02-08', 'Delivered', 46350.00),
(6,  1, 15, '2024-02-10', '2024-02-25', NULL,          'Pending',  18000.00),
(7,  6, 11, '2024-02-14', '2024-02-28', '2024-02-20', 'Delivered', 12500.00),
(8,  7,  5, '2024-03-01', '2024-03-15', NULL,          'Cancelled', 3200.00),
(9,  8, 15, '2024-03-05', '2024-03-20', '2024-03-12', 'Delivered', 27800.00),
(10, 9, 11, '2024-03-10', '2024-03-25', NULL,          'Processing', 5500.00);

-- ========================================
-- ใส่ข้อมูล order_items
-- ========================================
INSERT INTO order_items (item_id, order_id, product_id, quantity, unit_price, discount) VALUES
(1,  1,  1, 1, 45000.00, 0.00),
(2,  1,  2, 2,   850.00, 0.00),
(3,  1,  3, 1,  2500.00, 0.00),
(4,  2,  9, 1, 18000.00, 0.00),
(5,  2, 10, 1,  3200.00, 0.00),
(6,  2,  8, 1,  7500.00, 0.05),
(7,  3,  4, 1,  8900.00, 0.00),
(8,  3,  5, 1, 15000.00, 0.10),
(9,  4,  2, 1,   850.00, 0.00),
(10, 5,  1, 1, 45000.00, 0.05),
(11, 5,  6, 3,   450.00, 0.00),
(12, 6,  9, 1, 18000.00, 0.00),
(13, 7,  8, 1,  7500.00, 0.00),
(14, 7,  2, 2,   850.00, 0.00),
(15, 7, 11, 1,  2800.00, 0.00),
(16, 8, 10, 1,  3200.00, 0.00),
(17, 9,  1, 1, 45000.00, 0.10),
(18, 9,  2, 3,   850.00, 0.00),
(19, 10, 8, 1,  7500.00, 0.00);
```

---

## 1.10 SQL Query แรกของคุณ

ถึงเวลาลองรัน query แรกกันแล้ว!

```sql
-- ============================================================
-- EXAMPLE 1: Hello World ของ SQL
-- ============================================================

-- รันบน SQLite:
SELECT 'Hello, SQL World!' AS message;

-- ผลลัพธ์:
-- message
-- -------------------
-- Hello, SQL World!

-- ============================================================
-- EXAMPLE 2: ดูข้อมูลทั้งหมดในตาราง
-- ============================================================

-- ดูพนักงานทั้งหมด
SELECT * FROM employees;

-- ดูแผนกทั้งหมด
SELECT * FROM departments;

-- ดูลูกค้าทั้งหมด
SELECT * FROM customers;

-- ============================================================
-- EXAMPLE 3: ดูข้อมูลบางคอลัมน์
-- ============================================================

-- ดูชื่อ-นามสกุล และเงินเดือนของพนักงาน
SELECT first_name, last_name, salary
FROM employees;

-- ผลลัพธ์:
-- first_name  | last_name    | salary
-- ------------|--------------|----------
-- สมชาย       | นักเขียน     | 75000.00
-- สมหญิง      | ดีงาม        | 65000.00
-- ...

-- ============================================================
-- EXAMPLE 4: กรองข้อมูลด้วย WHERE
-- ============================================================

-- หาพนักงานที่เงินเดือนมากกว่า 60,000
SELECT first_name, last_name, salary
FROM employees
WHERE salary > 60000;

-- ============================================================
-- EXAMPLE 5: จัดเรียงข้อมูล
-- ============================================================

-- เรียงพนักงานจากเงินเดือนมากไปน้อย
SELECT first_name, last_name, salary
FROM employees
ORDER BY salary DESC;

-- ============================================================
-- EXAMPLE 6: นับจำนวนข้อมูล
-- ============================================================

-- นับจำนวนพนักงานทั้งหมด
SELECT COUNT(*) AS total_employees
FROM employees;

-- ผลลัพธ์:
-- total_employees
-- ---------------
-- 15

-- ============================================================
-- EXAMPLE 7: คำนวณค่าเฉลี่ย
-- ============================================================

-- หาเงินเดือนเฉลี่ยของพนักงาน
SELECT AVG(salary) AS average_salary
FROM employees;

-- ============================================================
-- EXAMPLE 8: Query แบบ JOIN (ดึงข้อมูลจาก 2 ตาราง)
-- ============================================================

-- ดูชื่อพนักงานพร้อมชื่อแผนก
SELECT 
    e.first_name,
    e.last_name,
    d.department_name
FROM employees e
JOIN departments d ON e.department_id = d.department_id;

-- ============================================================
-- EXAMPLE 9: Aggregation - สรุปข้อมูลตามกลุ่ม
-- ============================================================

-- หาจำนวนพนักงานในแต่ละแผนก
SELECT 
    d.department_name,
    COUNT(e.employee_id) AS headcount,
    AVG(e.salary) AS avg_salary
FROM employees e
JOIN departments d ON e.department_id = d.department_id
GROUP BY d.department_name
ORDER BY headcount DESC;

-- ============================================================
-- EXAMPLE 10: Query ที่ซับซ้อนขึ้น
-- ============================================================

-- หาพนักงานที่มีเงินเดือนสูงกว่าค่าเฉลี่ยของแผนก IT
SELECT first_name, last_name, salary
FROM employees
WHERE department_id = 1
  AND salary > (
      SELECT AVG(salary)
      FROM employees
      WHERE department_id = 1
  );
```

---

## 1.11 ความแตกต่างระหว่าง SQL Dialects

เนื่องจากมีหลาย RDBMS query บางอย่างเขียนต่างกัน:

```sql
-- ============================================================
-- การดึงข้อมูล 5 แถวแรก
-- ============================================================

-- SQL Standard (SQL:2008+)
SELECT * FROM employees
FETCH FIRST 5 ROWS ONLY;

-- PostgreSQL
SELECT * FROM employees
LIMIT 5;

-- MySQL
SELECT * FROM employees
LIMIT 5;

-- SQLite
SELECT * FROM employees
LIMIT 5;

-- SQL Server
SELECT TOP 5 * FROM employees;

-- Oracle
SELECT * FROM employees
WHERE ROWNUM <= 5;
-- หรือใน Oracle 12c+:
SELECT * FROM employees
FETCH FIRST 5 ROWS ONLY;

-- ============================================================
-- การรับวันที่ปัจจุบัน
-- ============================================================

-- PostgreSQL
SELECT CURRENT_DATE, NOW(), CURRENT_TIMESTAMP;

-- MySQL
SELECT CURDATE(), NOW(), CURRENT_TIMESTAMP;

-- SQLite
SELECT DATE('now'), DATETIME('now');

-- SQL Server
SELECT GETDATE(), CURRENT_TIMESTAMP;

-- Oracle
SELECT SYSDATE, CURRENT_DATE FROM DUAL;
```

---

## 1.12 SQL Best Practices เบื้องต้น

เริ่มสร้างนิสัยที่ดีตั้งแต่วันแรก!

```sql
-- ============================================================
-- 1. ใส่ Comment อธิบาย Query
-- ============================================================

-- Single-line comment
-- นี่คือ single-line comment

/* Multi-line comment
   อธิบายยาวๆ ได้ที่นี่
   เหมาะสำหรับ complex queries */

-- ============================================================
-- 2. Indent และ Formatting ที่ดี
-- ============================================================

-- ❌ ไม่ดี - อ่านยาก
SELECT e.first_name,e.last_name,d.department_name FROM employees e JOIN departments d ON e.department_id=d.department_id WHERE e.salary>50000 ORDER BY e.salary DESC;

-- ✓ ดี - อ่านง่าย
SELECT 
    e.first_name,
    e.last_name,
    d.department_name
FROM employees e
JOIN departments d 
    ON e.department_id = d.department_id
WHERE e.salary > 50000
ORDER BY e.salary DESC;

-- ============================================================
-- 3. ระบุ Column Names แทน SELECT *
-- ============================================================

-- ❌ หลีกเลี่ยงสำหรับ production code
SELECT * FROM employees;

-- ✓ ระบุ columns ที่ต้องการ
SELECT employee_id, first_name, last_name, salary
FROM employees;

-- ============================================================
-- 4. ใช้ Table Aliases ที่มีความหมาย
-- ============================================================

-- ❌ ไม่ดี - alias ไม่มีความหมาย
SELECT a.first_name, b.department_name
FROM employees a JOIN departments b ON a.department_id = b.department_id;

-- ✓ ดี - alias มีความหมาย
SELECT emp.first_name, dept.department_name
FROM employees emp 
JOIN departments dept ON emp.department_id = dept.department_id;

-- ============================================================
-- 5. WHERE clause ควรมีเสมอใน UPDATE/DELETE
-- ============================================================

-- ❌ อันตรายมาก! จะแก้ไข/ลบทุกแถว!
UPDATE employees SET salary = 0;
DELETE FROM employees;

-- ✓ ปลอดภัย - ระบุ condition
UPDATE employees SET salary = 75000 WHERE employee_id = 1;
DELETE FROM employees WHERE employee_id = 999;
```

---

## 1.13 สรุปบทที่ 1

ในบทนี้เราได้เรียนรู้:

```
✅ ฐานข้อมูลคืออะไร และทำไมถึงสำคัญ
✅ ประวัติของ Database และ SQL ตั้งแต่ 1960s ถึงปัจจุบัน
✅ ความแตกต่างระหว่าง RDBMS และ NoSQL
✅ SQL คืออะไร - Declarative language
✅ 4 ประเภทคำสั่ง SQL: DDL, DML, DCL, TCL
✅ RDBMS ยอดนิยม: PostgreSQL, MySQL, SQLite, SQL Server, Oracle
✅ การติดตั้ง SQLite
✅ การสร้างและ populate ฐานข้อมูลตัวอย่าง
✅ การรัน SQL queries พื้นฐาน
✅ SQL Best Practices เบื้องต้น
```

---

## 📝 แบบฝึกหัดท้ายบท

ทำแบบฝึกหัดเหล่านี้ก่อนดูเฉลย!

### แบบฝึกหัดที่ 1-5: ทฤษฎี
**คำถาม 1:** RDBMS ย่อมาจากอะไร? และ SQL ย่อมาจากอะไร?

**คำถาม 2:** อะไรคือความแตกต่างหลักระหว่าง RDBMS และ NoSQL Database? ให้ยกตัวอย่าง use case ของแต่ละประเภท

**คำถาม 3:** อธิบาย DDL, DML, DCL, TCL และให้ตัวอย่างคำสั่งของแต่ละประเภท

**คำถาม 4:** ทำไม SQLite ถึงเหมาะสำหรับการเริ่มเรียน SQL?

**คำถาม 5:** Edgar F. Codd คือใคร และมีความสำคัญอย่างไรต่อโลกของ Database?

### แบบฝึกหัดที่ 6-10: SQL Queries
หลังจาก setup ฐานข้อมูลตัวอย่างแล้ว ลองรัน queries เหล่านี้:

**คำถาม 6:** เขียน query เพื่อดูข้อมูลทั้งหมดจากตาราง `products`

**คำถาม 7:** เขียน query เพื่อนับจำนวนสินค้าทั้งหมดในตาราง `products`

**คำถาม 8:** เขียน query เพื่อดูชื่อสินค้าและราคา เรียงจากราคาแพงที่สุดไปถูกที่สุด

**คำถาม 9:** เขียน query เพื่อหาสินค้าที่มีราคาต่ำกว่า 1,000 บาท

**คำถาม 10:** เขียน query เพื่อหาราคาสินค้าเฉลี่ย, ราคาสูงสุด, และราคาต่ำสุด ในตาราง `products`

---

## ✅ เฉลยแบบฝึกหัด

### เฉลยที่ 1:
```
RDBMS = Relational Database Management System
SQL = Structured Query Language
```

### เฉลยที่ 2:
```
RDBMS:
- ข้อมูลเป็น structured (ตาราง)
- ใช้ SQL
- ACID transactions
- เหมาะสำหรับ: ระบบธนาคาร, ระบบ ERP, ระบบจองตั๋ว

NoSQL:
- ข้อมูล flexible (document, key-value, graph, column)
- Query language หลากหลาย
- Eventual consistency (โดยทั่วไป)
- เหมาะสำหรับ: Social media feed, Real-time analytics, IoT data
```

### เฉลยที่ 3:
```
DDL (Data Definition Language): CREATE, ALTER, DROP, TRUNCATE
DML (Data Manipulation Language): SELECT, INSERT, UPDATE, DELETE
DCL (Data Control Language): GRANT, REVOKE
TCL (Transaction Control Language): COMMIT, ROLLBACK, SAVEPOINT
```

### เฉลยที่ 4:
```
SQLite เหมาะสำหรับเริ่มเรียนเพราะ:
1. ไม่ต้องติดตั้ง server
2. เก็บข้อมูลในไฟล์เดียว
3. Zero configuration
4. มีใน Python และ smartphone ทุกเครื่อง
5. รองรับ SQL syntax ครบ
```

### เฉลยที่ 5:
```
Edgar F. Codd (1923-2003) เป็นนักวิทยาศาสตร์ IBM ที่คิดค้น 
Relational Model ในปี 1970 ทำให้เกิดแนวคิดการเก็บข้อมูลเป็นตาราง
และความสัมพันธ์ระหว่างตาราง ได้รับ Turing Award ในปี 1981
เป็นรากฐานของ SQL และ RDBMS ทุกตัวในปัจจุบัน
```

### เฉลยที่ 6:
```sql
SELECT * FROM products;
```

### เฉลยที่ 7:
```sql
SELECT COUNT(*) AS total_products FROM products;
-- ผลลัพธ์: 15
```

### เฉลยที่ 8:
```sql
SELECT product_name, unit_price
FROM products
ORDER BY unit_price DESC;
```

### เฉลยที่ 9:
```sql
SELECT product_name, unit_price
FROM products
WHERE unit_price < 1000;

-- ผลลัพธ์:
-- Wireless Mouse    850.00
-- SQL Programming Book  450.00
-- Python for Data Science  550.00
-- Whiteboard A4    150.00
-- Notebook Premium  250.00
-- Pen Set           120.00
```

### เฉลยที่ 10:
```sql
SELECT 
    AVG(unit_price) AS average_price,
    MAX(unit_price) AS max_price,
    MIN(unit_price) AS min_price
FROM products;

-- ผลลัพธ์:
-- average_price  | max_price | min_price
-- ----------------|-----------|----------
-- 6953.33...      | 45000.00  | 120.00
```

---

## 🔗 แหล่งข้อมูลเพิ่มเติม

- [Official SQLite Documentation](https://www.sqlite.org/docs.html)
- [DB Fiddle - Online SQL Editor](https://www.db-fiddle.com)
- [Edgar F. Codd's Original 1970 Paper](https://www.seas.upenn.edu/~zives/03f/cis550/codd.pdf)
- [History of SQL - Wikipedia](https://en.wikipedia.org/wiki/SQL#History)

---

## ➡️ บทถัดไป

**[Part 002: Setting Up Development Environment](part-002.md)**

ในบทถัดไปเราจะเรียนรู้การติดตั้ง PostgreSQL, MySQL, และเครื่องมือ GUI ต่างๆ เพื่อเตรียมพร้อมสำหรับการพัฒนาอย่างมืออาชีพ

---

*Part 001 of 120 | หลักสูตร SQL ครบวงจร*
