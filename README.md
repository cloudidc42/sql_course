# หลักสูตร SQL ครบวงจร: จากพื้นฐานสู่ระดับมืออาชีพ

> **Complete SQL Course: From Fundamentals to Professional Level**
> ออกแบบสำหรับผู้เรียนชาวไทย | Designed for Thai Learners

---

## 📚 เกี่ยวกับหลักสูตรนี้

หลักสูตรนี้ออกแบบมาเพื่อพาคุณจากศูนย์ไปสู่ระดับมืออาชีพในการใช้ SQL อย่างครบถ้วน เนื้อหาทั้งหมดเขียนเป็นภาษาไทย แต่ใช้ภาษาอังกฤษสำหรับโค้ดและคำศัพท์เทคนิค เพื่อให้คุณสามารถทำงานได้จริงในสภาพแวดล้อมระดับสากล

### จุดเด่นของหลักสูตร

- **120 บทเรียน** ครอบคลุมทุกหัวข้อตั้งแต่พื้นฐานถึงขั้นสูง
- **โค้ดตัวอย่างจริง** ที่ใช้งานได้จริงในทุกบทเรียน
- **ฐานข้อมูลตัวอย่างสม่ำเสมอ** ใช้ชุดข้อมูลเดียวกันตลอดหลักสูตร
- **แบบฝึกหัดและคำตอบ** ในทุกบทเรียน
- **รองรับหลาย RDBMS** PostgreSQL, MySQL, SQLite, SQL Server, Oracle
- **เนื้อหาเชิงปฏิบัติ** เน้นสถานการณ์จริงในการทำงาน

---

## 🎯 วัตถุประสงค์การเรียนรู้

เมื่อเรียนจบหลักสูตรนี้ คุณจะสามารถ:

1. **เขียน SQL queries** ได้อย่างมั่นใจตั้งแต่ระดับง่ายถึงซับซ้อน
2. **ออกแบบฐานข้อมูล** โดยใช้หลักการ Normalization และ ER Diagram
3. **ปรับปรุงประสิทธิภาพ** ของ queries ด้วย indexes และ execution plans
4. **ใช้ฟีเจอร์ขั้นสูง** เช่น Window Functions, CTEs, Stored Procedures
5. **ทำงานกับ RDBMS หลักๆ** PostgreSQL, MySQL, SQLite, SQL Server
6. **ออกแบบระบบ** ที่ secure, scalable, และ maintainable
7. **วิเคราะห์ข้อมูล** ด้วย SQL สำหรับ Business Intelligence

---

## 📋 ข้อกำหนดเบื้องต้น

### ความรู้ที่ควรมี
- ความเข้าใจพื้นฐานเกี่ยวกับคอมพิวเตอร์
- สามารถติดตั้งซอฟต์แวร์ได้เอง
- ความเข้าใจพื้นฐานเกี่ยวกับ logic และ mathematics

### ไม่จำเป็นต้องมีประสบการณ์
- ไม่จำเป็นต้องรู้ SQL มาก่อน
- ไม่จำเป็นต้องรู้การเขียนโปรแกรม
- ไม่จำเป็นต้องมีพื้นฐานฐานข้อมูล

---

## 🗂️ วิธีใช้หลักสูตรนี้

### สำหรับผู้เริ่มต้น (Beginner Path)
เรียนตามลำดับ Part 1-30 ก่อน แล้วจึงเลือกหัวข้อที่สนใจ

### สำหรับผู้มีพื้นฐาน (Intermediate Path)
ทดสอบตัวเองด้วยแบบฝึกหัดใน Part 1-10 ถ้าทำได้หมด ข้ามไป Part 31-70

### สำหรับผู้มีประสบการณ์ (Advanced Path)
เน้นที่ Part 71-120 สำหรับหัวข้อขั้นสูงและ performance optimization

### เคล็ดลับการเรียน
1. **ลงมือทำ** - รัน SQL ทุกตัวอย่างด้วยตัวเอง
2. **ทำแบบฝึกหัด** - อย่าข้ามแบบฝึกหัดท้ายบท
3. **ปรับแต่ง** - ลองเปลี่ยน parameters และดูผลลัพธ์
4. **บันทึก** - จดโน้ตสิ่งที่เรียนรู้ใหม่

---

## 💾 ฐานข้อมูลตัวอย่างที่ใช้ตลอดหลักสูตร

หลักสูตรนี้ใช้ระบบจัดการพนักงานและคำสั่งซื้อสมมติ ประกอบด้วยตารางหลัก:

| ตาราง | คำอธิบาย |
|-------|---------|
| `employees` | ข้อมูลพนักงาน |
| `departments` | ข้อมูลแผนก |
| `products` | ข้อมูลสินค้า |
| `customers` | ข้อมูลลูกค้า |
| `orders` | ข้อมูลคำสั่งซื้อ |
| `order_items` | รายการสินค้าในคำสั่งซื้อ |

---

## 📖 สารบัญทั้งหมด 120 บท

### 🔰 Module 1: พื้นฐาน SQL (Part 1-10)

| Part | ชื่อบท | คำอธิบาย |
|------|--------|---------|
| [001](part-001.md) | Introduction to Databases and SQL | ประวัติฐานข้อมูล, RDBMS vs NoSQL, ประเภท SQL, การติดตั้ง |
| [002](part-002.md) | Setting Up Development Environment | การติดตั้ง PostgreSQL/MySQL/SQLite, DBeaver, VS Code extensions |
| [003](part-003.md) | Basic SELECT - Your First Queries | SELECT syntax, เลือก columns, expressions พื้นฐาน |
| [004](part-004.md) | Filtering Data with WHERE Clause | comparison operators, BETWEEN, IN, LIKE, IS NULL |
| [005](part-005.md) | Sorting with ORDER BY | ASC/DESC, multiple columns, NULLS FIRST/LAST |
| [006](part-006.md) | LIMIT, OFFSET, and Pagination | pagination patterns, cursor-based pagination |
| [007](part-007.md) | Working with Column Aliases | AS keyword, expressions, table aliases |
| [008](part-008.md) | NULL Values - Understanding and Handling | three-valued logic, COALESCE, NULLIF |
| [009](part-009.md) | DISTINCT - Removing Duplicates | SELECT DISTINCT, COUNT DISTINCT, performance |
| [010](part-010.md) | String Functions Deep Dive | UPPER/LOWER, TRIM, SUBSTRING, CONCAT, REGEXP |

### 🔢 Module 2: Functions และ Aggregations (Part 11-20)

| Part | ชื่อบท | คำอธิบาย |
|------|--------|---------|
| [011](part-011.md) | Numeric Functions | ROUND, FLOOR, CEIL, ABS, MOD, POWER, SQRT |
| [012](part-012.md) | Date and Time Functions | NOW, DATEADD, DATEDIFF, FORMAT, EXTRACT |
| [013](part-013.md) | Aggregate Functions | COUNT, SUM, AVG, MIN, MAX - รายละเอียด |
| [014](part-014.md) | GROUP BY Clause | grouping data, multiple columns, expressions |
| [015](part-015.md) | HAVING Clause | filtering groups, HAVING vs WHERE |
| [016](part-016.md) | Type Casting and Conversion | CAST, CONVERT, implicit conversion, pitfalls |
| [017](part-017.md) | Conditional Expressions CASE | CASE WHEN, simple vs searched CASE |
| [018](part-018.md) | Math and Statistical Functions | VARIANCE, STDDEV, percentiles |
| [019](part-019.md) | Working with Dates - Advanced | time zones, date arithmetic, fiscal calendars |
| [020](part-020.md) | String Functions Advanced | regex, full-text basics, string splitting |

### 🔗 Module 3: Joins และ Subqueries (Part 21-35)

| Part | ชื่อบท | คำอธิบาย |
|------|--------|---------|
| [021](part-021.md) | INNER JOIN - Combining Tables | JOIN syntax, ON clause, multiple tables |
| [022](part-022.md) | LEFT JOIN and RIGHT JOIN | outer joins, preserving rows |
| [023](part-023.md) | FULL OUTER JOIN | combining both outer joins |
| [024](part-024.md) | CROSS JOIN and SELF JOIN | Cartesian products, self-referential queries |
| [025](part-025.md) | Multiple Joins and Complex Queries | chaining joins, readability |
| [026](part-026.md) | Subqueries Introduction | scalar, row, table subqueries |
| [027](part-027.md) | Correlated Subqueries | subqueries that reference outer query |
| [028](part-028.md) | EXISTS and NOT EXISTS | semi-joins, anti-joins |
| [029](part-029.md) | IN vs EXISTS vs JOIN Performance | choosing the right approach |
| [030](part-030.md) | UNION, INTERSECT, EXCEPT | set operations, use cases |
| [031](part-031.md) | Derived Tables | inline views, FROM subqueries |
| [032](part-032.md) | Common Table Expressions (CTEs) | WITH clause, readability, reuse |
| [033](part-033.md) | Recursive CTEs | hierarchical data, tree structures |
| [034](part-034.md) | Advanced Join Patterns | lateral joins, apply operators |
| [035](part-035.md) | Join Performance Optimization | hash join, nested loop, merge join |

### 📝 Module 4: Data Manipulation Language (Part 36-45)

| Part | ชื่อบท | คำอธิบาย |
|------|--------|---------|
| [036](part-036.md) | INSERT Statements | single row, multiple rows, INSERT SELECT |
| [037](part-037.md) | UPDATE Statements | basic update, update from join |
| [038](part-038.md) | DELETE Statements | basic delete, cascading, soft delete |
| [039](part-039.md) | MERGE / UPSERT | upsert patterns across databases |
| [040](part-040.md) | Transactions and ACID | BEGIN, COMMIT, ROLLBACK, savepoints |
| [041](part-041.md) | Isolation Levels | read uncommitted through serializable |
| [042](part-042.md) | Locking - Optimistic and Pessimistic | SELECT FOR UPDATE, deadlocks |
| [043](part-043.md) | Bulk Operations | bulk insert, bulk update strategies |
| [044](part-044.md) | Data Import and Export | CSV, JSON import/export |
| [045](part-045.md) | Error Handling in SQL | TRY/CATCH, EXCEPTION blocks |

### 🏗️ Module 5: Data Definition Language (Part 46-55)

| Part | ชื่อบท | คำอธิบาย |
|------|--------|---------|
| [046](part-046.md) | CREATE TABLE - Complete Guide | all options, storage parameters |
| [047](part-047.md) | Data Types - Complete Reference | numeric, string, date, boolean, binary |
| [048](part-048.md) | PRIMARY KEY Constraints | single, composite, auto-increment |
| [049](part-049.md) | FOREIGN KEY Constraints | referential integrity, cascade actions |
| [050](part-050.md) | UNIQUE, CHECK, DEFAULT | other constraint types |
| [051](part-051.md) | ALTER TABLE | add/drop/modify columns and constraints |
| [052](part-052.md) | DROP and TRUNCATE | removing objects safely |
| [053](part-053.md) | Sequences and Auto-increment | SERIAL, AUTO_INCREMENT, IDENTITY |
| [054](part-054.md) | Temporary Tables | session vs global, use cases |
| [055](part-055.md) | Schema Management | namespaces, schema design |

### ⚡ Module 6: Indexes และ Performance (Part 56-65)

| Part | ชื่อบท | คำอธิบาย |
|------|--------|---------|
| [056](part-056.md) | Indexes - Introduction | B-tree, how indexes work |
| [057](part-057.md) | Index Types | hash, GIN, GiST, full-text indexes |
| [058](part-058.md) | Composite Indexes | multi-column indexes, column order |
| [059](part-059.md) | Index Strategies | covering indexes, partial indexes |
| [060](part-060.md) | EXPLAIN and Query Plans | reading execution plans |
| [061](part-061.md) | Query Optimization Techniques | rewriting queries for performance |
| [062](part-062.md) | Statistics and the Query Planner | how the planner makes decisions |
| [063](part-063.md) | Partitioning | range, list, hash partitioning |
| [064](part-064.md) | Caching Strategies | query cache, application cache |
| [065](part-065.md) | Connection Pooling | PgBouncer, connection management |

### 🔧 Module 7: Views, Procedures, Functions (Part 66-75)

| Part | ชื่อบท | คำอธิบาย |
|------|--------|---------|
| [066](part-066.md) | Views - Creating and Using | simple views, complex views |
| [067](part-067.md) | Materialized Views | refreshing, use cases |
| [068](part-068.md) | Stored Procedures | PL/pgSQL, T-SQL, MySQL procedures |
| [069](part-069.md) | User-Defined Functions | scalar, table-valued functions |
| [070](part-070.md) | Triggers | BEFORE/AFTER, row/statement level |
| [071](part-071.md) | Cursors | explicit cursors, when to use |
| [072](part-072.md) | Dynamic SQL | EXECUTE, sp_executesql |
| [073](part-073.md) | Error Handling Advanced | exception types, logging |
| [074](part-074.md) | Scheduled Jobs | pg_cron, MySQL events |
| [075](part-075.md) | Procedures vs Functions vs Triggers | choosing the right approach |

### 🪟 Module 8: Window Functions (Part 76-85)

| Part | ชื่อบท | คำอธิบาย |
|------|--------|---------|
| [076](part-076.md) | Window Functions Introduction | OVER clause, PARTITION BY |
| [077](part-077.md) | ROW_NUMBER, RANK, DENSE_RANK | ranking and numbering |
| [078](part-078.md) | NTILE and PERCENT_RANK | percentile ranking |
| [079](part-079.md) | LEAD and LAG | accessing adjacent rows |
| [080](part-080.md) | FIRST_VALUE and LAST_VALUE | first/last in a window |
| [081](part-081.md) | Window Frames | ROWS vs RANGE, UNBOUNDED |
| [082](part-082.md) | Running Totals and Moving Averages | practical window aggregations |
| [083](part-083.md) | Window Functions with CTEs | combining techniques |
| [084](part-084.md) | Advanced Window Patterns | gaps and islands, sessionization |
| [085](part-085.md) | Performance of Window Functions | optimization tips |

### 🗄️ Module 9: Database Design (Part 86-95)

| Part | ชื่อบท | คำอธิบาย |
|------|--------|---------|
| [086](part-086.md) | Database Design Introduction | goals, principles |
| [087](part-087.md) | Entity-Relationship Diagrams | ER notation, tools |
| [088](part-088.md) | Normalization - 1NF | first normal form rules |
| [089](part-089.md) | Normalization - 2NF | second normal form, partial dependencies |
| [090](part-090.md) | Normalization - 3NF | third normal form, transitive dependencies |
| [091](part-091.md) | Normalization - BCNF and Beyond | higher normal forms |
| [092](part-092.md) | Denormalization | when and how to denormalize |
| [093](part-093.md) | Design Patterns | common patterns, anti-patterns |
| [094](part-094.md) | Data Warehouse Design | star schema, snowflake schema |
| [095](part-095.md) | Slowly Changing Dimensions | SCD types 0-6 |

### 🔒 Module 10: Security (Part 96-100)

| Part | ชื่อบท | คำอธิบาย |
|------|--------|---------|
| [096](part-096.md) | Security Introduction | threat model, principles |
| [097](part-097.md) | Users and Roles | CREATE USER, CREATE ROLE |
| [098](part-098.md) | GRANT and REVOKE | permission management |
| [099](part-099.md) | Row Level Security | PostgreSQL RLS policies |
| [100](part-100.md) | SQL Injection Prevention | parameterized queries, best practices |

### 🌐 Module 11: Advanced and Platform-Specific (Part 101-110)

| Part | ชื่อบท | คำอธิบาย |
|------|--------|---------|
| [101](part-101.md) | JSON in SQL - PostgreSQL | JSONB, operators, indexes |
| [102](part-102.md) | JSON in SQL - MySQL | JSON functions, generated columns |
| [103](part-103.md) | Arrays in PostgreSQL | array operations, unnest |
| [104](part-104.md) | Full-Text Search | tsvector, tsquery, ranking |
| [105](part-105.md) | Regular Expressions Advanced | regex patterns, replacement |
| [106](part-106.md) | Geospatial Data Basics | PostGIS introduction |
| [107](part-107.md) | Pivot and Unpivot | crosstab, conditional aggregation |
| [108](part-108.md) | Hierarchical Queries | CONNECT BY, recursive patterns |
| [109](part-109.md) | Temporal Tables | system-versioned, history tracking |
| [110](part-110.md) | Time Series Analysis | time-based aggregations, gaps |

### 📊 Module 12: Analytics and Reporting (Part 111-115)

| Part | ชื่อบท | คำอธิบาย |
|------|--------|---------|
| [111](part-111.md) | OLAP vs OLTP | analytical vs transactional |
| [112](part-112.md) | Conditional Aggregation | FILTER, CASE in aggregates |
| [113](part-113.md) | Statistical Analysis in SQL | correlation, regression |
| [114](part-114.md) | Cohort Analysis | retention, cohort patterns |
| [115](part-115.md) | Funnel Analysis | conversion funnels in SQL |

### 🏭 Module 13: Production Patterns (Part 116-120)

| Part | ชื่อบท | คำอธิบาย |
|------|--------|---------|
| [116](part-116.md) | Audit Trail Implementation | tracking changes, history |
| [117](part-117.md) | SQL Anti-Patterns | common mistakes to avoid |
| [118](part-118.md) | Best Practices and Style Guide | naming, formatting, documentation |
| [119](part-119.md) | Real-World Project: E-commerce | complete database design and queries |
| [120](part-120.md) | Real-World Project: Analytics Dashboard | BI queries, reporting system |

---

## 🛠️ เครื่องมือที่แนะนำ

### Database Systems
| ชื่อ | ประเภท | แนะนำสำหรับ |
|------|--------|------------|
| **SQLite** | ไฟล์เดี่ยว | เริ่มต้น, ทดสอบ |
| **PostgreSQL** | Open Source | Production, ฟีเจอร์ครบ |
| **MySQL/MariaDB** | Open Source | Web applications |
| **SQL Server** | Commercial | Windows, .NET |
| **Oracle** | Commercial | Enterprise |

### GUI Tools
| ชื่อ | Platform | ฟรี/เสียเงิน |
|------|----------|-------------|
| **DBeaver** | Windows/Mac/Linux | ฟรี |
| **TablePlus** | Mac/Windows | มี Free tier |
| **pgAdmin** | PostgreSQL | ฟรี |
| **MySQL Workbench** | MySQL | ฟรี |
| **HeidiSQL** | Windows | ฟรี |

### Online Playgrounds
- [db-fiddle.com](https://www.db-fiddle.com) - รองรับ PostgreSQL, MySQL, SQLite
- [sqlfiddle.com](http://sqlfiddle.com) - หลาย database engines
- [replit.com](https://replit.com) - เขียน SQL พร้อม code

---

## 🎓 วิธีเรียนให้ได้ผลสูงสุด

```
1. อ่านทฤษฎีให้เข้าใจก่อน
2. รัน CREATE TABLE + INSERT ทุกครั้งที่เริ่มบทใหม่
3. รันตัวอย่างทุกข้อด้วยตัวเอง
4. ลองปรับแต่ง query และสังเกตผลลัพธ์
5. ทำแบบฝึกหัดทุกข้อก่อนดูเฉลย
6. ทบทวนบทที่แล้วก่อนเริ่มบทใหม่
```

---

## 📞 ติดต่อและแหล่งข้อมูลเพิ่มเติม

### Official Documentation
- [PostgreSQL Docs](https://www.postgresql.org/docs/)
- [MySQL Docs](https://dev.mysql.com/doc/)
- [SQLite Docs](https://www.sqlite.org/docs.html)
- [SQL Server Docs](https://docs.microsoft.com/en-us/sql/)

### แหล่งเรียนรู้เพิ่มเติม
- [W3Schools SQL](https://www.w3schools.com/sql/)
- [SQLZoo](https://sqlzoo.net/)
- [Mode Analytics SQL Tutorial](https://mode.com/sql-tutorial/)
- [LeetCode SQL Problems](https://leetcode.com/problemset/database/)

---

## 📅 ตารางเรียนแนะนำ

### แผน 3 เดือน (Beginner to Intermediate)
| สัปดาห์ | Parts | เวลา/วัน |
|---------|-------|---------|
| 1-2 | 1-10 | 1-2 ชั่วโมง |
| 3-4 | 11-20 | 1-2 ชั่วโมง |
| 5-6 | 21-30 | 1-2 ชั่วโมง |
| 7-8 | 31-45 | 1-2 ชั่วโมง |
| 9-10 | 46-65 | 1-2 ชั่วโมง |
| 11-12 | 66-80 | 1-2 ชั่วโมง |

### แผน 6 เดือน (Full Course)
เรียนสัปดาห์ละ 20 บท ใช้เวลา 1 ชั่วโมงต่อวัน

---

*หลักสูตรนี้อัปเดตล่าสุด: 2026 | SQL Standard: SQL:2016*

*สงวนสิทธิ์ตามกฎหมาย - สามารถใช้เพื่อการศึกษาได้*
