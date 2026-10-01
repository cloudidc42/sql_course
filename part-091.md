# ส่วนที่ 91: Common Table Expressions (CTEs)

## บทนำ

Common Table Expressions หรือ CTEs เป็นหนึ่งในฟีเจอร์ที่ทรงพลังและอ่านง่ายที่สุดใน SQL สมัยใหม่ CTE ช่วยให้เราสามารถแบ่งคำสั่ง SQL ที่ซับซ้อนออกเป็นส่วนย่อยๆ ที่มีชื่อและเข้าใจง่าย เปรียบเสมือนการสร้าง "ตารางชั่วคราว" ที่มีชื่อและใช้ได้เฉพาะภายใน query เดียวกัน

---

## 91.1 ไวยากรณ์พื้นฐานของ WITH clause

CTE ถูกนิยามด้วย `WITH` keyword ตามด้วยชื่อ CTE และ query ที่นิยามเนื้อหา

### ไวยากรณ์พื้นฐาน

```sql
WITH cte_name AS (
    -- query ที่นิยาม CTE
    SELECT column1, column2
    FROM some_table
    WHERE condition
)
SELECT *
FROM cte_name;
```

### ตัวอย่างที่ 1: CTE พื้นฐานที่สุด

```sql
-- สร้างตัวอย่างข้อมูล
CREATE TABLE employees (
    employee_id   INT PRIMARY KEY,
    name          VARCHAR(100),
    department    VARCHAR(50),
    salary        DECIMAL(10,2),
    hire_date     DATE,
    manager_id    INT
);

INSERT INTO employees VALUES
(1,  'สมชาย ใจดี',      'IT',      85000,  '2020-01-15', NULL),
(2,  'สมหญิง รักงาน',   'HR',      65000,  '2019-03-20', 1),
(3,  'วิชัย เก่งกาจ',   'IT',      90000,  '2021-06-01', 1),
(4,  'มานี มีทรัพย์',   'Finance', 75000,  '2018-11-10', 1),
(5,  'ประเสริฐ ดีมาก',  'IT',      70000,  '2022-02-14', 3),
(6,  'อรุณี สวยงาม',    'HR',      60000,  '2020-07-22', 2),
(7,  'บุญมี มากทรัพย์', 'Finance', 80000,  '2017-05-30', 4),
(8,  'จรัส เจริญรุ่ง',  'IT',      95000,  '2016-09-15', 1),
(9,  'ลัดดา ดาวเรือง',  'HR',      55000,  '2023-01-08', 2),
(10, 'ไพศาล ใหญ่โต',   'Finance', 72000,  '2021-04-19', 4);

-- CTE พื้นฐาน: ดึงพนักงานในแผนก IT
WITH it_employees AS (
    SELECT employee_id, name, salary
    FROM employees
    WHERE department = 'IT'
)
SELECT *
FROM it_employees;
```

### ตัวอย่างที่ 2: CTE พร้อม alias columns

```sql
-- ระบุชื่อ columns ใน CTE
WITH dept_summary (department_name, total_employees, avg_salary) AS (
    SELECT 
        department,
        COUNT(*),
        AVG(salary)
    FROM employees
    GROUP BY department
)
SELECT 
    department_name,
    total_employees,
    ROUND(avg_salary, 2) AS average_salary
FROM dept_summary
ORDER BY avg_salary DESC;
```

---

## 91.2 CTE vs Subquery vs Temp Table

การเปรียบเทียบสามวิธีที่ใช้แทนกันได้ในหลายกรณี

### ตัวอย่างที่ 3: Subquery แบบเดิม

```sql
-- แบบ Subquery (อ่านยาก)
SELECT e.name, e.salary, dept_avg.avg_salary
FROM employees e
JOIN (
    SELECT department, AVG(salary) AS avg_salary
    FROM employees
    GROUP BY department
) dept_avg ON e.department = dept_avg.department
WHERE e.salary > dept_avg.avg_salary;
```

### ตัวอย่างที่ 4: เขียนใหม่ด้วย CTE (อ่านง่ายกว่า)

```sql
-- แบบ CTE (อ่านง่ายกว่ามาก)
WITH dept_avg AS (
    SELECT department, AVG(salary) AS avg_salary
    FROM employees
    GROUP BY department
)
SELECT e.name, e.salary, da.avg_salary
FROM employees e
JOIN dept_avg da ON e.department = da.department
WHERE e.salary > da.avg_salary;
```

### ตัวอย่างที่ 5: เปรียบเทียบกับ Temp Table

```sql
-- แบบ Temp Table (ต้องสร้างและลบ)
CREATE TEMPORARY TABLE dept_avg_temp AS
SELECT department, AVG(salary) AS avg_salary
FROM employees
GROUP BY department;

SELECT e.name, e.salary, t.avg_salary
FROM employees e
JOIN dept_avg_temp t ON e.department = t.department
WHERE e.salary > t.avg_salary;

DROP TEMPORARY TABLE dept_avg_temp;

-- แบบ CTE (ไม่ต้องสร้างหรือลบ)
WITH dept_avg AS (
    SELECT department, AVG(salary) AS avg_salary
    FROM employees
    GROUP BY department
)
SELECT e.name, e.salary, da.avg_salary
FROM employees e
JOIN dept_avg da ON e.department = da.department
WHERE e.salary > da.avg_salary;
```

### ตารางเปรียบเทียบ

| คุณสมบัติ | CTE | Subquery | Temp Table |
|-----------|-----|----------|------------|
| อ่านง่าย | ดีมาก | แย่ | ดี |
| ใช้ซ้ำใน query | ได้ | ไม่ได้ | ได้ |
| ขอบเขต (scope) | เฉพาะ query | เฉพาะที่ใช้ | ทั้ง session |
| สร้าง index ได้ | ไม่ | ไม่ | ได้ |
| เหมาะกับข้อมูลใหญ่ | ปานกลาง | ปานกลาง | ดี |

---

## 91.3 Multiple CTEs ใน Query เดียว

สามารถนิยาม CTE หลายตัวในคำสั่งเดียวโดยใช้เครื่องหมายจุลภาค (`,`)

### ตัวอย่างที่ 6: สอง CTEs พื้นฐาน

```sql
WITH 
high_earners AS (
    SELECT employee_id, name, salary, department
    FROM employees
    WHERE salary > 75000
),
it_dept AS (
    SELECT employee_id, name, salary
    FROM employees
    WHERE department = 'IT'
)
SELECT h.name, h.salary, h.department
FROM high_earners h
WHERE h.employee_id IN (SELECT employee_id FROM it_dept);
```

### ตัวอย่างที่ 7: สาม CTEs สำหรับ multi-step analysis

```sql
WITH 
-- CTE 1: คำนวณสถิติรายแผนก
dept_stats AS (
    SELECT 
        department,
        COUNT(*) AS employee_count,
        AVG(salary) AS avg_salary,
        MAX(salary) AS max_salary,
        MIN(salary) AS min_salary
    FROM employees
    GROUP BY department
),
-- CTE 2: จัดอันดับแผนกตามเงินเดือนเฉลี่ย
dept_ranked AS (
    SELECT 
        department,
        employee_count,
        avg_salary,
        RANK() OVER (ORDER BY avg_salary DESC) AS salary_rank
    FROM dept_stats
),
-- CTE 3: เลือกเฉพาะแผนกที่มีพนักงาน 3 คนขึ้นไป
qualified_depts AS (
    SELECT *
    FROM dept_ranked
    WHERE employee_count >= 3
)
SELECT *
FROM qualified_depts
ORDER BY salary_rank;
```

### ตัวอย่างที่ 8: CTEs ที่อ้างอิงกัน

```sql
-- CTE หลังสามารถอ้างอิง CTE ก่อนหน้าได้
WITH 
base_data AS (
    SELECT employee_id, name, salary, department, hire_date
    FROM employees
    WHERE hire_date >= '2019-01-01'
),
with_years AS (
    SELECT 
        employee_id,
        name,
        salary,
        department,
        EXTRACT(YEAR FROM CURRENT_DATE) - EXTRACT(YEAR FROM hire_date) AS years_worked
    FROM base_data
),
categorized AS (
    SELECT *,
        CASE 
            WHEN years_worked >= 5 THEN 'Senior'
            WHEN years_worked >= 3 THEN 'Mid-level'
            ELSE 'Junior'
        END AS level
    FROM with_years
)
SELECT level, COUNT(*) AS count, ROUND(AVG(salary), 2) AS avg_salary
FROM categorized
GROUP BY level
ORDER BY avg_salary DESC;
```

---

## 91.4 ประโยชน์ด้านความอ่านง่าย

### ตัวอย่างที่ 9: Query ที่ซับซ้อนโดยไม่มี CTE

```sql
-- ยากต่อการอ่านและดูแลรักษา
SELECT 
    d.department,
    d.total_salary,
    d.employee_count,
    ROUND(d.total_salary / d.employee_count, 2) AS avg_salary,
    t.company_total,
    ROUND(d.total_salary * 100.0 / t.company_total, 2) AS pct_of_total
FROM (
    SELECT department, SUM(salary) AS total_salary, COUNT(*) AS employee_count
    FROM employees
    GROUP BY department
) d
CROSS JOIN (
    SELECT SUM(salary) AS company_total
    FROM employees
) t
ORDER BY d.total_salary DESC;
```

### ตัวอย่างที่ 10: Query เดียวกันด้วย CTEs (อ่านง่ายมาก)

```sql
-- อ่านง่าย เข้าใจง่าย
WITH 
dept_totals AS (
    SELECT 
        department,
        SUM(salary)  AS total_salary,
        COUNT(*)     AS employee_count
    FROM employees
    GROUP BY department
),
company_total AS (
    SELECT SUM(salary) AS grand_total
    FROM employees
)
SELECT 
    dt.department,
    dt.total_salary,
    dt.employee_count,
    ROUND(dt.total_salary / dt.employee_count, 2)        AS avg_salary,
    ct.grand_total,
    ROUND(dt.total_salary * 100.0 / ct.grand_total, 2)   AS pct_of_total
FROM dept_totals dt
CROSS JOIN company_total ct
ORDER BY dt.total_salary DESC;
```

---

## 91.5 CTE สำหรับการ Deduplication

### ตัวอย่างที่ 11: ลบข้อมูลซ้ำด้วย CTE

```sql
-- สร้างตารางที่มีข้อมูลซ้ำ
CREATE TABLE customer_orders (
    order_id    INT,
    customer_id INT,
    product     VARCHAR(50),
    order_date  DATE,
    amount      DECIMAL(10,2)
);

INSERT INTO customer_orders VALUES
(1, 101, 'Laptop', '2024-01-01', 45000),
(2, 101, 'Laptop', '2024-01-01', 45000),  -- ซ้ำกับ row 1
(3, 102, 'Phone',  '2024-01-02', 15000),
(4, 103, 'Tablet', '2024-01-03', 20000),
(5, 103, 'Tablet', '2024-01-03', 20000),  -- ซ้ำกับ row 4
(6, 104, 'Mouse',  '2024-01-04',  800);

-- ระบุแถวซ้ำด้วย CTE
WITH duplicates AS (
    SELECT 
        order_id,
        ROW_NUMBER() OVER (
            PARTITION BY customer_id, product, order_date, amount
            ORDER BY order_id
        ) AS rn
    FROM customer_orders
)
SELECT order_id
FROM duplicates
WHERE rn > 1;  -- แถวที่เป็นซ้ำ (row number > 1)
```

### ตัวอย่างที่ 12: ลบแถวซ้ำออกจากตาราง

```sql
-- ลบแถวซ้ำโดยเก็บ order_id ที่น้อยที่สุดไว้
WITH duplicates AS (
    SELECT 
        order_id,
        ROW_NUMBER() OVER (
            PARTITION BY customer_id, product, order_date, amount
            ORDER BY order_id
        ) AS rn
    FROM customer_orders
)
DELETE FROM customer_orders
WHERE order_id IN (
    SELECT order_id FROM duplicates WHERE rn > 1
);

-- ตรวจสอบผลลัพธ์
SELECT * FROM customer_orders;
```

### ตัวอย่างที่ 13: Deduplication แบบเลือกเก็บแถวล่าสุด

```sql
-- สร้างข้อมูลตัวอย่าง: ประวัติราคาสินค้า
CREATE TABLE product_prices (
    price_id    INT,
    product_id  INT,
    price       DECIMAL(10,2),
    updated_at  TIMESTAMP
);

INSERT INTO product_prices VALUES
(1, 1001, 100.00, '2024-01-01 10:00:00'),
(2, 1001, 95.00,  '2024-01-15 14:30:00'),
(3, 1001, 98.00,  '2024-02-01 09:00:00'),
(4, 1002, 200.00, '2024-01-01 10:00:00'),
(5, 1002, 185.00, '2024-01-20 11:00:00');

-- เลือกเฉพาะราคาล่าสุดของแต่ละสินค้า
WITH latest_prices AS (
    SELECT 
        product_id,
        price,
        updated_at,
        ROW_NUMBER() OVER (
            PARTITION BY product_id
            ORDER BY updated_at DESC
        ) AS rn
    FROM product_prices
)
SELECT product_id, price, updated_at
FROM latest_prices
WHERE rn = 1;
```

---

## 91.6 CTE สำหรับ Multi-step Transformations

### ตัวอย่างที่ 14: การแปลงข้อมูลหลายขั้นตอน

```sql
CREATE TABLE sales (
    sale_id     INT PRIMARY KEY,
    salesperson VARCHAR(100),
    region      VARCHAR(50),
    product     VARCHAR(50),
    quantity    INT,
    unit_price  DECIMAL(10,2),
    sale_date   DATE
);

INSERT INTO sales VALUES
(1,  'สมชาย', 'เหนือ',  'A', 10, 100, '2024-01-05'),
(2,  'สมชาย', 'เหนือ',  'B', 5,  200, '2024-01-10'),
(3,  'สมหญิง','ใต้',    'A', 8,  100, '2024-01-07'),
(4,  'สมหญิง','ใต้',    'C', 3,  300, '2024-01-15'),
(5,  'วิชัย', 'กลาง',   'B', 12, 200, '2024-01-08'),
(6,  'วิชัย', 'กลาง',   'A', 7,  100, '2024-01-20'),
(7,  'มานี',  'ออก',    'C', 15, 300, '2024-02-01'),
(8,  'มานี',  'ออก',    'A', 4,  100, '2024-02-05'),
(9,  'สมชาย', 'เหนือ',  'C', 6,  300, '2024-02-10'),
(10, 'สมหญิง','ใต้',    'B', 9,  200, '2024-02-12');

-- ขั้นตอนที่ 1: คำนวณยอดขายรายการ
WITH step1_revenue AS (
    SELECT 
        sale_id,
        salesperson,
        region,
        product,
        sale_date,
        quantity * unit_price AS revenue
    FROM sales
),
-- ขั้นตอนที่ 2: รวมยอดรายพนักงาน
step2_by_person AS (
    SELECT 
        salesperson,
        SUM(revenue)   AS total_revenue,
        COUNT(*)       AS num_sales,
        AVG(revenue)   AS avg_sale_value
    FROM step1_revenue
    GROUP BY salesperson
),
-- ขั้นตอนที่ 3: จัดอันดับ
step3_ranked AS (
    SELECT 
        salesperson,
        total_revenue,
        num_sales,
        ROUND(avg_sale_value, 2) AS avg_sale_value,
        RANK() OVER (ORDER BY total_revenue DESC) AS revenue_rank
    FROM step2_by_person
)
SELECT * FROM step3_ranked;
```

### ตัวอย่างที่ 15: Pipeline การทำความสะอาดข้อมูล

```sql
-- สมมติว่ามีข้อมูลดิบที่ต้องทำความสะอาด
CREATE TABLE raw_customer_data (
    id          INT,
    full_name   VARCHAR(200),
    email       VARCHAR(200),
    phone       VARCHAR(50),
    city        VARCHAR(100)
);

INSERT INTO raw_customer_data VALUES
(1, '  สมชาย ใจดี  ',  'somchai@email.com',    '081-234-5678', 'กรุงเทพ'),
(2, 'SOMYING RAKNGAM',  'SOMYING@EMAIL.COM',    '0891234567',   'CHIANG MAI'),
(3, 'วิชัย',           NULL,                   '02-345-6789',  'นครราชสีมา'),
(4, '',                'manee@test.org',        '(066)9876543', 'ภูเก็ต');

-- ขั้นตอนที่ 1: ตัดช่องว่างและแปลงเป็น lowercase
WITH step1_clean AS (
    SELECT 
        id,
        TRIM(full_name)         AS full_name,
        LOWER(TRIM(email))      AS email,
        phone,
        TRIM(city)              AS city
    FROM raw_customer_data
),
-- ขั้นตอนที่ 2: กรองแถวที่ไม่มีชื่อหรือ email
step2_filter AS (
    SELECT *
    FROM step1_clean
    WHERE full_name <> '' 
      AND full_name IS NOT NULL
      AND email IS NOT NULL
),
-- ขั้นตอนที่ 3: ทำให้เบอร์โทรเป็นรูปแบบเดียวกัน
step3_normalize AS (
    SELECT 
        id,
        full_name,
        email,
        REGEXP_REPLACE(phone, '[^0-9]', '', 'g') AS phone_digits,
        city
    FROM step2_filter
)
SELECT 
    id,
    full_name,
    email,
    CASE 
        WHEN LENGTH(phone_digits) = 9 THEN '0' || phone_digits
        ELSE phone_digits
    END AS normalized_phone,
    city
FROM step3_normalize;
```

---

## 91.7 ขอบเขต (Scope) และกฎของ CTE

### กฎสำคัญของ CTE

1. CTE มีขอบเขตเฉพาะภายใน query เดียวที่นิยาม
2. CTE สามารถอ้างอิง CTE ก่อนหน้าได้ แต่ไม่สามารถอ้างอิง CTE ที่นิยามทีหลัง
3. CTE ไม่สามารถมีชื่อซ้ำกับตารางจริงหรือ CTE อื่นใน query เดียวกัน
4. CTE ที่ไม่ได้ใช้จะถูกละเว้น (optimizer อาจไม่ execute)

### ตัวอย่างที่ 16: การนำ CTE ไปใช้ใน INSERT, UPDATE, DELETE

```sql
-- ใช้ CTE กับ INSERT
WITH new_employees AS (
    SELECT 
        'พนักงานใหม่' || GENERATE_SERIES AS name,
        'IT' AS department,
        50000 + (RANDOM() * 30000)::INT AS salary,
        CURRENT_DATE AS hire_date
    FROM GENERATE_SERIES(11, 15)
)
INSERT INTO employees (employee_id, name, department, salary, hire_date)
SELECT 
    ROW_NUMBER() OVER () + 10,
    name, department, salary, hire_date
FROM new_employees;

-- ใช้ CTE กับ UPDATE
WITH high_performers AS (
    SELECT employee_id
    FROM employees
    WHERE salary > 80000
)
UPDATE employees
SET salary = salary * 1.10
WHERE employee_id IN (SELECT employee_id FROM high_performers);

-- ใช้ CTE กับ DELETE
WITH inactive_accounts AS (
    SELECT customer_id
    FROM customers
    WHERE last_purchase_date < CURRENT_DATE - INTERVAL '2 years'
)
DELETE FROM customers
WHERE customer_id IN (SELECT customer_id FROM inactive_accounts);
```

### ตัวอย่างที่ 17: CTE กับ RETURNING clause (PostgreSQL)

```sql
-- CTE พร้อม RETURNING ใน PostgreSQL
WITH deleted_orders AS (
    DELETE FROM customer_orders
    WHERE order_date < '2023-01-01'
    RETURNING *
),
archive_insert AS (
    INSERT INTO orders_archive
    SELECT *, NOW() AS archived_at
    FROM deleted_orders
    RETURNING order_id
)
SELECT COUNT(*) AS total_archived
FROM archive_insert;
```

---

## 91.8 CTE ซ้อนกัน (Nested CTE References)

### ตัวอย่างที่ 18: CTE ที่อ้างอิงกันหลายชั้น

```sql
CREATE TABLE transactions (
    txn_id      INT PRIMARY KEY,
    account_id  INT,
    amount      DECIMAL(12,2),
    txn_type    VARCHAR(10),   -- 'credit' หรือ 'debit'
    txn_date    DATE
);

INSERT INTO transactions VALUES
(1, 1001, 5000,  'credit', '2024-01-05'),
(2, 1001, -1500, 'debit',  '2024-01-10'),
(3, 1001, 3000,  'credit', '2024-01-15'),
(4, 1002, 8000,  'credit', '2024-01-03'),
(5, 1002, -2000, 'debit',  '2024-01-08'),
(6, 1002, -1000, 'debit',  '2024-01-20'),
(7, 1003, 10000, 'credit', '2024-01-01'),
(8, 1003, -5000, 'debit',  '2024-01-12');

WITH 
-- ขั้น 1: แยก credit และ debit
splits AS (
    SELECT 
        account_id,
        txn_date,
        CASE WHEN amount > 0 THEN amount ELSE 0 END AS credit,
        CASE WHEN amount < 0 THEN ABS(amount) ELSE 0 END AS debit
    FROM transactions
),
-- ขั้น 2: รวมรายบัญชี
account_summary AS (
    SELECT 
        account_id,
        SUM(credit)  AS total_credit,
        SUM(debit)   AS total_debit,
        SUM(credit) - SUM(debit) AS net_balance
    FROM splits
    GROUP BY account_id
),
-- ขั้น 3: จัดกลุ่มตาม balance
balance_categories AS (
    SELECT *,
        CASE
            WHEN net_balance > 5000 THEN 'High Balance'
            WHEN net_balance > 0    THEN 'Positive'
            ELSE 'Negative'
        END AS balance_category
    FROM account_summary
)
SELECT * FROM balance_categories ORDER BY net_balance DESC;
```

---

## 91.9 ประสิทธิภาพและการ Optimize

### ตัวอย่างที่ 19: CTE Materialization ใน PostgreSQL

```sql
-- PostgreSQL 12+: CTE ถูก inline โดย default (ไม่ materialized)
-- ใช้ MATERIALIZED เพื่อบังคับให้ execute ครั้งเดียว
WITH expensive_calc AS MATERIALIZED (
    SELECT 
        department,
        AVG(salary) AS avg_salary,
        PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY salary) AS median_salary
    FROM employees
    GROUP BY department
)
SELECT e.name, e.salary, ec.avg_salary, ec.median_salary
FROM employees e
JOIN expensive_calc ec ON e.department = ec.department;

-- NOT MATERIALIZED: บังคับให้ inline (ให้ optimizer ตัดสินใจ)
WITH always_inline AS NOT MATERIALIZED (
    SELECT * FROM employees WHERE salary > 50000
)
SELECT * FROM always_inline WHERE department = 'IT';
```

### ตัวอย่างที่ 20: เมื่อ CTE ถูกใช้หลายครั้ง

```sql
-- CTE ที่ถูกอ้างอิงหลายครั้ง - ควรใช้ MATERIALIZED
WITH dept_stats AS MATERIALIZED (
    SELECT 
        department,
        AVG(salary) AS avg_sal,
        STDDEV(salary) AS stddev_sal
    FROM employees
    GROUP BY department
)
SELECT 
    e.name,
    e.salary,
    d.avg_sal,
    d.stddev_sal,
    (e.salary - d.avg_sal) / NULLIF(d.stddev_sal, 0) AS z_score
FROM employees e
JOIN dept_stats d ON e.department = d.department
WHERE ABS((e.salary - d.avg_sal) / NULLIF(d.stddev_sal, 0)) > 1.5
ORDER BY ABS((e.salary - d.avg_sal) / NULLIF(d.stddev_sal, 0)) DESC;
```

---

## 91.10 ตัวอย่างขั้นสูง - Business Scenarios

### ตัวอย่างที่ 21: Customer Lifetime Value Analysis

```sql
CREATE TABLE orders (
    order_id    INT PRIMARY KEY,
    customer_id INT,
    order_date  DATE,
    total_amount DECIMAL(12,2)
);

CREATE TABLE customers (
    customer_id  INT PRIMARY KEY,
    name         VARCHAR(100),
    signup_date  DATE,
    country      VARCHAR(50)
);

-- (สมมติข้อมูลถูกใส่แล้ว)

-- คำนวณ CLV ด้วย CTEs
WITH 
customer_orders AS (
    SELECT 
        c.customer_id,
        c.name,
        c.signup_date,
        COUNT(o.order_id)      AS order_count,
        SUM(o.total_amount)    AS lifetime_value,
        MIN(o.order_date)      AS first_order,
        MAX(o.order_date)      AS last_order,
        AVG(o.total_amount)    AS avg_order_value
    FROM customers c
    LEFT JOIN orders o ON c.customer_id = o.customer_id
    GROUP BY c.customer_id, c.name, c.signup_date
),
customer_segments AS (
    SELECT *,
        NTILE(4) OVER (ORDER BY lifetime_value DESC) AS quartile
    FROM customer_orders
    WHERE order_count > 0
),
segment_labels AS (
    SELECT *,
        CASE quartile
            WHEN 1 THEN 'Champions'
            WHEN 2 THEN 'Loyal Customers'
            WHEN 3 THEN 'At Risk'
            WHEN 4 THEN 'Lost'
        END AS segment
    FROM customer_segments
)
SELECT 
    segment,
    COUNT(*)                          AS customer_count,
    ROUND(AVG(lifetime_value), 2)     AS avg_clv,
    ROUND(AVG(order_count), 1)        AS avg_orders,
    ROUND(AVG(avg_order_value), 2)    AS avg_order_value
FROM segment_labels
GROUP BY segment, quartile
ORDER BY quartile;
```

### ตัวอย่างที่ 22: Month-over-Month Growth Analysis

```sql
WITH monthly_sales AS (
    SELECT 
        DATE_TRUNC('month', order_date) AS month,
        COUNT(*)           AS order_count,
        SUM(total_amount)  AS revenue
    FROM orders
    GROUP BY DATE_TRUNC('month', order_date)
),
with_previous AS (
    SELECT 
        month,
        order_count,
        revenue,
        LAG(revenue) OVER (ORDER BY month) AS prev_revenue,
        LAG(order_count) OVER (ORDER BY month) AS prev_orders
    FROM monthly_sales
),
with_growth AS (
    SELECT 
        month,
        order_count,
        revenue,
        ROUND(
            (revenue - prev_revenue) * 100.0 / NULLIF(prev_revenue, 0), 
            2
        ) AS revenue_growth_pct,
        ROUND(
            (order_count - prev_orders) * 100.0 / NULLIF(prev_orders, 0),
            2
        ) AS order_growth_pct
    FROM with_previous
)
SELECT *
FROM with_growth
ORDER BY month;
```

### ตัวอย่างที่ 23: Funnel Analysis

```sql
CREATE TABLE user_events (
    event_id    INT PRIMARY KEY,
    user_id     INT,
    event_type  VARCHAR(50),
    event_time  TIMESTAMP
);

-- วิเคราะห์ funnel: view → add_to_cart → checkout → purchase
WITH 
event_users AS (
    SELECT user_id, event_type
    FROM user_events
    WHERE event_type IN ('view', 'add_to_cart', 'checkout', 'purchase')
    GROUP BY user_id, event_type
),
funnel_steps AS (
    SELECT 
        COUNT(DISTINCT CASE WHEN event_type = 'view' THEN user_id END) AS viewers,
        COUNT(DISTINCT CASE WHEN event_type = 'add_to_cart' THEN user_id END) AS cart_adders,
        COUNT(DISTINCT CASE WHEN event_type = 'checkout' THEN user_id END) AS checkouts,
        COUNT(DISTINCT CASE WHEN event_type = 'purchase' THEN user_id END) AS purchasers
    FROM event_users
),
funnel_rates AS (
    SELECT 
        viewers,
        cart_adders,
        checkouts,
        purchasers,
        ROUND(cart_adders * 100.0 / NULLIF(viewers, 0), 2)      AS view_to_cart_pct,
        ROUND(checkouts * 100.0 / NULLIF(cart_adders, 0), 2)    AS cart_to_checkout_pct,
        ROUND(purchasers * 100.0 / NULLIF(checkouts, 0), 2)     AS checkout_to_purchase_pct,
        ROUND(purchasers * 100.0 / NULLIF(viewers, 0), 2)       AS overall_conversion_pct
    FROM funnel_steps
)
SELECT * FROM funnel_rates;
```

### ตัวอย่างที่ 24: Inventory Reorder Analysis

```sql
CREATE TABLE inventory (
    product_id      INT PRIMARY KEY,
    product_name    VARCHAR(100),
    current_stock   INT,
    reorder_point   INT,
    reorder_quantity INT,
    unit_cost       DECIMAL(10,2)
);

CREATE TABLE daily_sales_rate (
    product_id      INT,
    avg_daily_sales DECIMAL(10,2)
);

WITH 
below_reorder AS (
    SELECT i.*, d.avg_daily_sales
    FROM inventory i
    JOIN daily_sales_rate d ON i.product_id = d.product_id
    WHERE i.current_stock <= i.reorder_point
),
days_remaining AS (
    SELECT *,
        CASE 
            WHEN avg_daily_sales > 0 
            THEN ROUND(current_stock / avg_daily_sales, 1)
            ELSE NULL
        END AS days_of_stock_remaining
    FROM below_reorder
),
priority_orders AS (
    SELECT *,
        reorder_quantity * unit_cost AS reorder_cost,
        CASE 
            WHEN days_of_stock_remaining <= 3 THEN 'URGENT'
            WHEN days_of_stock_remaining <= 7 THEN 'HIGH'
            ELSE 'NORMAL'
        END AS priority
    FROM days_remaining
)
SELECT 
    priority,
    product_id,
    product_name,
    current_stock,
    days_of_stock_remaining,
    reorder_quantity,
    reorder_cost
FROM priority_orders
ORDER BY 
    CASE priority WHEN 'URGENT' THEN 1 WHEN 'HIGH' THEN 2 ELSE 3 END,
    days_of_stock_remaining NULLS LAST;
```

### ตัวอย่างที่ 25: Employee Performance Dashboard

```sql
CREATE TABLE performance_reviews (
    review_id   INT PRIMARY KEY,
    employee_id INT,
    review_year INT,
    score       INT,    -- 1-5
    reviewer_id INT
);

WITH 
latest_reviews AS (
    SELECT 
        employee_id,
        review_year,
        AVG(score) AS avg_score
    FROM performance_reviews
    WHERE review_year = EXTRACT(YEAR FROM CURRENT_DATE) - 1
    GROUP BY employee_id, review_year
),
employee_performance AS (
    SELECT 
        e.employee_id,
        e.name,
        e.department,
        e.salary,
        lr.avg_score,
        PERCENT_RANK() OVER (
            PARTITION BY e.department 
            ORDER BY lr.avg_score
        ) AS dept_percentile
    FROM employees e
    LEFT JOIN latest_reviews lr ON e.employee_id = lr.employee_id
),
recommendations AS (
    SELECT *,
        CASE
            WHEN avg_score >= 4.5 THEN 'Fast Track Promotion'
            WHEN avg_score >= 4.0 THEN 'Standard Promotion'
            WHEN avg_score >= 3.0 THEN 'Merit Increase'
            WHEN avg_score >= 2.0 THEN 'Performance Plan'
            ELSE 'No Review Data'
        END AS recommendation
    FROM employee_performance
)
SELECT 
    department,
    COUNT(*) AS team_size,
    ROUND(AVG(avg_score), 2) AS team_avg_score,
    COUNT(CASE WHEN recommendation = 'Fast Track Promotion' THEN 1 END) AS fast_track,
    COUNT(CASE WHEN recommendation = 'Performance Plan' THEN 1 END) AS needs_improvement
FROM recommendations
GROUP BY department
ORDER BY team_avg_score DESC;
```

---

## 91.11 CTE Pattern สำหรับ Date Generation

### ตัวอย่างที่ 26: สร้าง Date Range ด้วย CTE

```sql
-- PostgreSQL: สร้าง calendar ด้วย generate_series
WITH date_range AS (
    SELECT 
        generate_series(
            '2024-01-01'::date,
            '2024-12-31'::date,
            '1 day'::interval
        )::date AS calendar_date
),
calendar AS (
    SELECT 
        calendar_date,
        EXTRACT(DOW FROM calendar_date) AS day_of_week,
        EXTRACT(MONTH FROM calendar_date) AS month,
        EXTRACT(QUARTER FROM calendar_date) AS quarter,
        TO_CHAR(calendar_date, 'Day') AS day_name,
        CASE WHEN EXTRACT(DOW FROM calendar_date) IN (0, 6) THEN TRUE ELSE FALSE END AS is_weekend
    FROM date_range
)
SELECT 
    month,
    COUNT(*) AS total_days,
    COUNT(CASE WHEN NOT is_weekend THEN 1 END) AS working_days,
    COUNT(CASE WHEN is_weekend THEN 1 END) AS weekend_days
FROM calendar
GROUP BY month
ORDER BY month;
```

### ตัวอย่างที่ 27: เติมวันที่หายไปใน time series

```sql
-- เติม 0 สำหรับวันที่ไม่มียอดขาย
WITH date_spine AS (
    SELECT generate_series(
        '2024-01-01'::date,
        '2024-01-31'::date,
        '1 day'
    )::date AS sales_date
),
actual_sales AS (
    SELECT 
        order_date,
        COUNT(*) AS order_count,
        SUM(total_amount) AS daily_revenue
    FROM orders
    WHERE order_date BETWEEN '2024-01-01' AND '2024-01-31'
    GROUP BY order_date
)
SELECT 
    d.sales_date,
    COALESCE(a.order_count, 0)    AS order_count,
    COALESCE(a.daily_revenue, 0)  AS daily_revenue
FROM date_spine d
LEFT JOIN actual_sales a ON d.sales_date = a.order_date
ORDER BY d.sales_date;
```

---

## 91.12 Advanced CTE Patterns

### ตัวอย่างที่ 28: Pivot แบบง่ายด้วย CTE

```sql
-- Pivot ยอดขายแบ่งตามไตรมาส
WITH quarterly_sales AS (
    SELECT 
        salesperson,
        EXTRACT(QUARTER FROM sale_date) AS quarter,
        SUM(quantity * unit_price) AS revenue
    FROM sales
    GROUP BY salesperson, EXTRACT(QUARTER FROM sale_date)
)
SELECT 
    salesperson,
    MAX(CASE WHEN quarter = 1 THEN revenue ELSE 0 END) AS Q1,
    MAX(CASE WHEN quarter = 2 THEN revenue ELSE 0 END) AS Q2,
    MAX(CASE WHEN quarter = 3 THEN revenue ELSE 0 END) AS Q3,
    MAX(CASE WHEN quarter = 4 THEN revenue ELSE 0 END) AS Q4
FROM quarterly_sales
GROUP BY salesperson
ORDER BY salesperson;
```

### ตัวอย่างที่ 29: CTE สำหรับ Running Total

```sql
WITH ordered_transactions AS (
    SELECT 
        txn_id,
        account_id,
        amount,
        txn_date,
        ROW_NUMBER() OVER (PARTITION BY account_id ORDER BY txn_date, txn_id) AS rn
    FROM transactions
),
running_balance AS (
    SELECT 
        txn_id,
        account_id,
        amount,
        txn_date,
        SUM(amount) OVER (
            PARTITION BY account_id
            ORDER BY txn_date, txn_id
            ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
        ) AS running_balance
    FROM transactions
)
SELECT * FROM running_balance ORDER BY account_id, txn_date;
```

### ตัวอย่างที่ 30: CTE สำหรับ String Aggregation

```sql
-- รวม skills ของแต่ละพนักงาน
CREATE TABLE employee_skills (
    employee_id INT,
    skill       VARCHAR(50)
);

INSERT INTO employee_skills VALUES
(1, 'Python'), (1, 'SQL'), (1, 'Excel'),
(2, 'SQL'), (2, 'PowerBI'),
(3, 'Java'), (3, 'SQL'), (3, 'Spring');

WITH skill_groups AS (
    SELECT 
        employee_id,
        STRING_AGG(skill, ', ' ORDER BY skill) AS skills_list,
        COUNT(*) AS skill_count
    FROM employee_skills
    GROUP BY employee_id
)
SELECT 
    e.name,
    e.department,
    sg.skill_count,
    sg.skills_list
FROM employees e
JOIN skill_groups sg ON e.employee_id = sg.employee_id
ORDER BY sg.skill_count DESC;
```

### ตัวอย่างที่ 31: CTE สำหรับ Cohort Retention

```sql
-- วิเคราะห์ retention ของลูกค้า
WITH 
first_purchase AS (
    SELECT 
        customer_id,
        DATE_TRUNC('month', MIN(order_date)) AS cohort_month
    FROM orders
    GROUP BY customer_id
),
monthly_activity AS (
    SELECT 
        o.customer_id,
        DATE_TRUNC('month', o.order_date) AS activity_month
    FROM orders o
    GROUP BY o.customer_id, DATE_TRUNC('month', o.order_date)
),
cohort_data AS (
    SELECT 
        fp.cohort_month,
        ma.activity_month,
        COUNT(DISTINCT ma.customer_id) AS active_customers,
        EXTRACT(YEAR FROM AGE(ma.activity_month, fp.cohort_month)) * 12 +
        EXTRACT(MONTH FROM AGE(ma.activity_month, fp.cohort_month)) AS months_since_cohort
    FROM first_purchase fp
    JOIN monthly_activity ma ON fp.customer_id = ma.customer_id
    WHERE ma.activity_month >= fp.cohort_month
    GROUP BY fp.cohort_month, ma.activity_month
),
cohort_sizes AS (
    SELECT cohort_month, COUNT(*) AS cohort_size
    FROM first_purchase
    GROUP BY cohort_month
)
SELECT 
    cd.cohort_month,
    cs.cohort_size,
    cd.months_since_cohort,
    cd.active_customers,
    ROUND(cd.active_customers * 100.0 / cs.cohort_size, 2) AS retention_rate
FROM cohort_data cd
JOIN cohort_sizes cs ON cd.cohort_month = cs.cohort_month
ORDER BY cd.cohort_month, cd.months_since_cohort;
```

---

## 91.13 CTE สำหรับ Data Quality Checks

### ตัวอย่างที่ 32: ตรวจสอบคุณภาพข้อมูลอย่างครอบคลุม

```sql
-- Data quality report ด้วย CTEs
WITH 
total_count AS (
    SELECT COUNT(*) AS n FROM employees
),
null_checks AS (
    SELECT
        SUM(CASE WHEN name IS NULL THEN 1 ELSE 0 END)       AS null_names,
        SUM(CASE WHEN email IS NULL THEN 1 ELSE 0 END)      AS null_emails,
        SUM(CASE WHEN department IS NULL THEN 1 ELSE 0 END) AS null_depts,
        SUM(CASE WHEN salary IS NULL THEN 1 ELSE 0 END)     AS null_salaries
    FROM employees
),
range_checks AS (
    SELECT
        SUM(CASE WHEN salary < 0 THEN 1 ELSE 0 END)          AS negative_salary,
        SUM(CASE WHEN salary > 1000000 THEN 1 ELSE 0 END)     AS unrealistic_salary,
        SUM(CASE WHEN hire_date > CURRENT_DATE THEN 1 ELSE 0 END) AS future_hire_date
    FROM employees
),
duplicate_check AS (
    SELECT COUNT(*) - COUNT(DISTINCT employee_id) AS duplicate_ids
    FROM employees
)
SELECT 
    t.n AS total_rows,
    n.null_names, n.null_emails, n.null_depts, n.null_salaries,
    r.negative_salary, r.unrealistic_salary, r.future_hire_date,
    d.duplicate_ids
FROM total_count t
CROSS JOIN null_checks n
CROSS JOIN range_checks r
CROSS JOIN duplicate_check d;
```

### ตัวอย่างที่ 33: CTE สำหรับ Anomaly Detection

```sql
-- ตรวจจับค่าผิดปกติด้วย Z-score
WITH stats AS (
    SELECT 
        department,
        AVG(salary)    AS mean_sal,
        STDDEV(salary) AS std_sal
    FROM employees
    GROUP BY department
),
z_scores AS (
    SELECT 
        e.employee_id,
        e.name,
        e.department,
        e.salary,
        s.mean_sal,
        s.std_sal,
        ABS(e.salary - s.mean_sal) / NULLIF(s.std_sal, 0) AS z_score
    FROM employees e
    JOIN stats s ON e.department = s.department
)
SELECT *
FROM z_scores
WHERE z_score > 2   -- เงินเดือนที่ผิดปกติ (> 2 standard deviations)
ORDER BY z_score DESC;
```

---

## 91.14 CTE สำหรับ Reporting

### ตัวอย่างที่ 34: Executive Summary Report

```sql
WITH 
-- ยอดขายเดือนนี้
current_month AS (
    SELECT 
        SUM(total_amount) AS revenue,
        COUNT(*) AS orders
    FROM orders
    WHERE DATE_TRUNC('month', order_date) = DATE_TRUNC('month', CURRENT_DATE)
),
-- ยอดขายเดือนที่แล้ว
prev_month AS (
    SELECT 
        SUM(total_amount) AS revenue,
        COUNT(*) AS orders
    FROM orders
    WHERE DATE_TRUNC('month', order_date) = DATE_TRUNC('month', CURRENT_DATE) - INTERVAL '1 month'
),
-- ลูกค้าใหม่เดือนนี้
new_customers AS (
    SELECT COUNT(DISTINCT customer_id) AS count
    FROM customers
    WHERE DATE_TRUNC('month', signup_date) = DATE_TRUNC('month', CURRENT_DATE)
),
-- สินค้าขายดีสุด
top_product AS (
    SELECT product, SUM(quantity) AS total_qty
    FROM sales
    WHERE sale_date >= DATE_TRUNC('month', CURRENT_DATE)
    GROUP BY product
    ORDER BY total_qty DESC
    LIMIT 1
)
SELECT 
    cm.revenue AS this_month_revenue,
    pm.revenue AS last_month_revenue,
    ROUND((cm.revenue - pm.revenue) * 100.0 / NULLIF(pm.revenue, 0), 2) AS mom_growth_pct,
    cm.orders AS this_month_orders,
    nc.count AS new_customers,
    tp.product AS best_selling_product
FROM current_month cm
CROSS JOIN prev_month pm
CROSS JOIN new_customers nc
CROSS JOIN top_product tp;
```

### ตัวอย่างที่ 35: Regional Sales Breakdown

```sql
WITH 
regional_sales AS (
    SELECT 
        region,
        DATE_TRUNC('month', sale_date) AS month,
        SUM(quantity * unit_price) AS revenue
    FROM sales
    GROUP BY region, DATE_TRUNC('month', sale_date)
),
regional_totals AS (
    SELECT region, SUM(revenue) AS total_revenue
    FROM regional_sales
    GROUP BY region
),
grand_total AS (
    SELECT SUM(total_revenue) AS grand_total
    FROM regional_totals
),
with_share AS (
    SELECT 
        rt.region,
        rt.total_revenue,
        ROUND(rt.total_revenue * 100.0 / gt.grand_total, 2) AS market_share,
        RANK() OVER (ORDER BY rt.total_revenue DESC) AS rank
    FROM regional_totals rt
    CROSS JOIN grand_total gt
)
SELECT * FROM with_share ORDER BY rank;
```

---

## 91.15 ตัวอย่างเพิ่มเติม

### ตัวอย่างที่ 36: CTE สำหรับ Graph Traversal แบบง่าย

```sql
-- หา dependencies ของ project tasks
CREATE TABLE task_dependencies (
    task_id         INT,
    depends_on_task INT
);

INSERT INTO task_dependencies VALUES
(2, 1), (3, 1), (4, 2), (4, 3), (5, 4);

-- หา tasks ทั้งหมดที่ต้องทำก่อน task 5
WITH RECURSIVE prereqs AS (
    -- Base case: tasks ที่ task 5 ขึ้นตรง
    SELECT depends_on_task AS task_id, 1 AS level
    FROM task_dependencies
    WHERE task_id = 5
    
    UNION ALL
    
    -- Recursive: prerequisites ของ prerequisites
    SELECT td.depends_on_task, p.level + 1
    FROM task_dependencies td
    JOIN prereqs p ON td.task_id = p.task_id
)
SELECT DISTINCT task_id, level
FROM prereqs
ORDER BY level, task_id;
```

### ตัวอย่างที่ 37: CTE สำหรับ Pagination

```sql
-- Pagination ด้วย CTE
WITH paginated AS (
    SELECT 
        employee_id,
        name,
        department,
        salary,
        ROW_NUMBER() OVER (ORDER BY employee_id) AS row_num
    FROM employees
),
page_info AS (
    SELECT 
        COUNT(*) AS total_rows,
        CEIL(COUNT(*) * 1.0 / 3) AS total_pages  -- 3 แถวต่อหน้า
    FROM employees
)
SELECT 
    p.employee_id, p.name, p.department, p.salary,
    pi.total_rows, pi.total_pages,
    CEIL(p.row_num * 1.0 / 3) AS page_number
FROM paginated p
CROSS JOIN page_info pi
WHERE p.row_num BETWEEN 4 AND 6;  -- หน้าที่ 2 (แถว 4-6)
```

### ตัวอย่างที่ 38: CTE สำหรับ Email Report Generation

```sql
-- สร้าง email body สำหรับส่งรายงาน
WITH 
summary_stats AS (
    SELECT 
        COUNT(*) AS total_emp,
        SUM(CASE WHEN department = 'IT' THEN 1 ELSE 0 END) AS it_count,
        AVG(salary) AS avg_salary
    FROM employees
),
highest_paid AS (
    SELECT name, salary
    FROM employees
    ORDER BY salary DESC
    LIMIT 1
)
SELECT 
    'รายงานสรุปพนักงาน' AS report_title,
    CURRENT_TIMESTAMP AS generated_at,
    s.total_emp || ' คน' AS total_employees,
    s.it_count || ' คน' AS it_employees,
    'THB ' || ROUND(s.avg_salary, 2) AS average_salary,
    h.name || ' (THB ' || h.salary || ')' AS highest_paid_employee
FROM summary_stats s
CROSS JOIN highest_paid h;
```

### ตัวอย่างที่ 39: CTE เพื่อ Debug Complex Queries

```sql
-- ใช้ CTE เป็น "checkpoint" เพื่อ debug
WITH 
step1_check AS (
    SELECT employee_id, name, salary
    FROM employees
    -- uncomment เพื่อดูผลลัพธ์ขั้นนี้: SELECT * FROM step1_check;
),
step2_check AS (
    SELECT *, 
           salary * 1.1 AS new_salary,
           'Raise applied' AS note
    FROM step1_check
    WHERE salary < 75000
    -- uncomment: SELECT * FROM step2_check;
),
step3_final AS (
    SELECT employee_id, name, new_salary
    FROM step2_check
)
SELECT * FROM step3_final;
-- เพื่อ debug: เปลี่ยน FROM step3_final เป็น FROM step1_check หรือ step2_check
```

### ตัวอย่างที่ 40: CTE ที่ใช้กับ UNION

```sql
-- รวม report จากหลาย CTE ด้วย UNION
WITH 
dept_IT AS (
    SELECT 'IT Department' AS category, COUNT(*) AS headcount, AVG(salary) AS avg_salary
    FROM employees WHERE department = 'IT'
),
dept_HR AS (
    SELECT 'HR Department', COUNT(*), AVG(salary)
    FROM employees WHERE department = 'HR'
),
dept_Finance AS (
    SELECT 'Finance Department', COUNT(*), AVG(salary)
    FROM employees WHERE department = 'Finance'
),
all_company AS (
    SELECT 'Company Total', COUNT(*), AVG(salary)
    FROM employees
)
SELECT * FROM dept_IT
UNION ALL
SELECT * FROM dept_HR
UNION ALL
SELECT * FROM dept_Finance
UNION ALL
SELECT * FROM all_company;
```

---

## แบบฝึกหัดท้ายบท

### แบบฝึกหัดที่ 1
เขียน CTE เพื่อหาพนักงาน 3 อันดับแรกที่มีเงินเดือนสูงสุดในแต่ละแผนก

**คำตอบ:**
```sql
WITH ranked_employees AS (
    SELECT 
        name,
        department,
        salary,
        DENSE_RANK() OVER (PARTITION BY department ORDER BY salary DESC) AS salary_rank
    FROM employees
)
SELECT name, department, salary, salary_rank
FROM ranked_employees
WHERE salary_rank <= 3
ORDER BY department, salary_rank;
```

### แบบฝึกหัดที่ 2
สร้าง CTE ที่แสดงพนักงานที่มีเงินเดือนสูงกว่าค่าเฉลี่ยของแผนกตัวเองและสูงกว่าค่าเฉลี่ยของบริษัท

**คำตอบ:**
```sql
WITH 
dept_avg AS (
    SELECT department, AVG(salary) AS dept_avg_salary
    FROM employees
    GROUP BY department
),
company_avg AS (
    SELECT AVG(salary) AS company_avg_salary
    FROM employees
)
SELECT 
    e.name,
    e.department,
    e.salary,
    ROUND(da.dept_avg_salary, 2) AS dept_average,
    ROUND(ca.company_avg_salary, 2) AS company_average
FROM employees e
JOIN dept_avg da ON e.department = da.department
CROSS JOIN company_avg ca
WHERE e.salary > da.dept_avg_salary
  AND e.salary > ca.company_avg_salary
ORDER BY e.salary DESC;
```

### แบบฝึกหัดที่ 3
เขียน CTE pipeline ที่คำนวณ: (1) ยอดขายรายเดือน (2) การเติบโต MoM (3) สถานะการเติบโต (เพิ่ม/ลด/คงที่)

**คำตอบ:**
```sql
WITH 
monthly_totals AS (
    SELECT 
        DATE_TRUNC('month', sale_date) AS month,
        SUM(quantity * unit_price) AS revenue
    FROM sales
    GROUP BY DATE_TRUNC('month', sale_date)
),
with_growth AS (
    SELECT 
        month,
        revenue,
        LAG(revenue) OVER (ORDER BY month) AS prev_month_revenue,
        ROUND(
            (revenue - LAG(revenue) OVER (ORDER BY month)) * 100.0 /
            NULLIF(LAG(revenue) OVER (ORDER BY month), 0),
        2) AS growth_pct
    FROM monthly_totals
),
categorized AS (
    SELECT *,
        CASE
            WHEN growth_pct > 5    THEN 'เพิ่มขึ้นมาก'
            WHEN growth_pct > 0    THEN 'เพิ่มขึ้นเล็กน้อย'
            WHEN growth_pct = 0    THEN 'คงที่'
            WHEN growth_pct IS NULL THEN 'ไม่มีข้อมูลเดือนก่อน'
            ELSE 'ลดลง'
        END AS growth_status
    FROM with_growth
)
SELECT month, revenue, prev_month_revenue, growth_pct, growth_status
FROM categorized
ORDER BY month;
```

### แบบฝึกหัดที่ 4
ใช้ CTE เพื่อลบ duplicate records ออกจากตาราง โดยเก็บไว้เฉพาะ record ที่มี ID ต่ำที่สุด

**คำตอบ:**
```sql
-- สมมติตาราง contacts มี duplicate ที่ email เหมือนกัน
WITH duplicates_to_remove AS (
    SELECT contact_id,
           ROW_NUMBER() OVER (
               PARTITION BY email 
               ORDER BY contact_id ASC
           ) AS rn
    FROM contacts
)
DELETE FROM contacts
WHERE contact_id IN (
    SELECT contact_id 
    FROM duplicates_to_remove 
    WHERE rn > 1
);
```

### แบบฝึกหัดที่ 5
เขียน CTE สำหรับสร้าง Report ที่แสดง: แผนก, จำนวนพนักงาน, เงินเดือนเฉลี่ย, พนักงานที่เงินเดือนสูงสุด, พนักงานที่เงินเดือนต่ำสุด

**คำตอบ:**
```sql
WITH 
dept_stats AS (
    SELECT 
        department,
        COUNT(*) AS headcount,
        AVG(salary) AS avg_salary
    FROM employees
    GROUP BY department
),
dept_top AS (
    SELECT DISTINCT ON (department) 
        department,
        name AS top_earner,
        salary AS top_salary
    FROM employees
    ORDER BY department, salary DESC
),
dept_bottom AS (
    SELECT DISTINCT ON (department)
        department,
        name AS lowest_earner,
        salary AS lowest_salary
    FROM employees
    ORDER BY department, salary ASC
)
SELECT 
    s.department,
    s.headcount,
    ROUND(s.avg_salary, 2) AS avg_salary,
    t.top_earner,
    t.top_salary,
    b.lowest_earner,
    b.lowest_salary
FROM dept_stats s
JOIN dept_top t USING (department)
JOIN dept_bottom b USING (department)
ORDER BY s.avg_salary DESC;
```

### แบบฝึกหัดที่ 6
สร้าง CTE ที่วิเคราะห์ว่าพนักงานคนใดมีเงินเดือนที่สอง (second highest) ในแต่ละแผนก

**คำตอบ:**
```sql
WITH salary_ranked AS (
    SELECT 
        employee_id,
        name,
        department,
        salary,
        DENSE_RANK() OVER (PARTITION BY department ORDER BY salary DESC) AS rnk
    FROM employees
)
SELECT employee_id, name, department, salary
FROM salary_ranked
WHERE rnk = 2
ORDER BY department;
```

### แบบฝึกหัดที่ 7
เขียน CTE ที่คำนวณ cumulative percentage ของยอดขายแยกตาม region (แสดงแต่ละ region และ % สะสมจนถึง region นั้น)

**คำตอบ:**
```sql
WITH 
regional_totals AS (
    SELECT 
        region,
        SUM(quantity * unit_price) AS revenue
    FROM sales
    GROUP BY region
),
grand_total AS (
    SELECT SUM(revenue) AS total FROM regional_totals
),
with_pct AS (
    SELECT 
        region,
        revenue,
        ROUND(revenue * 100.0 / t.total, 2) AS pct_of_total,
        SUM(revenue) OVER (ORDER BY revenue DESC) AS cumulative_revenue,
        ROUND(
            SUM(revenue) OVER (ORDER BY revenue DESC) * 100.0 / t.total,
        2) AS cumulative_pct
    FROM regional_totals
    CROSS JOIN grand_total t
)
SELECT 
    region,
    revenue,
    pct_of_total,
    cumulative_revenue,
    cumulative_pct
FROM with_pct
ORDER BY revenue DESC;
```

### แบบฝึกหัดที่ 8
ใช้ CTE เพื่อหา "gaps" ในหมายเลข order_id (หมายเลขที่หายไป)

**คำตอบ:**
```sql
WITH 
id_range AS (
    SELECT 
        MIN(order_id) AS min_id,
        MAX(order_id) AS max_id
    FROM orders
),
all_ids AS (
    SELECT generate_series(min_id, max_id) AS expected_id
    FROM id_range
),
missing_ids AS (
    SELECT a.expected_id
    FROM all_ids a
    LEFT JOIN orders o ON a.expected_id = o.order_id
    WHERE o.order_id IS NULL
)
SELECT 
    expected_id AS missing_order_id,
    expected_id - 1 AS previous_existing_id,
    expected_id + 1 AS next_existing_id
FROM missing_ids
ORDER BY expected_id;
```

### แบบฝึกหัดที่ 9
สร้าง CTE ที่แสดงผลลัพธ์เป็น "email digest" สรุปข้อมูลพนักงานใหม่ที่เพิ่งจ้างในเดือนนี้

**คำตอบ:**
```sql
WITH 
new_hires AS (
    SELECT 
        name,
        department,
        salary,
        hire_date,
        ROW_NUMBER() OVER (ORDER BY hire_date DESC) AS hire_order
    FROM employees
    WHERE DATE_TRUNC('month', hire_date) = DATE_TRUNC('month', CURRENT_DATE)
),
summary AS (
    SELECT 
        COUNT(*) AS total_new_hires,
        STRING_AGG(name || ' (' || department || ')', ', ' ORDER BY hire_date) AS hire_list
    FROM new_hires
)
SELECT 
    '🎉 พนักงานใหม่ประจำเดือน ' || TO_CHAR(CURRENT_DATE, 'Month YYYY') AS title,
    'มีพนักงานใหม่ทั้งหมด ' || s.total_new_hires || ' คน' AS summary,
    'รายชื่อ: ' || s.hire_list AS details
FROM summary s;
```

### แบบฝึกหัดที่ 10
เขียน CTE ที่ซับซ้อนเพื่อสร้าง "Department Scorecard" ที่แสดงคะแนนรวมจาก: จำนวนพนักงาน (30%), เงินเดือนเฉลี่ย (40%), อัตราการเก็บรักษาพนักงาน (30%)

**คำตอบ:**
```sql
WITH 
dept_headcount AS (
    SELECT department, COUNT(*) AS headcount
    FROM employees
    GROUP BY department
),
dept_salary AS (
    SELECT department, AVG(salary) AS avg_salary
    FROM employees
    GROUP BY department
),
dept_tenure AS (
    SELECT 
        department,
        AVG(EXTRACT(YEAR FROM AGE(CURRENT_DATE, hire_date))) AS avg_tenure_years
    FROM employees
    GROUP BY department
),
max_values AS (
    SELECT 
        MAX(h.headcount) AS max_headcount,
        MAX(s.avg_salary) AS max_avg_salary,
        MAX(t.avg_tenure_years) AS max_tenure
    FROM dept_headcount h
    CROSS JOIN dept_salary s
    CROSS JOIN dept_tenure t
    LIMIT 1  -- ใช้ DISTINCT หรือ LIMIT 1 เพื่อหลีกเลี่ยง Cartesian product ที่ไม่ต้องการ
),
scores AS (
    SELECT 
        h.department,
        h.headcount,
        ROUND(s.avg_salary, 2) AS avg_salary,
        ROUND(t.avg_tenure_years, 1) AS avg_tenure,
        -- คำนวณคะแนน (normalize เป็น 0-100 แล้วถ่วงน้ำหนัก)
        ROUND(
            (h.headcount * 1.0 / m.max_headcount * 100) * 0.30 +
            (s.avg_salary / m.max_avg_salary * 100) * 0.40 +
            (t.avg_tenure_years / NULLIF(m.max_tenure, 0) * 100) * 0.30
        , 2) AS dept_score
    FROM dept_headcount h
    JOIN dept_salary s USING (department)
    JOIN dept_tenure t USING (department)
    CROSS JOIN max_values m
)
SELECT 
    department,
    headcount,
    avg_salary,
    avg_tenure,
    dept_score,
    RANK() OVER (ORDER BY dept_score DESC) AS score_rank
FROM scores
ORDER BY dept_score DESC;
```

---

## สรุปบทที่ 91

CTEs เป็นเครื่องมือที่ทรงพลังสำหรับ:
- **ความอ่านง่าย**: แบ่งงานซับซ้อนเป็นขั้นตอนที่มีชื่อชัดเจน
- **การนำกลับมาใช้**: อ้างอิง CTE เดียวกันหลายครั้งใน query
- **การ Debug**: ตรวจสอบผลลัพธ์ทีละขั้นตอน
- **Multi-step Transformation**: แปลงข้อมูลผ่านหลาย pipeline
- **Deduplication**: ระบุและลบข้อมูลซ้ำ

ในบทถัดไป เราจะเรียนรู้เกี่ยวกับ Recursive CTEs ซึ่งเปิดความสามารถในการ traverse ข้อมูลแบบลำดับชั้น เช่น org charts, category trees, และ bill of materials
