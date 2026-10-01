# Part 031: Aggregate Functions - COUNT, SUM, AVG, MIN, MAX

## บทนำ (Introduction)

Aggregate Functions หรือฟังก์ชันการรวมข้อมูล เป็นหนึ่งในเครื่องมือที่สำคัญที่สุดใน SQL สำหรับการวิเคราะห์ข้อมูล ฟังก์ชันเหล่านี้ช่วยให้เราสามารถ:

- **นับจำนวน** รายการในชุดข้อมูล
- **รวมค่า** ตัวเลขทั้งหมด
- **คำนวณค่าเฉลี่ย** ของกลุ่มข้อมูล
- **หาค่าต่ำสุด/สูงสุด** ในกลุ่มข้อมูล

ในบทนี้เราจะเรียนรู้ฟังก์ชัน Aggregate ทั้ง 5 ตัวหลัก: `COUNT`, `SUM`, `AVG`, `MIN`, `MAX` พร้อมกับตัวอย่างการใช้งานจริงกว่า 50 ตัวอย่าง

---

## การตั้งค่าฐานข้อมูล (Database Setup)

```sql
-- สร้างฐานข้อมูลและตาราง
CREATE DATABASE IF NOT EXISTS ecommerce_db;
USE ecommerce_db;

-- ตาราง departments
CREATE TABLE IF NOT EXISTS departments (
    department_id   INT PRIMARY KEY AUTO_INCREMENT,
    department_name VARCHAR(100) NOT NULL,
    location        VARCHAR(100),
    budget          DECIMAL(15,2),
    created_at      DATE
);

-- ตาราง employees
CREATE TABLE IF NOT EXISTS employees (
    employee_id    INT PRIMARY KEY AUTO_INCREMENT,
    first_name     VARCHAR(50) NOT NULL,
    last_name      VARCHAR(50) NOT NULL,
    email          VARCHAR(100) UNIQUE,
    phone          VARCHAR(20),
    hire_date      DATE NOT NULL,
    job_title      VARCHAR(100),
    salary         DECIMAL(10,2),
    department_id  INT,
    manager_id     INT,
    FOREIGN KEY (department_id) REFERENCES departments(department_id)
);

-- ตาราง customers
CREATE TABLE IF NOT EXISTS customers (
    customer_id   INT PRIMARY KEY AUTO_INCREMENT,
    first_name    VARCHAR(50) NOT NULL,
    last_name     VARCHAR(50) NOT NULL,
    email         VARCHAR(100) UNIQUE,
    phone         VARCHAR(20),
    address       TEXT,
    city          VARCHAR(50),
    province      VARCHAR(50),
    postal_code   VARCHAR(10),
    registered_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    loyalty_points INT DEFAULT 0
);

-- ตาราง products
CREATE TABLE IF NOT EXISTS products (
    product_id    INT PRIMARY KEY AUTO_INCREMENT,
    product_name  VARCHAR(200) NOT NULL,
    category      VARCHAR(50),
    brand         VARCHAR(50),
    price         DECIMAL(10,2) NOT NULL,
    cost          DECIMAL(10,2),
    stock_qty     INT DEFAULT 0,
    min_stock     INT DEFAULT 10,
    is_active     BOOLEAN DEFAULT TRUE,
    created_at    DATE
);

-- ตาราง orders
CREATE TABLE IF NOT EXISTS orders (
    order_id      INT PRIMARY KEY AUTO_INCREMENT,
    customer_id   INT NOT NULL,
    order_date    DATETIME DEFAULT CURRENT_TIMESTAMP,
    status        ENUM('pending','processing','shipped','delivered','cancelled') DEFAULT 'pending',
    shipping_fee  DECIMAL(10,2) DEFAULT 0,
    discount_amt  DECIMAL(10,2) DEFAULT 0,
    total_amount  DECIMAL(10,2),
    payment_method VARCHAR(50),
    notes         TEXT,
    FOREIGN KEY (customer_id) REFERENCES customers(customer_id)
);

-- ตาราง order_items
CREATE TABLE IF NOT EXISTS order_items (
    item_id       INT PRIMARY KEY AUTO_INCREMENT,
    order_id      INT NOT NULL,
    product_id    INT NOT NULL,
    quantity      INT NOT NULL,
    unit_price    DECIMAL(10,2) NOT NULL,
    discount_pct  DECIMAL(5,2) DEFAULT 0,
    FOREIGN KEY (order_id) REFERENCES orders(order_id),
    FOREIGN KEY (product_id) REFERENCES products(product_id)
);
```

### ข้อมูลตัวอย่าง (Sample Data)

```sql
-- เพิ่มข้อมูล departments (10 แผนก)
INSERT INTO departments (department_name, location, budget, created_at) VALUES
('ฝ่ายขาย',           'กรุงเทพฯ ชั้น 3',  5000000.00, '2018-01-01'),
('ฝ่ายการตลาด',       'กรุงเทพฯ ชั้น 4',  3000000.00, '2018-01-01'),
('ฝ่ายไอที',          'กรุงเทพฯ ชั้น 5',  4000000.00, '2018-03-01'),
('ฝ่ายบัญชีการเงิน',  'กรุงเทพฯ ชั้น 2',  2500000.00, '2018-01-01'),
('ฝ่ายคลังสินค้า',    'สมุทรปราการ',       3500000.00, '2018-06-01'),
('ฝ่ายทรัพยากรบุคคล', 'กรุงเทพฯ ชั้น 2',  1500000.00, '2018-01-01'),
('ฝ่ายลูกค้าสัมพันธ์','กรุงเทพฯ ชั้น 1',  2000000.00, '2019-01-01'),
('ฝ่ายจัดซื้อ',       'กรุงเทพฯ ชั้น 3',  1800000.00, '2018-09-01'),
('ฝ่ายวิจัยพัฒนา',    'กรุงเทพฯ ชั้น 6',  6000000.00, '2019-06-01'),
('ฝ่ายกฎหมาย',        'กรุงเทพฯ ชั้น 2',  1200000.00, '2020-01-01');

-- เพิ่มข้อมูล employees (20 พนักงาน)
INSERT INTO employees (first_name, last_name, email, phone, hire_date, job_title, salary, department_id, manager_id) VALUES
('สมชาย',   'ใจดี',      'somchai@email.com',   '081-111-1111', '2018-03-01', 'ผู้จัดการฝ่ายขาย',      85000, 1, NULL),
('สมหญิง',  'รักงาน',    'somying@email.com',   '081-222-2222', '2018-04-01', 'เจ้าหน้าที่ขาย',        45000, 1, 1),
('วิชาญ',   'เก่งมาก',   'wichan@email.com',    '081-333-3333', '2019-01-15', 'เจ้าหน้าที่ขาย',        42000, 1, 1),
('นภา',     'สวยใส',     'napa@email.com',      '081-444-4444', '2019-06-01', 'เจ้าหน้าที่ขาย',        40000, 1, 1),
('ประยุทธ', 'ทำงานดี',   'prayuth@email.com',   '081-555-5555', '2018-05-01', 'ผู้จัดการฝ่ายการตลาด',  90000, 2, NULL),
('มาลี',    'หัวใจดี',   'malee@email.com',     '081-666-6666', '2019-02-01', 'นักการตลาด',             48000, 2, 5),
('อนุชา',   'ขยันมาก',   'anucha@email.com',    '081-777-7777', '2019-08-01', 'นักการตลาดดิจิทัล',     52000, 2, 5),
('ชลธิชา',  'ฉลาดเฉียบ', 'chon@email.com',      '081-888-8888', '2018-07-01', 'ผู้จัดการฝ่ายไอที',     95000, 3, NULL),
('ธนากร',   'โปรแกรมเมอร์','thanakorn@email.com','081-999-9999', '2019-03-01', 'นักพัฒนาซอฟต์แวร์',    65000, 3, 8),
('พิมพ์ใจ', 'คิดเร็ว',   'pimjai@email.com',    '082-111-1111', '2020-01-01', 'นักพัฒนาซอฟต์แวร์',    60000, 3, 8),
('รุ่งโรจน์','เฉลียวฉลาด','rung@email.com',      '082-222-2222', '2020-06-01', 'วิศวกรระบบ',            70000, 3, 8),
('สุดา',    'ใส่ใจงาน',  'suda@email.com',      '082-333-3333', '2018-08-01', 'ผู้จัดการฝ่ายบัญชี',    88000, 4, NULL),
('กมลา',    'ตั้งใจ',    'kamala@email.com',    '082-444-4444', '2019-04-01', 'นักบัญชี',               43000, 4, 12),
('วิบูลย์', 'รอบคอบ',    'wiboon@email.com',    '082-555-5555', '2019-09-01', 'นักบัญชี',               42000, 4, 12),
('ปรีชา',   'มีวินัย',   'preecha@email.com',   '082-666-6666', '2018-11-01', 'ผู้จัดการคลังสินค้า',   75000, 5, NULL),
('อรอุมา',  'ขยันทำงาน', 'orn@email.com',       '082-777-7777', '2020-02-01', 'เจ้าหน้าที่คลังสินค้า', 35000, 5, 15),
('สุภัทร',  'ตรงเวลา',   'supat@email.com',     '082-888-8888', '2020-07-01', 'เจ้าหน้าที่คลังสินค้า', 35000, 5, 15),
('วันชัย',  'หัวไว',     'wanchai@email.com',   '082-999-9999', '2018-02-01', 'ผู้จัดการฝ่าย HR',      82000, 6, NULL),
('นุชนาถ',  'ใจเย็น',    'nuch@email.com',      '083-111-1111', '2019-05-01', 'เจ้าหน้าที่ HR',        40000, 6, 18),
('ศิริลักษณ์','ทำงานเก่ง','siri@email.com',      '083-222-2222', '2021-01-01', 'เจ้าหน้าที่ HR',        NULL,  6, 18);

-- เพิ่มข้อมูล customers (15 ลูกค้า)
INSERT INTO customers (first_name, last_name, email, phone, city, province, registered_at, loyalty_points) VALUES
('อลิสา',   'รักการซื้อ', 'alisa@gmail.com',   '090-001-0001', 'กรุงเทพฯ',   'กรุงเทพฯ',  '2020-01-15 10:00:00', 1500),
('บอย',     'ชอบช้อปปิ้ง','boy@gmail.com',     '090-002-0002', 'เชียงใหม่',  'เชียงใหม่', '2020-03-20 11:00:00', 800),
('ซินดี้',  'แฟชั่นนิสต้า','cindy@gmail.com',  '090-003-0003', 'กรุงเทพฯ',   'กรุงเทพฯ',  '2020-06-10 09:00:00', 2200),
('เดวิด',   'ซื้อบ่อย',  'david@gmail.com',   '090-004-0004', 'ขอนแก่น',    'ขอนแก่น',   '2021-01-05 14:00:00', 300),
('อีฟ',     'สาวประหยัด','eve@gmail.com',      '090-005-0005', 'ภูเก็ต',     'ภูเก็ต',    '2021-03-12 15:00:00', 650),
('ฟ้า',     'สั่งเยอะ',  'fah@gmail.com',     '090-006-0006', 'กรุงเทพฯ',   'กรุงเทพฯ',  '2021-05-18 16:00:00', 4500),
('โก',      'ลูกค้าใหม่', 'go@gmail.com',      '090-007-0007', 'นนทบุรี',    'นนทบุรี',   '2021-08-22 10:00:00', 100),
('ฮีโร่',   'นักกีฬา',   'hero@gmail.com',    '090-008-0008', 'สมุทรปราการ','สมุทรปราการ','2021-09-30 11:00:00', 900),
('ไอซ์',    'หนาวเย็น',  'ice@gmail.com',     '090-009-0009', 'เชียงราย',   'เชียงราย',  '2022-01-10 09:00:00', 200),
('เจ',      'เจ้าของร้าน','jay@gmail.com',     '090-010-0010', 'กรุงเทพฯ',   'กรุงเทพฯ',  '2022-02-14 13:00:00', 3200),
('กวาง',    'ดาราสาว',   'kwang@gmail.com',   '090-011-0011', 'กรุงเทพฯ',   'กรุงเทพฯ',  '2022-04-01 10:00:00', 7800),
('ลีน',     'นักเรียน',  'lin@gmail.com',     '090-012-0012', 'ปทุมธานี',   'ปทุมธานี',  '2022-06-15 14:00:00', 150),
('มาร์ค',   'โปรแกรมเมอร์','mark@gmail.com',  '090-013-0013', 'กรุงเทพฯ',   'กรุงเทพฯ',  '2022-08-20 11:00:00', 600),
('นก',      'พนักงานออฟฟิศ','nok@gmail.com',   '090-014-0014', 'นครปฐม',    'นครปฐม',    '2022-10-05 16:00:00', 450),
('โอ',      'ช่างภาพ',   'oh@gmail.com',      '090-015-0015', 'กรุงเทพฯ',   'กรุงเทพฯ',  '2023-01-20 09:00:00', 1100);

-- เพิ่มข้อมูล products (20 สินค้า)
INSERT INTO products (product_name, category, brand, price, cost, stock_qty, min_stock, is_active, created_at) VALUES
('iPhone 15 Pro',        'สมาร์ทโฟน',  'Apple',    45900, 35000, 50,  10, TRUE,  '2023-09-15'),
('Samsung Galaxy S24',   'สมาร์ทโฟน',  'Samsung',  35900, 27000, 75,  10, TRUE,  '2024-01-17'),
('MacBook Air M3',       'โน้ตบุ๊ค',    'Apple',    42900, 33000, 30,   5, TRUE,  '2024-03-08'),
('Dell XPS 15',          'โน้ตบุ๊ค',    'Dell',     38900, 29000, 25,   5, TRUE,  '2023-10-01'),
('iPad Pro 12.9',        'แท็บเล็ต',   'Apple',    38900, 29000, 40,   8, TRUE,  '2024-05-07'),
('AirPods Pro 2',        'หูฟัง',      'Apple',     8990,  5500, 100, 20, TRUE,  '2022-09-23'),
('Sony WH-1000XM5',      'หูฟัง',      'Sony',      9990,  6500, 60,  15, TRUE,  '2022-05-12'),
('LG OLED 65"',          'ทีวี',       'LG',        59900, 45000, 15,   3, TRUE,  '2023-03-15'),
('Samsung 4K 55"',       'ทีวี',       'Samsung',   29900, 22000, 20,   5, TRUE,  '2023-01-10'),
('Dyson V15',            'เครื่องดูดฝุ่น','Dyson',  24900, 18000, 35,  10, TRUE,  '2023-06-01'),
('เครื่องชงกาแฟ Breville','เครื่องใช้ไฟฟ้า','Breville',15900, 11000, 45, 10, TRUE, '2022-11-15'),
('Nintendo Switch OLED', 'เกมคอนโซล',  'Nintendo', 14900,  9500, 80,  15, TRUE,  '2021-10-08'),
('PlayStation 5',        'เกมคอนโซล',  'Sony',     19900, 15000, 20,   5, TRUE,  '2020-11-12'),
('Kindle Paperwhite',    'อีบุ๊ค',      'Amazon',    5490,  3200, 55,  10, TRUE,  '2022-10-19'),
('GoPro Hero 12',        'กล้อง',      'GoPro',    16900, 12000, 30,   8, TRUE,  '2023-09-06'),
('Canon EOS R50',        'กล้อง',      'Canon',    28900, 21000, 25,   5, TRUE,  '2022-10-25'),
('Microsoft Surface Pro','แท็บเล็ต',   'Microsoft',36900, 27000, 15,   3, TRUE,  '2023-06-20'),
('Garmin Fenix 7',       'สมาร์ทวอทช์','Garmin',   21900, 16000, 40,  10, TRUE,  '2022-01-18'),
('Apple Watch Series 9', 'สมาร์ทวอทช์','Apple',    14900, 10000, 65,  15, TRUE,  '2023-09-22'),
('Fitbit Charge 6',      'สมาร์ทวอทช์','Fitbit',    6990,  4500, 90,  20, FALSE, '2023-10-04');

-- เพิ่มข้อมูล orders (25 ออเดอร์)
INSERT INTO orders (customer_id, order_date, status, shipping_fee, discount_amt, total_amount, payment_method) VALUES
(1,  '2024-01-05 10:30:00', 'delivered',   50,    0,    46000, 'บัตรเครดิต'),
(3,  '2024-01-08 14:15:00', 'delivered',   50, 1000,    17980, 'PromptPay'),
(6,  '2024-01-12 09:45:00', 'delivered',    0,  500,    45450, 'บัตรเครดิต'),
(2,  '2024-01-15 16:20:00', 'delivered',   50,    0,    36000, 'บัตรเครดิต'),
(10, '2024-01-20 11:00:00', 'delivered',   50, 2000,    22950, 'บัตรเดบิต'),
(11, '2024-01-25 13:30:00', 'shipped',      0,    0,    43000, 'บัตรเครดิต'),
(1,  '2024-02-03 10:00:00', 'delivered',   50,    0,     9090, 'บัตรเครดิต'),
(5,  '2024-02-08 15:45:00', 'delivered',   50,  500,    25400, 'PromptPay'),
(7,  '2024-02-14 09:00:00', 'cancelled',   50,    0,    15000, 'บัตรเครดิต'),
(3,  '2024-02-18 14:30:00', 'delivered',    0, 1500,    38450, 'บัตรเครดิต'),
(8,  '2024-02-22 11:15:00', 'processing',  50,    0,    10090, 'บัตรเดบิต'),
(11, '2024-03-01 10:00:00', 'delivered',    0,    0,    79900, 'บัตรเครดิต'),
(4,  '2024-03-05 13:00:00', 'delivered',   50,    0,    15050, 'PromptPay'),
(6,  '2024-03-10 16:45:00', 'shipped',     50, 1000,    43950, 'บัตรเครดิต'),
(9,  '2024-03-15 09:30:00', 'pending',     50,    0,     5540, 'บัตรเดบิต'),
(12, '2024-03-20 14:00:00', 'delivered',   50,    0,     9040, 'PromptPay'),
(13, '2024-03-25 11:30:00', 'delivered',   50,  500,    29450, 'บัตรเครดิต'),
(1,  '2024-04-01 10:15:00', 'delivered',    0,    0,    14950, 'บัตรเครดิต'),
(14, '2024-04-05 15:00:00', 'processing',  50,    0,    21950, 'PromptPay'),
(11, '2024-04-10 09:45:00', 'delivered',    0, 2000,    57900, 'บัตรเครดิต'),
(2,  '2024-04-15 14:30:00', 'delivered',   50,    0,    17990, 'บัตรเดบิต'),
(6,  '2024-04-20 11:00:00', 'shipped',     50, 1500,    24450, 'บัตรเครดิต'),
(15, '2024-04-25 16:15:00', 'pending',     50,    0,     7040, 'PromptPay'),
(10, '2024-05-01 10:30:00', 'delivered',    0, 1000,    37950, 'บัตรเครดิต'),
(3,  '2024-05-05 13:45:00', 'processing',  50,    0,    46000, 'บัตรเครดิต');

-- เพิ่มข้อมูล order_items (50 รายการ)
INSERT INTO order_items (order_id, product_id, quantity, unit_price, discount_pct) VALUES
(1,  1,  1, 45900, 0),
(2,  6,  1,  8990, 0),
(2,  7,  1,  9990, 0),
(3,  2,  1, 35900, 0),
(3,  6,  1,  8990, 5),
(4,  2,  1, 35900, 0),
(5,  10, 1, 24900, 0),
(6,  3,  1, 42900, 0),
(7,  6,  1,  8990, 0),
(8,  10, 1, 24900, 0),
(9,  12, 1, 14900, 0),
(10, 3,  1, 42900, 0),
(11, 6,  1,  8990, 0),
(11, 14, 1,  5490, 0),
(12, 8,  1, 59900, 0),
(12, 9,  1, 29900, 0),
(13, 12, 1, 14900, 0),
(14, 5,  1, 38900, 0),
(14, 6,  1,  8990, 0),
(15, 14, 1,  5490, 0),
(16, 7,  1,  9990, 0),
(17, 4,  1, 38900, 0),
(18, 12, 1, 14900, 0),
(19, 10, 1, 24900, 0),
(20, 8,  1, 59900, 0),
(21, 2,  1, 35900, 0),
(21, 7,  1,  9990, 0),
(22, 10, 1, 24900, 0),
(23, 14, 1,  5490, 0),
(23, 19, 1, 14900, 0),
(24, 3,  1, 42900, 0),
(25, 1,  1, 45900, 0),
(1,  6,  1,  8990, 0),
(4,  6,  1,  8990, 0),
(5,  19, 1, 14900, 0),
(6,  19, 1, 14900, 0),
(8,  9,  1, 29900, 0),
(10, 6,  1,  8990, 0),
(12, 12, 1, 14900, 0),
(13, 14, 1,  5490, 0),
(14, 17, 0, 36900, 0),
(16, 14, 1,  5490, 0),
(17, 18, 1, 21900, 0),
(20, 4,  1, 38900, 0),
(21, 16, 0, 28900, 0),
(22, 7,  1,  9990, 0),
(23, 6,  1,  8990, 0),
(24, 17, 0, 36900, 0),
(24, 18, 0, 21900, 0),
(25, 3,  1, 42900, 0);
```

---

## 1. COUNT Function - นับจำนวนแถว

### 1.1 COUNT(*) - นับทุกแถวรวม NULL

`COUNT(*)` นับจำนวนแถวทั้งหมดในตาราง รวมถึงแถวที่มี NULL ด้วย

```sql
-- ตัวอย่างที่ 1: นับจำนวนพนักงานทั้งหมด
SELECT COUNT(*) AS total_employees
FROM employees;

-- ผลลัพธ์:
-- total_employees
-- ---------------
-- 20
```

```sql
-- ตัวอย่างที่ 2: นับจำนวนสินค้าทั้งหมด
SELECT COUNT(*) AS total_products
FROM products;

-- ผลลัพธ์:
-- total_products
-- --------------
-- 20
```

```sql
-- ตัวอย่างที่ 3: นับจำนวนออเดอร์ทั้งหมด
SELECT COUNT(*) AS total_orders
FROM orders;

-- ผลลัพธ์:
-- total_orders
-- ------------
-- 25
```

### 1.2 COUNT(column) - นับเฉพาะค่าที่ไม่ใช่ NULL

`COUNT(column)` จะข้ามแถวที่มีค่า NULL ในคอลัมน์นั้น

```sql
-- ตัวอย่างที่ 4: ความแตกต่างระหว่าง COUNT(*) และ COUNT(column)
-- สังเกตว่า ศิริลักษณ์ มี salary = NULL
SELECT 
    COUNT(*)           AS total_rows,
    COUNT(salary)      AS has_salary,
    COUNT(*) - COUNT(salary) AS null_salary_count
FROM employees;

-- ผลลัพธ์:
-- total_rows | has_salary | null_salary_count
-- -----------+------------+------------------
-- 20         | 19         | 1
```

```sql
-- ตัวอย่างที่ 5: นับพนักงานที่มีข้อมูลเบอร์โทรศัพท์
SELECT 
    COUNT(*)     AS total_employees,
    COUNT(phone) AS employees_with_phone
FROM employees;
```

```sql
-- ตัวอย่างที่ 6: นับออเดอร์ที่มีหมายเหตุ (notes)
SELECT 
    COUNT(*)     AS total_orders,
    COUNT(notes) AS orders_with_notes
FROM orders;
```

### 1.3 COUNT(DISTINCT column) - นับค่าที่ไม่ซ้ำ

```sql
-- ตัวอย่างที่ 7: นับจำนวนแผนกที่มีพนักงาน
SELECT COUNT(DISTINCT department_id) AS departments_with_employees
FROM employees;

-- ผลลัพธ์:
-- departments_with_employees
-- --------------------------
-- 6
```

```sql
-- ตัวอย่างที่ 8: นับจำนวนลูกค้าที่สั่งซื้อ
SELECT COUNT(DISTINCT customer_id) AS unique_customers
FROM orders;

-- ผลลัพธ์:
-- unique_customers
-- ----------------
-- 14
```

```sql
-- ตัวอย่างที่ 9: นับจำนวนสินค้าที่มีการสั่งซื้อ
SELECT COUNT(DISTINCT product_id) AS products_ordered
FROM order_items;
```

```sql
-- ตัวอย่างที่ 10: เปรียบเทียบ COUNT(*), COUNT(col), COUNT(DISTINCT col)
SELECT 
    COUNT(*)                    AS total_orders,
    COUNT(DISTINCT customer_id) AS unique_customers,
    COUNT(DISTINCT status)      AS distinct_statuses,
    COUNT(DISTINCT payment_method) AS payment_methods
FROM orders;
```

---

## 2. SUM Function - รวมค่าตัวเลข

### 2.1 SUM พื้นฐาน

```sql
-- ตัวอย่างที่ 11: รวมยอดขายทั้งหมด
SELECT SUM(total_amount) AS total_revenue
FROM orders;

-- ผลลัพธ์:
-- total_revenue
-- -------------
-- 761930.00
```

```sql
-- ตัวอย่างที่ 12: รวมเงินเดือนพนักงานทั้งหมด
SELECT SUM(salary) AS total_salary_expense
FROM employees;
```

```sql
-- ตัวอย่างที่ 13: รวมค่าจัดส่งทั้งหมด
SELECT 
    SUM(shipping_fee)  AS total_shipping_fees,
    SUM(discount_amt)  AS total_discounts
FROM orders;
```

### 2.2 SUM กับ NULL

SUM จะข้ามค่า NULL โดยอัตโนมัติ

```sql
-- ตัวอย่างที่ 14: SUM ข้าม NULL อัตโนมัติ
-- ศิริลักษณ์ มี salary = NULL แต่ SUM ยังทำงานได้
SELECT 
    SUM(salary)    AS total_salary,
    COUNT(salary)  AS employees_with_salary,
    COUNT(*)       AS total_employees
FROM employees;
```

```sql
-- ตัวอย่างที่ 15: ใช้ COALESCE เพื่อนับ NULL เป็น 0
SELECT 
    SUM(COALESCE(salary, 0)) AS total_salary_including_zero
FROM employees;
```

### 2.3 SUM กับ Expression (การคำนวณ)

```sql
-- ตัวอย่างที่ 16: คำนวณกำไรรวมจากสินค้า
SELECT 
    SUM(price - cost)       AS total_unit_profit,
    SUM((price - cost) * stock_qty) AS total_potential_profit
FROM products;
```

```sql
-- ตัวอย่างที่ 17: คำนวณมูลค่าสต็อกสินค้า
SELECT 
    SUM(price * stock_qty) AS total_stock_value_retail,
    SUM(cost  * stock_qty) AS total_stock_value_cost
FROM products;
```

```sql
-- ตัวอย่างที่ 18: รวมยอดขายสุทธิ (หลังหักส่วนลด)
SELECT 
    SUM(total_amount)    AS gross_revenue,
    SUM(discount_amt)    AS total_discounts,
    SUM(total_amount) - SUM(discount_amt) AS net_revenue
FROM orders;
```

---

## 3. AVG Function - ค่าเฉลี่ย

### 3.1 AVG พื้นฐาน

```sql
-- ตัวอย่างที่ 19: เงินเดือนเฉลี่ยของพนักงาน
SELECT AVG(salary) AS avg_salary
FROM employees;

-- หมายเหตุ: AVG จะข้าม NULL โดยอัตโนมัติ (ไม่รวม ศิริลักษณ์)
```

```sql
-- ตัวอย่างที่ 20: ราคาสินค้าเฉลี่ย
SELECT 
    AVG(price)          AS avg_price,
    AVG(cost)           AS avg_cost,
    AVG(price - cost)   AS avg_profit_per_item
FROM products;
```

```sql
-- ตัวอย่างที่ 21: ยอดสั่งซื้อเฉลี่ยต่อออเดอร์
SELECT 
    AVG(total_amount)   AS avg_order_value,
    AVG(shipping_fee)   AS avg_shipping_fee,
    AVG(discount_amt)   AS avg_discount
FROM orders;
```

### 3.2 ปัญหาความแม่นยำของ AVG (Precision Issues)

```sql
-- ตัวอย่างที่ 22: AVG อาจมีทศนิยมยาวมาก ใช้ ROUND เพื่อแก้ปัญหา
SELECT 
    AVG(salary)            AS avg_salary_raw,
    ROUND(AVG(salary), 2)  AS avg_salary_rounded,
    ROUND(AVG(salary), 0)  AS avg_salary_integer
FROM employees;
```

```sql
-- ตัวอย่างที่ 23: เปรียบเทียบ AVG vs SUM/COUNT
SELECT 
    AVG(salary)          AS avg_from_avg_function,
    SUM(salary) / COUNT(salary) AS avg_calculated_manually
FROM employees;

-- ทั้งสองวิธีให้ผลเหมือนกัน
```

```sql
-- ตัวอย่างที่ 24: AVG ที่รวม NULL ด้วย (ใช้ COUNT(*) แทน COUNT(col))
SELECT 
    AVG(salary)                          AS avg_excluding_null,
    SUM(COALESCE(salary, 0)) / COUNT(*)  AS avg_including_null_as_zero
FROM employees;
```

---

## 4. MIN และ MAX Function

### 4.1 MIN/MAX กับตัวเลข

```sql
-- ตัวอย่างที่ 25: เงินเดือนต่ำสุดและสูงสุด
SELECT 
    MIN(salary) AS min_salary,
    MAX(salary) AS max_salary,
    MAX(salary) - MIN(salary) AS salary_range
FROM employees;
```

```sql
-- ตัวอย่างที่ 26: ราคาสินค้าต่ำสุดและสูงสุด
SELECT 
    MIN(price) AS cheapest_product,
    MAX(price) AS most_expensive_product
FROM products;
```

```sql
-- ตัวอย่างที่ 27: ยอดออเดอร์ต่ำสุดและสูงสุด
SELECT 
    MIN(total_amount) AS smallest_order,
    MAX(total_amount) AS largest_order
FROM orders;
```

### 4.2 MIN/MAX กับ String

```sql
-- ตัวอย่างที่ 28: MIN/MAX กับ String จะเรียงตามตัวอักษร (Alphabetical)
SELECT 
    MIN(first_name) AS first_name_alphabetically,
    MAX(first_name) AS last_name_alphabetically
FROM employees;
```

```sql
-- ตัวอย่างที่ 29: MIN/MAX กับชื่อสินค้า
SELECT 
    MIN(product_name) AS first_product_alphabetically,
    MAX(product_name) AS last_product_alphabetically
FROM products;
```

### 4.3 MIN/MAX กับวันที่ (Date)

```sql
-- ตัวอย่างที่ 30: วันที่รับพนักงานคนแรกและคนสุดท้าย
SELECT 
    MIN(hire_date) AS first_hire_date,
    MAX(hire_date) AS latest_hire_date,
    DATEDIFF(MAX(hire_date), MIN(hire_date)) AS days_between
FROM employees;
```

```sql
-- ตัวอย่างที่ 31: ออเดอร์แรกและล่าสุด
SELECT 
    MIN(order_date) AS first_order,
    MAX(order_date) AS latest_order
FROM orders;
```

---

## 5. รวมหลาย Aggregate Functions

```sql
-- ตัวอย่างที่ 32: สรุปภาพรวมธุรกิจในคำสั่งเดียว
SELECT 
    COUNT(*)                     AS total_orders,
    COUNT(DISTINCT customer_id)  AS unique_customers,
    SUM(total_amount)            AS total_revenue,
    AVG(total_amount)            AS avg_order_value,
    MIN(total_amount)            AS min_order,
    MAX(total_amount)            AS max_order
FROM orders
WHERE status != 'cancelled';
```

```sql
-- ตัวอย่างที่ 33: สรุปข้อมูลพนักงาน
SELECT 
    COUNT(*)           AS total_employees,
    COUNT(salary)      AS employees_with_salary,
    SUM(salary)        AS total_payroll,
    ROUND(AVG(salary), 2) AS avg_salary,
    MIN(salary)        AS min_salary,
    MAX(salary)        AS max_salary
FROM employees;
```

```sql
-- ตัวอย่างที่ 34: สรุปข้อมูลสินค้า
SELECT 
    COUNT(*)                          AS total_products,
    COUNT(CASE WHEN is_active THEN 1 END) AS active_products,
    SUM(stock_qty)                    AS total_stock,
    ROUND(AVG(price), 2)              AS avg_price,
    MIN(price)                        AS min_price,
    MAX(price)                        AS max_price,
    SUM(price * stock_qty)            AS total_inventory_value
FROM products;
```

---

## 6. Aggregate Functions ใน WHERE (ผิด!) vs HAVING (ถูก!)

### ข้อผิดพลาดที่พบบ่อยที่สุด!

```sql
-- ❌ ผิด! ไม่สามารถใช้ Aggregate Function ใน WHERE
SELECT department_id, AVG(salary)
FROM employees
WHERE AVG(salary) > 60000;  -- ERROR: Invalid use of group function

-- ✅ ถูก! ใช้ HAVING แทน
SELECT department_id, AVG(salary) AS avg_salary
FROM employees
GROUP BY department_id
HAVING AVG(salary) > 60000;
```

```sql
-- ตัวอย่างที่ 35: ความแตกต่างระหว่าง WHERE และ HAVING
-- WHERE กรองก่อน Aggregate
-- HAVING กรองหลัง Aggregate

-- หาลูกค้าที่สั่งซื้อมากกว่า 3 ครั้ง
SELECT 
    customer_id,
    COUNT(*) AS order_count,
    SUM(total_amount) AS total_spent
FROM orders
WHERE status != 'cancelled'      -- WHERE กรองก่อนนับ
GROUP BY customer_id
HAVING COUNT(*) >= 2;            -- HAVING กรองหลังนับ
```

---

## 7. Aggregate กับ Expression ขั้นสูง

```sql
-- ตัวอย่างที่ 36: Conditional Aggregation เบื้องต้น
SELECT 
    COUNT(*) AS total_orders,
    COUNT(CASE WHEN status = 'delivered' THEN 1 END) AS delivered,
    COUNT(CASE WHEN status = 'pending'   THEN 1 END) AS pending,
    COUNT(CASE WHEN status = 'cancelled' THEN 1 END) AS cancelled
FROM orders;
```

```sql
-- ตัวอย่างที่ 37: SUM กับ CASE (นับเฉพาะบางเงื่อนไข)
SELECT 
    SUM(CASE WHEN status = 'delivered' THEN total_amount ELSE 0 END) AS delivered_revenue,
    SUM(CASE WHEN status = 'cancelled' THEN total_amount ELSE 0 END) AS cancelled_revenue,
    SUM(CASE WHEN payment_method = 'บัตรเครดิต' THEN total_amount ELSE 0 END) AS credit_card_revenue
FROM orders;
```

```sql
-- ตัวอย่างที่ 38: คำนวณเปอร์เซ็นต์จาก Aggregate
SELECT 
    COUNT(*) AS total_orders,
    COUNT(CASE WHEN status = 'delivered' THEN 1 END) AS delivered_count,
    ROUND(
        COUNT(CASE WHEN status = 'delivered' THEN 1 END) * 100.0 / COUNT(*), 
        2
    ) AS delivery_rate_pct
FROM orders;
```

```sql
-- ตัวอย่างที่ 39: หาค่าเฉลี่ยของสินค้าที่มีราคาสูงกว่าราคาเฉลี่ย
-- ใช้ Subquery ร่วมกับ AVG
SELECT AVG(price) AS avg_expensive_products
FROM products
WHERE price > (SELECT AVG(price) FROM products);
```

```sql
-- ตัวอย่างที่ 40: รวมยอดสั่งซื้อต่อสินค้า
SELECT 
    p.product_name,
    COUNT(oi.item_id)          AS times_ordered,
    SUM(oi.quantity)           AS total_qty_sold,
    SUM(oi.quantity * oi.unit_price) AS total_revenue
FROM products p
LEFT JOIN order_items oi ON p.product_id = oi.product_id
GROUP BY p.product_id, p.product_name
ORDER BY total_revenue DESC;
```

---

## 8. ตัวอย่างการใช้งานจริง (Real Business Examples)

```sql
-- ตัวอย่างที่ 41: รายงานสรุปประจำเดือน
SELECT 
    YEAR(order_date)  AS year,
    MONTH(order_date) AS month,
    COUNT(*)          AS total_orders,
    SUM(total_amount) AS monthly_revenue,
    AVG(total_amount) AS avg_order_value,
    MIN(total_amount) AS smallest_order,
    MAX(total_amount) AS largest_order
FROM orders
WHERE status IN ('delivered', 'shipped')
GROUP BY YEAR(order_date), MONTH(order_date)
ORDER BY year, month;
```

```sql
-- ตัวอย่างที่ 42: วิเคราะห์ช่องทางการชำระเงิน
SELECT 
    payment_method,
    COUNT(*) AS order_count,
    SUM(total_amount) AS total_revenue,
    ROUND(AVG(total_amount), 2) AS avg_order_value,
    ROUND(COUNT(*) * 100.0 / SUM(COUNT(*)) OVER (), 2) AS pct_of_total
FROM orders
GROUP BY payment_method
ORDER BY total_revenue DESC;
```

```sql
-- ตัวอย่างที่ 43: TOP 5 ลูกค้าที่ซื้อมากที่สุด
SELECT 
    c.customer_id,
    CONCAT(c.first_name, ' ', c.last_name) AS customer_name,
    COUNT(o.order_id)    AS total_orders,
    SUM(o.total_amount)  AS total_spent,
    MAX(o.order_date)    AS last_order_date
FROM customers c
INNER JOIN orders o ON c.customer_id = o.customer_id
WHERE o.status != 'cancelled'
GROUP BY c.customer_id, c.first_name, c.last_name
ORDER BY total_spent DESC
LIMIT 5;
```

```sql
-- ตัวอย่างที่ 44: สินค้าขายดีที่สุด
SELECT 
    p.product_id,
    p.product_name,
    p.category,
    COUNT(oi.item_id)     AS times_in_orders,
    SUM(oi.quantity)      AS total_units_sold,
    SUM(oi.quantity * oi.unit_price) AS total_revenue,
    AVG(oi.unit_price)    AS avg_selling_price
FROM products p
INNER JOIN order_items oi ON p.product_id = oi.product_id
GROUP BY p.product_id, p.product_name, p.category
ORDER BY total_units_sold DESC
LIMIT 10;
```

```sql
-- ตัวอย่างที่ 45: วิเคราะห์สต็อกสินค้า
SELECT 
    category,
    COUNT(*)          AS product_count,
    SUM(stock_qty)    AS total_stock,
    AVG(stock_qty)    AS avg_stock_per_product,
    MIN(stock_qty)    AS min_stock,
    MAX(stock_qty)    AS max_stock,
    SUM(price * stock_qty) AS inventory_value
FROM products
WHERE is_active = TRUE
GROUP BY category
ORDER BY inventory_value DESC;
```

```sql
-- ตัวอย่างที่ 46: รายงานแผนกและเงินเดือน
SELECT 
    d.department_name,
    COUNT(e.employee_id) AS headcount,
    SUM(e.salary)        AS total_salary,
    AVG(e.salary)        AS avg_salary,
    MIN(e.salary)        AS min_salary,
    MAX(e.salary)        AS max_salary
FROM departments d
LEFT JOIN employees e ON d.department_id = e.department_id
GROUP BY d.department_id, d.department_name
ORDER BY total_salary DESC;
```

```sql
-- ตัวอย่างที่ 47: อัตราการยกเลิกออเดอร์
SELECT 
    COUNT(*) AS total_orders,
    SUM(CASE WHEN status = 'cancelled' THEN 1 ELSE 0 END) AS cancelled_orders,
    ROUND(
        SUM(CASE WHEN status = 'cancelled' THEN 1 ELSE 0 END) * 100.0 / COUNT(*),
        2
    ) AS cancellation_rate
FROM orders;
```

```sql
-- ตัวอย่างที่ 48: ลูกค้าที่ยังไม่เคยสั่งซื้อ
SELECT 
    COUNT(*) AS customers_without_orders
FROM customers c
WHERE NOT EXISTS (
    SELECT 1 FROM orders o 
    WHERE o.customer_id = c.customer_id
);
```

```sql
-- ตัวอย่างที่ 49: รายงาน KPI หลัก (Executive Dashboard)
SELECT 
    -- Revenue KPIs
    SUM(total_amount)                         AS gross_revenue,
    SUM(CASE WHEN status NOT IN ('cancelled') 
        THEN total_amount ELSE 0 END)          AS net_revenue,
    
    -- Order KPIs
    COUNT(*)                                  AS total_orders,
    COUNT(CASE WHEN status = 'delivered' 
        THEN 1 END)                           AS completed_orders,
    COUNT(CASE WHEN status = 'cancelled' 
        THEN 1 END)                           AS cancelled_orders,
    
    -- Customer KPIs
    COUNT(DISTINCT customer_id)               AS unique_customers,
    
    -- Average KPIs
    ROUND(AVG(total_amount), 2)               AS avg_order_value,
    ROUND(AVG(shipping_fee), 2)               AS avg_shipping_fee
FROM orders;
```

```sql
-- ตัวอย่างที่ 50: วิเคราะห์กำไรขั้นต้นจากสินค้า
SELECT 
    p.category,
    COUNT(DISTINCT p.product_id)    AS product_count,
    SUM(oi.quantity)                AS total_units_sold,
    SUM(oi.quantity * oi.unit_price) AS total_revenue,
    SUM(oi.quantity * p.cost)       AS total_cost,
    SUM(oi.quantity * oi.unit_price) - 
    SUM(oi.quantity * p.cost)       AS gross_profit,
    ROUND(
        (SUM(oi.quantity * oi.unit_price) - SUM(oi.quantity * p.cost)) * 100.0 / 
        SUM(oi.quantity * oi.unit_price),
        2
    )                               AS gross_profit_margin_pct
FROM products p
INNER JOIN order_items oi ON p.product_id = oi.product_id
GROUP BY p.category
ORDER BY gross_profit DESC;
```

---

## 9. ข้อผิดพลาดที่พบบ่อยและวิธีหลีกเลี่ยง

### ข้อผิดพลาด #1: ใช้ Aggregate ใน WHERE

```sql
-- ❌ ผิด
SELECT * FROM orders WHERE SUM(total_amount) > 50000;

-- ✅ ถูก - ใช้ Subquery
SELECT * FROM orders WHERE total_amount > (SELECT AVG(total_amount) FROM orders);

-- ✅ ถูก - ใช้ HAVING กับ GROUP BY
SELECT customer_id, SUM(total_amount) 
FROM orders 
GROUP BY customer_id
HAVING SUM(total_amount) > 50000;
```

### ข้อผิดพลาด #2: สับสนระหว่าง COUNT(*) และ COUNT(column)

```sql
-- ❌ อาจให้ผลผิดพลาดถ้า manager_id มี NULL
SELECT COUNT(manager_id) FROM employees;  -- นับเฉพาะที่ไม่ใช่ NULL

-- ✅ เลือกใช้ให้ถูกต้องตามความต้องการ
SELECT 
    COUNT(*)          AS total_including_null,
    COUNT(manager_id) AS total_excluding_null
FROM employees;
```

### ข้อผิดพลาด #3: หารด้วยศูนย์ใน AVG calculation

```sql
-- ❌ อาจเกิด Division by Zero ถ้าไม่มีข้อมูล
SELECT SUM(salary) / COUNT(salary) FROM employees WHERE department_id = 999;

-- ✅ ใช้ NULLIF ป้องกัน
SELECT SUM(salary) / NULLIF(COUNT(salary), 0) FROM employees WHERE department_id = 999;
-- ถ้า COUNT = 0 จะได้ NULL แทน Error
```

### ข้อผิดพลาด #4: AVG ไม่ตรงกับที่คาดหวังเพราะ NULL

```sql
-- สถานการณ์: พนักงาน 5 คน มีเงินเดือน [50000, 60000, NULL, NULL, NULL]
-- AVG จะให้ผลเป็น 55000 ไม่ใช่ 22000!
-- เพราะ NULL ถูกข้ามไป

-- ถ้าต้องการ AVG ที่รวม NULL เป็น 0:
SELECT SUM(COALESCE(salary, 0)) / COUNT(*) AS avg_with_null_as_zero
FROM employees;
```

---

## สรุปบทที่ 31 (Summary)

| Function | จุดประสงค์ | NULL Handling |
|----------|-----------|---------------|
| `COUNT(*)` | นับทุกแถว | รวม NULL |
| `COUNT(col)` | นับเฉพาะ non-NULL | ข้าม NULL |
| `COUNT(DISTINCT col)` | นับค่าไม่ซ้ำ | ข้าม NULL |
| `SUM(col)` | รวมค่า | ข้าม NULL |
| `AVG(col)` | ค่าเฉลี่ย | ข้าม NULL |
| `MIN(col)` | ค่าต่ำสุด | ข้าม NULL |
| `MAX(col)` | ค่าสูงสุด | ข้าม NULL |

---

## แบบฝึกหัดท้ายบท (Exercises)

### แบบฝึกหัดที่ 1
นับจำนวนสินค้าทั้งหมด, สินค้าที่ active, และสินค้าที่ inactive

```sql
-- เฉลย:
SELECT 
    COUNT(*) AS total_products,
    COUNT(CASE WHEN is_active = TRUE  THEN 1 END) AS active_products,
    COUNT(CASE WHEN is_active = FALSE THEN 1 END) AS inactive_products
FROM products;
```

### แบบฝึกหัดที่ 2
หายอดขายรวม ยอดขายเฉลี่ยต่อออเดอร์ และจำนวนออเดอร์ทั้งหมดที่มีสถานะ 'delivered'

```sql
-- เฉลย:
SELECT 
    COUNT(*)           AS delivered_orders,
    SUM(total_amount)  AS total_revenue,
    ROUND(AVG(total_amount), 2) AS avg_order_value
FROM orders
WHERE status = 'delivered';
```

### แบบฝึกหัดที่ 3
หาเงินเดือนสูงสุดและต่ำสุด พร้อมทั้งความแตกต่าง (range) ของเงินเดือน

```sql
-- เฉลย:
SELECT 
    MAX(salary) AS max_salary,
    MIN(salary) AS min_salary,
    MAX(salary) - MIN(salary) AS salary_range
FROM employees
WHERE salary IS NOT NULL;
```

### แบบฝึกหัดที่ 4
นับจำนวนลูกค้าที่ลงทะเบียนในแต่ละปี

```sql
-- เฉลย:
SELECT 
    YEAR(registered_at) AS registration_year,
    COUNT(*) AS new_customers
FROM customers
GROUP BY YEAR(registered_at)
ORDER BY registration_year;
```

### แบบฝึกหัดที่ 5
หามูลค่ารวมของสินค้าในสต็อก (price * stock_qty) แยกตาม category

```sql
-- เฉลย:
SELECT 
    category,
    COUNT(*) AS product_count,
    SUM(stock_qty) AS total_stock,
    SUM(price * stock_qty) AS total_stock_value
FROM products
WHERE is_active = TRUE
GROUP BY category
ORDER BY total_stock_value DESC;
```

### แบบฝึกหัดที่ 6
หาจำนวนออเดอร์และยอดขายรวมแยกตามวิธีการชำระเงิน เรียงตามยอดขายมากที่สุดก่อน

```sql
-- เฉลย:
SELECT 
    payment_method,
    COUNT(*) AS order_count,
    SUM(total_amount) AS total_revenue,
    ROUND(AVG(total_amount), 2) AS avg_order_value
FROM orders
GROUP BY payment_method
ORDER BY total_revenue DESC;
```

### แบบฝึกหัดที่ 7
คำนวณอัตราส่วนของออเดอร์ที่ delivered เทียบกับทั้งหมด (เป็นเปอร์เซ็นต์)

```sql
-- เฉลย:
SELECT 
    COUNT(*) AS total_orders,
    COUNT(CASE WHEN status = 'delivered' THEN 1 END) AS delivered_orders,
    ROUND(
        COUNT(CASE WHEN status = 'delivered' THEN 1 END) * 100.0 / COUNT(*),
        2
    ) AS delivery_pct
FROM orders;
```

### แบบฝึกหัดที่ 8
หาลูกค้าที่มี loyalty_points สูงสุด ต่ำสุด และค่าเฉลี่ย

```sql
-- เฉลย:
SELECT 
    MAX(loyalty_points) AS max_points,
    MIN(loyalty_points) AS min_points,
    ROUND(AVG(loyalty_points), 2) AS avg_points,
    SUM(loyalty_points) AS total_points_issued
FROM customers;
```

### แบบฝึกหัดที่ 9
หาจำนวนสินค้าที่ราคาต่ำกว่าราคาเฉลี่ย และสูงกว่าราคาเฉลี่ย

```sql
-- เฉลย:
SELECT 
    AVG(price) AS avg_price,
    COUNT(CASE WHEN price < (SELECT AVG(price) FROM products) THEN 1 END) AS below_avg_count,
    COUNT(CASE WHEN price > (SELECT AVG(price) FROM products) THEN 1 END) AS above_avg_count
FROM products;
```

### แบบฝึกหัดที่ 10
สร้าง Dashboard สรุป: จำนวนพนักงานทั้งหมด, เงินเดือนรวม, ค่าเฉลี่ยเงินเดือน, พนักงานที่มีเงินเดือนสูงกว่าเฉลี่ย

```sql
-- เฉลย:
SELECT 
    COUNT(*)       AS total_employees,
    COUNT(salary)  AS employees_with_salary,
    SUM(salary)    AS total_payroll,
    ROUND(AVG(salary), 2) AS avg_salary,
    COUNT(CASE WHEN salary > (SELECT AVG(salary) FROM employees) THEN 1 END) AS above_avg_count,
    ROUND(
        COUNT(CASE WHEN salary > (SELECT AVG(salary) FROM employees) THEN 1 END) * 100.0 / COUNT(salary),
        2
    ) AS pct_above_avg
FROM employees;
```

---

*จบบทที่ 031 - Aggregate Functions: COUNT, SUM, AVG, MIN, MAX*

*บทถัดไป: Part 032 - GROUP BY: การจัดกลุ่มข้อมูล*
