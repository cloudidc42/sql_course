# Part 48: Subqueries in UPDATE, DELETE, and INSERT

## 48.1 DML กับ Subqueries

Subqueries ไม่ได้ใช้แค่กับ SELECT เท่านั้น แต่ยังใช้กับ DML statements ได้ด้วย:
- `UPDATE ... WHERE (subquery)`
- `UPDATE ... SET col = (subquery)`
- `DELETE ... WHERE (subquery)`
- `INSERT ... SELECT (subquery)`

---

## 48.2 UPDATE กับ Subquery ใน WHERE

### ตัวอย่างที่ 1: UPDATE ข้อมูลตาม subquery

```sql
-- เพิ่มเงินเดือน 10% ให้พนักงานที่อยู่ในแผนกที่มีงบประมาณสูงกว่า 5 ล้าน
UPDATE employees
SET    salary = salary * 1.10
WHERE  department_id IN (
    SELECT department_id
    FROM   departments
    WHERE  budget > 5000000
);
```

### ตัวอย่างที่ 2: UPDATE โดยใช้ aggregate subquery

```sql
-- เพิ่มเงินเดือน 5% ให้พนักงานที่มีเงินเดือนต่ำกว่าค่าเฉลี่ย
UPDATE employees
SET    salary = salary * 1.05
WHERE  salary < (
    SELECT AVG(salary)
    FROM   employees
    -- ⚠️ MySQL ไม่อนุญาตให้ UPDATE ตาราง และ subquery จากตารางเดียวกัน!
);
-- ERROR: You can't specify target table 'employees' for update in FROM clause
```

### ตัวอย่างที่ 3: แก้ไข MySQL Self-reference ด้วย Derived Table

```sql
-- แก้ไขด้วย derived table (MySQL workaround):
UPDATE employees
SET    salary = salary * 1.05
WHERE  salary < (
    SELECT avg_sal
    FROM (
        SELECT AVG(salary) AS avg_sal FROM employees
    ) AS emp_avg  -- ← ห่อด้วย derived table
);
```

### ตัวอย่างที่ 4: UPDATE กับ EXISTS

```sql
-- อัพเดท status ออเดอร์ที่มีสินค้า Electronics
UPDATE orders o
SET    o.status = 'priority'
WHERE  EXISTS (
    SELECT 1
    FROM   order_items oi
    JOIN   products p ON p.product_id = oi.product_id
    WHERE  oi.order_id = o.order_id
      AND  p.category = 'Electronics'
      AND  p.price > 20000
);
```

### ตัวอย่างที่ 5: UPDATE กับ NOT EXISTS

```sql
-- Mark สินค้าที่ไม่เคยขายว่า discontinued
-- สมมติมีคอลัมน์ is_active
ALTER TABLE products ADD COLUMN is_active TINYINT DEFAULT 1;

UPDATE products p
SET    is_active = 0
WHERE  NOT EXISTS (
    SELECT 1
    FROM   order_items oi
    WHERE  oi.product_id = p.product_id
);
```

### ตัวอย่างที่ 6: UPDATE กับ IN และ aggregate

```sql
-- เพิ่ม stock ให้สินค้าที่ขายดีใน top 3
UPDATE products
SET    stock_qty = stock_qty + 50
WHERE  product_id IN (
    SELECT product_id
    FROM (
        SELECT product_id
        FROM   order_items
        GROUP  BY product_id
        ORDER  BY SUM(quantity) DESC
        LIMIT  3
    ) AS top_products
);
```

---

## 48.3 UPDATE กับ Subquery ใน SET

### ตัวอย่างที่ 7: SET ค่าจาก subquery

```sql
-- สมมติมีตาราง order_totals
-- UPDATE total_amount จากการคำนวณ order_items
UPDATE orders o
SET    total_amount = (
    SELECT SUM(oi.quantity * oi.unit_price)
    FROM   order_items oi
    WHERE  oi.order_id = o.order_id
)
WHERE  total_amount IS NULL OR total_amount = 0;
```

### ตัวอย่างที่ 8: SET หลายคอลัมน์จาก subquery

```sql
-- สร้างตาราง employee_stats
CREATE TABLE employee_stats (
    employee_id    INT PRIMARY KEY,
    dept_avg_salary DECIMAL(10,2),
    dept_rank       INT
);

INSERT INTO employee_stats (employee_id)
SELECT employee_id FROM employees;

-- Update หลายคอลัมน์จาก subquery
UPDATE employee_stats es
SET
    dept_avg_salary = (
        SELECT AVG(salary)
        FROM   employees
        WHERE  department_id = (
            SELECT department_id FROM employees WHERE employee_id = es.employee_id
        )
    ),
    dept_rank = (
        SELECT COUNT(*) + 1
        FROM   employees e2
        WHERE  e2.department_id = (
            SELECT department_id FROM employees WHERE employee_id = es.employee_id
        )
          AND  e2.salary > (
            SELECT salary FROM employees WHERE employee_id = es.employee_id
        )
    );
```

### ตัวอย่างที่ 9: UPDATE กับ Correlated Subquery ใน SET

```sql
-- เพิ่ม budget แผนกตาม % ของเงินเดือนรวม
UPDATE departments d
SET    budget = budget + (
    SELECT SUM(salary) * 0.1
    FROM   employees e
    WHERE  e.department_id = d.department_id
);
```

---

## 48.4 UPDATE ด้วย JOIN (Alternative)

### ตัวอย่างที่ 10: UPDATE + JOIN (MySQL)

```sql
-- MySQL อนุญาต UPDATE + JOIN:
-- เพิ่มเงินเดือน 8% ให้พนักงานในแผนกที่อยู่ใน Bangkok
UPDATE employees e
JOIN   departments d ON d.department_id = e.department_id
SET    e.salary = e.salary * 1.08
WHERE  d.location = 'Bangkok';

-- เทียบกับ subquery approach:
UPDATE employees
SET    salary = salary * 1.08
WHERE  department_id IN (
    SELECT department_id FROM departments WHERE location = 'Bangkok'
);
```

---

## 48.5 DELETE กับ Subquery

### ตัวอย่างที่ 11: DELETE กับ IN

```sql
-- ลบออเดอร์ที่ pending นานกว่า 30 วัน
DELETE FROM orders
WHERE  status = 'pending'
  AND  order_id IN (
    SELECT order_id
    FROM (
        SELECT order_id
        FROM   orders
        WHERE  status = 'pending'
          AND  order_date < DATE_SUB(CURDATE(), INTERVAL 30 DAY)
    ) AS old_pending
    -- ⚠️ MySQL: ต้องห่อ derived table เมื่อ DELETE ตารางเดียวกับ subquery
);
```

### ตัวอย่างที่ 12: DELETE กับ NOT IN

```sql
-- ลบ order_items ที่ไม่มี order อ้างอิง (orphan records)
DELETE FROM order_items
WHERE  order_id NOT IN (
    SELECT order_id FROM orders
);
```

### ตัวอย่างที่ 13: DELETE กับ EXISTS

```sql
-- ลบลูกค้าที่ไม่เคยสั่งซื้อ (ระวัง!)
-- ⚠️ ต้อง backup ก่อน!
DELETE FROM customers
WHERE  NOT EXISTS (
    SELECT 1 FROM orders WHERE customer_id = customers.customer_id
);
```

### ตัวอย่างที่ 14: DELETE กับ NOT EXISTS

```sql
-- ลบ order_items ของออเดอร์ที่ไม่มีอยู่แล้ว
DELETE FROM order_items oi
WHERE  NOT EXISTS (
    SELECT 1
    FROM   orders o
    WHERE  o.order_id = oi.order_id
);
```

### ตัวอย่างที่ 15: DELETE กับ Correlated Subquery

```sql
-- ลบ order_items ที่ product ราคาต่ำกว่า 500 และออเดอร์ไม่ได้เป็น completed
DELETE FROM order_items
WHERE  product_id IN (
    SELECT product_id FROM products WHERE price < 500
)
  AND  order_id IN (
    SELECT order_id
    FROM (
        SELECT order_id FROM orders WHERE status <> 'completed'
    ) AS non_complete
);
```

### ตัวอย่างที่ 16: DELETE + JOIN (MySQL)

```sql
-- DELETE กับ JOIN ใน MySQL:
DELETE oi
FROM   order_items oi
JOIN   orders o ON o.order_id = oi.order_id
WHERE  o.status = 'cancelled'
  AND  o.order_date < DATE_SUB(CURDATE(), INTERVAL 1 YEAR);
```

---

## 48.6 INSERT ... SELECT

### ตัวอย่างที่ 17: INSERT ... SELECT พื้นฐาน

```sql
-- สร้างตาราง backup
CREATE TABLE orders_archive (
    order_id     INT,
    customer_id  INT,
    order_date   DATE,
    total_amount DECIMAL(12,2),
    status       VARCHAR(50),
    archived_at  DATETIME DEFAULT CURRENT_TIMESTAMP
);

-- Copy ออเดอร์เก่ามากกว่า 1 ปีไปยัง archive
INSERT INTO orders_archive (order_id, customer_id, order_date, total_amount, status)
SELECT order_id, customer_id, order_date, total_amount, status
FROM   orders
WHERE  order_date < DATE_SUB(CURDATE(), INTERVAL 1 YEAR)
  AND  status = 'completed';
```

### ตัวอย่างที่ 18: INSERT ... SELECT กับ subquery

```sql
-- สร้าง summary table
CREATE TABLE monthly_sales_summary (
    month_year   VARCHAR(7),
    order_count  INT,
    total_revenue DECIMAL(15,2),
    avg_order    DECIMAL(12,2),
    created_at   DATETIME DEFAULT CURRENT_TIMESTAMP
);

INSERT INTO monthly_sales_summary (month_year, order_count, total_revenue, avg_order)
SELECT
    DATE_FORMAT(order_date, '%Y-%m') AS month_year,
    COUNT(*)                          AS order_count,
    SUM(total_amount)                AS total_revenue,
    AVG(total_amount)                AS avg_order
FROM orders
GROUP BY DATE_FORMAT(order_date, '%Y-%m')
ORDER BY month_year;
```

### ตัวอย่างที่ 19: INSERT ... SELECT กับ JOIN

```sql
-- สร้าง customer purchase report
CREATE TABLE customer_summary (
    customer_id   INT PRIMARY KEY,
    customer_name VARCHAR(100),
    total_orders  INT,
    total_spent   DECIMAL(12,2),
    first_order   DATE,
    last_order    DATE,
    avg_order     DECIMAL(12,2)
);

INSERT INTO customer_summary
SELECT
    c.customer_id,
    CONCAT(c.first_name, ' ', c.last_name),
    COUNT(o.order_id),
    COALESCE(SUM(o.total_amount), 0),
    MIN(o.order_date),
    MAX(o.order_date),
    AVG(o.total_amount)
FROM customers c
LEFT JOIN orders o ON o.customer_id = c.customer_id
GROUP BY c.customer_id, c.first_name, c.last_name;
```

### ตัวอย่างที่ 20: INSERT ... SELECT กับ Subquery ใน SELECT

```sql
-- INSERT พร้อม computed columns
INSERT INTO employee_stats (employee_id, dept_avg_salary, dept_rank)
SELECT
    e.employee_id,
    (SELECT AVG(salary) FROM employees WHERE department_id = e.department_id),
    (SELECT COUNT(*) + 1
     FROM   employees e2
     WHERE  e2.department_id = e.department_id
       AND  e2.salary > e.salary)
FROM employees e;
```

---

## 48.7 INSERT ... SELECT กับ Transformation

### ตัวอย่างที่ 21: Data Migration

```sql
-- ย้ายข้อมูลจากตารางเก่าไปตารางใหม่พร้อม transform
CREATE TABLE products_v2 (
    product_id    INT PRIMARY KEY,
    product_name  VARCHAR(200),
    category      VARCHAR(100),
    price         DECIMAL(10,2),
    price_tier    VARCHAR(20),
    total_sold    INT DEFAULT 0
);

INSERT INTO products_v2 (product_id, product_name, category, price, price_tier, total_sold)
SELECT
    p.product_id,
    p.product_name,
    p.category,
    p.price,
    CASE
        WHEN p.price > 20000 THEN 'Premium'
        WHEN p.price > 5000  THEN 'Mid-range'
        ELSE                      'Budget'
    END AS price_tier,
    COALESCE((
        SELECT SUM(oi.quantity)
        FROM   order_items oi
        WHERE  oi.product_id = p.product_id
    ), 0) AS total_sold
FROM products p;
```

### ตัวอย่างที่ 22: Selective INSERT

```sql
-- Insert เฉพาะ products ที่ขายได้
INSERT INTO products_v2 (product_id, product_name, category, price, price_tier)
SELECT
    product_id,
    product_name,
    category,
    price,
    CASE
        WHEN price > 20000 THEN 'Premium'
        WHEN price > 5000  THEN 'Mid-range'
        ELSE 'Budget'
    END
FROM products
WHERE product_id IN (
    SELECT DISTINCT product_id FROM order_items
);
```

---

## 48.8 UPSERT / MERGE Preview

### ตัวอย่างที่ 23: INSERT ... ON DUPLICATE KEY UPDATE (MySQL)

```sql
-- MySQL: UPSERT pattern
INSERT INTO customer_summary (customer_id, customer_name, total_orders, total_spent)
SELECT
    c.customer_id,
    CONCAT(c.first_name, ' ', c.last_name),
    COUNT(o.order_id),
    SUM(o.total_amount)
FROM customers c
LEFT JOIN orders o ON o.customer_id = c.customer_id
GROUP BY c.customer_id, c.first_name, c.last_name
ON DUPLICATE KEY UPDATE
    total_orders = VALUES(total_orders),
    total_spent  = VALUES(total_spent);
```

### ตัวอย่างที่ 24: INSERT IGNORE

```sql
-- Insert ข้ามถ้า duplicate
INSERT IGNORE INTO orders_archive (order_id, customer_id, order_date, total_amount, status)
SELECT order_id, customer_id, order_date, total_amount, status
FROM   orders
WHERE  order_date < DATE_SUB(CURDATE(), INTERVAL 6 MONTH);
```

---

## 48.9 ตัวอย่าง DML Complex

### ตัวอย่างที่ 25: Multi-step Data Maintenance

```sql
-- Step 1: Mark ออเดอร์ที่ pending เกิน 7 วัน เป็น expired
UPDATE orders
SET    status = 'expired'
WHERE  status = 'pending'
  AND  order_date < DATE_SUB(CURDATE(), INTERVAL 7 DAY);

-- Step 2: Archive expired orders
INSERT INTO orders_archive (order_id, customer_id, order_date, total_amount, status)
SELECT order_id, customer_id, order_date, total_amount, status
FROM   orders
WHERE  status = 'expired';

-- Step 3: Delete archived orders from main table
DELETE FROM orders
WHERE  status = 'expired'
  AND  order_id IN (
    SELECT order_id
    FROM (
        SELECT order_id FROM orders_archive
    ) AS archived_ids
);
```

---

## แบบฝึกหัดบทที่ 48

**ข้อ 1:** UPDATE เงินเดือนพนักงาน +15% ที่อยู่ในแผนกที่มีงบประมาณมากกว่า 6 ล้าน

```sql
-- เฉลย:
UPDATE employees
SET    salary = salary * 1.15
WHERE  department_id IN (
    SELECT department_id
    FROM   departments
    WHERE  budget > 6000000
);
```

**ข้อ 2:** UPDATE status ออเดอร์เป็น 'review_needed' สำหรับออเดอร์ที่มียอดสูงกว่าค่าเฉลี่ย

```sql
-- เฉลย:
UPDATE orders
SET    status = 'review_needed'
WHERE  total_amount > (
    SELECT avg_amount
    FROM (
        SELECT AVG(total_amount) AS avg_amount FROM orders
    ) AS avg_tbl
)
  AND  status = 'pending';
```

**ข้อ 3:** DELETE order_items ที่ product_id ไม่มีในตาราง products

```sql
-- เฉลย:
DELETE FROM order_items
WHERE  product_id NOT IN (
    SELECT product_id FROM products
);
```

**ข้อ 4:** INSERT ข้อมูล product summary ลงตารางใหม่โดยรวมยอดขาย

```sql
-- เฉลย:
CREATE TABLE IF NOT EXISTS product_sales_summary (
    product_id    INT,
    product_name  VARCHAR(200),
    total_revenue DECIMAL(12,2),
    total_qty     INT,
    order_count   INT
);

INSERT INTO product_sales_summary
SELECT
    p.product_id,
    p.product_name,
    COALESCE(SUM(oi.quantity * oi.unit_price), 0),
    COALESCE(SUM(oi.quantity), 0),
    COUNT(DISTINCT oi.order_id)
FROM products p
LEFT JOIN order_items oi ON oi.product_id = p.product_id
GROUP BY p.product_id, p.product_name;
```

**ข้อ 5:** UPDATE stock_qty ของสินค้าลดลงตามยอดสั่งซื้อ pending

```sql
-- เฉลย:
UPDATE products p
SET    stock_qty = stock_qty - (
    SELECT COALESCE(SUM(oi.quantity), 0)
    FROM   order_items oi
    JOIN   orders o ON o.order_id = oi.order_id
    WHERE  oi.product_id = p.product_id
      AND  o.status = 'pending'
)
WHERE  EXISTS (
    SELECT 1
    FROM   order_items oi
    JOIN   orders o ON o.order_id = oi.order_id
    WHERE  oi.product_id = p.product_id
      AND  o.status = 'pending'
);
```

**ข้อ 6:** DELETE ลูกค้าที่ไม่มีออเดอร์และสมัครมาแล้วมากกว่า 1 ปี

```sql
-- เฉลย:
DELETE FROM customers
WHERE  created_at < DATE_SUB(CURDATE(), INTERVAL 1 YEAR)
  AND  NOT EXISTS (
    SELECT 1
    FROM   orders o
    WHERE  o.customer_id = customers.customer_id
  );
```

**ข้อ 7:** INSERT ข้อมูลออเดอร์เก่ากว่า 6 เดือนไป archive table

```sql
-- เฉลย:
INSERT INTO orders_archive (order_id, customer_id, order_date, total_amount, status)
SELECT order_id, customer_id, order_date, total_amount, status
FROM   orders
WHERE  order_date < DATE_SUB(CURDATE(), INTERVAL 6 MONTH)
  AND  status IN ('completed', 'cancelled')
  AND  order_id NOT IN (SELECT order_id FROM orders_archive);
```

**ข้อ 8:** UPDATE ราคาสินค้า +5% สำหรับ category ที่มียอดขายรวมมากกว่า 50,000

```sql
-- เฉลย:
UPDATE products p
SET    price = price * 1.05
WHERE  category IN (
    SELECT category
    FROM (
        SELECT p2.category,
               SUM(oi.quantity * oi.unit_price) AS total_rev
        FROM   products p2
        JOIN   order_items oi ON oi.product_id = p2.product_id
        GROUP  BY p2.category
        HAVING SUM(oi.quantity * oi.unit_price) > 50000
    ) AS top_categories
);
```

**ข้อ 9:** INSERT department_summary table จากข้อมูลพนักงาน

```sql
-- เฉลย:
CREATE TABLE IF NOT EXISTS department_summary (
    department_id   INT,
    department_name VARCHAR(100),
    emp_count       INT,
    avg_salary      DECIMAL(10,2),
    total_salary    DECIMAL(12,2),
    min_salary      DECIMAL(10,2),
    max_salary      DECIMAL(10,2)
);

INSERT INTO department_summary
SELECT
    d.department_id,
    d.department_name,
    COUNT(e.employee_id),
    ROUND(AVG(e.salary), 2),
    SUM(e.salary),
    MIN(e.salary),
    MAX(e.salary)
FROM departments d
LEFT JOIN employees e ON e.department_id = d.department_id
GROUP BY d.department_id, d.department_name;
```

**ข้อ 10:** เขียน data maintenance script: UPDATE, INSERT archive, DELETE เก่า

```sql
-- เฉลย: Full maintenance script

-- 1. Mark old pending orders as expired
UPDATE orders
SET    status = 'expired'
WHERE  status = 'pending'
  AND  order_date < DATE_SUB(CURDATE(), INTERVAL 14 DAY);

-- 2. Archive completed orders older than 6 months
INSERT INTO orders_archive (order_id, customer_id, order_date, total_amount, status)
SELECT order_id, customer_id, order_date, total_amount, status
FROM   orders
WHERE  status IN ('completed', 'expired')
  AND  order_date < DATE_SUB(CURDATE(), INTERVAL 6 MONTH)
  AND  order_id NOT IN (SELECT order_id FROM orders_archive);

-- 3. Delete archived orders from main table
DELETE FROM orders
WHERE  order_id IN (
    SELECT order_id
    FROM (
        SELECT order_id FROM orders_archive
    ) AS archived
)
  AND  status IN ('completed', 'expired')
  AND  order_date < DATE_SUB(CURDATE(), INTERVAL 6 MONTH);
```

---

*จบบทที่ 48: Subqueries in UPDATE, DELETE, and INSERT*
*บทถัดไป: Part 49 - Subquery Optimization*
