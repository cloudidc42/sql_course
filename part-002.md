# Part 002: Setting Up Development Environment

> **หลักสูตร SQL ครบวงจร | Part 2 of 120**

---

## 🎯 สิ่งที่จะได้เรียนรู้ในบทนี้

- การติดตั้ง PostgreSQL บน Windows, macOS, Linux
- การติดตั้ง MySQL และ MariaDB
- การติดตั้ง SQLite
- การใช้ DBeaver - GUI ฟรีที่รองรับทุก database
- การใช้ TablePlus
- การใช้ pgAdmin สำหรับ PostgreSQL
- การใช้ VS Code กับ SQL extensions
- Online SQL Playgrounds
- การสร้าง database แรก
- Connection strings และ connection parameters

**เวลาที่ใช้เรียน**: ประมาณ 2-4 ชั่วโมง (ขึ้นกับ OS และเครื่อง)

---

## 2.1 ภาพรวมของ Development Environment

ก่อนเริ่มเรียน SQL อย่างจริงจัง เราต้องมีเครื่องมือที่ถูกต้อง

```
Development Environment สำหรับ SQL:
├── Database Server (Backend)
│   ├── PostgreSQL - แนะนำมากที่สุด
│   ├── MySQL/MariaDB
│   └── SQLite - ไม่ต้องมี server
│
├── Client Tools (Frontend)
│   ├── Command Line Interface (CLI)
│   ├── GUI Tools (DBeaver, TablePlus, pgAdmin)
│   └── Code Editor (VS Code + extensions)
│
└── Online Options (ไม่ต้องติดตั้ง)
    ├── DB Fiddle
    ├── SQLite Online
    └── Neon (PostgreSQL cloud)
```

### แนะนำสำหรับผู้เริ่มต้น

```
ตัวเลือกที่ง่ายที่สุด (เริ่มได้ใน 5 นาที):
└── SQLite + DB Browser for SQLite

ตัวเลือกที่ดีที่สุดระยะยาว:
└── PostgreSQL + DBeaver

ตัวเลือกสำหรับ Web Developer:
└── MySQL + TablePlus หรือ DBeaver
```

---

## 2.2 การติดตั้ง PostgreSQL

PostgreSQL เป็น RDBMS ที่แนะนำมากที่สุดเพราะ:
- ฟีเจอร์ครบที่สุด
- Standard SQL compliance สูง
- ฟรีและ Open Source
- ใช้ใน Production จริงได้

### Windows - ติดตั้งด้วย Installer

**วิธีที่ 1: EDB Installer (แนะนำ)**

1. ไปที่ https://www.postgresql.org/download/windows/
2. คลิก "Download the installer"
3. เลือกเวอร์ชันล่าสุด (เช่น 16.x) และ Windows x86-64
4. รันไฟล์ installer
5. ตั้งค่าตามนี้:
   - Installation Directory: `C:\Program Files\PostgreSQL\16`
   - Data Directory: `C:\Program Files\PostgreSQL\16\data`
   - Password for superuser: จำให้ดี! (เช่น `postgres123`)
   - Port: `5432` (default)
   - Locale: `Thai, Thailand`

```
ตรวจสอบหลังติดตั้ง (เปิด Command Prompt):
> psql --version
psql (PostgreSQL) 16.1

> psql -U postgres
Password for user postgres: [ใส่ password ที่ตั้งไว้]
postgres=#
```

**วิธีที่ 2: Chocolatey Package Manager**

```powershell
# ติดตั้ง Chocolatey ก่อน (ถ้ายังไม่มี)
Set-ExecutionPolicy Bypass -Scope Process -Force
[System.Net.ServicePointManager]::SecurityProtocol = [System.Net.ServicePointManager]::SecurityProtocol -bor 3072
iex ((New-Object System.Net.WebClient).DownloadString('https://community.chocolatey.org/install.ps1'))

# ติดตั้ง PostgreSQL
choco install postgresql --params '/Password:postgres123'
```

**วิธีที่ 3: Scoop**

```powershell
# ติดตั้ง Scoop ก่อน
irm get.scoop.sh | iex

# ติดตั้ง PostgreSQL
scoop install postgresql
```

### macOS - ติดตั้ง PostgreSQL

**วิธีที่ 1: Homebrew (แนะนำ)**

```bash
# ติดตั้ง Homebrew ก่อน (ถ้ายังไม่มี)
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

# ติดตั้ง PostgreSQL
brew install postgresql@16

# เพิ่ม PATH
echo 'export PATH="/opt/homebrew/opt/postgresql@16/bin:$PATH"' >> ~/.zshrc
source ~/.zshrc

# เริ่ม PostgreSQL service
brew services start postgresql@16

# ทดสอบ
psql --version
psql postgres
```

**วิธีที่ 2: Postgres.app (ง่ายที่สุดบน Mac)**

1. ดาวน์โหลดจาก https://postgresapp.com
2. Drag ไปวางใน Applications folder
3. เปิด app และคลิก "Initialize"
4. เพิ่ม PATH ตามที่ app บอก

```bash
# เพิ่มใน ~/.zshrc หรือ ~/.bash_profile
export PATH="/Applications/Postgres.app/Contents/Versions/latest/bin:$PATH"
```

### Linux - ติดตั้ง PostgreSQL

**Ubuntu / Debian:**

```bash
# อัปเดต package list
sudo apt update

# ติดตั้ง PostgreSQL
sudo apt install postgresql postgresql-contrib

# เริ่ม service
sudo systemctl start postgresql
sudo systemctl enable postgresql

# ตรวจสอบ status
sudo systemctl status postgresql

# เข้าใช้งาน
sudo -u postgres psql
```

**Fedora / RHEL / CentOS:**

```bash
# ติดตั้ง
sudo dnf install postgresql-server postgresql-contrib

# Initialize database
sudo postgresql-setup --initdb

# เริ่ม service
sudo systemctl start postgresql
sudo systemctl enable postgresql

# เข้าใช้งาน
sudo -u postgres psql
```

**ใช้ Docker (ทุก OS):**

```bash
# ติดตั้ง Docker ก่อน จาก docker.com

# รัน PostgreSQL container
docker run --name my-postgres \
    -e POSTGRES_PASSWORD=postgres123 \
    -e POSTGRES_USER=postgres \
    -e POSTGRES_DB=sql_course \
    -p 5432:5432 \
    -d postgres:16

# เชื่อมต่อ
docker exec -it my-postgres psql -U postgres

# ทดสอบ
SELECT version();
```

### การตั้งค่า PostgreSQL หลังติดตั้ง

```sql
-- เชื่อมต่อในฐานะ superuser
-- psql -U postgres

-- สร้าง user ใหม่ (ไม่แนะนำให้ใช้ postgres โดยตรง)
CREATE USER sql_student WITH PASSWORD 'password123';

-- สร้าง database
CREATE DATABASE sql_course;
CREATE DATABASE company_db;

-- ให้สิทธิ์
GRANT ALL PRIVILEGES ON DATABASE sql_course TO sql_student;
GRANT ALL PRIVILEGES ON DATABASE company_db TO sql_student;

-- ดู databases ทั้งหมด
\l

-- เปลี่ยนไปใช้ database
\c sql_course

-- ดู tables
\dt

-- ออก
\q
```

### PostgreSQL Connection String

```
Format:
postgresql://[user[:password]@][host][:port][/dbname]

ตัวอย่าง:
postgresql://postgres:postgres123@localhost:5432/sql_course
postgresql://sql_student:password123@localhost:5432/company_db

สำหรับ remote server:
postgresql://user:pass@db.example.com:5432/production_db
```

---

## 2.3 การติดตั้ง MySQL

### Windows

**วิธีที่ 1: MySQL Installer**

1. ดาวน์โหลดจาก https://dev.mysql.com/downloads/installer/
2. เลือก "MySQL Installer for Windows"
3. รัน installer และเลือก "Developer Default"
4. ตั้ง root password: จำให้ดี!
5. ทดสอบ:

```cmd
mysql -u root -p
Enter password: [ใส่ password]
mysql>
```

**วิธีที่ 2: XAMPP (มา PHP, Apache, MySQL)**

ดาวน์โหลดจาก https://www.apachefriends.org/index.html

### macOS

```bash
# ติดตั้งผ่าน Homebrew
brew install mysql

# เริ่ม service
brew services start mysql

# Secure installation
mysql_secure_installation

# เชื่อมต่อ
mysql -u root -p
```

### Linux

```bash
# Ubuntu/Debian
sudo apt install mysql-server mysql-client

# เริ่ม service
sudo systemctl start mysql
sudo systemctl enable mysql

# Secure setup
sudo mysql_secure_installation

# เชื่อมต่อ
sudo mysql -u root -p
```

### MySQL Connection String

```
Format:
mysql://[user[:password]@][host][:port][/database]

ตัวอย่าง:
mysql://root:password@localhost:3306/sql_course
mysql://user:pass@db.example.com:3306/production
```

### การสร้าง Database ใน MySQL

```sql
-- เชื่อมต่อ: mysql -u root -p

-- ดู databases ที่มีอยู่
SHOW DATABASES;

-- สร้าง database ใหม่
CREATE DATABASE sql_course CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;

-- สร้าง user ใหม่
CREATE USER 'sql_student'@'localhost' IDENTIFIED BY 'password123';

-- ให้สิทธิ์
GRANT ALL PRIVILEGES ON sql_course.* TO 'sql_student'@'localhost';
FLUSH PRIVILEGES;

-- ใช้ database
USE sql_course;

-- ดู tables
SHOW TABLES;
```

---

## 2.4 การติดตั้ง SQLite

SQLite ง่ายที่สุด ไม่ต้องมี server!

### Windows

```
1. ไปที่ https://www.sqlite.org/download.html
2. ดาวน์โหลด "sqlite-tools-win32-x86-*.zip"
3. แตกไฟล์ไปที่ C:\sqlite\
4. เพิ่ม C:\sqlite ใน System PATH:
   - คลิกขวา My Computer > Properties
   - Advanced System Settings > Environment Variables
   - System Variables > Path > Edit
   - เพิ่ม C:\sqlite
5. เปิด Command Prompt ใหม่:

> sqlite3 --version
3.43.2 2023-10-10...
```

### macOS

```bash
# มาพร้อม macOS แล้ว!
sqlite3 --version

# หรือติดตั้งเวอร์ชันใหม่กว่า
brew install sqlite
```

### Linux

```bash
# Ubuntu/Debian
sudo apt install sqlite3

# Fedora
sudo dnf install sqlite

# ตรวจสอบ
sqlite3 --version
```

### การใช้ SQLite Command Line

```bash
# สร้าง/เปิด database file
sqlite3 company.db

# คำสั่งภายใน SQLite
sqlite> .help             -- แสดง help ทั้งหมด
sqlite> .databases        -- แสดง databases ที่เปิดอยู่
sqlite> .tables           -- แสดง tables ทั้งหมด
sqlite> .schema           -- แสดง CREATE statements ทั้งหมด
sqlite> .schema employees -- แสดง schema ของตาราง employees
sqlite> .mode column      -- แสดงผลแบบ column-aligned
sqlite> .headers on       -- แสดง column headers
sqlite> .width 15 20 10   -- กำหนดความกว้าง column
sqlite> .output file.txt  -- redirect output ไปไฟล์
sqlite> .output stdout    -- กลับมาแสดงใน terminal
sqlite> .read script.sql  -- รัน SQL จากไฟล์
sqlite> .quit             -- ออก (หรือ .exit)

-- ตัวอย่างการใช้งาน
sqlite> .mode column
sqlite> .headers on
sqlite> CREATE TABLE test (id INTEGER, name TEXT);
sqlite> INSERT INTO test VALUES (1, 'Hello');
sqlite> SELECT * FROM test;
id  name
--  -----
1   Hello
```

### SQLite ใน Python (ไม่ต้องติดตั้งอะไร)

```python
import sqlite3

# สร้าง/เปิด database
conn = sqlite3.connect('sql_course.db')
cursor = conn.cursor()

# สร้างตาราง
cursor.execute('''
    CREATE TABLE IF NOT EXISTS employees (
        id INTEGER PRIMARY KEY,
        name TEXT NOT NULL,
        salary REAL
    )
''')

# Insert ข้อมูล
cursor.execute("INSERT INTO employees VALUES (1, 'สมชาย', 45000)")
cursor.execute("INSERT INTO employees VALUES (2, 'สมหญิง', 55000)")
conn.commit()

# Query
cursor.execute("SELECT * FROM employees")
rows = cursor.fetchall()
for row in rows:
    print(row)

# Output:
# (1, 'สมชาย', 45000)
# (2, 'สมหญิง', 55000)

conn.close()
```

---

## 2.5 DBeaver - GUI ที่แนะนำ

DBeaver เป็น GUI tool ฟรีที่รองรับ database เกือบทุกชนิด

```
DBeaver รองรับ:
├── PostgreSQL
├── MySQL / MariaDB
├── SQLite
├── SQL Server
├── Oracle
├── MongoDB
├── Redis
└── อีกกว่า 100 database!
```

### การติดตั้ง DBeaver

**ทุก OS:**
1. ดาวน์โหลดจาก https://dbeaver.io/download/
2. เลือก "Community Edition" (ฟรี)
3. ติดตั้งตามปกติ

**macOS (Homebrew):**
```bash
brew install --cask dbeaver-community
```

**Windows (Chocolatey):**
```powershell
choco install dbeaver
```

### การเชื่อมต่อ PostgreSQL ด้วย DBeaver

```
ขั้นตอน:
1. เปิด DBeaver
2. คลิก New Database Connection (Ctrl+Shift+N หรือ ไอคอน +)
3. เลือก "PostgreSQL"
4. กรอกข้อมูล:
   ├── Host: localhost
   ├── Port: 5432
   ├── Database: sql_course
   ├── Username: postgres
   └── Password: [password ที่ตั้งไว้]
5. คลิก "Test Connection"
6. ถ้าผ่านแล้วคลิก "Finish"

ถ้า test ไม่ผ่าน:
- ตรวจสอบว่า PostgreSQL service กำลังรันอยู่
- ตรวจสอบ password
- ตรวจสอบ firewall settings
```

### การเชื่อมต่อ SQLite ด้วย DBeaver

```
ขั้นตอน:
1. คลิก New Database Connection
2. เลือก "SQLite"
3. คลิก "Browse" และเลือก .db file
   (หรือพิมพ์ path ของไฟล์ใหม่)
4. คลิก "Test Connection"
5. คลิก "Finish"
```

### การใช้ DBeaver SQL Editor

```
เปิด SQL Editor:
- คลิกขวาที่ connection > SQL Editor
- หรือ กด Ctrl+` (backtick)

Keyboard Shortcuts สำคัญ:
├── Ctrl+Enter  - รัน query ที่ cursor อยู่
├── Ctrl+Shift+Enter - รัน query ทั้งหมด
├── Ctrl+/      - Toggle comment
├── Ctrl+Space  - Auto-complete
├── Ctrl+F      - Find
├── F5          - Refresh
└── Ctrl+D      - Duplicate line
```

---

## 2.6 TablePlus

TablePlus เป็น GUI tool ที่สวยงามและเร็วมาก มี Free tier

```
TablePlus:
├── Platform: macOS, Windows, Linux
├── ราคา: ฟรี (limited) / $79 (license)
├── Website: tableplus.com
└── รองรับ: PostgreSQL, MySQL, SQLite, SQL Server, Redis, MongoDB
```

### การติดตั้ง TablePlus

**macOS:**
```bash
brew install --cask tableplus
```

**Windows/Linux:**
ดาวน์โหลดจาก https://tableplus.com/download

### Connection ใน TablePlus

```
สร้าง Connection ใหม่:
1. เปิด TablePlus
2. กด Cmd+N (Mac) หรือ Ctrl+N (Windows)
3. เลือกประเภท database
4. กรอกข้อมูล connection
5. คลิก "Test" แล้ว "Connect"
```

---

## 2.7 pgAdmin - Official PostgreSQL GUI

pgAdmin มาพร้อมกับ PostgreSQL installer หรือติดตั้งแยก

```
pgAdmin 4:
├── Platform: Web-based (ทุก OS)
├── ราคา: ฟรี
└── Website: pgadmin.org
```

### การเข้าใช้ pgAdmin

```
ถ้าติดตั้งพร้อม PostgreSQL:
1. ค้นหา "pgAdmin 4" ใน Start Menu / Applications
2. เปิดผ่าน Browser
3. กรอก Master Password (ตั้งครั้งแรก)
4. เพิ่ม Server connection:
   - Right-click "Servers" > Register > Server
   - Name: My PostgreSQL
   - Host: localhost
   - Port: 5432
   - Username: postgres
   - Password: [password ที่ตั้งไว้]
5. คลิก Save
```

---

## 2.8 VS Code สำหรับ SQL

Visual Studio Code พร้อม extensions ที่ดี

### Extensions ที่แนะนำ

```
1. SQLTools
   ID: mtxr.sqltools
   - Connect กับ databases โดยตรงจาก VS Code
   - Auto-complete
   - Query execution

2. SQLTools PostgreSQL/MySQL/SQLite Drivers
   - SQLTools PostgreSQL: mtxr.sqltools-driver-pg
   - SQLTools MySQL: mtxr.sqltools-driver-mysql
   - SQLTools SQLite: mtxr.sqltools-driver-sqlite

3. SQL Formatter
   ID: adpyke.codesnap
   - Format SQL code อัตโนมัติ

4. Rainbow CSV
   ID: mechatroner.rainbow-csv
   - ช่วย highlight CSV files

5. SQL Server (mssql)
   ID: ms-mssql.mssql
   - Official extension สำหรับ SQL Server
```

### การติดตั้ง SQLTools

```bash
# ผ่าน VS Code Command Palette (Ctrl+P)
ext install mtxr.sqltools
ext install mtxr.sqltools-driver-pg
ext install mtxr.sqltools-driver-mysql
ext install mtxr.sqltools-driver-sqlite
```

### การตั้งค่า Connection ใน SQLTools

```json
// settings.json (เปิดด้วย Ctrl+Shift+P > "Open Settings JSON")
{
  "sqltools.connections": [
    {
      "name": "My PostgreSQL",
      "driver": "PostgreSQL",
      "server": "localhost",
      "port": 5432,
      "database": "sql_course",
      "username": "postgres",
      "password": "postgres123"
    },
    {
      "name": "My SQLite",
      "driver": "SQLite",
      "database": "/path/to/company.db"
    },
    {
      "name": "My MySQL",
      "driver": "MySQL",
      "server": "localhost",
      "port": 3306,
      "database": "sql_course",
      "username": "root",
      "password": "password123"
    }
  ]
}
```

### การใช้ SQLTools ใน VS Code

```
Keyboard Shortcuts:
Ctrl+Shift+P > SQLTools: New Connection    - สร้าง connection ใหม่
Ctrl+Shift+P > SQLTools: Run Query         - รัน query ที่ select ไว้
Ctrl+E, Ctrl+E                             - รัน query ปัจจุบัน
Ctrl+E, Ctrl+F                             - Format SQL

สร้างไฟล์ .sql แล้วรัน:
1. สร้างไฟล์ query.sql
2. เปิด SQL Tools panel (sidebar)
3. เลือก connection
4. เขียน query
5. กด Ctrl+E, Ctrl+E เพื่อรัน
```

---

## 2.9 Online SQL Playgrounds

ไม่ต้องติดตั้งอะไรเลย เรียนได้ทันที!

### DB Fiddle (แนะนำมากที่สุด)

```
URL: https://www.db-fiddle.com

รองรับ:
├── PostgreSQL (หลายเวอร์ชัน)
├── MySQL (หลายเวอร์ชัน)
└── SQLite

การใช้งาน:
1. เลือก Database Engine ด้านบน
2. พิมพ์ DDL (CREATE TABLE, INSERT) ในช่องซ้าย
3. พิมพ์ Query ในช่องขวา
4. คลิก "Run"
5. ดูผลลัพธ์ด้านล่าง
6. แชร์ URL เพื่อแชร์ query ได้

ตัวอย่าง URL พร้อม query:
https://www.db-fiddle.com/f/example123
```

### SQLite Online

```
URL: https://sqliteonline.com

ข้อดี:
- ง่ายมาก
- Import CSV ได้
- Export ได้
- ไม่ต้อง register
```

### SQL Playground ของ W3Schools

```
URL: https://www.w3schools.com/sql/trysql.asp?filename=trysql_select_all

ข้อดี:
- มีตัวอย่างข้อมูลให้แล้ว (Northwind database)
- เหมาะสำหรับทดลอง queries
- ไม่ต้อง setup อะไร
```

### Replit

```
URL: https://replit.com

- รองรับหลาย database
- มี IDE ครบ
- ทำงานร่วมกับคนอื่นได้
- ฟรีสำหรับ basic use
```

### Neon - PostgreSQL Cloud (ฟรี)

```
URL: https://neon.tech

- PostgreSQL แท้ๆ ใน cloud
- ฟรี forever tier
- มี connection string ให้ใช้กับ tools
- เหมาะสำหรับเรียน + project จริง

Setup:
1. Register ที่ neon.tech
2. สร้าง project
3. Copy connection string
4. ใช้กับ DBeaver หรือ pgAdmin
```

---

## 2.10 การสร้าง Database แรกและทดสอบ Connection

มาลองสร้าง database และทดสอบ connection กัน!

### ใน PostgreSQL (psql command line)

```sql
-- เชื่อมต่อ: psql -U postgres

-- 1. สร้าง database
CREATE DATABASE sql_course
    ENCODING = 'UTF8'
    LC_COLLATE = 'en_US.UTF-8'
    LC_CTYPE = 'en_US.UTF-8';

-- 2. เชื่อมต่อไปยัง database
\c sql_course

-- 3. สร้าง schema (optional)
CREATE SCHEMA IF NOT EXISTS course;

-- 4. ตรวจสอบ
SELECT current_database();
-- sql_course

SELECT version();
-- PostgreSQL 16.1 on x86_64-pc-linux-gnu...

-- 5. สร้างตารางทดสอบ
CREATE TABLE test_connection (
    id SERIAL PRIMARY KEY,
    message TEXT,
    created_at TIMESTAMP DEFAULT NOW()
);

-- 6. Insert ข้อมูลทดสอบ
INSERT INTO test_connection (message) VALUES ('Hello, PostgreSQL!');

-- 7. Query
SELECT * FROM test_connection;
-- id | message            | created_at
-- ---|--------------------|-----------
--  1 | Hello, PostgreSQL! | 2024-01-15 10:30:00

-- 8. ลบตารางทดสอบ
DROP TABLE test_connection;

-- 9. ออก
\q
```

### ใน MySQL (mysql command line)

```sql
-- เชื่อมต่อ: mysql -u root -p

-- 1. สร้าง database
CREATE DATABASE sql_course
    CHARACTER SET utf8mb4
    COLLATE utf8mb4_unicode_ci;

-- 2. ใช้ database
USE sql_course;

-- 3. ตรวจสอบ
SELECT DATABASE();
-- sql_course

SELECT VERSION();
-- 8.0.35

-- 4. สร้างตารางทดสอบ
CREATE TABLE test_connection (
    id INT AUTO_INCREMENT PRIMARY KEY,
    message VARCHAR(255),
    created_at DATETIME DEFAULT NOW()
);

-- 5. Insert
INSERT INTO test_connection (message) VALUES ('Hello, MySQL!');

-- 6. Query
SELECT * FROM test_connection;

-- 7. ลบ
DROP TABLE test_connection;

-- 8. ออก
EXIT;
```

### ใน SQLite

```bash
# เปิด/สร้าง database
sqlite3 sql_course.db

# ตรวจสอบ version
sqlite> SELECT sqlite_version();
-- 3.43.2

# เปิด mode ที่อ่านง่าย
sqlite> .mode column
sqlite> .headers on

# สร้างตารางทดสอบ
sqlite> CREATE TABLE test_connection (
   ...>     id INTEGER PRIMARY KEY AUTOINCREMENT,
   ...>     message TEXT,
   ...>     created_at DATETIME DEFAULT CURRENT_TIMESTAMP
   ...> );

# Insert
sqlite> INSERT INTO test_connection (message) VALUES ('Hello, SQLite!');

# Query
sqlite> SELECT * FROM test_connection;
-- id  message         created_at
-- --  --------------  -------------------
-- 1   Hello, SQLite!  2024-01-15 10:30:00

# ลบ
sqlite> DROP TABLE test_connection;

# ออก
sqlite> .quit
```

---

## 2.11 การสร้างฐานข้อมูลหลักสำหรับหลักสูตร

มาสร้างฐานข้อมูลที่จะใช้ตลอดหลักสูตรนี้กัน!

### สร้างไฟล์ SQL script

บันทึกเป็น `setup_database.sql`:

```sql
-- ============================================================
-- SQL Course Database Setup Script
-- รันไฟล์นี้เพื่อสร้างและเติมข้อมูลฐานข้อมูลตัวอย่าง
-- ============================================================

-- ลบตารางเก่าถ้ามี (รันได้หลายครั้ง)
DROP TABLE IF EXISTS order_items;
DROP TABLE IF EXISTS orders;
DROP TABLE IF EXISTS products;
DROP TABLE IF EXISTS customers;
DROP TABLE IF EXISTS employees;
DROP TABLE IF EXISTS departments;

-- ============================================================
-- สร้าง Tables
-- ============================================================

CREATE TABLE departments (
    department_id   INTEGER PRIMARY KEY,
    department_name VARCHAR(50) NOT NULL,
    location        VARCHAR(100),
    budget          DECIMAL(15, 2),
    manager_id      INTEGER,
    created_at      DATE DEFAULT CURRENT_DATE
);

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
    is_active      BOOLEAN DEFAULT TRUE
);

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

CREATE TABLE products (
    product_id    INTEGER PRIMARY KEY,
    product_name  VARCHAR(100) NOT NULL,
    category      VARCHAR(50),
    unit_price    DECIMAL(10, 2) NOT NULL,
    units_in_stock INTEGER DEFAULT 0,
    discontinued  BOOLEAN DEFAULT FALSE,
    description   TEXT
);

CREATE TABLE orders (
    order_id      INTEGER PRIMARY KEY,
    customer_id   INTEGER NOT NULL,
    employee_id   INTEGER,
    order_date    DATE NOT NULL DEFAULT CURRENT_DATE,
    required_date DATE,
    shipped_date  DATE,
    status        VARCHAR(20) DEFAULT 'Pending',
    total_amount  DECIMAL(15, 2),
    notes         TEXT
);

CREATE TABLE order_items (
    item_id      INTEGER PRIMARY KEY,
    order_id     INTEGER NOT NULL,
    product_id   INTEGER NOT NULL,
    quantity     INTEGER NOT NULL DEFAULT 1,
    unit_price   DECIMAL(10, 2) NOT NULL,
    discount     DECIMAL(5, 4) DEFAULT 0
);

-- ============================================================
-- ใส่ข้อมูล
-- ============================================================

-- departments
INSERT INTO departments VALUES
(1, 'Information Technology', 'Building A, Floor 3', 5000000.00, 1, '2015-01-01'),
(2, 'Human Resources',        'Building B, Floor 1', 2000000.00, 2, '2015-01-01'),
(3, 'Finance',                'Building A, Floor 2', 3000000.00, 3, '2015-01-01'),
(4, 'Marketing',              'Building C, Floor 2', 4000000.00, 4, '2015-01-01'),
(5, 'Sales',                  'Building C, Floor 1', 8000000.00, 5, '2015-01-01'),
(6, 'Operations',             'Building D',          6000000.00, 12, '2015-01-01'),
(7, 'Research & Development', 'Building E',          7000000.00, 13, '2015-01-01');

-- employees
INSERT INTO employees VALUES
(1,  'สมชาย',   'นักเขียน',  'somchai.n@company.com',   '081-111-1111', '2018-03-15', 'Senior Developer',    75000.00, 1, NULL,  TRUE),
(2,  'สมหญิง',  'ดีงาม',     'somying.d@company.com',   '081-222-2222', '2019-07-01', 'HR Manager',          65000.00, 2, NULL,  TRUE),
(3,  'วิภา',    'รักงาน',    'wipa.r@company.com',      '081-333-3333', '2017-01-10', 'CFO',                 120000.00, 3, NULL,  TRUE),
(4,  'ประยูร',  'มีสุข',     'prayoon.m@company.com',   '081-444-4444', '2020-05-20', 'Marketing Manager',   70000.00, 4, NULL,  TRUE),
(5,  'กิตติ',   'เก่งมาก',   'kitti.k@company.com',     '081-555-5555', '2016-09-30', 'Sales Director',      95000.00, 5, NULL,  TRUE),
(6,  'มาลี',    'สวยงาม',    'malee.s@company.com',     '081-666-6666', '2021-02-14', 'Junior Developer',    45000.00, 1, 1,     TRUE),
(7,  'อนันต์',  'ใจดี',      'anan.j@company.com',      '081-777-7777', '2020-11-01', 'Developer',           58000.00, 1, 1,     TRUE),
(8,  'รัตนา',   'ขยันมาก',   'rattana.k@company.com',   '081-888-8888', '2019-03-22', 'HR Specialist',       42000.00, 2, 2,     TRUE),
(9,  'ชาญชัย', 'ฉลาดเฉลียว', 'chanchai.c@company.com',  '081-999-9999', '2018-08-15', 'Senior Accountant',   55000.00, 3, 3,     TRUE),
(10, 'นงนุช',   'น่ารักมาก', 'nongnuch.n@company.com',  '082-111-1111', '2022-01-03', 'Content Creator',     38000.00, 4, 4,     TRUE),
(11, 'ธนพล',   'รวยแน่',    'thanaphol.r@company.com', '082-222-2222', '2021-06-15', 'Sales Executive',     48000.00, 5, 5,     TRUE),
(12, 'พิมพ์ใจ', 'หวานใจ',    'pimjai.h@company.com',    '082-333-3333', '2020-09-01', 'Operations Manager',  68000.00, 6, NULL,  TRUE),
(13, 'สุรชัย',  'เด่นมาก',   'surachai.d@company.com',  '082-444-4444', '2019-12-20', 'R&D Lead',            85000.00, 7, NULL,  TRUE),
(14, 'จินตนา',  'คิดเก่ง',   'jintana.k@company.com',   '082-555-5555', '2023-03-01', 'Data Analyst',        52000.00, 1, 1,     TRUE),
(15, 'ไพโรจน์', 'กว้างขวาง', 'pairoj.k@company.com',    '082-666-6666', '2017-07-17', 'Senior Sales',        72000.00, 5, 5,     TRUE);

-- customers
INSERT INTO customers VALUES
(1,  'บริษัท เทค สตาร์ จำกัด',       'นภดล',  'วงศ์ใหญ่',  'noppadol@techstar.co.th',   '02-111-1111', '123 Sukhumvit Rd', 'Bangkok',             'Thailand', '2023-01-15', TRUE),
(2,  'ร้านสินค้า ABC',               'อรณิช', 'สว่างใจ',   'oranit@abc.co.th',          '02-222-2222', '456 Nimman Rd',    'Chiang Mai',          'Thailand', '2023-02-20', TRUE),
(3,  'บริษัท Global Trade จำกัด',    'วิรัตน์','พานิชย์',   'wirat@globaltrade.th',      '02-333-3333', '789 Silom Rd',     'Bangkok',             'Thailand', '2023-03-10', TRUE),
(4,  NULL,                           'สุภา',  'แก้วใส',    'supa.k@gmail.com',          '08-444-4444', '321 Beach Rd',     'Phuket',              'Thailand', '2023-04-05', TRUE),
(5,  'ห้างหุ้นส่วน ไทยดี',           'ปิยะ',  'รักไทย',    'piya@thaidee.co.th',        '02-555-5555', '555 Mittraphap Rd','Khon Kaen',           'Thailand', '2023-05-12', TRUE),
(6,  'บริษัท Smart Solution จำกัด',  'ธีร์',  'ฉลาดยิ่ง',  'thee@smartsol.co.th',       '02-666-6666', '100 Ratchada Rd',  'Bangkok',             'Thailand', '2023-06-08', TRUE),
(7,  NULL,                           'กนกวรรณ','มีมาก',    'kanokwan@gmail.com',         '08-777-7777', '200 Mitraphap Rd', 'Nakhon Ratchasima',   'Thailand', '2023-07-22', TRUE),
(8,  'บริษัท ใหม่ดี จำกัด',          'ชัยณรงค์','ก้าวหน้า', 'chainaong@maidee.co.th',   '02-888-8888', '300 Vibhavadi Rd', 'Bangkok',             'Thailand', '2023-08-30', TRUE),
(9,  'ร้านค้าออนไลน์ Happy Shop',    'พรรณิภา','สุขใจ',    'pannipa@happyshop.th',      '08-999-9999', '400 Phahon Rd',    'Chiang Rai',          'Thailand', '2023-09-15', TRUE),
(10, 'บริษัท Mega Corp จำกัด',       'อุดม',  'เจริญทรัพย์','udom@megacorp.co.th',      '02-000-0000', '500 Asoke Rd',     'Bangkok',             'Thailand', '2023-10-01', TRUE);

-- products
INSERT INTO products VALUES
(1,  'Laptop Pro 15"',          'Electronics', 45000.00, 25, FALSE, 'High-performance laptop for professionals'),
(2,  'Wireless Mouse',          'Electronics',   850.00, 150, FALSE, 'Ergonomic wireless mouse'),
(3,  'USB-C Hub',               'Electronics',  2500.00,  80, FALSE, '7-port USB-C hub'),
(4,  'Office Chair Premium',    'Furniture',    8900.00,  30, FALSE, 'Ergonomic office chair with lumbar support'),
(5,  'Standing Desk',           'Furniture',   15000.00,  15, FALSE, 'Adjustable height standing desk'),
(6,  'SQL Programming Book',    'Books',          450.00, 200, FALSE, 'Comprehensive SQL guide'),
(7,  'Python for Data Science', 'Books',          550.00, 180, FALSE, 'Python programming for data science'),
(8,  'Noise Cancelling Headphone','Electronics', 7500.00, 45, FALSE, 'Premium noise-cancelling headphones'),
(9,  'Monitor 27" 4K',          'Electronics', 18000.00,  20, FALSE, '4K IPS display 27 inch'),
(10, 'Keyboard Mechanical',     'Electronics',  3200.00,  60, FALSE, 'Mechanical keyboard with RGB'),
(11, 'Webcam HD',               'Electronics',  2800.00,  40, FALSE, '1080p webcam with microphone'),
(12, 'Desk Lamp LED',           'Furniture',    1200.00, 100, FALSE, 'Adjustable LED desk lamp'),
(13, 'Whiteboard A4',           'Stationery',    150.00, 500, FALSE, 'Reusable A4 whiteboard'),
(14, 'Notebook Premium',        'Stationery',    250.00, 300, FALSE, 'Premium hardcover notebook'),
(15, 'Pen Set',                 'Stationery',    120.00, 400, FALSE, 'Set of 10 ballpoint pens');

-- orders
INSERT INTO orders VALUES
(1,  1, 11, '2024-01-05', '2024-01-15', '2024-01-10', 'Delivered',  48700.00, NULL),
(2,  2,  5, '2024-01-08', '2024-01-20', '2024-01-15', 'Delivered',  22500.00, NULL),
(3,  3, 15, '2024-01-12', '2024-01-25', NULL,          'Processing',  9200.00, 'Rush order'),
(4,  4, 11, '2024-01-15', '2024-01-30', '2024-01-22', 'Delivered',    850.00, NULL),
(5,  5,  5, '2024-02-01', '2024-02-15', '2024-02-08', 'Delivered',  46350.00, NULL),
(6,  1, 15, '2024-02-10', '2024-02-25', NULL,          'Pending',   18000.00, 'Waiting for approval'),
(7,  6, 11, '2024-02-14', '2024-02-28', '2024-02-20', 'Delivered',  12500.00, NULL),
(8,  7,  5, '2024-03-01', '2024-03-15', NULL,          'Cancelled',   3200.00, 'Customer cancelled'),
(9,  8, 15, '2024-03-05', '2024-03-20', '2024-03-12', 'Delivered',  27800.00, NULL),
(10, 9, 11, '2024-03-10', '2024-03-25', NULL,          'Processing',  5500.00, NULL);

-- order_items
INSERT INTO order_items VALUES
(1,  1,  1, 1, 45000.00, 0.0000),
(2,  1,  2, 2,   850.00, 0.0000),
(3,  1,  3, 1,  2500.00, 0.0000),
(4,  2,  9, 1, 18000.00, 0.0000),
(5,  2, 10, 1,  3200.00, 0.0000),
(6,  2,  8, 1,  7500.00, 0.0500),
(7,  3,  4, 1,  8900.00, 0.0000),
(8,  3,  5, 1, 15000.00, 0.1000),
(9,  4,  2, 1,   850.00, 0.0000),
(10, 5,  1, 1, 45000.00, 0.0500),
(11, 5,  6, 3,   450.00, 0.0000),
(12, 6,  9, 1, 18000.00, 0.0000),
(13, 7,  8, 1,  7500.00, 0.0000),
(14, 7,  2, 2,   850.00, 0.0000),
(15, 7, 11, 1,  2800.00, 0.0000),
(16, 8, 10, 1,  3200.00, 0.0000),
(17, 9,  1, 1, 45000.00, 0.1000),
(18, 9,  2, 3,   850.00, 0.0000),
(19,10,  8, 1,  7500.00, 0.0000);

-- ยืนยันการ setup
SELECT 'Setup complete!' AS status;
SELECT 'Tables created: departments, employees, customers, products, orders, order_items' AS info;
```

### วิธีรัน Script

**PostgreSQL:**
```bash
psql -U postgres -d sql_course -f setup_database.sql
```

**MySQL:**
```bash
mysql -u root -p sql_course < setup_database.sql
```

**SQLite:**
```bash
sqlite3 company.db < setup_database.sql
```

**Python (SQLite):**
```python
import sqlite3

with open('setup_database.sql', 'r', encoding='utf-8') as f:
    sql_script = f.read()

conn = sqlite3.connect('company.db')
conn.executescript(sql_script)
conn.commit()
conn.close()

print("Database setup complete!")
```

---

## 2.12 ทดสอบว่า Environment ทำงานถูกต้อง

หลังจาก setup เสร็จแล้ว ลองรัน queries เหล่านี้เพื่อยืนยัน:

```sql
-- Test 1: นับจำนวน records ในแต่ละตาราง
SELECT 'departments' AS table_name, COUNT(*) AS record_count FROM departments
UNION ALL
SELECT 'employees',  COUNT(*) FROM employees
UNION ALL
SELECT 'customers',  COUNT(*) FROM customers
UNION ALL
SELECT 'products',   COUNT(*) FROM products
UNION ALL
SELECT 'orders',     COUNT(*) FROM orders
UNION ALL
SELECT 'order_items',COUNT(*) FROM order_items;

-- ผลลัพธ์ที่ควรได้:
-- table_name   | record_count
-- -------------|-------------
-- departments  | 7
-- employees    | 15
-- customers    | 10
-- products     | 15
-- orders       | 10
-- order_items  | 19

-- Test 2: ดูพนักงานแต่ละแผนก
SELECT d.department_name, COUNT(e.employee_id) AS headcount
FROM departments d
LEFT JOIN employees e ON d.department_id = e.department_id
GROUP BY d.department_name
ORDER BY headcount DESC;

-- ผลลัพธ์:
-- Information Technology | 5
-- Sales                  | 3
-- Human Resources        | 2
-- Finance                | 2
-- Marketing              | 2
-- Operations             | 1
-- Research & Development | 1
```

---

## 2.13 Troubleshooting - การแก้ปัญหาที่พบบ่อย

### PostgreSQL ไม่เริ่ม

```bash
# Windows
net start postgresql-x64-16

# macOS
brew services start postgresql@16

# Linux
sudo systemctl start postgresql
sudo systemctl status postgresql

# ดู logs
# Linux
sudo journalctl -u postgresql

# ดู error
sudo -u postgres pg_lsclusters
```

### Connection Refused

```
ปัญหา: Could not connect to server: Connection refused

สาเหตุที่เป็นไปได้:
1. PostgreSQL service ไม่ได้รัน
   → sudo systemctl start postgresql

2. Port ผิด
   → ตรวจสอบว่า port เป็น 5432

3. pg_hba.conf ไม่อนุญาต
   → แก้ไฟล์ pg_hba.conf

4. Firewall บล็อก
   → เพิ่ม rule ใน firewall
```

### Password Authentication Failed

```sql
-- วิธีแก้ใน PostgreSQL:
-- เปลี่ยนเป็น peer authentication ชั่วคราว
-- แก้ /etc/postgresql/16/main/pg_hba.conf
-- local all postgres peer → local all postgres trust
-- จากนั้น restart และเปลี่ยน password

-- เชื่อมต่อและเปลี่ยน password
ALTER USER postgres WITH PASSWORD 'new_password';
```

### Character Encoding ปัญหาภาษาไทย

```sql
-- PostgreSQL: ตรวจสอบ encoding
SHOW client_encoding;
SHOW server_encoding;

-- เปลี่ยน encoding สำหรับ session
SET client_encoding TO 'UTF8';

-- MySQL: ตรวจสอบ charset
SHOW VARIABLES LIKE 'character_set%';

-- ถ้าสร้าง database ใหม่:
CREATE DATABASE sql_course CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;

-- SQLite: ใช้ UTF-8 โดยอัตโนมัติ
```

---

## 2.14 สรุปบทที่ 2

ในบทนี้เราได้ setup:

```
✅ PostgreSQL พร้อมใช้งาน
✅ MySQL/SQLite พร้อมใช้งาน
✅ DBeaver GUI tool
✅ VS Code + SQLTools extension
✅ Online playgrounds รู้จัก
✅ ฐานข้อมูลตัวอย่างสร้างเสร็จแล้ว
✅ ทดสอบ connection ผ่าน
```

---

## 📝 แบบฝึกหัดท้ายบท

### แบบฝึกหัดที่ 1-5: Setup Tasks

**คำถาม 1:** ติดตั้ง SQLite หรือ PostgreSQL และรัน `SELECT version();` สำเร็จ

**คำถาม 2:** สร้าง database ชื่อ `sql_course` และ run script `setup_database.sql`

**คำถาม 3:** เชื่อมต่อ database ด้วย DBeaver และดู tables ที่มีทั้งหมด

**คำถาม 4:** เปิด VS Code ติดตั้ง SQLTools extension และเชื่อมต่อ database

**คำถาม 5:** ลองรัน query นี้ใน 3 วิธีต่างกัน (CLI, DBeaver, VS Code):
```sql
SELECT COUNT(*) FROM employees;
```

### แบบฝึกหัดที่ 6-10: SQL Queries

**คำถาม 6:** รัน query เพื่อดูข้อมูลทั้งหมดในตาราง `departments`

**คำถาม 7:** รัน query เพื่อดู products ทั้งหมด เรียงตาม `unit_price` จากแพงไปถูก

**คำถาม 8:** รัน query นี้และอธิบายผลลัพธ์:
```sql
SELECT table_name 
FROM information_schema.tables 
WHERE table_schema = 'public';
```

**คำถาม 9:** สร้างตารางใหม่ชื่อ `my_notes` ด้วย columns: id, title, content, created_at

**คำถาม 10:** Insert 3 rows ลงใน `my_notes` แล้ว SELECT ทั้งหมดออกมา

---

## ✅ เฉลยแบบฝึกหัด

### เฉลยที่ 6:
```sql
SELECT * FROM departments;
```

### เฉลยที่ 7:
```sql
SELECT product_name, unit_price, category
FROM products
ORDER BY unit_price DESC;
```

### เฉลยที่ 8:
```
Query นี้ดูรายชื่อตารางทั้งหมดใน schema 'public' ของ PostgreSQL
information_schema เป็น built-in schema ที่เก็บ metadata ของ database
ผลลัพธ์จะแสดงชื่อตาราง: departments, employees, customers, etc.
```

### เฉลยที่ 9:
```sql
CREATE TABLE my_notes (
    id INTEGER PRIMARY KEY,
    title VARCHAR(100) NOT NULL,
    content TEXT,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP
);
```

### เฉลยที่ 10:
```sql
INSERT INTO my_notes (id, title, content) VALUES
(1, 'SQL Basics', 'เรียนรู้ SELECT, FROM, WHERE'),
(2, 'JOIN Types', 'INNER, LEFT, RIGHT, FULL OUTER'),
(3, 'Indexes', 'ช่วยให้ query เร็วขึ้น');

SELECT * FROM my_notes;
```

---

## ➡️ บทถัดไป

**[Part 003: Basic SELECT - Your First Queries](part-003.md)**

ในบทถัดไปเราจะเจาะลึก SELECT statement ซึ่งเป็นคำสั่งที่ใช้บ่อยที่สุดใน SQL!

---

*Part 002 of 120 | หลักสูตร SQL ครบวงจร*
