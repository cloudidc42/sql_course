# Part 081: Views - Virtual Tables (วิวส์ - ตารางเสมือน)

## บทนำ

Views (วิวส์) คือตารางเสมือน (Virtual Tables) ที่ถูกนิยามโดย SQL query แทนที่จะเก็บข้อมูลจริงๆ เหมือนตารางปกติ View จะเก็บเพียง query definition เอาไว้ และทุกครั้งที่คุณเรียกใช้ View มันจะรัน query นั้นและส่งผลลัพธ์กลับมา

Views เปรียบเสมือนหน้าต่างที่มองเข้าไปยังข้อมูล โดยสามารถแสดงข้อมูลบางส่วน เปลี่ยนรูปแบบ หรือรวมข้อมูลจากหลายตารางเข้าด้วยกัน

---

## 1. CREATE VIEW Syntax (ไวยากรณ์การสร้าง View)

### 1.1 รูปแบบพื้นฐาน

```sql
-- PostgreSQL / MySQL / SQL Server
CREATE VIEW view_name AS
SELECT column1, column2, ...
FROM table_name
WHERE condition;

-- ตัวอย่างที่ 1: View พื้นฐาน
CREATE VIEW active_employees AS
SELECT employee_id, first_name, last_name, email, department_id
FROM employees
WHERE status = 'active';

-- เรียกใช้ View เหมือนตารางปกติ
SELECT * FROM active_employees;
SELECT first_name, last_name FROM active_employees WHERE department_id = 10;
```

### 1.2 CREATE OR REPLACE VIEW

```sql
-- PostgreSQL
CREATE OR REPLACE VIEW active_employees AS
SELECT employee_id, first_name, last_name, email, department_id, hire_date
FROM employees
WHERE status = 'active';

-- MySQL
CREATE OR REPLACE VIEW active_employees AS
SELECT employee_id, first_name, last_name, email, department_id, hire_date
FROM employees
WHERE status = 'active';

-- SQL Server ไม่มี OR REPLACE ต้องใช้ ALTER VIEW
ALTER VIEW active_employees AS
SELECT employee_id, first_name, last_name, email, department_id, hire_date
FROM employees
WHERE status = 'active';
```

### 1.3 การตั้งชื่อ Column ใน View

```sql
-- ตัวอย่างที่ 2: กำหนดชื่อ column ใน view
CREATE VIEW employee_summary (emp_id, full_name, dept, salary_grade) AS
SELECT 
    employee_id,
    first_name || ' ' || last_name,
    department_name,
    CASE 
        WHEN salary < 30000 THEN 'Junior'
        WHEN salary < 60000 THEN 'Mid-level'
        ELSE 'Senior'
    END
FROM employees e
JOIN departments d ON e.department_id = d.department_id;
```

---

## 2. ทำไมต้องใช้ Views (Why Use Views)

### 2.1 ความปลอดภัย (Security)

```sql
-- ตัวอย่างที่ 3: ซ่อนข้อมูลที่ละเอียดอ่อน
-- ตารางต้นฉบับมี salary และข้อมูลส่วนตัว
CREATE TABLE employees (
    employee_id INT PRIMARY KEY,
    first_name VARCHAR(50),
    last_name VARCHAR(50),
    email VARCHAR(100),
    salary DECIMAL(10,2),
    ssn VARCHAR(11),        -- เลขประจำตัวประชาชน
    bank_account VARCHAR(20),
    department_id INT,
    hire_date DATE,
    status VARCHAR(20)
);

-- สร้าง View ที่ไม่แสดงข้อมูลที่ละเอียดอ่อน
CREATE VIEW employees_public AS
SELECT 
    employee_id,
    first_name,
    last_name,
    email,
    department_id,
    hire_date
FROM employees
WHERE status = 'active';

-- ให้สิทธิ์ผู้ใช้ทั่วไปเข้าถึงเฉพาะ View
GRANT SELECT ON employees_public TO public_user;

-- ไม่ให้สิทธิ์เข้าตารางต้นฉบับ
REVOKE ALL ON employees FROM public_user;
```

### 2.2 ความเรียบง่าย (Simplicity)

```sql
-- ตัวอย่างที่ 4: ซ่อนความซับซ้อน
-- Query ซับซ้อนที่ต้อง JOIN หลายตาราง
CREATE VIEW order_details_full AS
SELECT 
    o.order_id,
    o.order_date,
    c.first_name || ' ' || c.last_name AS customer_name,
    c.email AS customer_email,
    p.product_name,
    p.category,
    od.quantity,
    od.unit_price,
    od.quantity * od.unit_price AS line_total,
    o.status AS order_status,
    s.shipper_name,
    o.shipped_date
FROM orders o
JOIN customers c ON o.customer_id = c.customer_id
JOIN order_details od ON o.order_id = od.order_id
JOIN products p ON od.product_id = p.product_id
LEFT JOIN shippers s ON o.shipper_id = s.shipper_id;

-- ตอนนี้ query ง่ายมาก
SELECT * FROM order_details_full WHERE customer_name LIKE 'สมชาย%';
```

### 2.3 การนำกลับมาใช้ใหม่ (Reusability)

```sql
-- ตัวอย่างที่ 5: สร้าง base view ที่ใช้ซ้ำได้
CREATE VIEW active_products AS
SELECT 
    product_id,
    product_name,
    category_id,
    unit_price,
    units_in_stock,
    supplier_id
FROM products
WHERE discontinued = 0
  AND units_in_stock > 0;

-- นำ View ไปใช้ใน Query อื่นๆ
-- Query 1: สินค้าที่ราคาต่ำกว่า 100
SELECT product_name, unit_price FROM active_products WHERE unit_price < 100;

-- Query 2: จำนวนสินค้าต่อหมวดหมู่
SELECT category_id, COUNT(*) as product_count 
FROM active_products 
GROUP BY category_id;

-- Query 3: สินค้าที่ stock น้อย
SELECT product_name, units_in_stock 
FROM active_products 
WHERE units_in_stock < 10;
```

---

## 3. Simple Views (View พื้นฐาน)

```sql
-- ตัวอย่างที่ 6: View กรองข้อมูลเฉพาะแผนก
CREATE VIEW it_department_employees AS
SELECT employee_id, first_name, last_name, position, hire_date
FROM employees
WHERE department_id = (
    SELECT department_id FROM departments WHERE department_name = 'IT'
);

-- ตัวอย่างที่ 7: View แสดงเฉพาะ column ที่จำเป็น
CREATE VIEW customer_contact_list AS
SELECT 
    customer_id,
    company_name,
    contact_name,
    phone,
    email,
    city,
    country
FROM customers
WHERE active = TRUE
ORDER BY company_name;

-- ตัวอย่างที่ 8: View ที่มีการคำนวณ
CREATE VIEW product_inventory_value AS
SELECT 
    product_id,
    product_name,
    unit_price,
    units_in_stock,
    unit_price * units_in_stock AS inventory_value,
    CASE 
        WHEN units_in_stock = 0 THEN 'Out of Stock'
        WHEN units_in_stock < 10 THEN 'Low Stock'
        WHEN units_in_stock < 50 THEN 'Medium Stock'
        ELSE 'Well Stocked'
    END AS stock_status
FROM products;

-- ตัวอย่างที่ 9: View สำหรับรายงานยอดขายรายวัน
CREATE VIEW daily_sales_summary AS
SELECT 
    DATE(order_date) AS sale_date,
    COUNT(DISTINCT order_id) AS total_orders,
    COUNT(DISTINCT customer_id) AS unique_customers,
    SUM(total_amount) AS daily_revenue
FROM orders
WHERE status != 'cancelled'
GROUP BY DATE(order_date);

-- ตัวอย่างที่ 10: View สำหรับสินค้าที่ต้องสั่งเพิ่ม
CREATE VIEW reorder_list AS
SELECT 
    p.product_id,
    p.product_name,
    p.units_in_stock,
    p.reorder_level,
    p.units_on_order,
    s.company_name AS supplier_name,
    s.phone AS supplier_phone
FROM products p
JOIN suppliers s ON p.supplier_id = s.supplier_id
WHERE p.units_in_stock <= p.reorder_level
  AND p.discontinued = 0;
```

---

## 4. Complex Views with JOINs and Aggregation (View ที่ซับซ้อน)

```sql
-- ตัวอย่างที่ 11: View รายงานพนักงานพร้อมข้อมูลแผนก
CREATE VIEW employee_department_report AS
SELECT 
    e.employee_id,
    e.first_name,
    e.last_name,
    e.position,
    e.salary,
    e.hire_date,
    d.department_name,
    d.location,
    m.first_name || ' ' || m.last_name AS manager_name,
    DATEDIFF(CURDATE(), e.hire_date) / 365 AS years_of_service
FROM employees e
JOIN departments d ON e.department_id = d.department_id
LEFT JOIN employees m ON e.manager_id = m.employee_id;

-- ตัวอย่างที่ 12: View สรุปยอดขายต่อ Sales Rep
CREATE VIEW sales_rep_performance AS
SELECT 
    e.employee_id,
    e.first_name || ' ' || e.last_name AS sales_rep,
    COUNT(o.order_id) AS total_orders,
    COUNT(DISTINCT o.customer_id) AS unique_customers,
    SUM(od.quantity * od.unit_price) AS total_revenue,
    AVG(od.quantity * od.unit_price) AS avg_order_value,
    MAX(o.order_date) AS last_order_date
FROM employees e
LEFT JOIN orders o ON e.employee_id = o.employee_id
LEFT JOIN order_details od ON o.order_id = od.order_id
WHERE e.position LIKE '%Sales%'
GROUP BY e.employee_id, e.first_name, e.last_name;

-- ตัวอย่างที่ 13: View สำหรับรายงานสินค้าขายดี Top 10
CREATE VIEW top_selling_products AS
SELECT 
    p.product_id,
    p.product_name,
    c.category_name,
    SUM(od.quantity) AS total_quantity_sold,
    SUM(od.quantity * od.unit_price) AS total_revenue,
    COUNT(DISTINCT od.order_id) AS number_of_orders,
    RANK() OVER (ORDER BY SUM(od.quantity) DESC) AS sales_rank
FROM products p
JOIN categories c ON p.category_id = c.category_id
JOIN order_details od ON p.product_id = od.product_id
JOIN orders o ON od.order_id = o.order_id
WHERE o.status = 'completed'
GROUP BY p.product_id, p.product_name, c.category_name;

-- ตัวอย่างที่ 14: View Customer Lifetime Value
CREATE VIEW customer_lifetime_value AS
SELECT 
    c.customer_id,
    c.first_name || ' ' || c.last_name AS customer_name,
    c.email,
    c.city,
    c.country,
    COUNT(o.order_id) AS total_orders,
    SUM(o.total_amount) AS lifetime_value,
    AVG(o.total_amount) AS avg_order_value,
    MIN(o.order_date) AS first_order_date,
    MAX(o.order_date) AS last_order_date,
    CASE 
        WHEN SUM(o.total_amount) > 10000 THEN 'Gold'
        WHEN SUM(o.total_amount) > 5000 THEN 'Silver'
        ELSE 'Bronze'
    END AS customer_tier
FROM customers c
LEFT JOIN orders o ON c.customer_id = o.customer_id
WHERE o.status = 'completed' OR o.status IS NULL
GROUP BY c.customer_id, c.first_name, c.last_name, c.email, c.city, c.country;

-- ตัวอย่างที่ 15: View รายงานการเงินรายเดือน
CREATE VIEW monthly_financial_report AS
SELECT 
    YEAR(o.order_date) AS year,
    MONTH(o.order_date) AS month,
    MONTHNAME(o.order_date) AS month_name,
    COUNT(DISTINCT o.order_id) AS total_orders,
    SUM(od.quantity * od.unit_price) AS gross_revenue,
    SUM(od.quantity * od.unit_price * (1 - od.discount)) AS net_revenue,
    SUM(od.quantity * p.unit_cost) AS total_cost,
    SUM(od.quantity * od.unit_price * (1 - od.discount)) - 
    SUM(od.quantity * p.unit_cost) AS gross_profit,
    (SUM(od.quantity * od.unit_price * (1 - od.discount)) - 
     SUM(od.quantity * p.unit_cost)) / 
    NULLIF(SUM(od.quantity * od.unit_price * (1 - od.discount)), 0) * 100 AS profit_margin_pct
FROM orders o
JOIN order_details od ON o.order_id = od.order_id
JOIN products p ON od.product_id = p.product_id
WHERE o.status = 'completed'
GROUP BY YEAR(o.order_date), MONTH(o.order_date), MONTHNAME(o.order_date)
ORDER BY year, month;
```

---

## 5. WITH CHECK OPTION

`WITH CHECK OPTION` ใช้เพื่อป้องกันไม่ให้ INSERT หรือ UPDATE ผ่าน View สร้างหรือแก้ไขข้อมูลที่ไม่ตรงกับ WHERE clause ของ View

```sql
-- ตัวอย่างที่ 16: WITH CHECK OPTION พื้นฐาน
CREATE VIEW active_products_only AS
SELECT product_id, product_name, unit_price, discontinued
FROM products
WHERE discontinued = 0
WITH CHECK OPTION;

-- นี่จะ ERROR เพราะ discontinued = 1 ไม่ตรงกับ WHERE clause
INSERT INTO active_products_only (product_name, unit_price, discontinued)
VALUES ('Test Product', 99.99, 1);
-- ERROR: Check option violated for view 'active_products_only'

-- นี่จะสำเร็จ
INSERT INTO active_products_only (product_name, unit_price, discontinued)
VALUES ('New Active Product', 99.99, 0);

-- ตัวอย่างที่ 17: LOCAL vs CASCADED CHECK OPTION
-- CASCADED (default) - ตรวจสอบทั้ง view ปัจจุบันและ view ที่ base
CREATE VIEW it_dept_employees AS
SELECT employee_id, first_name, last_name, department_id, salary
FROM employees
WHERE department_id = 10;

-- View ที่ซ้อนบน it_dept_employees
CREATE VIEW senior_it_employees AS
SELECT * FROM it_dept_employees
WHERE salary > 50000
WITH CASCADED CHECK OPTION;
-- CASCADED = ตรวจสอบเงื่อนไขทั้ง salary > 50000 AND department_id = 10

-- LOCAL - ตรวจสอบเฉพาะ view ปัจจุบัน
CREATE VIEW senior_it_employees_local AS
SELECT * FROM it_dept_employees
WHERE salary > 50000
WITH LOCAL CHECK OPTION;
-- LOCAL = ตรวจสอบเฉพาะ salary > 50000 (ไม่ตรวจ department_id)

-- ตัวอย่างที่ 18: WITH CHECK OPTION กับ View ที่มีเงื่อนไขซับซ้อน
CREATE VIEW current_year_orders AS
SELECT order_id, customer_id, order_date, total_amount, status
FROM orders
WHERE YEAR(order_date) = YEAR(CURDATE())
  AND status != 'cancelled'
WITH CHECK OPTION;

-- INSERT นี้จะ fail ถ้าปีไม่ตรง
INSERT INTO current_year_orders (customer_id, order_date, total_amount, status)
VALUES (1, '2020-01-01', 500.00, 'pending');
-- ERROR: ปี 2020 ไม่ตรงกับปีปัจจุบัน
```

---

## 6. Dropping and Altering Views

```sql
-- ตัวอย่างที่ 19: DROP VIEW
DROP VIEW active_employees;

-- ถ้า View ไม่มีอยู่จะ ERROR ใช้ IF EXISTS เพื่อป้องกัน
DROP VIEW IF EXISTS active_employees;

-- ลบหลาย Views พร้อมกัน
DROP VIEW IF EXISTS 
    active_employees,
    order_details_full,
    customer_lifetime_value;

-- ตัวอย่างที่ 20: ALTER VIEW (SQL Server)
ALTER VIEW active_employees AS
SELECT employee_id, first_name, last_name, email, department_id, hire_date, status
FROM employees
WHERE status = 'active'
  AND termination_date IS NULL;

-- PostgreSQL ใช้ CREATE OR REPLACE VIEW
CREATE OR REPLACE VIEW active_employees AS
SELECT employee_id, first_name, last_name, email, department_id, hire_date, status
FROM employees
WHERE status = 'active'
  AND termination_date IS NULL;

-- MySQL ใช้ CREATE OR REPLACE VIEW หรือ ALTER VIEW
ALTER VIEW active_employees AS
SELECT employee_id, first_name, last_name, email, department_id, hire_date, status
FROM employees
WHERE status = 'active'
  AND termination_date IS NULL;

-- ตัวอย่างที่ 21: เปลี่ยน View options
-- PostgreSQL - เปลี่ยน security_barrier option
ALTER VIEW employee_salary_view SET (security_barrier = true);

-- SQL Server - เพิ่ม SCHEMABINDING
ALTER VIEW active_employees
WITH SCHEMABINDING AS
SELECT employee_id, first_name, last_name, email
FROM dbo.employees
WHERE status = 'active';
```

---

## 7. View Naming Conventions (การตั้งชื่อ View)

```sql
-- Best Practices สำหรับการตั้งชื่อ View

-- 1. ใช้ prefix v_ หรือ vw_ เพื่อแยกจากตารางปกติ
CREATE VIEW v_active_employees AS SELECT ...;
CREATE VIEW vw_sales_report AS SELECT ...;

-- 2. ตั้งชื่อให้บ่งบอกเนื้อหา
CREATE VIEW v_monthly_revenue_by_region AS SELECT ...;

-- 3. ตัวอย่างการตั้งชื่อตาม pattern ต่างๆ
-- pattern: v_[subject]_[filter/type]
CREATE VIEW v_employees_active AS SELECT ...;
CREATE VIEW v_orders_pending AS SELECT ...;
CREATE VIEW v_products_low_stock AS SELECT ...;

-- pattern: v_rpt_[report_name] สำหรับ report views
CREATE VIEW v_rpt_sales_monthly AS SELECT ...;
CREATE VIEW v_rpt_employee_performance AS SELECT ...;

-- pattern: v_sec_[name] สำหรับ security views
CREATE VIEW v_sec_employee_public AS SELECT ...;
CREATE VIEW v_sec_customer_masked AS SELECT ...;

-- ตัวอย่างที่ 22: Naming conventions แบบ Schema-based
-- ใน PostgreSQL แยก schema สำหรับ views
CREATE SCHEMA reports;
CREATE SCHEMA security_views;

CREATE VIEW reports.monthly_sales AS
SELECT 
    YEAR(order_date) AS year,
    MONTH(order_date) AS month,
    SUM(total_amount) AS revenue
FROM orders
GROUP BY YEAR(order_date), MONTH(order_date);

CREATE VIEW security_views.employee_no_salary AS
SELECT employee_id, first_name, last_name, department_id, hire_date
FROM employees;
```

---

## 8. Cascading Views (View ที่ซ้อนกัน)

```sql
-- ตัวอย่างที่ 23: Cascading Views
-- Base View
CREATE VIEW v_all_products AS
SELECT 
    product_id,
    product_name,
    category_id,
    unit_price,
    units_in_stock,
    discontinued
FROM products;

-- View ที่ต่อยอดจาก Base View
CREATE VIEW v_active_products AS
SELECT product_id, product_name, category_id, unit_price, units_in_stock
FROM v_all_products
WHERE discontinued = 0;

-- View ที่ต่อยอดอีกชั้น
CREATE VIEW v_affordable_active_products AS
SELECT product_id, product_name, category_id, unit_price, units_in_stock
FROM v_active_products
WHERE unit_price < 50;

-- ตัวอย่างที่ 24: Cascading Views สำหรับรายงาน
-- Level 1: ข้อมูล Raw
CREATE VIEW v_raw_sales AS
SELECT 
    o.order_id,
    o.order_date,
    o.customer_id,
    od.product_id,
    od.quantity,
    od.unit_price,
    od.discount
FROM orders o
JOIN order_details od ON o.order_id = od.order_id
WHERE o.status = 'completed';

-- Level 2: คำนวณยอดขาย
CREATE VIEW v_calculated_sales AS
SELECT 
    order_id,
    order_date,
    customer_id,
    product_id,
    quantity,
    unit_price,
    discount,
    quantity * unit_price AS gross_amount,
    quantity * unit_price * (1 - discount) AS net_amount
FROM v_raw_sales;

-- Level 3: สรุปยอดขายต่อ order
CREATE VIEW v_order_totals AS
SELECT 
    order_id,
    order_date,
    customer_id,
    SUM(gross_amount) AS total_gross,
    SUM(net_amount) AS total_net,
    COUNT(product_id) AS item_count
FROM v_calculated_sales
GROUP BY order_id, order_date, customer_id;

-- Level 4: สรุปรายลูกค้า
CREATE VIEW v_customer_sales_summary AS
SELECT 
    c.customer_id,
    c.first_name || ' ' || c.last_name AS customer_name,
    COUNT(ot.order_id) AS total_orders,
    SUM(ot.total_gross) AS lifetime_gross,
    SUM(ot.total_net) AS lifetime_net,
    AVG(ot.total_net) AS avg_order_value
FROM customers c
LEFT JOIN v_order_totals ot ON c.customer_id = ot.customer_id
GROUP BY c.customer_id, c.first_name, c.last_name;
```

---

## 9. Information Schema for Views

```sql
-- ตัวอย่างที่ 25: ดูรายการ Views ทั้งหมด
-- PostgreSQL
SELECT 
    table_schema AS schema_name,
    table_name AS view_name,
    view_definition
FROM information_schema.views
WHERE table_schema NOT IN ('information_schema', 'pg_catalog')
ORDER BY table_schema, table_name;

-- MySQL
SELECT 
    TABLE_SCHEMA AS database_name,
    TABLE_NAME AS view_name,
    VIEW_DEFINITION,
    IS_UPDATABLE,
    SECURITY_TYPE
FROM information_schema.VIEWS
WHERE TABLE_SCHEMA = 'your_database'
ORDER BY TABLE_NAME;

-- SQL Server
SELECT 
    SCHEMA_NAME(v.schema_id) AS schema_name,
    v.name AS view_name,
    m.definition AS view_definition,
    v.create_date,
    v.modify_date
FROM sys.views v
JOIN sys.sql_modules m ON v.object_id = m.object_id
ORDER BY schema_name, view_name;

-- ตัวอย่างที่ 26: ดู View columns
-- PostgreSQL
SELECT 
    view_name,
    column_name,
    ordinal_position,
    data_type,
    is_nullable
FROM information_schema.columns
WHERE table_name = 'your_view_name'
ORDER BY ordinal_position;

-- ตัวอย่างที่ 27: ดูว่า View ใช้ตารางอะไรบ้าง (PostgreSQL)
SELECT DISTINCT 
    v.table_name AS view_name,
    vtu.table_name AS referenced_table
FROM information_schema.views v
JOIN information_schema.view_table_usage vtu 
    ON v.table_name = vtu.view_name
    AND v.table_schema = vtu.view_schema
WHERE v.table_schema = 'public'
ORDER BY v.table_name, vtu.table_name;

-- ตัวอย่างที่ 28: ตรวจสอบว่า View สามารถ update ได้หรือไม่ (MySQL)
SELECT 
    TABLE_NAME AS view_name,
    IS_UPDATABLE,
    CHECK_OPTION
FROM information_schema.VIEWS
WHERE TABLE_SCHEMA = DATABASE()
ORDER BY TABLE_NAME;

-- ตัวอย่างที่ 29: ดู View dependencies (SQL Server)
SELECT 
    OBJECT_NAME(referencing_id) AS view_name,
    referenced_entity_name AS referenced_object,
    referenced_class_desc
FROM sys.sql_expression_dependencies
WHERE OBJECT_NAME(referencing_id) IN (
    SELECT name FROM sys.views
)
ORDER BY view_name, referenced_entity_name;
```

---

## 10. Security Through Views (Column-Level Security)

```sql
-- ตัวอย่างที่ 30: Column-Level Security ด้วย Views
-- ซ่อน salary จากผู้ใช้ทั่วไป
CREATE VIEW v_employees_basic AS
SELECT 
    employee_id,
    first_name,
    last_name,
    email,
    department_id,
    position,
    hire_date
FROM employees;
-- ไม่มี salary, ssn, bank_account

-- สร้าง View ที่ HR เท่านั้นที่เห็น salary
CREATE VIEW v_employees_hr AS
SELECT 
    employee_id,
    first_name,
    last_name,
    email,
    department_id,
    position,
    hire_date,
    salary,
    performance_rating
FROM employees;

-- ตัวอย่างที่ 31: Row-Level Security ด้วย Views
-- แต่ละ manager เห็นเฉพาะพนักงานในทีมตัวเอง
CREATE VIEW v_my_team_employees AS
SELECT 
    employee_id,
    first_name,
    last_name,
    email,
    position,
    hire_date
FROM employees
WHERE manager_id = (
    SELECT employee_id FROM employees 
    WHERE email = CURRENT_USER()  -- MySQL: USER()
);

-- PostgreSQL: ใช้ current_user
CREATE VIEW v_my_team_employees AS
SELECT 
    e.employee_id,
    e.first_name,
    e.last_name,
    e.email,
    e.position,
    e.hire_date
FROM employees e
WHERE e.manager_id = (
    SELECT employee_id FROM employees 
    WHERE email = current_user
);

-- ตัวอย่างที่ 32: Data Masking ด้วย Views
CREATE VIEW v_customers_masked AS
SELECT 
    customer_id,
    first_name,
    CONCAT(SUBSTR(last_name, 1, 1), REPEAT('*', LENGTH(last_name) - 1)) AS last_name,
    CONCAT(SUBSTR(email, 1, 3), '****@', SPLIT_PART(email, '@', 2)) AS email,
    CONCAT('***-***-', RIGHT(phone, 4)) AS phone,
    city,
    country
FROM customers;

-- ตัวอย่างที่ 33: Security View สำหรับ API Access
-- View สำหรับ public API (ไม่แสดงข้อมูลส่วนตัว)
CREATE VIEW v_api_products AS
SELECT 
    product_id,
    product_name,
    category_id,
    unit_price,
    units_in_stock > 0 AS in_stock,  -- แสดงแค่ว่ามีของหรือเปล่า ไม่บอกจำนวน
    description,
    image_url
FROM products
WHERE discontinued = 0;

-- ตัวอย่างที่ 34: Security View ด้วย PostgreSQL Security Barrier
-- SECURITY BARRIER ป้องกัน information leakage
CREATE VIEW v_public_employees
WITH (security_barrier = TRUE) AS
SELECT 
    employee_id,
    first_name,
    last_name,
    department_id
FROM employees
WHERE is_public_profile = TRUE;

-- ตัวอย่างที่ 35: View สำหรับ Audit Trail
CREATE VIEW v_sensitive_data_access AS
SELECT 
    a.access_id,
    a.access_time,
    a.user_name,
    a.table_accessed,
    a.action_type,
    a.row_count_affected
FROM audit_log a
WHERE a.table_accessed IN ('employees', 'customers', 'financial_transactions')
  AND a.action_type IN ('SELECT', 'UPDATE', 'DELETE');

-- ตัวอย่างที่ 36: DEFINER vs INVOKER Security
-- MySQL: DEFINER = view รันด้วยสิทธิ์ของคนสร้าง (default)
CREATE DEFINER = 'admin'@'localhost' 
SQL SECURITY DEFINER
VIEW v_all_employee_salaries AS
SELECT employee_id, first_name, last_name, salary
FROM employees;

-- INVOKER = view รันด้วยสิทธิ์ของคนใช้
CREATE 
SQL SECURITY INVOKER
VIEW v_my_department AS
SELECT employee_id, first_name, last_name, department_id
FROM employees
WHERE department_id = get_user_department();
```

---

## 11. ตัวอย่าง Views เพิ่มเติม

```sql
-- ตัวอย่างที่ 37: View สำหรับ Dashboard
CREATE VIEW v_dashboard_kpi AS
SELECT 
    (SELECT COUNT(*) FROM customers WHERE created_at >= DATE_SUB(NOW(), INTERVAL 30 DAY)) AS new_customers_30d,
    (SELECT COUNT(*) FROM orders WHERE order_date >= DATE_SUB(NOW(), INTERVAL 30 DAY)) AS orders_30d,
    (SELECT SUM(total_amount) FROM orders WHERE order_date >= DATE_SUB(NOW(), INTERVAL 30 DAY) AND status = 'completed') AS revenue_30d,
    (SELECT COUNT(*) FROM products WHERE units_in_stock < reorder_level AND discontinued = 0) AS low_stock_products,
    (SELECT COUNT(*) FROM orders WHERE status = 'pending') AS pending_orders;

-- ตัวอย่างที่ 38: View สำหรับการ Reporting แบบ Pivot
CREATE VIEW v_sales_by_category_month AS
SELECT 
    YEAR(o.order_date) AS year,
    MONTH(o.order_date) AS month,
    c.category_name,
    SUM(od.quantity * od.unit_price) AS revenue
FROM orders o
JOIN order_details od ON o.order_id = od.order_id
JOIN products p ON od.product_id = p.product_id
JOIN categories c ON p.category_id = c.category_id
WHERE o.status = 'completed'
GROUP BY YEAR(o.order_date), MONTH(o.order_date), c.category_name;

-- ตัวอย่างที่ 39: View ที่ใช้ Window Functions
CREATE VIEW v_employee_salary_rank AS
SELECT 
    employee_id,
    first_name,
    last_name,
    department_id,
    salary,
    RANK() OVER (ORDER BY salary DESC) AS company_rank,
    RANK() OVER (PARTITION BY department_id ORDER BY salary DESC) AS dept_rank,
    PERCENT_RANK() OVER (ORDER BY salary) AS salary_percentile,
    salary - AVG(salary) OVER (PARTITION BY department_id) AS diff_from_dept_avg
FROM employees
WHERE status = 'active';

-- ตัวอย่างที่ 40: View สำหรับการ Monitor Database Performance (PostgreSQL)
CREATE VIEW v_slow_queries AS
SELECT 
    query,
    calls,
    total_exec_time / 1000 AS total_seconds,
    mean_exec_time / 1000 AS avg_seconds,
    stddev_exec_time / 1000 AS stddev_seconds,
    rows
FROM pg_stat_statements
WHERE mean_exec_time > 1000  -- queries ที่ใช้เวลามากกว่า 1 วินาที
ORDER BY mean_exec_time DESC;
```

---

## 12. Performance Considerations (ข้อควรพิจารณาด้านประสิทธิภาพ)

```sql
-- Views ไม่เพิ่มประสิทธิภาพในตัวเอง
-- แต่สามารถช่วยได้โดยอ้อม

-- ตัวอย่างที่ 41: View ที่ใช้ Index ได้อย่างมีประสิทธิภาพ
-- สร้าง Index ก่อน
CREATE INDEX idx_employees_dept_status ON employees(department_id, status);

-- View ที่ Filter ตาม indexed columns
CREATE VIEW v_active_it_employees AS
SELECT employee_id, first_name, last_name, position, salary
FROM employees
WHERE department_id = 10  -- มี index
  AND status = 'active';  -- มี index
-- Query optimizer จะใช้ index นี้

-- ตัวอย่างที่ 42: ข้อควรระวัง - View ที่ทำให้ Index ไม่ทำงาน
CREATE VIEW v_upper_names AS
SELECT 
    employee_id,
    UPPER(first_name) AS first_name,  -- Function บน indexed column ทำให้ index ไม่ทำงาน
    last_name,
    department_id
FROM employees;

-- แนะนำ: สร้าง Function-based Index แทน (PostgreSQL)
CREATE INDEX idx_employees_upper_name ON employees(UPPER(first_name));
```

---

## 13. สรุป: Best Practices สำหรับ Views

```sql
-- 1. ใช้ View เพื่อ abstraction ไม่ใช่เพื่อประสิทธิภาพ
-- 2. หลีกเลี่ยง View ซ้อน View มากเกินไป (ยาก debug)
-- 3. ตั้งชื่อให้ชัดเจนและสม่ำเสมอ
-- 4. Document View definition ใน comments
-- 5. ใช้ WITH CHECK OPTION เมื่อต้องการ data integrity
-- 6. พิจารณา Materialized View เมื่อต้องการประสิทธิภาพ

-- ตัวอย่างที่ 43: View พร้อม comments
CREATE VIEW v_customer_orders_summary AS
-- สรุปข้อมูลคำสั่งซื้อของลูกค้าแต่ละราย
-- ใช้สำหรับ: Customer Dashboard, Sales Reports
-- อัพเดทล่าสุด: 2024-01-01
-- สร้างโดย: ทีมพัฒนา
SELECT 
    c.customer_id,
    c.company_name,
    c.contact_name,
    COUNT(o.order_id) AS total_orders,
    SUM(CASE WHEN o.status = 'pending' THEN 1 ELSE 0 END) AS pending_orders,
    SUM(CASE WHEN o.status = 'completed' THEN 1 ELSE 0 END) AS completed_orders,
    SUM(CASE WHEN o.status = 'cancelled' THEN 1 ELSE 0 END) AS cancelled_orders,
    SUM(o.total_amount) AS total_spent,
    MAX(o.order_date) AS last_order_date
FROM customers c
LEFT JOIN orders o ON c.customer_id = o.customer_id
GROUP BY c.customer_id, c.company_name, c.contact_name;
```

---

## แบบฝึกหัด (Exercises)

**ข้อ 1:** สร้าง View ชื่อ `v_high_value_customers` ที่แสดงลูกค้าที่มียอดซื้อรวมมากกว่า 10,000 บาท พร้อมแสดง customer_id, ชื่อ, email, total_spent

**คำตอบข้อ 1:**
```sql
CREATE VIEW v_high_value_customers AS
SELECT 
    c.customer_id,
    c.first_name || ' ' || c.last_name AS full_name,
    c.email,
    SUM(o.total_amount) AS total_spent
FROM customers c
JOIN orders o ON c.customer_id = o.customer_id
WHERE o.status = 'completed'
GROUP BY c.customer_id, c.first_name, c.last_name, c.email
HAVING SUM(o.total_amount) > 10000
ORDER BY total_spent DESC;
```

**ข้อ 2:** สร้าง View ชื่อ `v_products_with_category` ที่ JOIN products กับ categories แสดงชื่อสินค้าและชื่อหมวดหมู่ พร้อม WITH CHECK OPTION

**คำตอบข้อ 2:**
```sql
CREATE VIEW v_products_with_category AS
SELECT 
    p.product_id,
    p.product_name,
    c.category_name,
    p.unit_price,
    p.units_in_stock,
    p.discontinued
FROM products p
JOIN categories c ON p.category_id = c.category_id
WHERE p.discontinued = 0
WITH CHECK OPTION;
```

**ข้อ 3:** เขียน Query เพื่อดูรายการ Views ทั้งหมดใน Database พร้อม definition

**คำตอบข้อ 3:**
```sql
-- MySQL
SELECT TABLE_NAME, VIEW_DEFINITION, IS_UPDATABLE
FROM information_schema.VIEWS
WHERE TABLE_SCHEMA = DATABASE();

-- PostgreSQL
SELECT table_name, view_definition
FROM information_schema.views
WHERE table_schema = 'public';
```

**ข้อ 4:** สร้าง View สำหรับ Security โดยซ่อน salary และ ssn จาก employees table

**คำตอบข้อ 4:**
```sql
CREATE VIEW v_employees_public AS
SELECT 
    employee_id,
    first_name,
    last_name,
    email,
    department_id,
    position,
    hire_date
FROM employees
WHERE status = 'active';
-- salary และ ssn ถูกซ่อนไว้
```

**ข้อ 5:** สร้าง Cascading Views 3 ชั้น: ชั้น 1 = raw orders, ชั้น 2 = orders พร้อม line totals, ชั้น 3 = สรุปต่อ order

**คำตอบข้อ 5:**
```sql
-- ชั้น 1
CREATE VIEW v_orders_raw AS
SELECT o.order_id, o.order_date, o.customer_id, o.status,
       od.product_id, od.quantity, od.unit_price, od.discount
FROM orders o JOIN order_details od ON o.order_id = od.order_id;

-- ชั้น 2
CREATE VIEW v_orders_with_line_totals AS
SELECT *, 
       quantity * unit_price AS gross_line,
       quantity * unit_price * (1 - discount) AS net_line
FROM v_orders_raw;

-- ชั้น 3
CREATE VIEW v_order_summaries AS
SELECT order_id, order_date, customer_id, status,
       SUM(gross_line) AS total_gross,
       SUM(net_line) AS total_net,
       COUNT(product_id) AS items
FROM v_orders_with_line_totals
GROUP BY order_id, order_date, customer_id, status;
```

**ข้อ 6:** สร้าง View ที่ใช้ Window Function แสดง rank ของ products ตาม revenue

**คำตอบข้อ 6:**
```sql
CREATE VIEW v_product_revenue_rank AS
SELECT 
    p.product_id,
    p.product_name,
    SUM(od.quantity * od.unit_price) AS total_revenue,
    RANK() OVER (ORDER BY SUM(od.quantity * od.unit_price) DESC) AS revenue_rank,
    DENSE_RANK() OVER (
        PARTITION BY p.category_id 
        ORDER BY SUM(od.quantity * od.unit_price) DESC
    ) AS category_rank
FROM products p
JOIN order_details od ON p.product_id = od.product_id
JOIN orders o ON od.order_id = o.order_id
WHERE o.status = 'completed'
GROUP BY p.product_id, p.product_name, p.category_id;
```

**ข้อ 7:** แก้ไข View ที่มีอยู่แล้วโดยเพิ่ม column phone ลงใน View `v_customer_contact_list`

**คำตอบข้อ 7:**
```sql
-- PostgreSQL / MySQL
CREATE OR REPLACE VIEW v_customer_contact_list AS
SELECT 
    customer_id,
    company_name,
    contact_name,
    phone,        -- เพิ่ม column นี้
    email,
    city,
    country
FROM customers
WHERE active = TRUE;

-- SQL Server
ALTER VIEW v_customer_contact_list AS
SELECT 
    customer_id,
    company_name,
    contact_name,
    phone,
    email,
    city,
    country
FROM customers
WHERE active = 1;
```

**ข้อ 8:** สร้าง View ที่ซ่อน/mask ข้อมูล email ให้เหลือเฉพาะ 3 ตัวแรกและ domain

**คำตอบข้อ 8:**
```sql
CREATE VIEW v_customers_email_masked AS
SELECT 
    customer_id,
    first_name,
    last_name,
    CONCAT(
        LEFT(email, 3), 
        '***@', 
        SUBSTRING_INDEX(email, '@', -1)
    ) AS email_masked,
    phone,
    city
FROM customers;
```

**ข้อ 9:** DROP View ทั้งหมดที่ขึ้นต้นด้วย `v_temp_` (ต้องดูรายชื่อก่อน)

**คำตอบข้อ 9:**
```sql
-- ดูรายชื่อก่อน
SELECT TABLE_NAME FROM information_schema.VIEWS
WHERE TABLE_SCHEMA = DATABASE()
  AND TABLE_NAME LIKE 'v_temp_%';

-- DROP ทีละอัน
DROP VIEW IF EXISTS v_temp_sales;
DROP VIEW IF EXISTS v_temp_employees;
-- หรือใช้ Dynamic SQL
```

**ข้อ 10:** สร้าง View ที่แสดงสินค้าที่ยังไม่ถูก order เลยในช่วง 3 เดือนที่ผ่านมา

**คำตอบข้อ 10:**
```sql
CREATE VIEW v_inactive_products AS
SELECT 
    p.product_id,
    p.product_name,
    p.category_id,
    p.unit_price,
    p.units_in_stock,
    MAX(o.order_date) AS last_order_date
FROM products p
LEFT JOIN order_details od ON p.product_id = od.product_id
LEFT JOIN orders o ON od.order_id = o.order_id
WHERE p.discontinued = 0
GROUP BY p.product_id, p.product_name, p.category_id, p.unit_price, p.units_in_stock
HAVING MAX(o.order_date) < DATE_SUB(NOW(), INTERVAL 3 MONTH)
    OR MAX(o.order_date) IS NULL;
```

---

*จบ Part 081: Views - Virtual Tables*
