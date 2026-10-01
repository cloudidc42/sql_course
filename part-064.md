# Part 064: Composite and Multi-Column Indexes

## Composite Index และ Multi-Column Index

---

## บทนำ

Composite Index (หรือ Multi-Column Index) คือ index ที่ครอบคลุมมากกว่า 1 คอลัมน์ การเรียงลำดับคอลัมน์ใน composite index ส่งผลต่อประสิทธิภาพอย่างมาก ผิดลำดับ = ไม่ได้ใช้ index!

---

## 1. Composite Index พื้นฐาน

```sql
-- ตัวอย่างที่ 1: สร้าง composite index
CREATE INDEX idx_employees_dept_salary ON employees(department_id, salary);

-- Index นี้เก็บข้อมูลโดยเรียงตาม:
-- 1. department_id ก่อน
-- 2. ภายใน department_id เดียวกัน เรียงตาม salary

-- โครงสร้างใน B-tree:
/*
[dept=1, salary=30000] → row pointer
[dept=1, salary=35000] → row pointer
[dept=1, salary=40000] → row pointer
[dept=2, salary=25000] → row pointer
[dept=2, salary=55000] → row pointer
...
*/
```

---

## 2. Leftmost Prefix Rule (กฎ Prefix ซ้ายสุด)

กฎสำคัญที่สุดของ composite index: **index จะถูกใช้เฉพาะเมื่อ query ใช้คอลัมน์จากซ้ายสุด**

```sql
-- Composite index: (A, B, C)
CREATE INDEX idx_abc ON orders(customer_id, order_date, status);

-- ตัวอย่างที่ 2: ✓ ใช้ index เต็มที่
SELECT * FROM orders WHERE customer_id = 1 AND order_date = '2024-01-01' AND status = 'paid';

-- ตัวอย่างที่ 3: ✓ ใช้ index บางส่วน (A, B)
SELECT * FROM orders WHERE customer_id = 1 AND order_date = '2024-01-01';

-- ตัวอย่างที่ 4: ✓ ใช้ index บางส่วน (A เท่านั้น)
SELECT * FROM orders WHERE customer_id = 1;

-- ตัวอย่างที่ 5: ✗ ไม่ใช้ index! (ข้าม A)
SELECT * FROM orders WHERE order_date = '2024-01-01' AND status = 'paid';

-- ตัวอย่างที่ 6: ✗ ไม่ใช้ index! (ข้าม A และ B)
SELECT * FROM orders WHERE status = 'paid';

-- ตัวอย่างที่ 7: ✓ ใช้ index บางส่วน (A เท่านั้น แม้ระบุ C ด้วย)
SELECT * FROM orders WHERE customer_id = 1 AND status = 'paid';
-- หมายเหตุ: index scan บน A แล้ว filter บน C
```

### ภาพประกอบ Leftmost Prefix Rule

```
Composite Index: (last_name, first_name, birth_date)

Query Patterns:
┌─────────────────────────────────────────────────────────────────┐
│ WHERE last_name = 'Smith'                          → 1 column ✓ │
│ WHERE last_name = 'Smith' AND first_name = 'John'  → 2 columns ✓│
│ WHERE last_name = 'Smith' AND first_name = 'John'               │
│       AND birth_date = '1990-01-01'                → 3 columns ✓│
│                                                                  │
│ WHERE first_name = 'John'                          → 0 columns ✗│
│ WHERE first_name = 'John' AND birth_date = '1990'  → 0 columns ✗│
│ WHERE last_name = 'Smith' AND birth_date = '1990'  → 1 column ✓ │
│   (index ใช้ last_name แล้ว filter birth_date เอง)              │
└─────────────────────────────────────────────────────────────────┘
```

---

## 3. ลำดับคอลัมน์สำคัญมาก: 25+ ตัวอย่าง

### กรณีที่ 1: Equality ก่อน Range

```sql
-- ตัวอย่างที่ 8: ❌ ลำดับผิด
CREATE INDEX idx_wrong ON orders(order_date, customer_id);
-- Query: customer_id = 1 AND order_date >= '2024-01-01'
-- ปัญหา: order_date เป็น range → index หยุดใช้หลัง range column!

-- ตัวอย่างที่ 9: ✅ ลำดับถูก  
CREATE INDEX idx_correct ON orders(customer_id, order_date);
-- Query: customer_id = 1 AND order_date >= '2024-01-01'
-- Index ใช้ customer_id (equality) แล้วค้นหา range ใน order_date ✓
```

### กรณีที่ 2: High Selectivity ก่อน

```sql
-- ตัวอย่างที่ 10: ❌ ลำดับผิด (gender ก่อน dept_id)
CREATE INDEX idx_wrong2 ON employees(gender, department_id);
-- gender มีแค่ M/F → selectivity ต่ำมาก
-- ต้องค้นหาใน half of index แล้วค้นหา dept_id อีกที

-- ตัวอย่างที่ 11: ✅ ลำดับถูก (dept_id ก่อน gender)
CREATE INDEX idx_correct2 ON employees(department_id, gender);
-- department_id มี high selectivity → แยกข้อมูลได้ดีกว่า
```

### กรณีที่ 3: JOIN + Filter Conditions

```sql
-- Query ที่ต้องการ:
SELECT o.*, c.name
FROM orders o
JOIN customers c ON o.customer_id = c.customer_id
WHERE c.country = 'TH' AND o.status = 'pending'
ORDER BY o.created_at DESC;

-- ตัวอย่างที่ 12: ✅ Index บน orders สำหรับ query นี้
CREATE INDEX idx_orders_status_date ON orders(status, created_at DESC);
-- status (equality) ก่อน created_at (sort)

-- ตัวอย่างที่ 13: ✅ Index บน customers สำหรับ JOIN
CREATE INDEX idx_customers_country ON customers(country, customer_id);
-- ช่วย filter country แล้ว JOIN กับ orders
```

### กรณีที่ 4: WHERE vs ORDER BY

```sql
-- Query: หาสินค้าในหมวดหมู่เรียงตามราคา
SELECT * FROM products 
WHERE category_id = 5 AND is_active = TRUE
ORDER BY price ASC;

-- ตัวอย่างที่ 14: ❌ ไม่ดี
CREATE INDEX idx_products_bad ON products(price, category_id, is_active);
-- price เป็น sort column ไม่ควรอยู่แรก

-- ตัวอย่างที่ 15: ✅ ดี
CREATE INDEX idx_products_good ON products(category_id, is_active, price);
-- category_id (equality) → is_active (equality) → price (sort)
```

### กรณีที่ 5: Multi-table Join Optimization

```sql
-- ตัวอย่างที่ 16-20: Orders system

-- Orders ที่ต้องค้นหาหลายแบบ:

-- Query A: ดู orders ของ customer ในช่วงเวลา
SELECT * FROM orders WHERE customer_id = ? AND created_at >= ? ORDER BY created_at DESC;
CREATE INDEX idx_orders_cust_date ON orders(customer_id, created_at DESC);

-- Query B: ดู orders ตาม status
SELECT * FROM orders WHERE status = ? ORDER BY created_at DESC;
CREATE INDEX idx_orders_status_date ON orders(status, created_at DESC);

-- Query C: ดู orders ของ customer ตาม status
SELECT * FROM orders WHERE customer_id = ? AND status = ?;
CREATE INDEX idx_orders_cust_status ON orders(customer_id, status);

-- Query D: Dashboard: orders ใหม่ล่าสุด ทุก status
SELECT * FROM orders ORDER BY created_at DESC LIMIT 100;
CREATE INDEX idx_orders_date_only ON orders(created_at DESC);

-- Query E: Report: จำนวน orders ต่อ customer ต่อเดือน
SELECT customer_id, DATE_TRUNC('month', created_at), COUNT(*)
FROM orders 
GROUP BY 1, 2;
-- ไม่มี index ดีพอ → อาจต้องพิจารณา materialized view
```

### กรณีที่ 6: คอลัมน์ที่ใช้ใน LIKE

```sql
-- ตัวอย่างที่ 21: Index สำหรับ LIKE prefix
CREATE INDEX idx_products_sku ON products(sku, product_id);
SELECT product_id, name FROM products WHERE sku LIKE 'ELEC%';
-- LIKE prefix ใช้ sku column ใน index ได้
-- product_id ใน index ทำให้เป็น covering index!

-- ตัวอย่างที่ 22: Case-insensitive search
CREATE INDEX idx_products_lower_sku ON products(LOWER(sku));
SELECT * FROM products WHERE LOWER(sku) = LOWER('elec-001');
```

---

## 4. Covering Index Strategy

Covering Index คือ index ที่มีข้อมูลทุกอย่างที่ query ต้องการ ทำให้ไม่ต้อง fetch ตารางเลย (Index Only Scan)

```sql
-- Query: ดูชื่อและ salary ของพนักงานใน department
SELECT first_name, last_name, salary 
FROM employees 
WHERE department_id = 5;

-- ตัวอย่างที่ 23: ✅ Covering index
CREATE INDEX idx_emp_dept_cover 
ON employees(department_id, first_name, last_name, salary);
-- Index มีทุกคอลัมน์ที่ query ต้องการ:
-- WHERE: department_id ✓
-- SELECT: first_name, last_name, salary ✓
-- ไม่ต้อง fetch table เลย! = Index Only Scan

-- ตรวจสอบ:
EXPLAIN SELECT first_name, last_name, salary 
FROM employees WHERE department_id = 5;
/*
Index Only Scan using idx_emp_dept_cover on employees
  (cost=0.43..45.67 rows=50 width=60)
  Index Cond: (department_id = 5)
  Heap Fetches: 0  ← ไม่ต้อง fetch heap เลย!
*/
```

### ตัวอย่าง Covering Index แบบต่างๆ

```sql
-- ตัวอย่างที่ 24: Covering index สำหรับ COUNT query
CREATE INDEX idx_orders_cust_status_cover 
ON orders(customer_id, status, order_id);

SELECT customer_id, status, COUNT(order_id)
FROM orders 
WHERE customer_id = 1001
GROUP BY customer_id, status;
-- Index Only Scan ✓

-- ตัวอย่างที่ 25: Covering index สำหรับ aggregation
CREATE INDEX idx_sales_product_date_amount 
ON sales(product_id, sale_date, amount);

SELECT product_id, SUM(amount)
FROM sales 
WHERE product_id = 5 AND sale_date >= '2024-01-01'
GROUP BY product_id;
-- Index Only Scan ✓ (ทุก column อยู่ใน index)
```

---

## 5. INCLUDE Columns (PostgreSQL 11+, SQL Server)

INCLUDE columns คือวิธีเพิ่มคอลัมน์เข้า index โดยไม่ส่งผลต่อ sort order ของ index เหมาะสำหรับ covering index ที่ไม่ต้องการ search คอลัมน์นั้น

```sql
-- ตัวอย่างที่ 26: INCLUDE columns ใน PostgreSQL
-- Query: ค้นหาพนักงานตาม department แล้วดู salary และ phone
SELECT first_name, last_name, salary, phone
FROM employees
WHERE department_id = 5 AND is_active = TRUE;

-- วิธีที่ 1: ใส่ทุก column เป็น index key (ไม่ดี)
CREATE INDEX idx_emp_cover_v1 
ON employees(department_id, is_active, first_name, last_name, salary, phone);
-- ปัญหา: salary และ phone เป็น key columns → เสีย space + slow writes

-- วิธีที่ 2: ✅ ใช้ INCLUDE
CREATE INDEX idx_emp_cover_v2 
ON employees(department_id, is_active) 
INCLUDE (first_name, last_name, salary, phone);
-- department_id, is_active: key columns (ใช้ค้นหา/เรียง)
-- first_name, last_name, salary, phone: included columns (แค่เก็บค่า)
-- ผล: covering index แต่ index เล็กกว่า!
```

```sql
-- ตัวอย่างที่ 27: เปรียบเทียบขนาด
-- Index version 1 (ทุก column เป็น key):
-- ขนาด: ใหญ่กว่าเพราะ sort key มีข้อมูลซ้ำๆ

-- Index version 2 (INCLUDE):
-- ขนาด: เล็กกว่า เพราะ key columns สั้นกว่า
-- Write performance: ดีกว่า เพราะ sort order ง่ายกว่า

-- ตัวอย่างที่ 28: INCLUDE กับ aggregate functions
CREATE INDEX idx_orders_customer_date 
ON orders(customer_id, order_date)
INCLUDE (total_amount, status);

-- Query: order history พร้อม total
SELECT order_date, total_amount, status
FROM orders
WHERE customer_id = 1001
ORDER BY order_date DESC;
-- Index Only Scan ✓
```

---

## 6. Composite Index กับ Range Queries

```sql
-- กฎสำคัญ: เมื่อมีคอลัมน์ range condition ใน composite index
-- คอลัมน์ถัดไปจะไม่ถูกใช้เป็น index key (ยังใช้เป็น filter ได้)

-- ตัวอย่างที่ 29
CREATE INDEX idx_orders_cust_date_status ON orders(customer_id, order_date, status);

-- Query A: ✓ ใช้ index ทั้ง 3 columns
WHERE customer_id = 1 AND order_date = '2024-01-15' AND status = 'paid'

-- Query B: ✓ ใช้ index 2 columns (customer_id, order_date range)
WHERE customer_id = 1 AND order_date >= '2024-01-01' AND order_date < '2024-02-01'
-- status ไม่ถูกใช้เป็น index key (แต่ถูก filter หลัง range scan)

-- Query C: ✓ ใช้ index 2 columns และ filter status
WHERE customer_id = 1 AND order_date BETWEEN '2024-01-01' AND '2024-12-31' AND status = 'paid'
-- range บน order_date หยุดการใช้ index key สำหรับ status
-- แต่ status ยังถูก filter (จาก index data เอง ในกรณี covering index)
```

### ตัวอย่างเปรียบเทียบ Performance

```sql
-- Setup: 10 ล้าน orders

-- ตัวอย่างที่ 30: ทดสอบลำดับคอลัมน์

-- Index A (date ก่อน)
CREATE INDEX idx_A ON orders(order_date, customer_id, status);
-- Index B (customer_id ก่อน)  
CREATE INDEX idx_B ON orders(customer_id, order_date, status);

-- Query: customer_id = 1001 AND order_date >= '2024-01-01'
EXPLAIN ANALYZE SELECT * FROM orders 
WHERE customer_id = 1001 AND order_date >= '2024-01-01';

-- ด้วย idx_A (date ก่อน):
-- ต้องค้นหา date range ทั้งหมดก่อน (อาจ 5M rows)
-- แล้ว filter customer_id
-- ผล: Slow! ใช้ index แต่ไม่มีประสิทธิภาพ

-- ด้วย idx_B (customer_id ก่อน):
-- ค้นหา customer_id = 1001 ก่อน (อาจ 1000 rows)
-- แล้ว filter date range
-- ผล: Fast! ใช้ index อย่างมีประสิทธิภาพ
```

---

## 7. Common Composite Index Patterns

### Pattern 1: Customer-centric queries

```sql
-- ตัวอย่างที่ 31: E-commerce order lookup
CREATE INDEX idx_orders_customer_lookup 
ON orders(customer_id, created_at DESC, status)
INCLUDE (order_id, total_amount);

-- รองรับ queries:
-- "ดู orders ทั้งหมดของ customer X" 
-- "ดู orders ล่าสุดของ customer X"
-- "ดู pending orders ของ customer X"
```

### Pattern 2: Time-series Analytics

```sql
-- ตัวอย่างที่ 32: Analytics queries
CREATE INDEX idx_events_time_type 
ON events(event_date, event_type, user_id)
INCLUDE (value, duration);

-- รองรับ queries:
-- "events ในช่วงวันที่"
-- "events ประเภทนี้ในช่วงวันที่"
-- Daily/monthly aggregates
```

### Pattern 3: Multi-tenant SaaS

```sql
-- ตัวอย่างที่ 33: Multi-tenant pattern
CREATE INDEX idx_documents_tenant_user 
ON documents(tenant_id, user_id, created_at DESC)
INCLUDE (doc_type, title, file_size);

-- กฎสำคัญ: tenant_id ต้องเป็น column แรกเสมอ!
-- เพราะทุก query จะมี tenant_id ใน WHERE
```

### Pattern 4: Status + Date หรือ Date + Status

```sql
-- ตัวอย่างที่ 34: เลือกตาม query frequency
-- ถ้า query บ่อยกว่า: "status = X AND date range"
CREATE INDEX idx_tasks_status_date ON tasks(status, due_date);
-- ถ้า query บ่อยกว่า: "date range ทุก status"
CREATE INDEX idx_tasks_date_status ON tasks(due_date, status);
```

### Pattern 5: Hierarchical Data

```sql
-- ตัวอย่างที่ 35: Organization hierarchy
CREATE INDEX idx_employees_hierarchy 
ON employees(company_id, department_id, manager_id, employee_id);

-- Query: ดูพนักงานทั้งหมดในแผนก
WHERE company_id = 1 AND department_id = 5

-- Query: ดูทีมงานของ manager
WHERE company_id = 1 AND department_id = 5 AND manager_id = 100
```

---

## 8. Index Selectivity ใน Composite Indexes

```sql
-- ตัวอย่างที่ 36: ประเมิน selectivity แต่ละ column
SELECT 
    COUNT(DISTINCT customer_id) AS customer_cardinality,
    COUNT(DISTINCT status) AS status_cardinality,
    COUNT(DISTINCT DATE_TRUNC('day', created_at)) AS date_cardinality,
    COUNT(*) AS total_rows,
    COUNT(*) / COUNT(DISTINCT customer_id) AS avg_orders_per_customer,
    COUNT(*) / COUNT(DISTINCT status) AS avg_orders_per_status
FROM orders;

/*
ผลลัพธ์สมมติ:
customer_cardinality: 100,000
status_cardinality: 5
date_cardinality: 365
total_rows: 5,000,000

avg_orders_per_customer: 50    → customer_id selectivity ดี
avg_orders_per_status: 1,000,000 → status selectivity แย่!

สรุป: customer_id ควรอยู่ก่อน status ใน composite index
*/
```

```sql
-- ตัวอย่างที่ 37: วิเคราะห์ query patterns เพื่อออกแบบ index
-- ดู query ที่ถูก execute บ่อยที่สุดใน PostgreSQL
SELECT 
    query,
    calls,
    mean_exec_time,
    rows
FROM pg_stat_statements
WHERE query LIKE '%orders%'
ORDER BY calls DESC
LIMIT 10;
-- ใช้ข้อมูลนี้ออกแบบ composite indexes ที่เหมาะสม
```

---

## 9. Multi-Column Index Best Practices

```sql
-- ตัวอย่างที่ 38: Index สำหรับ reporting query
-- Report: ยอดขายรายวัน แยกตาม category และ region
SELECT 
    DATE_TRUNC('day', sale_date) AS day,
    category_id,
    region_id,
    SUM(amount) AS total
FROM sales
WHERE sale_date >= '2024-01-01'
GROUP BY 1, 2, 3;

-- ✓ Index ที่เหมาะ:
CREATE INDEX idx_sales_date_cat_region 
ON sales(sale_date, category_id, region_id, amount);
-- date (range), category_id (group), region_id (group), amount (SUM)
-- กลายเป็น covering index สำหรับ query นี้!
```

```sql
-- ตัวอย่างที่ 39: OR conditions
-- ปัญหา: OR ใน WHERE ทำให้ composite index ไม่ทำงานเต็มที่
SELECT * FROM employees WHERE department_id = 1 OR department_id = 2;

-- วิธีแก้: แยกเป็น UNION
SELECT * FROM employees WHERE department_id = 1
UNION ALL
SELECT * FROM employees WHERE department_id = 2;
-- แต่ละ query ใช้ index ของ department_id ✓
```

```sql
-- ตัวอย่างที่ 40: Composite index กับ NULL values
-- PostgreSQL: B-tree เก็บ NULL ค่าได้
CREATE INDEX idx_tasks_assignee_due 
ON tasks(assignee_id, due_date);

-- Query ที่มี NULL check:
SELECT * FROM tasks WHERE assignee_id IS NULL AND due_date < NOW();
-- ใช้ index ได้! NULL ถูก index โดย PostgreSQL B-tree
```

---

## 10. ตัวอย่างเต็ม: ออกแบบ Indexes สำหรับระบบจริง

```sql
-- ระบบจองโรงแรม - Index Design

-- Tables:
CREATE TABLE hotels (
    hotel_id INT PRIMARY KEY,
    name VARCHAR(200),
    city VARCHAR(100),
    country_code CHAR(2),
    star_rating TINYINT,
    price_per_night DECIMAL(10,2)
);

CREATE TABLE rooms (
    room_id INT PRIMARY KEY,
    hotel_id INT,
    room_type VARCHAR(50),
    capacity INT,
    price_per_night DECIMAL(10,2),
    is_available BOOLEAN DEFAULT TRUE
);

CREATE TABLE bookings (
    booking_id BIGINT PRIMARY KEY,
    user_id INT,
    room_id INT,
    hotel_id INT,
    check_in DATE,
    check_out DATE,
    status VARCHAR(20),
    total_price DECIMAL(10,2),
    created_at TIMESTAMP
);

-- ตัวอย่างที่ 41-50: Indexes สำหรับ Hotel Booking

-- 1. ค้นหาโรงแรมในเมือง
CREATE INDEX idx_hotels_city_stars ON hotels(city, star_rating, price_per_night);

-- 2. ค้นหาโรงแรมในประเทศ เรียงตามราคา
CREATE INDEX idx_hotels_country_price ON hotels(country_code, price_per_night);

-- 3. ดูห้องที่ว่างในโรงแรม
CREATE INDEX idx_rooms_hotel_avail ON rooms(hotel_id, is_available, room_type);

-- 4. ตรวจสอบการจองในช่วงวันที่ (conflict check)
CREATE INDEX idx_bookings_room_dates ON bookings(room_id, check_in, check_out)
WHERE status NOT IN ('cancelled', 'refunded');

-- 5. ดู bookings ของ user
CREATE INDEX idx_bookings_user_date ON bookings(user_id, created_at DESC)
INCLUDE (hotel_id, check_in, check_out, status, total_price);

-- 6. Hotel revenue report
CREATE INDEX idx_bookings_hotel_date ON bookings(hotel_id, check_in)
INCLUDE (total_price, status);

-- 7. Admin: ดู bookings ที่รอ check-in วันนี้
CREATE INDEX idx_bookings_checkin_status ON bookings(check_in, status)
WHERE status = 'confirmed';

-- 8. User booking history
CREATE INDEX idx_bookings_user_status ON bookings(user_id, status, created_at DESC);
```

---

## แบบฝึกหัด (10 ข้อ)

**ข้อ 1:** Composite index (a, b, c) ถูกสร้างบนตาราง t ข้อไหนใช้ index ได้?
- (a) `WHERE a = 1`
- (b) `WHERE b = 2`
- (c) `WHERE a = 1 AND c = 3`
- (d) `WHERE a = 1 AND b = 2 AND c = 3`
- (e) `WHERE b = 2 AND c = 3`

**เฉลยข้อ 1:**
- (a) ✓ ใช้ column a (leftmost prefix)
- (b) ✗ ข้าม a (leftmost column)
- (c) ✓ ใช้ column a (b ข้าม แต่ a ยังใช้ได้)
- (d) ✓ ใช้ทุก column
- (e) ✗ ข้าม a (leftmost column)

---

**ข้อ 2:** ออกแบบ composite index สำหรับ query: `SELECT name, email FROM users WHERE country = 'TH' AND age BETWEEN 25 AND 35 ORDER BY name`

**เฉลยข้อ 2:**
```sql
CREATE INDEX idx_users_country_age_name 
ON users(country, age, name)
INCLUDE (email);
-- country (equality) → age (range) → name (sort, แต่จะถูกใช้ได้บางส่วนเพราะ age เป็น range)
-- INCLUDE email เพื่อเป็น covering index
```

---

**ข้อ 3:** อธิบาย Covering Index คืออะไร และทำไมจึงเร็วกว่า index ทั่วไป

**เฉลยข้อ 3:**
Covering Index คือ index ที่มีข้อมูลทุกคอลัมน์ที่ query ต้องการ (ทั้ง WHERE, SELECT, ORDER BY) ทำให้ database สามารถตอบ query ได้จาก index โดยตรงโดยไม่ต้อง fetch data จาก table (heap) ซึ่งเรียกว่า "Index Only Scan" เร็วกว่าเพราะลด I/O ได้มาก

---

**ข้อ 4:** ทำไม INCLUDE columns ถึงดีกว่าการใส่ทุก column เป็น key columns ใน composite index?

**เฉลยข้อ 4:**
- Key columns ส่งผลต่อ sort order ของ index → ใหญ่กว่า, write overhead สูงกว่า
- INCLUDE columns แค่เก็บค่าไว้ใน leaf nodes → ไม่กระทบ sort order
- INCLUDE ทำให้ index เล็กกว่า → fast I/O, fit in cache ได้ดีกว่า
- ใช้ INCLUDE สำหรับ columns ที่ต้องการแค่ "อ่าน" ไม่ใช่ "ค้นหา"

---

**ข้อ 5:** ตาราง `log_events` มี 500M rows columns: event_id, user_id, event_type, severity, created_at ออกแบบ index ที่เหมาะสมสำหรับ: (a) user activity feed (b) error monitoring (c) daily summary report

**เฉลยข้อ 5:**
```sql
-- (a) User activity feed
CREATE INDEX idx_log_user_feed 
ON log_events(user_id, created_at DESC)
INCLUDE (event_type, severity);

-- (b) Error monitoring (partial index)
CREATE INDEX idx_log_errors 
ON log_events(created_at DESC, user_id)
WHERE severity IN ('error', 'critical');

-- (c) Daily summary (BRIN + partial covering)
CREATE INDEX idx_log_daily_brin ON log_events USING BRIN(created_at);
-- BRIN สำหรับ date range scans บน time-series
```

---

**ข้อ 6:** อธิบายว่า range condition ใน composite index ทำให้คอลัมน์ถัดไปไม่ถูกใช้เป็น index key อย่างไร

**เฉลยข้อ 6:**
B-tree index เรียงข้อมูลตาม composite key เมื่อมีการค้นหา range บนคอลัมน์ที่ n คอลัมน์ที่ n+1 ไป "กระจาย" ไม่ได้เรียงต่อเนื่องกัน เช่น:
Index (dept_id, salary, hire_date)
```
dept=1, salary=30000, hire_date=2020-01-15
dept=1, salary=30000, hire_date=2022-06-30
dept=1, salary=40000, hire_date=2019-03-01  ← hire_date ไม่ต่อเนื่อง!
```
เมื่อ query `dept_id = 1 AND salary BETWEEN 30000 AND 50000`, hire_date ในช่วง salary นั้นไม่ได้เรียงต่อเนื่อง ดังนั้นไม่สามารถใช้ binary search บน hire_date ได้

---

**ข้อ 7:** เมื่อใดควรสร้าง index แยกต่างหาก 2 ตัว แทนที่จะสร้าง 1 composite index?

**เฉลยข้อ 7:**
ควรสร้าง 2 indexes แยกเมื่อ:
1. มี 2 queries ที่ต้องการ different leading columns
2. Queries ใช้ OR condition ระหว่างคอลัมน์
3. คอลัมน์หนึ่งถูกใช้บ่อยมาก ส่วนอีกคอลัมน์ใช้บ้าง
4. Table มีขนาดใหญ่และ write-heavy (composite index ใหญ่กว่า)

ตัวอย่าง: queries ทั้ง "customer_id = ?" และ "product_id = ?" ต่างก็บ่อย
```sql
CREATE INDEX idx_orders_customer ON orders(customer_id, created_at);
CREATE INDEX idx_orders_product ON orders(product_id, created_at);
-- ดีกว่า composite (customer_id, product_id) ที่ตอบ query เดียวได้ดี
```

---

**ข้อ 8:** เขียน query เพื่อหา composite indexes ที่มีมากกว่า 3 columns ใน PostgreSQL (อาจ over-indexed)

**เฉลยข้อ 8:**
```sql
SELECT 
    t.relname AS table_name,
    i.relname AS index_name,
    COUNT(a.attnum) AS column_count,
    array_agg(a.attname ORDER BY array_position(ix.indkey, a.attnum)) AS columns
FROM pg_class t
JOIN pg_index ix ON t.oid = ix.indrelid
JOIN pg_class i ON i.oid = ix.indexrelid
JOIN pg_attribute a ON a.attrelid = t.oid AND a.attnum = ANY(ix.indkey)
WHERE t.relkind = 'r'
GROUP BY t.relname, i.relname, ix.indkey
HAVING COUNT(a.attnum) > 3
ORDER BY column_count DESC;
```

---

**ข้อ 9:** ออกแบบ composite index สำหรับ multi-tenant SaaS ที่มี queries: (a) list all docs of user in tenant (b) search docs by type in tenant (c) recent docs across tenant

**เฉลยข้อ 9:**
```sql
-- (a) tenant + user + date
CREATE INDEX idx_docs_tenant_user 
ON documents(tenant_id, user_id, created_at DESC)
INCLUDE (doc_type, title);

-- (b) tenant + type + date  
CREATE INDEX idx_docs_tenant_type 
ON documents(tenant_id, doc_type, created_at DESC)
INCLUDE (title, user_id);

-- (c) tenant + date (cross-user)
CREATE INDEX idx_docs_tenant_recent 
ON documents(tenant_id, created_at DESC)
INCLUDE (user_id, doc_type, title);
-- Note: (a) and (c) indexes overlap - ต้องประเมิน query frequency
```

---

**ข้อ 10:** สร้าง benchmark เพื่อเปรียบเทียบประสิทธิภาพ index (A, B) vs (B, A) สำหรับ query ที่มีทั้ง equality และ range

**เฉลยข้อ 10:**
```sql
-- Setup
CREATE TABLE test_compare AS
SELECT 
    generate_series(1, 1000000) AS id,
    (random() * 100)::int AS dept_id,    -- 100 departments
    (random() * 1000000)::int AS salary,  -- variable salary
    NOW() - (random() * INTERVAL '3650 days') AS hire_date;

-- สร้าง 2 indexes
CREATE INDEX idx_test_dept_salary ON test_compare(dept_id, salary);
CREATE INDEX idx_test_salary_dept ON test_compare(salary, dept_id);

-- Query: WHERE dept_id = 5 AND salary > 50000
EXPLAIN ANALYZE SELECT * FROM test_compare WHERE dept_id = 5 AND salary > 50000;
-- ผล: idx_test_dept_salary เร็วกว่า (dept_id เป็น equality, salary เป็น range)

-- Query: WHERE salary BETWEEN 40000 AND 60000 (no dept filter)
EXPLAIN ANALYZE SELECT * FROM test_compare WHERE salary BETWEEN 40000 AND 60000;
-- ผล: idx_test_salary_dept เร็วกว่า (salary เป็น leading column)
```

---

## สรุป

ใน Part 064 เราได้เรียนรู้:

1. **Leftmost Prefix Rule** - กฎสำคัญที่สุดของ composite index
2. **ลำดับคอลัมน์** - equality ก่อน range, high selectivity ก่อน
3. **25+ ตัวอย่าง** แสดงผลของลำดับคอลัมน์
4. **Covering Index** - ไม่ต้อง fetch table เลย = Index Only Scan
5. **INCLUDE columns** - เพิ่มข้อมูลโดยไม่ส่งผลต่อ sort key
6. **Range Query** - คอลัมน์หลัง range column ไม่ถูกใช้เป็น key
7. **Common Patterns** - Customer-centric, Time-series, Multi-tenant
8. **Selectivity Analysis** - วิเคราะห์ก่อนออกแบบ index

ใน Part 065 เราจะเรียนรู้การอ่าน Query Execution Plans เพื่อเข้าใจว่า database ใช้ index อย่างไร
