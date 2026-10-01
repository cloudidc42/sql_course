# Part 41: Introduction to Subqueries (การแนะนำ Subqueries)

## การตั้งค่าฐานข้อมูลตัวอย่าง

```sql
-- สร้างตารางสำหรับระบบ E-Commerce
CREATE TABLE departments (
    department_id   INT PRIMARY KEY AUTO_INCREMENT,
    department_name VARCHAR(100) NOT NULL,
    budget          DECIMAL(15,2),
    location        VARCHAR(100)
);

CREATE TABLE employees (
    employee_id   INT PRIMARY KEY AUTO_INCREMENT,
    first_name    VARCHAR(50) NOT NULL,
    last_name     VARCHAR(50) NOT NULL,
    email         VARCHAR(100) UNIQUE,
    salary        DECIMAL(10,2),
    department_id INT,
    hire_date     DATE,
    manager_id    INT,
    FOREIGN KEY (department_id) REFERENCES departments(department_id)
);

CREATE TABLE customers (
    customer_id   INT PRIMARY KEY AUTO_INCREMENT,
    first_name    VARCHAR(50) NOT NULL,
    last_name     VARCHAR(50) NOT NULL,
    email         VARCHAR(100) UNIQUE,
    phone         VARCHAR(20),
    city          VARCHAR(100),
    country       VARCHAR(100),
    created_at    DATETIME DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE products (
    product_id    INT PRIMARY KEY AUTO_INCREMENT,
    product_name  VARCHAR(200) NOT NULL,
    category      VARCHAR(100),
    price         DECIMAL(10,2) NOT NULL,
    stock_qty     INT DEFAULT 0,
    supplier_id   INT
);

CREATE TABLE orders (
    order_id    INT PRIMARY KEY AUTO_INCREMENT,
    customer_id INT NOT NULL,
    order_date  DATE NOT NULL,
    status      VARCHAR(50) DEFAULT 'pending',
    total_amount DECIMAL(12,2),
    FOREIGN KEY (customer_id) REFERENCES customers(customer_id)
);

CREATE TABLE order_items (
    item_id    INT PRIMARY KEY AUTO_INCREMENT,
    order_id   INT NOT NULL,
    product_id INT NOT NULL,
    quantity   INT NOT NULL,
    unit_price DECIMAL(10,2) NOT NULL,
    FOREIGN KEY (order_id) REFERENCES orders(order_id),
    FOREIGN KEY (product_id) REFERENCES products(product_id)
);

-- ข้อมูลตัวอย่าง
INSERT INTO departments VALUES
(1, 'Sales',           5000000.00, 'Bangkok'),
(2, 'Engineering',     8000000.00, 'Bangkok'),
(3, 'Marketing',       3000000.00, 'Chiang Mai'),
(4, 'HR',              2000000.00, 'Bangkok'),
(5, 'Finance',         4000000.00, 'Bangkok');

INSERT INTO employees VALUES
(1,  'Somchai',  'Jaidee',    'somchai@company.com',  85000, 1, '2019-03-15', NULL),
(2,  'Wanida',   'Kaewkla',   'wanida@company.com',   72000, 1, '2020-06-01', 1),
(3,  'Prasert',  'Thongdee',  'prasert@company.com',  95000, 2, '2018-01-10', NULL),
(4,  'Malee',    'Sombat',    'malee@company.com',    88000, 2, '2019-09-20', 3),
(5,  'Nattapol', 'Wongsiri',  'nattapol@company.com', 78000, 3, '2021-02-14', NULL),
(6,  'Sunisa',   'Phakdee',   'sunisa@company.com',   65000, 3, '2021-07-01', 5),
(7,  'Chaiwat',  'Boonchu',   'chaiwat@company.com',  91000, 2, '2017-11-05', 3),
(8,  'Pimchanok','Rattana',   'pimchanok@company.com',70000, 4, '2022-01-15', NULL),
(9,  'Theerawat','Suwan',     'theerawat@company.com',82000, 1, '2020-04-20', 1),
(10, 'Apinya',   'Moonthong', 'apinya@company.com',   76000, 5, '2021-10-01', NULL);

INSERT INTO customers VALUES
(1,  'Araya',   'Siriporn',  'araya@email.com',   '081-111-1111', 'Bangkok',    'Thailand', '2022-01-15'),
(2,  'Bodin',   'Charoenwong','bodin@email.com',  '082-222-2222', 'Chiang Mai', 'Thailand', '2022-03-20'),
(3,  'Chotika', 'Watthana',  'chotika@email.com', '083-333-3333', 'Bangkok',    'Thailand', '2022-05-10'),
(4,  'Danai',   'Prayong',   'danai@email.com',   '084-444-4444', 'Phuket',     'Thailand', '2022-07-01'),
(5,  'Ekachai', 'Boonmee',   'ekachai@email.com', '085-555-5555', 'Bangkok',    'Thailand', '2022-09-15'),
(6,  'Fah',     'Nontachai', 'fah@email.com',     '086-666-6666', 'Khon Kaen',  'Thailand', '2023-01-05'),
(7,  'Gan',     'Thiparat',  'gan@email.com',     '087-777-7777', 'Bangkok',    'Thailand', '2023-02-14'),
(8,  'Hathai',  'Sukkasem',  'hathai@email.com',  '088-888-8888', 'Chiang Mai', 'Thailand', '2023-03-01'),
(9,  'Ittipat', 'Kruakaew',  'ittipat@email.com', '089-999-9999', 'Bangkok',    'Thailand', '2023-04-20'),
(10, 'Jiraporn','Siamwong',  'jiraporn@email.com','090-000-0000', 'Pattaya',    'Thailand', '2023-05-10');

INSERT INTO products VALUES
(1,  'Laptop Pro 15',     'Electronics',  45000.00,  50,  1),
(2,  'Wireless Mouse',    'Electronics',   890.00,  200,  1),
(3,  'USB-C Hub',         'Electronics',  1500.00,  150,  1),
(4,  'Office Chair',      'Furniture',    8500.00,   30,  2),
(5,  'Standing Desk',     'Furniture',   15000.00,   20,  2),
(6,  'Notebook A4',       'Stationery',    120.00,  500,  3),
(7,  'Ballpoint Pen Set', 'Stationery',    250.00,  300,  3),
(8,  'Coffee Maker',      'Appliances',   3500.00,   40,  4),
(9,  'Air Purifier',      'Appliances',   7800.00,   25,  4),
(10, 'Smartphone X',      'Electronics', 28000.00,   80,  1),
(11, 'Tablet Pro',        'Electronics', 18000.00,   60,  1),
(12, 'Keyboard Mech',     'Electronics',  3200.00,  120,  1),
(13, 'Monitor 27"',       'Electronics', 12000.00,   35,  1),
(14, 'Webcam HD',         'Electronics',  2200.00,   90,  1),
(15, 'Headphones Pro',    'Electronics',  5500.00,   70,  1);

INSERT INTO orders VALUES
(1,  1, '2024-01-15', 'completed', 46890.00),
(2,  2, '2024-01-20', 'completed', 16500.00),
(3,  3, '2024-02-01', 'completed',  8750.00),
(4,  4, '2024-02-15', 'completed', 53000.00),
(5,  5, '2024-03-01', 'completed',  3740.00),
(6,  1, '2024-03-10', 'completed', 28000.00),
(7,  2, '2024-03-20', 'shipped',   15000.00),
(8,  6, '2024-04-01', 'completed',  9390.00),
(9,  7, '2024-04-15', 'completed', 45000.00),
(10, 3, '2024-04-20', 'pending',    1200.00),
(11, 8, '2024-05-01', 'completed', 20200.00),
(12, 9, '2024-05-15', 'completed', 33700.00),
(13,10, '2024-05-20', 'shipped',    7700.00),
(14, 1, '2024-06-01', 'completed', 12000.00),
(15, 5, '2024-06-10', 'pending',    5500.00);

INSERT INTO order_items VALUES
(1,  1,  1, 1, 45000.00),
(2,  1,  2, 1,   890.00),
(3,  2,  5, 1, 15000.00),
(4,  2,  4, 1,  1500.00),
(5,  3,  4, 1,  8500.00),
(6,  3,  7, 1,   250.00),
(7,  4,  1, 1, 45000.00),
(8,  4, 13, 1, 12000.00),
(9,  5,  2, 2,   890.00),
(10, 5,  6, 8,   120.00),
(11, 6, 10, 1, 28000.00),
(12, 7,  5, 1, 15000.00),
(13, 8, 12, 1,  3200.00),
(14, 8,  8, 1,  3500.00),
(15, 8,  7, 2,   250.00),
(16, 9,  1, 1, 45000.00),
(17,10,  6,10,   120.00),
(18,11, 11, 1, 18000.00),
(19,11,  3, 1,  1500.00),
(20,12, 10, 1, 28000.00),
(21,12, 15, 1,  5500.00),
(22,13, 15, 1,  5500.00),
(23,13,  2, 2,   890.00),
(24,14, 13, 1, 12000.00),
(25,15, 15, 1,  5500.00);
```

---

## 41.1 Subquery คืออะไร?

**Subquery** (หรือ nested query / inner query) คือ query ที่อยู่ภายใน query อื่น ซึ่งเรียกว่า outer query หรือ main query

```
Outer Query
┌─────────────────────────────────────┐
│  SELECT ...                         │
│  FROM   ...                         │
│  WHERE  column > (                  │
│    ┌─────────────────────────────┐  │
│    │  SELECT MAX(column)         │  │
│    │  FROM   some_table          │  │
│    └─────────────────────────────┘  │
│  )                 ↑                │
└───────────────── Subquery ──────────┘
```

### ตัวอย่างที่ 1: แนวคิดพื้นฐาน

```sql
-- คำถาม: พนักงานคนไหนมีเงินเดือนสูงกว่าค่าเฉลี่ย?

-- วิธีทำแบบ 2 ขั้นตอน (ไม่มี subquery):
-- ขั้น 1: หาค่าเฉลี่ยเงินเดือน
SELECT AVG(salary) FROM employees;
-- ผลลัพธ์: 80200.00

-- ขั้น 2: ใช้ค่าที่ได้
SELECT first_name, last_name, salary
FROM   employees
WHERE  salary > 80200.00;

-- วิธีทำแบบ subquery (1 ขั้นตอน):
SELECT first_name, last_name, salary
FROM   employees
WHERE  salary > (SELECT AVG(salary) FROM employees);
```

---

## 41.2 ประเภทของ Subquery

### ประเภทที่ 1: Scalar Subquery (คืนค่าเดียว)

```sql
-- คืนค่าเดียว (1 แถว, 1 คอลัมน์)
SELECT first_name, last_name, salary,
       (SELECT AVG(salary) FROM employees) AS avg_salary
FROM   employees;
```

### ประเภทที่ 2: Row Subquery (คืนหลายคอลัมน์ แต่ 1 แถว)

```sql
-- คืน 1 แถว หลายคอลัมน์
SELECT *
FROM   employees
WHERE  (salary, department_id) = (
    SELECT MAX(salary), department_id
    FROM   employees
    WHERE  department_id = 2
    GROUP  BY department_id
);
```

### ประเภทที่ 3: Table Subquery (คืนหลายแถว)

```sql
-- คืนหลายแถว ใช้กับ IN, NOT IN, EXISTS, ANY, ALL
SELECT product_name, price
FROM   products
WHERE  product_id IN (
    SELECT DISTINCT product_id
    FROM   order_items
);
```

### ประเภทที่ 4: Correlated Subquery (อ้างอิง outer query)

```sql
-- อ้างอิงคอลัมน์จาก outer query (e.salary)
SELECT first_name, last_name, salary
FROM   employees e
WHERE  salary > (
    SELECT AVG(salary)
    FROM   employees
    WHERE  department_id = e.department_id
);
```

---

## 41.3 ตำแหน่งที่ใช้ Subquery ได้

### ตำแหน่งที่ 1: ใน SELECT clause

```sql
-- ตัวอย่างที่ 5: แสดงยอดขายรวมของแต่ละสินค้าพร้อมค่าเฉลี่ย
SELECT
    product_name,
    price,
    (SELECT SUM(oi.quantity)
     FROM   order_items oi
     WHERE  oi.product_id = p.product_id) AS total_sold,
    (SELECT AVG(price) FROM products)     AS avg_price
FROM products p;
```

### ตำแหน่งที่ 2: ใน FROM clause (Derived Table)

```sql
-- ตัวอย่างที่ 6: Derived table ใน FROM
SELECT dept_summary.department_id,
       dept_summary.avg_salary,
       d.department_name
FROM (
    SELECT department_id,
           AVG(salary) AS avg_salary,
           COUNT(*)    AS emp_count
    FROM   employees
    GROUP  BY department_id
) AS dept_summary
JOIN departments d ON d.department_id = dept_summary.department_id;
```

### ตำแหน่งที่ 3: ใน WHERE clause

```sql
-- ตัวอย่างที่ 7: Subquery ใน WHERE
SELECT customer_id, first_name, last_name
FROM   customers
WHERE  customer_id IN (
    SELECT customer_id
    FROM   orders
    WHERE  total_amount > 20000
);
```

### ตำแหน่งที่ 4: ใน HAVING clause

```sql
-- ตัวอย่างที่ 8: Subquery ใน HAVING
SELECT department_id,
       AVG(salary) AS avg_dept_salary
FROM   employees
GROUP  BY department_id
HAVING AVG(salary) > (
    SELECT AVG(salary) FROM employees
);
```

### ตำแหน่งที่ 5: ใน ORDER BY

```sql
-- ตัวอย่างที่ 9: Subquery ใน ORDER BY (น้อยมากในทางปฏิบัติ)
SELECT customer_id, first_name, last_name
FROM   customers c
ORDER  BY (
    SELECT COUNT(*)
    FROM   orders o
    WHERE  o.customer_id = c.customer_id
) DESC;
```

---

## 41.4 Subquery vs JOIN

### ตัวอย่างที่ 10: แบบ JOIN

```sql
-- หาลูกค้าที่มีออเดอร์มากกว่า 1 ออเดอร์ -- แบบ JOIN
SELECT DISTINCT c.customer_id, c.first_name, c.last_name
FROM   customers c
JOIN   orders o ON o.customer_id = c.customer_id
GROUP  BY c.customer_id, c.first_name, c.last_name
HAVING COUNT(o.order_id) > 1;
```

### ตัวอย่างที่ 11: แบบ Subquery

```sql
-- หาลูกค้าที่มีออเดอร์มากกว่า 1 ออเดอร์ -- แบบ Subquery
SELECT customer_id, first_name, last_name
FROM   customers
WHERE  customer_id IN (
    SELECT customer_id
    FROM   orders
    GROUP  BY customer_id
    HAVING COUNT(*) > 1
);
```

---

## 41.5 ตัวอย่างพื้นฐาน 30+ ข้อ

### กลุ่มที่ 1: Scalar Subquery ใน WHERE

```sql
-- ตัวอย่างที่ 12: เงินเดือนสูงกว่าค่าเฉลี่ย
SELECT first_name, last_name, salary
FROM   employees
WHERE  salary > (SELECT AVG(salary) FROM employees)
ORDER  BY salary DESC;

-- ตัวอย่างที่ 13: ราคาสินค้าสูงกว่าราคาสูงสุดของ Stationery
SELECT product_name, category, price
FROM   products
WHERE  price > (
    SELECT MAX(price)
    FROM   products
    WHERE  category = 'Stationery'
);

-- ตัวอย่างที่ 14: ออเดอร์ที่มียอดสูงกว่าออเดอร์เฉลี่ย
SELECT order_id, customer_id, total_amount
FROM   orders
WHERE  total_amount > (SELECT AVG(total_amount) FROM orders)
ORDER  BY total_amount DESC;

-- ตัวอย่างที่ 15: สินค้าที่มีราคาต่ำกว่าราคาต่ำสุดของ Electronics
SELECT product_name, category, price
FROM   products
WHERE  price < (
    SELECT MIN(price)
    FROM   products
    WHERE  category = 'Electronics'
);
```

### กลุ่มที่ 2: Scalar Subquery ใน SELECT

```sql
-- ตัวอย่างที่ 16: แสดงเงินเดือนพร้อมความต่างจากค่าเฉลี่ย
SELECT
    first_name,
    last_name,
    salary,
    (SELECT AVG(salary) FROM employees)       AS company_avg,
    salary - (SELECT AVG(salary) FROM employees) AS diff_from_avg
FROM employees
ORDER BY salary DESC;

-- ตัวอย่างที่ 17: แสดงยอดออเดอร์ล่าสุดของแต่ละลูกค้า
SELECT
    customer_id,
    first_name,
    last_name,
    (SELECT MAX(order_date)
     FROM   orders
     WHERE  customer_id = c.customer_id) AS last_order_date
FROM customers c;

-- ตัวอย่างที่ 18: จำนวนออเดอร์ของแต่ละลูกค้า
SELECT
    c.customer_id,
    c.first_name,
    c.last_name,
    (SELECT COUNT(*)
     FROM   orders o
     WHERE  o.customer_id = c.customer_id) AS order_count
FROM customers c
ORDER BY order_count DESC;
```

### กลุ่มที่ 3: IN / NOT IN

```sql
-- ตัวอย่างที่ 19: ลูกค้าที่เคยสั่งซื้อ
SELECT first_name, last_name, email
FROM   customers
WHERE  customer_id IN (
    SELECT DISTINCT customer_id FROM orders
);

-- ตัวอย่างที่ 20: ลูกค้าที่ไม่เคยสั่งซื้อ
SELECT first_name, last_name, email
FROM   customers
WHERE  customer_id NOT IN (
    SELECT DISTINCT customer_id FROM orders
);

-- ตัวอย่างที่ 21: สินค้าที่เคยถูกสั่งซื้อ
SELECT product_name, category, price
FROM   products
WHERE  product_id IN (
    SELECT DISTINCT product_id FROM order_items
);

-- ตัวอย่างที่ 22: สินค้าที่ยังไม่เคยถูกสั่งซื้อ
SELECT product_name, category, price
FROM   products
WHERE  product_id NOT IN (
    SELECT DISTINCT product_id FROM order_items
);

-- ตัวอย่างที่ 23: พนักงานในแผนกที่มีงบประมาณสูงกว่า 3 ล้าน
SELECT first_name, last_name, department_id
FROM   employees
WHERE  department_id IN (
    SELECT department_id
    FROM   departments
    WHERE  budget > 3000000
);
```

### กลุ่มที่ 4: Subquery ใน FROM (Derived Table)

```sql
-- ตัวอย่างที่ 24: สรุปยอดขายต่อหมวดหมู่
SELECT
    category_sales.category,
    category_sales.total_revenue,
    category_sales.product_count
FROM (
    SELECT
        p.category,
        SUM(oi.quantity * oi.unit_price) AS total_revenue,
        COUNT(DISTINCT p.product_id)     AS product_count
    FROM   products p
    JOIN   order_items oi ON oi.product_id = p.product_id
    GROUP  BY p.category
) AS category_sales
ORDER BY total_revenue DESC;

-- ตัวอย่างที่ 25: ค่าเฉลี่ยของผลรวม (aggregation of aggregation)
SELECT AVG(customer_total) AS avg_customer_spend
FROM (
    SELECT customer_id, SUM(total_amount) AS customer_total
    FROM   orders
    GROUP  BY customer_id
) AS customer_totals;
```

### กลุ่มที่ 5: Subquery ใน HAVING

```sql
-- ตัวอย่างที่ 26: แผนกที่มีเงินเดือนเฉลี่ยสูงกว่าค่าเฉลี่ยทั้งบริษัท
SELECT
    d.department_name,
    AVG(e.salary) AS avg_salary
FROM   employees e
JOIN   departments d ON d.department_id = e.department_id
GROUP  BY d.department_id, d.department_name
HAVING AVG(e.salary) > (SELECT AVG(salary) FROM employees)
ORDER  BY avg_salary DESC;

-- ตัวอย่างที่ 27: สินค้าที่ขายได้มากกว่าค่าเฉลี่ยยอดขาย
SELECT
    p.product_name,
    SUM(oi.quantity) AS total_qty_sold
FROM   products p
JOIN   order_items oi ON oi.product_id = p.product_id
GROUP  BY p.product_id, p.product_name
HAVING SUM(oi.quantity) > (
    SELECT AVG(qty_per_product)
    FROM (
        SELECT product_id, SUM(quantity) AS qty_per_product
        FROM   order_items
        GROUP  BY product_id
    ) AS product_qty
);
```

### กลุ่มที่ 6: Correlated Subquery เบื้องต้น

```sql
-- ตัวอย่างที่ 28: พนักงานที่มีเงินเดือนสูงสุดในแผนกตัวเอง
SELECT first_name, last_name, salary, department_id
FROM   employees e1
WHERE  salary = (
    SELECT MAX(salary)
    FROM   employees e2
    WHERE  e2.department_id = e1.department_id
);

-- ตัวอย่างที่ 29: ออเดอร์ที่มียอดสูงกว่าค่าเฉลี่ยออเดอร์ของลูกค้าคนนั้น
SELECT order_id, customer_id, total_amount, order_date
FROM   orders o
WHERE  total_amount > (
    SELECT AVG(total_amount)
    FROM   orders
    WHERE  customer_id = o.customer_id
);

-- ตัวอย่างที่ 30: ลูกค้าที่มีออเดอร์ล่าสุดใน 90 วัน
SELECT first_name, last_name, email
FROM   customers c
WHERE  EXISTS (
    SELECT 1
    FROM   orders o
    WHERE  o.customer_id = c.customer_id
      AND  o.order_date >= DATE_SUB(CURDATE(), INTERVAL 90 DAY)
);
```

---

## 41.6 การอ่าน Subquery ให้เข้าใจ

### ตัวอย่างที่ 31: อ่านจากใน → นอก

```sql
-- อ่าน subquery ก่อน (ด้านใน) แล้วค่อยอ่าน outer query
SELECT product_name, price
FROM   products
WHERE  price > (
    -- STEP 1: หาราคาเฉลี่ยของ Electronics
    SELECT AVG(price)
    FROM   products
    WHERE  category = 'Electronics'
    -- ผลลัพธ์: 16697.14
)
-- STEP 2: ดึงสินค้าที่มีราคา > 16697.14
;
```

### ตัวอย่างที่ 32: Nested หลายชั้น

```sql
-- Subquery ซ้อน 2 ชั้น
SELECT first_name, last_name
FROM   customers
WHERE  customer_id IN (
    -- ชั้นที่ 1: ลูกค้าที่มีออเดอร์ที่มี product จาก Electronics
    SELECT DISTINCT o.customer_id
    FROM   orders o
    WHERE  o.order_id IN (
        -- ชั้นที่ 2: ออเดอร์ที่มีสินค้า Electronics
        SELECT DISTINCT oi.order_id
        FROM   order_items oi
        JOIN   products p ON p.product_id = oi.product_id
        WHERE  p.category = 'Electronics'
    )
);
```

---

## 41.7 เมื่อไหร่ควรใช้ Subquery vs JOIN

### ตัวอย่างที่ 33: กรณีที่ Subquery อ่านง่ายกว่า

```sql
-- คำถาม: ลูกค้าที่ไม่เคยสั่งซื้อเลย
-- แบบ Subquery -- อ่านง่ายกว่า
SELECT first_name, last_name
FROM   customers
WHERE  customer_id NOT IN (SELECT customer_id FROM orders);

-- แบบ LEFT JOIN -- อ่านยากกว่าเล็กน้อย
SELECT c.first_name, c.last_name
FROM   customers c
LEFT   JOIN orders o ON o.customer_id = c.customer_id
WHERE  o.order_id IS NULL;
```

### ตัวอย่างที่ 34: กรณีที่ JOIN เหมาะกว่า

```sql
-- คำถาม: แสดงชื่อลูกค้าพร้อมจำนวนออเดอร์
-- แบบ JOIN -- มีประสิทธิภาพกว่า
SELECT c.first_name, c.last_name, COUNT(o.order_id) AS order_count
FROM   customers c
LEFT   JOIN orders o ON o.customer_id = c.customer_id
GROUP  BY c.customer_id, c.first_name, c.last_name;

-- แบบ Correlated Subquery -- ช้ากว่า (รันซ้ำทุกแถว)
SELECT
    first_name,
    last_name,
    (SELECT COUNT(*) FROM orders WHERE customer_id = c.customer_id) AS order_count
FROM customers c;
```

### ตัวอย่างที่ 35: Subquery เหมาะกับการแบ่ง logic

```sql
-- ตรวจสอบสินค้าที่ขายดีที่สุดในแต่ละหมวดหมู่
SELECT p.product_name, p.category, p.price
FROM   products p
WHERE  p.product_id IN (
    SELECT product_id
    FROM   order_items
    GROUP  BY product_id
    HAVING SUM(quantity) >= (
        -- เงื่อนไขซับซ้อนที่อ่านเข้าใจง่ายขึ้น
        SELECT AVG(total_qty)
        FROM (
            SELECT product_id, SUM(quantity) AS total_qty
            FROM   order_items
            GROUP  BY product_id
        ) AS t
    )
);
```

---

## 41.8 ข้อผิดพลาดที่พบบ่อย

### ตัวอย่างที่ 36: Subquery คืนหลายแถว (ERROR)

```sql
-- ผิด! Subquery คืนหลายแถว ใช้กับ = ไม่ได้
SELECT first_name, salary
FROM   employees
WHERE  salary = (
    SELECT salary FROM employees WHERE department_id = 1
    -- คืนหลายแถว!
);

-- ถูก: ใช้ IN แทน
SELECT first_name, salary
FROM   employees
WHERE  salary IN (
    SELECT salary FROM employees WHERE department_id = 1
);

-- หรือใช้ aggregate
SELECT first_name, salary
FROM   employees
WHERE  salary = (
    SELECT MAX(salary) FROM employees WHERE department_id = 1
);
```

### ตัวอย่างที่ 37: ลืมตั้งชื่อ Derived Table (ERROR)

```sql
-- ผิด! Derived table ต้องมีชื่อ (alias)
SELECT *
FROM (
    SELECT department_id, AVG(salary)
    FROM   employees
    GROUP  BY department_id
);
-- ERROR: Every derived table must have its own alias

-- ถูก: ใส่ alias
SELECT *
FROM (
    SELECT department_id, AVG(salary) AS avg_sal
    FROM   employees
    GROUP  BY department_id
) AS dept_avg;
```

### ตัวอย่างที่ 38: Subquery ที่ไม่มีข้อมูล (NULL)

```sql
-- ถ้า subquery ไม่คืนข้อมูล
SELECT first_name, salary
FROM   employees
WHERE  salary > (
    SELECT MAX(salary)
    FROM   employees
    WHERE  department_id = 999  -- department ไม่มีอยู่จริง
);
-- ผลลัพธ์: ไม่มีแถว (NULL > NULL = FALSE)
-- ไม่ใช่ error แต่ไม่คืนผลลัพธ์

-- แก้ไขด้วย COALESCE
SELECT first_name, salary
FROM   employees
WHERE  salary > COALESCE(
    (SELECT MAX(salary) FROM employees WHERE department_id = 999),
    0
);
```

---

## 41.9 Subquery ใน MySQL, PostgreSQL, SQL Server

```sql
-- MySQL / MariaDB
SELECT *
FROM   employees
WHERE  salary > (SELECT AVG(salary) FROM employees);

-- PostgreSQL -- เหมือนกัน
SELECT *
FROM   employees
WHERE  salary > (SELECT AVG(salary) FROM employees);

-- SQL Server -- เหมือนกัน
SELECT *
FROM   employees
WHERE  salary > (SELECT AVG(salary) FROM employees);

-- ตัวอย่างที่ 39: MySQL ต้องตั้งชื่อ subquery ใน FROM เสมอ
-- MySQL
SELECT *
FROM (SELECT * FROM employees WHERE salary > 80000) AS high_earners;

-- PostgreSQL ไม่บังคับ alias แต่แนะนำ
SELECT *
FROM (SELECT * FROM employees WHERE salary > 80000);
```

---

## 41.10 สรุปตาราง: Subquery Types

```
┌─────────────────┬──────────────────────────────┬──────────────────────┐
│ ประเภท          │ คืนค่า                       │ ใช้กับ              │
├─────────────────┼──────────────────────────────┼──────────────────────┤
│ Scalar          │ 1 แถว, 1 คอลัมน์             │ =, >, <, >=, <=, <> │
│ Row             │ 1 แถว, หลายคอลัมน์           │ =, <>, IN            │
│ Table/Column    │ หลายแถว, 1 คอลัมน์           │ IN, NOT IN, ANY, ALL │
│ Table           │ หลายแถว, หลายคอลัมน์         │ FROM clause          │
│ Correlated      │ อ้างอิง outer query           │ WHERE, SELECT, HAVING│
└─────────────────┴──────────────────────────────┴──────────────────────┘
```

### ตัวอย่างที่ 40: สรุปการใช้งานทุกแบบ

```sql
-- Scalar in WHERE
SELECT * FROM employees
WHERE  salary > (SELECT AVG(salary) FROM employees);

-- Scalar in SELECT
SELECT first_name,
       (SELECT department_name FROM departments WHERE department_id = e.department_id) AS dept
FROM   employees e;

-- Table subquery in FROM
SELECT dept_name, avg_sal
FROM (
    SELECT d.department_name AS dept_name, AVG(e.salary) AS avg_sal
    FROM   employees e JOIN departments d ON d.department_id = e.department_id
    GROUP  BY d.department_id, d.department_name
) AS dept_stats
WHERE avg_sal > 80000;

-- Multi-row in WHERE with IN
SELECT * FROM products
WHERE category IN (SELECT DISTINCT category FROM products WHERE price > 10000);

-- Correlated subquery
SELECT * FROM employees e
WHERE salary = (SELECT MAX(salary) FROM employees WHERE department_id = e.department_id);
```

---

## แบบฝึกหัดบทที่ 41

**ข้อ 1:** เขียน subquery ค้นหาพนักงานที่มีเงินเดือนสูงกว่าเงินเดือนเฉลี่ยของแผนก Engineering

```sql
-- เฉลย:
SELECT first_name, last_name, salary
FROM   employees
WHERE  salary > (
    SELECT AVG(salary)
    FROM   employees e
    JOIN   departments d ON d.department_id = e.department_id
    WHERE  d.department_name = 'Engineering'
);
```

**ข้อ 2:** ใช้ subquery ใน SELECT แสดงรายชื่อสินค้าพร้อมจำนวนครั้งที่ถูกสั่งซื้อ

```sql
-- เฉลย:
SELECT
    product_id,
    product_name,
    price,
    (SELECT COUNT(*)
     FROM   order_items oi
     WHERE  oi.product_id = p.product_id) AS times_ordered
FROM products p
ORDER BY times_ordered DESC;
```

**ข้อ 3:** หาลูกค้าที่อยู่ในเมือง Bangkok และเคยสั่งซื้อสินค้าประเภท Electronics

```sql
-- เฉลย:
SELECT first_name, last_name, city
FROM   customers
WHERE  city = 'Bangkok'
  AND  customer_id IN (
    SELECT DISTINCT o.customer_id
    FROM   orders o
    JOIN   order_items oi ON oi.order_id = o.order_id
    JOIN   products p     ON p.product_id = oi.product_id
    WHERE  p.category = 'Electronics'
);
```

**ข้อ 4:** ใช้ derived table หาหมวดหมู่สินค้าที่มียอดขายรวมสูงกว่า 50,000 บาท

```sql
-- เฉลย:
SELECT category, total_revenue
FROM (
    SELECT p.category,
           SUM(oi.quantity * oi.unit_price) AS total_revenue
    FROM   products p
    JOIN   order_items oi ON oi.product_id = p.product_id
    GROUP  BY p.category
) AS cat_rev
WHERE total_revenue > 50000
ORDER BY total_revenue DESC;
```

**ข้อ 5:** หาพนักงานที่มีเงินเดือนสูงสุดในแต่ละแผนก (correlated subquery)

```sql
-- เฉลย:
SELECT first_name, last_name, salary, department_id
FROM   employees e
WHERE  salary = (
    SELECT MAX(salary)
    FROM   employees
    WHERE  department_id = e.department_id
)
ORDER BY department_id;
```

**ข้อ 6:** แสดงออเดอร์ทั้งหมดพร้อมจำนวนรายการสินค้าในแต่ละออเดอร์

```sql
-- เฉลย:
SELECT
    o.order_id,
    o.order_date,
    o.status,
    o.total_amount,
    (SELECT COUNT(*)
     FROM   order_items oi
     WHERE  oi.order_id = o.order_id) AS item_count
FROM orders o
ORDER BY order_date;
```

**ข้อ 7:** หาสินค้าที่มีราคาสูงกว่าราคาเฉลี่ยของหมวดหมู่เดียวกัน

```sql
-- เฉลย:
SELECT product_name, category, price
FROM   products p
WHERE  price > (
    SELECT AVG(price)
    FROM   products
    WHERE  category = p.category
)
ORDER BY category, price;
```

**ข้อ 8:** ใช้ HAVING กับ subquery หาลูกค้าที่มียอดสั่งซื้อรวมสูงกว่าค่าเฉลี่ยยอดสั่งซื้อของลูกค้าทุกคน

```sql
-- เฉลย:
SELECT customer_id, SUM(total_amount) AS total_spent
FROM   orders
GROUP  BY customer_id
HAVING SUM(total_amount) > (
    SELECT AVG(customer_total)
    FROM (
        SELECT customer_id, SUM(total_amount) AS customer_total
        FROM   orders
        GROUP  BY customer_id
    ) AS t
);
```

**ข้อ 9:** หาแผนกที่มีจำนวนพนักงานมากกว่าค่าเฉลี่ยจำนวนพนักงานต่อแผนก

```sql
-- เฉลย:
SELECT d.department_name, COUNT(e.employee_id) AS emp_count
FROM   departments d
JOIN   employees e ON e.department_id = d.department_id
GROUP  BY d.department_id, d.department_name
HAVING COUNT(e.employee_id) > (
    SELECT AVG(emp_per_dept)
    FROM (
        SELECT department_id, COUNT(*) AS emp_per_dept
        FROM   employees
        GROUP  BY department_id
    ) AS dept_counts
);
```

**ข้อ 10:** เขียน query แสดงชื่อลูกค้า, จำนวนออเดอร์, ยอดรวมทั้งหมด, และออเดอร์ล่าสุด โดยใช้ subquery

```sql
-- เฉลย:
SELECT
    c.first_name,
    c.last_name,
    (SELECT COUNT(*)
     FROM   orders
     WHERE  customer_id = c.customer_id) AS order_count,
    (SELECT COALESCE(SUM(total_amount), 0)
     FROM   orders
     WHERE  customer_id = c.customer_id) AS total_spent,
    (SELECT MAX(order_date)
     FROM   orders
     WHERE  customer_id = c.customer_id) AS last_order_date
FROM customers c
ORDER BY total_spent DESC;
```

---

*จบบทที่ 41: Introduction to Subqueries*
*บทถัดไป: Part 42 - Scalar Subqueries in SELECT and WHERE*
