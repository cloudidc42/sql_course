# ภาค 24: FULL OUTER JOIN

## FULL OUTER JOIN คืออะไร?

**FULL OUTER JOIN** (หรือ FULL JOIN) คือการรวมตารางที่:
- เก็บ **ทุกแถว** จาก **ทั้งสองตาราง**
- ถ้าตาราง LEFT ไม่มีคู่ทางขวา → NULL สำหรับ columns ของตาราง RIGHT
- ถ้าตาราง RIGHT ไม่มีคู่ทางซ้าย → NULL สำหรับ columns ของตาราง LEFT
- เป็นการรวม LEFT JOIN + RIGHT JOIN เข้าด้วยกัน

```
แผนภาพ Venn Diagram:

Table A               Table B
  ┌─────────────────────────────┐
  │ ╔══════╗ ╔═══╗ ╔═════════╗ │
  │ ║A only║ ║A∩B║ ║ B only  ║ │
  │ ║(NULL ║ ║   ║ ║ (NULL   ║ │
  │ ║for B)║ ║   ║ ║  for A) ║ │
  │ ╚══════╝ ╚═══╝ ╚═════════╝ │
  └─────────────────────────────┘

FULL OUTER JOIN = A only + A∩B + B only
                = ทุกแถวจากทั้งสองตาราง
```

---

## Syntax ของ FULL OUTER JOIN

```sql
-- PostgreSQL, SQL Server, DB2 (รองรับ FULL OUTER JOIN โดยตรง)
SELECT columns
FROM table_a
FULL OUTER JOIN table_b ON table_a.key = table_b.key;

-- หรือย่อ
SELECT columns
FROM table_a
FULL JOIN table_b ON table_a.key = table_b.key;
```

---

## FULL OUTER JOIN ใน MySQL (ไม่รองรับ!)

**MySQL ไม่รองรับ FULL OUTER JOIN** โดยตรง! ต้องใช้ UNION ของ LEFT JOIN + RIGHT JOIN แทน

```sql
-- MySQL workaround: ใช้ UNION
SELECT columns
FROM table_a
LEFT JOIN table_b ON table_a.key = table_b.key

UNION  -- ใช้ UNION (ไม่ใช่ UNION ALL) เพื่อลบ duplicates

SELECT columns
FROM table_a
RIGHT JOIN table_b ON table_a.key = table_b.key;

-- หรือ UNION ALL + ตัด duplicate ด้วย WHERE
SELECT columns
FROM table_a
LEFT JOIN table_b ON table_a.key = table_b.key

UNION ALL

SELECT columns
FROM table_a
RIGHT JOIN table_b ON table_a.key = table_b.key
WHERE table_a.key IS NULL;  -- เอาเฉพาะส่วนที่ RIGHT only
```

---

## เปรียบเทียบ JOIN ทุกประเภท

```
ข้อมูลตัวอย่าง:
Table A (departments):  Table B (employees):
ID | Name               ID | Emp    | dept_id
1  | Engineering        1  | Emp_A  | 1
2  | Marketing          2  | Emp_B  | 10
10 | Executive          3  | Emp_C  | 99  ← dept ที่ไม่มีใน A
99 | Ghost Dept         (ไม่มีพนักงานใน dept 2)

INNER JOIN:    LEFT JOIN:         RIGHT JOIN:        FULL JOIN:
Engineering    Engineering        Engineering        Engineering
Executive      Executive          Executive          Executive
Emp_B          Emp_B              Emp_B              Emp_B
               Marketing (NULL)   Emp_C (NULL)      Marketing (NULL)
               Ghost Dept (NULL)                     Ghost Dept (NULL)
                                                     Emp_C (NULL)
```

---

## ตัวอย่าง FULL OUTER JOIN (1-20)

### ตัวอย่างที่ 1: Basic FULL OUTER JOIN

```sql
-- PostgreSQL
SELECT 
    COALESCE(d.dept_name, 'No Department') AS department,
    COALESCE(e.first_name || ' ' || e.last_name, 'No Employee') AS employee
FROM departments d
FULL OUTER JOIN employees e ON d.dept_id = e.dept_id
ORDER BY department, employee;

-- MySQL workaround
SELECT 
    COALESCE(d.dept_name, 'No Department') AS department,
    COALESCE(CONCAT(e.first_name, ' ', e.last_name), 'No Employee') AS employee
FROM departments d
LEFT JOIN employees e ON d.dept_id = e.dept_id
UNION
SELECT 
    COALESCE(d.dept_name, 'No Department'),
    COALESCE(CONCAT(e.first_name, ' ', e.last_name), 'No Employee')
FROM departments d
RIGHT JOIN employees e ON d.dept_id = e.dept_id;
```

### ตัวอย่างที่ 2: FULL JOIN สำหรับ Data Reconciliation

```sql
-- เปรียบเทียบ expected vs actual: แผนกทั้งหมด + พนักงานทั้งหมด
SELECT 
    d.dept_id AS dept_id_from_dept,
    d.dept_name,
    e.emp_id,
    e.dept_id AS dept_id_from_emp,
    CASE 
        WHEN d.dept_id IS NULL THEN 'Employee has invalid dept_id'
        WHEN e.emp_id IS NULL THEN 'Department has no employees'
        ELSE 'Matched'
    END AS status
FROM departments d
FULL OUTER JOIN employees e ON d.dept_id = e.dept_id
WHERE d.dept_id IS NULL OR e.emp_id IS NULL  -- แสดงเฉพาะ unmatched
ORDER BY status, d.dept_id;
```

### ตัวอย่างที่ 3: Products vs Order Items Reconciliation

```sql
-- เปรียบเทียบ products ที่มีกับที่ถูกสั่ง
SELECT 
    COALESCE(p.product_id::TEXT, 'N/A') AS product_id,
    COALESCE(p.product_name, 'Unknown Product') AS product_name,
    COALESCE(p.price::TEXT, 'N/A') AS list_price,
    COALESCE(oi.order_id::TEXT, 'Never Ordered') AS order_info,
    CASE 
        WHEN p.product_id IS NULL THEN 'Ordered but Product Deleted'
        WHEN oi.order_id IS NULL THEN 'Product Never Ordered'
        ELSE 'Active Product'
    END AS product_status
FROM products p
FULL OUTER JOIN order_items oi ON p.product_id = oi.product_id
ORDER BY product_status, p.product_id;
```

### ตัวอย่างที่ 4: FULL JOIN กับ COALESCE

```sql
-- รวมข้อมูลจากสองตาราง แสดงค่าที่มีจากฝั่งใดก็ได้
SELECT 
    COALESCE(d.dept_id, e.dept_id) AS dept_id,  -- ใช้ค่าจากฝั่งที่มี
    COALESCE(d.dept_name, 'No Name') AS dept_name,
    COUNT(e.emp_id) AS employee_count,
    COALESCE(SUM(e.salary), 0) AS total_salary
FROM departments d
FULL OUTER JOIN employees e ON d.dept_id = e.dept_id
GROUP BY COALESCE(d.dept_id, e.dept_id), d.dept_name
ORDER BY dept_id;
```

### ตัวอย่างที่ 5: FULL OUTER JOIN สำหรับ Symmetric Difference

```sql
-- Symmetric Difference: แถวที่อยู่ในตารางใดตารางหนึ่ง แต่ไม่ทั้งคู่
-- ใช้หา data mismatches

-- หา dept_id ที่อยู่ใน departments แต่ไม่ใน employees
-- และ dept_id ที่อยู่ใน employees แต่ไม่ใน departments
SELECT 
    COALESCE(d.dept_id::TEXT, '') AS dept_table_id,
    COALESCE(d.dept_name, '') AS dept_name,
    COALESCE(e.dept_id::TEXT, '') AS emp_dept_id,
    CASE 
        WHEN e.emp_id IS NULL THEN 'In departments ONLY'
        WHEN d.dept_id IS NULL THEN 'In employees ONLY'
    END AS location
FROM departments d
FULL OUTER JOIN (
    SELECT DISTINCT dept_id FROM employees WHERE dept_id IS NOT NULL
) e ON d.dept_id = e.dept_id
WHERE d.dept_id IS NULL OR e.dept_id IS NULL;
```

### ตัวอย่างที่ 6: Monthly Sales Comparison (FULL JOIN)

```sql
-- เปรียบเทียบยอดขายปี 2023 vs 2024 ทุกเดือน
WITH sales_2023 AS (
    SELECT 
        EXTRACT(MONTH FROM order_date) AS month,
        SUM(total_amount) AS revenue_2023
    FROM orders
    WHERE EXTRACT(YEAR FROM order_date) = 2023
      AND status = 'completed'
    GROUP BY month
),
sales_2024 AS (
    SELECT 
        EXTRACT(MONTH FROM order_date) AS month,
        SUM(total_amount) AS revenue_2024
    FROM orders
    WHERE EXTRACT(YEAR FROM order_date) = 2024
      AND status = 'completed'
    GROUP BY month
)
SELECT 
    COALESCE(s23.month, s24.month) AS month,
    COALESCE(s23.revenue_2023, 0) AS revenue_2023,
    COALESCE(s24.revenue_2024, 0) AS revenue_2024,
    COALESCE(s24.revenue_2024, 0) - COALESCE(s23.revenue_2023, 0) AS yoy_growth
FROM sales_2023 s23
FULL OUTER JOIN sales_2024 s24 ON s23.month = s24.month
ORDER BY month;
```

### ตัวอย่างที่ 7: Customer Order Matching

```sql
-- Match customers กับ orders (แสดงทั้งที่ไม่มีคู่)
SELECT 
    c.customer_id,
    c.first_name || ' ' || c.last_name AS customer,
    o.order_id,
    o.order_date,
    o.total_amount,
    CASE 
        WHEN c.customer_id IS NULL THEN 'Order has no customer'
        WHEN o.order_id IS NULL THEN 'Customer has no orders'
        ELSE 'Matched'
    END AS match_status
FROM customers c
FULL OUTER JOIN orders o ON c.customer_id = o.customer_id
ORDER BY match_status, c.customer_id;
```

### ตัวอย่างที่ 8: MySQL FULL JOIN Workaround แบบมีประสิทธิภาพ

```sql
-- MySQL: FULL OUTER JOIN ด้วย UNION ALL + exclusion
-- (เร็วกว่า UNION เพราะไม่ต้อง dedup)
SELECT 
    d.dept_id,
    d.dept_name,
    e.emp_id,
    e.first_name
FROM departments d
LEFT JOIN employees e ON d.dept_id = e.dept_id

UNION ALL

SELECT 
    d.dept_id,
    d.dept_name,
    e.emp_id,
    e.first_name
FROM departments d
RIGHT JOIN employees e ON d.dept_id = e.dept_id
WHERE d.dept_id IS NULL;  -- เฉพาะ RIGHT only (ป้องกัน duplicate)
```

### ตัวอย่างที่ 9: Inventory vs Sales Full Report

```sql
-- Products + ยอดขาย: รวมที่ไม่ match ทั้งสองด้าน
SELECT 
    COALESCE(p.product_id, oi_agg.product_id) AS product_id,
    COALESCE(p.product_name, 'Product Not Found') AS product_name,
    COALESCE(p.price, 0) AS list_price,
    COALESCE(p.stock_quantity, 0) AS stock,
    COALESCE(oi_agg.total_sold, 0) AS total_sold,
    COALESCE(oi_agg.revenue, 0) AS revenue
FROM products p
FULL OUTER JOIN (
    SELECT 
        product_id,
        SUM(quantity) AS total_sold,
        SUM(quantity * unit_price - discount) AS revenue
    FROM order_items
    GROUP BY product_id
) oi_agg ON p.product_id = oi_agg.product_id
ORDER BY revenue DESC;
```

### ตัวอย่างที่ 10: Department Salary Budget Analysis (FULL)

```sql
-- วิเคราะห์งบประมาณ: รวมแผนกว่าง + พนักงานไม่มีแผนก
SELECT 
    COALESCE(d.dept_name, 'Unassigned') AS department,
    COALESCE(d.budget, 0) AS budget,
    COUNT(e.emp_id) AS headcount,
    COALESCE(SUM(e.salary), 0) AS monthly_payroll,
    COALESCE(SUM(e.salary) * 12, 0) AS annual_payroll,
    COALESCE(d.budget, 0) - COALESCE(SUM(e.salary) * 12, 0) AS budget_remaining,
    CASE 
        WHEN d.dept_id IS NULL THEN 'Employees without Dept'
        WHEN SUM(e.salary) IS NULL THEN 'Empty Department'
        WHEN SUM(e.salary) * 12 > d.budget THEN 'Over Budget'
        ELSE 'Within Budget'
    END AS status
FROM departments d
FULL OUTER JOIN employees e ON d.dept_id = e.dept_id
GROUP BY d.dept_id, d.dept_name, d.budget
ORDER BY annual_payroll DESC;
```

---

## ตัวอย่าง FULL OUTER JOIN สำหรับ Real-World Scenarios

### ตัวอย่างที่ 11: Data Migration Verification

```sql
-- ตรวจสอบข้อมูลหลัง migrate: เปรียบเทียบ old กับ new
CREATE TEMP TABLE customers_old (
    customer_id INT,
    name VARCHAR(100),
    email VARCHAR(100)
);
CREATE TEMP TABLE customers_new (
    customer_id INT,
    full_name VARCHAR(100),
    contact_email VARCHAR(100)
);

INSERT INTO customers_old VALUES
(1, 'Somchai Jaidee', 'somchai@old.com'),
(2, 'Wanchai Thong', 'wanchai@old.com'),
(3, 'Old Only Customer', 'old@old.com');  -- มีแค่ใน old

INSERT INTO customers_new VALUES
(1, 'Somchai Jaidee', 'somchai@new.com'),  -- email ต่างกัน
(2, 'Wanchai Thong', 'wanchai@new.com'),
(4, 'New Only Customer', 'new@new.com');   -- มีแค่ใน new

-- ตรวจสอบ
SELECT 
    COALESCE(o.customer_id::TEXT, 'N/A') AS old_id,
    o.name AS old_name,
    o.email AS old_email,
    COALESCE(n.customer_id::TEXT, 'N/A') AS new_id,
    n.full_name AS new_name,
    n.contact_email AS new_email,
    CASE 
        WHEN o.customer_id IS NULL THEN 'NEW RECORD ONLY'
        WHEN n.customer_id IS NULL THEN 'OLD RECORD ONLY (not migrated)'
        WHEN o.email != n.contact_email THEN 'EMAIL MISMATCH'
        ELSE 'OK'
    END AS migration_status
FROM customers_old o
FULL OUTER JOIN customers_new n ON o.customer_id = n.customer_id
ORDER BY migration_status;
```

### ตัวอย่างที่ 12: Budget vs Actual Reconciliation

```sql
-- สมมติมีตาราง budget_plan กับ actual_spending
CREATE TEMP TABLE budget_plan (
    dept_id INT,
    budget_item VARCHAR(100),
    planned_amount DECIMAL(15,2)
);
CREATE TEMP TABLE actual_spending (
    dept_id INT,
    expense_item VARCHAR(100),
    actual_amount DECIMAL(15,2)
);

INSERT INTO budget_plan VALUES
(1, 'Salaries', 2000000),
(1, 'Equipment', 500000),
(1, 'Training', 100000),
(2, 'Marketing Campaigns', 800000);

INSERT INTO actual_spending VALUES
(1, 'Salaries', 1950000),
(1, 'Equipment', 650000),  -- over budget
(1, 'Office Supplies', 50000),  -- not in plan
(3, 'Emergency Repair', 200000);  -- dept not in budget

SELECT 
    COALESCE(bp.dept_id, asp.dept_id) AS dept_id,
    COALESCE(bp.budget_item, asp.expense_item) AS item,
    COALESCE(bp.planned_amount, 0) AS planned,
    COALESCE(asp.actual_amount, 0) AS actual,
    COALESCE(asp.actual_amount, 0) - COALESCE(bp.planned_amount, 0) AS variance,
    CASE 
        WHEN bp.budget_item IS NULL THEN 'UNPLANNED EXPENSE'
        WHEN asp.expense_item IS NULL THEN 'UNSPENT BUDGET'
        WHEN asp.actual_amount > bp.planned_amount THEN 'OVER BUDGET'
        ELSE 'WITHIN BUDGET'
    END AS status
FROM budget_plan bp
FULL OUTER JOIN actual_spending asp 
    ON bp.dept_id = asp.dept_id 
    AND bp.budget_item = asp.expense_item
ORDER BY dept_id, item;
```

### ตัวอย่างที่ 13: Employee Skills Matrix

```sql
-- สร้าง skills matrix: พนักงานทุกคน × ทักษะทุกอย่าง
-- (สมมติมีตาราง skills และ employee_skills)
CREATE TEMP TABLE all_skills AS
SELECT DISTINCT skill_name FROM (
    VALUES 
    ('Python'), ('SQL'), ('Java'), ('Excel'), ('Communication')
) AS s(skill_name);

-- Cross join แสดง matrix ทุก combination (แต่ใช้ FULL OUTER เพื่อความครบถ้วน)
SELECT 
    e.first_name || ' ' || e.last_name AS employee,
    s.skill_name,
    CASE 
        WHEN es.emp_id IS NOT NULL THEN 'Has Skill'
        ELSE 'Missing'
    END AS skill_status
FROM employees e
CROSS JOIN all_skills s
LEFT JOIN employee_skills es ON e.emp_id = es.emp_id AND s.skill_name = es.skill_name
WHERE e.dept_id = 1  -- Engineering only
ORDER BY employee, s.skill_name;
```

### ตัวอย่างที่ 14: Audit Trail Comparison

```sql
-- เปรียบเทียบ snapshots สองช่วงเวลา
-- สมมติมี product_snapshot ที่ถ่ายภาพ products ในแต่ละไตรมาส
CREATE TEMP TABLE products_q1 AS
SELECT product_id, product_name, price, stock_quantity
FROM products;

CREATE TEMP TABLE products_q2 AS
SELECT product_id, product_name, price * 1.05 AS price, 
       CASE WHEN stock_quantity > 10 THEN stock_quantity - 5 ELSE 0 END AS stock_quantity
FROM products;

SELECT 
    COALESCE(q1.product_id, q2.product_id) AS product_id,
    COALESCE(q1.product_name, q2.product_name) AS product_name,
    q1.price AS q1_price,
    q2.price AS q2_price,
    q2.price - q1.price AS price_change,
    q1.stock_quantity AS q1_stock,
    q2.stock_quantity AS q2_stock,
    CASE 
        WHEN q1.product_id IS NULL THEN 'New Product in Q2'
        WHEN q2.product_id IS NULL THEN 'Discontinued in Q2'
        WHEN q1.price != q2.price THEN 'Price Changed'
        ELSE 'No Change'
    END AS change_type
FROM products_q1 q1
FULL OUTER JOIN products_q2 q2 ON q1.product_id = q2.product_id
ORDER BY change_type, product_id;
```

### ตัวอย่างที่ 15: Complete Sales vs Inventory Report

```sql
-- รายงานสต็อกและยอดขายครบถ้วน
WITH product_sales AS (
    SELECT 
        p.product_id,
        SUM(oi.quantity) AS total_sold,
        SUM(oi.quantity * oi.unit_price - oi.discount) AS revenue
    FROM order_items oi
    JOIN orders o ON oi.order_id = o.order_id
    WHERE o.status IN ('completed', 'shipped')
    RIGHT JOIN products p ON oi.product_id = p.product_id  -- ทุก product
    GROUP BY p.product_id
)
SELECT 
    p.product_id,
    p.product_name,
    p.category,
    p.price,
    p.stock_quantity,
    COALESCE(ps.total_sold, 0) AS units_sold,
    COALESCE(ps.revenue, 0) AS sales_revenue,
    CASE 
        WHEN COALESCE(ps.total_sold, 0) = 0 THEN 'Dead Stock'
        WHEN p.stock_quantity = 0 THEN 'Sold Out'
        WHEN p.stock_quantity < 10 THEN 'Low Stock'
        ELSE 'OK'
    END AS stock_status
FROM products p
LEFT JOIN product_sales ps ON p.product_id = ps.product_id
ORDER BY COALESCE(ps.revenue, 0) DESC;
```

---

## FULL JOIN กับ COALESCE Patterns

### ตัวอย่างที่ 16: Merge Two Tables

```sql
-- รวมข้อมูลจากสองตาราง (เหมือน UPSERT ใน read mode)
-- ถ้า key มีในทั้งสองตาราง → ใช้ค่าจากตาราง B (override)
-- ถ้า key มีแค่ใน A → ใช้ค่า A
-- ถ้า key มีแค่ใน B → ใช้ค่า B

CREATE TEMP TABLE base_data (
    id INT, name VARCHAR(50), value DECIMAL(10,2)
);
CREATE TEMP TABLE override_data (
    id INT, name VARCHAR(50), value DECIMAL(10,2)
);

INSERT INTO base_data VALUES (1,'A',100),(2,'B',200),(3,'C',300);
INSERT INTO override_data VALUES (2,'B_UPDATED',250),(4,'D',400);

SELECT 
    COALESCE(o.id, b.id) AS id,
    COALESCE(o.name, b.name) AS name,    -- B overrides A
    COALESCE(o.value, b.value) AS value  -- B overrides A
FROM base_data b
FULL OUTER JOIN override_data o ON b.id = o.id
ORDER BY id;
-- id=1: A (base only) → 100
-- id=2: B_UPDATED (override wins) → 250
-- id=3: C (base only) → 300
-- id=4: D (override only) → 400
```

### ตัวอย่างที่ 17: แสดง Difference ชัดเจน

```sql
-- แสดงความแตกต่างระหว่างสองชุดข้อมูล
WITH left_only AS (
    SELECT 
        d.dept_id,
        d.dept_name,
        NULL::INT AS emp_id,
        NULL::VARCHAR AS emp_name,
        'Dept without employee' AS diff_type
    FROM departments d
    LEFT JOIN employees e ON d.dept_id = e.dept_id
    WHERE e.emp_id IS NULL
),
right_only AS (
    SELECT 
        NULL::INT AS dept_id,
        NULL::VARCHAR AS dept_name,
        e.emp_id,
        e.first_name || ' ' || e.last_name AS emp_name,
        'Employee with invalid dept' AS diff_type
    FROM employees e
    LEFT JOIN departments d ON e.dept_id = d.dept_id
    WHERE d.dept_id IS NULL
)
SELECT * FROM left_only
UNION ALL
SELECT * FROM right_only
ORDER BY diff_type;
```

### ตัวอย่างที่ 18: Cross-System Reconciliation

```sql
-- สมมติมีข้อมูลจาก 2 ระบบ: CRM และ ERP
CREATE TEMP TABLE crm_customers (
    customer_id INT,
    customer_name VARCHAR(100),
    total_orders INT
);
CREATE TEMP TABLE erp_customers (
    customer_id INT,
    company_name VARCHAR(100),
    account_balance DECIMAL(15,2)
);

INSERT INTO crm_customers VALUES
(1, 'Somchai Corp', 5),
(2, 'Wanchai Ltd', 3),
(4, 'CRM Only Customer', 1);  -- มีแค่ใน CRM

INSERT INTO erp_customers VALUES
(1, 'SOMCHAI CORP', 150000),
(2, 'WANCHAI LTD', 75000),
(5, NULL, 200000);  -- มีแค่ใน ERP (และไม่มีชื่อ)

SELECT 
    COALESCE(crm.customer_id, erp.customer_id) AS customer_id,
    crm.customer_name AS crm_name,
    erp.company_name AS erp_name,
    crm.total_orders,
    erp.account_balance,
    CASE 
        WHEN crm.customer_id IS NULL THEN 'ERP ONLY - Not in CRM'
        WHEN erp.customer_id IS NULL THEN 'CRM ONLY - Not in ERP'
        ELSE 'MATCHED'
    END AS reconciliation_status
FROM crm_customers crm
FULL OUTER JOIN erp_customers erp ON crm.customer_id = erp.customer_id
ORDER BY reconciliation_status, customer_id;
```

---

## Symmetric Difference ด้วย FULL JOIN

**Symmetric Difference** คือ แถวที่อยู่ในตารางใดตารางหนึ่ง แต่ไม่ทั้งคู่ (A XOR B)

```sql
-- Symmetric Difference: A ∪ B minus A ∩ B
-- แถวที่ "ไม่ match" ทั้งสองฝั่ง

SELECT 
    d.dept_id AS dept_table_dept_id,
    d.dept_name,
    e_dept.dept_id AS emp_table_dept_id
FROM departments d
FULL OUTER JOIN (
    SELECT DISTINCT dept_id FROM employees
) e_dept ON d.dept_id = e_dept.dept_id
WHERE d.dept_id IS NULL OR e_dept.dept_id IS NULL;
-- แสดงเฉพาะ dept_id ที่ไม่ match ในทั้งสองตาราง
```

---

## ตัวอย่างที่ 19-20: ธุรกิจขั้นสูง

### ตัวอย่างที่ 19: Employee Coverage Report

```sql
-- รายงานครอบคลุม: departments + employees (ทุกอย่าง)
SELECT 
    COALESCE(d.dept_id::TEXT, 'N/A') AS dept_id,
    COALESCE(d.dept_name, 'Unassigned') AS dept_name,
    COALESCE(d.location, 'Unknown') AS location,
    COALESCE(d.budget::TEXT, 'No Budget') AS budget,
    e.emp_id,
    COALESCE(e.first_name || ' ' || e.last_name, 'No Employee') AS employee_name,
    e.job_title,
    e.salary,
    CASE 
        WHEN d.dept_id IS NULL THEN '⚠ No Dept'
        WHEN e.emp_id IS NULL THEN 'ℹ Empty Dept'
        ELSE '✓ OK'
    END AS status
FROM departments d
FULL OUTER JOIN employees e ON d.dept_id = e.dept_id
ORDER BY d.dept_id NULLS LAST, e.emp_id;
```

### ตัวอย่างที่ 20: Complete Business Health Report

```sql
-- รายงานสุขภาพธุรกิจ: สินค้าทุกชิ้น + order ทุกรายการ
WITH order_summary AS (
    SELECT 
        product_id,
        COUNT(DISTINCT order_id) AS order_count,
        SUM(quantity) AS total_qty,
        SUM(quantity * unit_price - discount) AS total_revenue
    FROM order_items
    GROUP BY product_id
)
SELECT 
    COALESCE(p.product_id, os.product_id) AS product_id,
    COALESCE(p.product_name, 'Deleted Product') AS product_name,
    COALESCE(p.category, 'Unknown') AS category,
    COALESCE(p.price, 0) AS current_price,
    COALESCE(p.stock_quantity, 0) AS stock,
    COALESCE(os.order_count, 0) AS times_ordered,
    COALESCE(os.total_qty, 0) AS units_sold,
    COALESCE(os.total_revenue, 0) AS revenue,
    CASE 
        WHEN p.product_id IS NULL THEN 'GHOST - Product deleted but in orders'
        WHEN os.product_id IS NULL THEN 'DORMANT - Never sold'
        WHEN p.stock_quantity < 5 THEN 'CRITICAL - Low stock, high demand'
        WHEN os.total_qty > 10 AND p.stock_quantity > 50 THEN 'STAR - Good stock, selling well'
        ELSE 'NORMAL'
    END AS product_health
FROM products p
FULL OUTER JOIN order_summary os ON p.product_id = os.product_id
ORDER BY revenue DESC;
```

---

## แบบฝึกหัดภาค 24

**ข้อ 1:** เขียน FULL OUTER JOIN ระหว่าง departments กับ employees

```sql
-- เฉลย
SELECT 
    COALESCE(d.dept_id::TEXT, 'N/A') AS dept_id,
    COALESCE(d.dept_name, 'No Dept') AS department,
    COALESCE(e.emp_id::TEXT, 'N/A') AS emp_id,
    COALESCE(e.first_name || ' ' || e.last_name, 'No Employee') AS employee
FROM departments d
FULL OUTER JOIN employees e ON d.dept_id = e.dept_id
ORDER BY department, employee;
```

**ข้อ 2:** จงเขียน MySQL workaround สำหรับ FULL OUTER JOIN ในข้อ 1

```sql
-- เฉลย MySQL
SELECT 
    COALESCE(d.dept_id, 0) AS dept_id,
    COALESCE(d.dept_name, 'No Dept') AS department,
    COALESCE(e.emp_id, 0) AS emp_id,
    COALESCE(CONCAT(e.first_name, ' ', e.last_name), 'No Employee') AS employee
FROM departments d
LEFT JOIN employees e ON d.dept_id = e.dept_id
UNION ALL
SELECT 
    COALESCE(d.dept_id, 0),
    COALESCE(d.dept_name, 'No Dept'),
    COALESCE(e.emp_id, 0),
    COALESCE(CONCAT(e.first_name, ' ', e.last_name), 'No Employee')
FROM departments d
RIGHT JOIN employees e ON d.dept_id = e.dept_id
WHERE d.dept_id IS NULL;  -- เฉพาะ RIGHT ONLY
```

**ข้อ 3:** หา Symmetric Difference ระหว่าง customers (customer_id) กับ orders (customer_id)

```sql
-- เฉลย
SELECT 
    COALESCE(c.customer_id::TEXT, 'No Customer Record') AS customer_id,
    c.first_name,
    c.last_name,
    o.order_id,
    CASE 
        WHEN c.customer_id IS NULL THEN 'Order without Customer Record'
        WHEN o.order_id IS NULL THEN 'Customer with No Orders'
    END AS mismatch_type
FROM customers c
FULL OUTER JOIN orders o ON c.customer_id = o.customer_id
WHERE c.customer_id IS NULL OR o.customer_id IS NULL;
```

**ข้อ 4:** เปรียบเทียบยอดขายต่อ category ระหว่าง Q1 (Jan-Mar) กับ Q2 (Apr-Jun)

```sql
-- เฉลย
WITH q1_sales AS (
    SELECT p.category, SUM(oi.quantity * oi.unit_price - oi.discount) AS q1_revenue
    FROM products p
    JOIN order_items oi ON p.product_id = oi.product_id
    JOIN orders o ON oi.order_id = o.order_id
    WHERE o.order_date BETWEEN '2024-01-01' AND '2024-03-31'
      AND o.status IN ('completed', 'shipped')
    GROUP BY p.category
),
q2_sales AS (
    SELECT p.category, SUM(oi.quantity * oi.unit_price - oi.discount) AS q2_revenue
    FROM products p
    JOIN order_items oi ON p.product_id = oi.product_id
    JOIN orders o ON oi.order_id = o.order_id
    WHERE o.order_date BETWEEN '2024-04-01' AND '2024-06-30'
      AND o.status IN ('completed', 'shipped')
    GROUP BY p.category
)
SELECT 
    COALESCE(q1.category, q2.category) AS category,
    COALESCE(q1.q1_revenue, 0) AS q1_revenue,
    COALESCE(q2.q2_revenue, 0) AS q2_revenue,
    COALESCE(q2.q2_revenue, 0) - COALESCE(q1.q1_revenue, 0) AS growth,
    CASE 
        WHEN q1.category IS NULL THEN 'New in Q2'
        WHEN q2.category IS NULL THEN 'No Sales in Q2'
        WHEN q2.q2_revenue > q1.q1_revenue THEN 'Growing'
        ELSE 'Declining'
    END AS trend
FROM q1_sales q1
FULL OUTER JOIN q2_sales q2 ON q1.category = q2.category
ORDER BY trend, COALESCE(q2.q2_revenue, q1.q1_revenue) DESC;
```

**ข้อ 5:** จงอธิบายว่าเมื่อใดควรใช้ FULL OUTER JOIN แทน LEFT JOIN

```sql
-- เฉลย: ใช้ FULL OUTER JOIN เมื่อ:
-- 1. ต้องการ reconciliation ข้อมูลจากสองแหล่ง
-- 2. ไม่แน่ใจว่าข้อมูลฝั่งใดครบถ้วนกว่า
-- 3. ต้องการหา symmetric difference (ข้อมูลที่ไม่ match ทั้งสองฝั่ง)
-- 4. เปรียบเทียบข้อมูลสองช่วงเวลา

-- ตัวอย่าง: ตรวจสอบว่า orders มี customer record ครบหรือไม่
-- และ customers ทุกคนมี order หรือไม่
SELECT 
    COALESCE(c.customer_id, o.customer_id) AS id,
    CASE 
        WHEN c.customer_id IS NULL THEN 'Missing customer record'
        WHEN o.customer_id IS NULL THEN 'No orders'
        ELSE 'OK'
    END AS data_quality
FROM customers c
FULL OUTER JOIN (SELECT DISTINCT customer_id FROM orders) o 
    ON c.customer_id = o.customer_id
ORDER BY data_quality;
```

**ข้อ 6:** ใช้ COALESCE กับ FULL OUTER JOIN เพื่อสร้าง unified product list

```sql
-- เฉลย
SELECT 
    COALESCE(p.product_id, oi_agg.product_id) AS product_id,
    COALESCE(p.product_name, 'DELETED PRODUCT') AS product_name,
    COALESCE(p.category, 'N/A') AS category,
    COALESCE(p.price, 0) AS price,
    COALESCE(oi_agg.order_count, 0) AS times_in_orders,
    COALESCE(oi_agg.total_revenue, 0) AS revenue
FROM products p
FULL OUTER JOIN (
    SELECT product_id, 
           COUNT(*) AS order_count,
           SUM(quantity * unit_price - discount) AS total_revenue
    FROM order_items
    GROUP BY product_id
) oi_agg ON p.product_id = oi_agg.product_id
ORDER BY revenue DESC;
```

**ข้อ 7:** สร้าง Monthly Revenue Comparison ด้วย FULL OUTER JOIN

```sql
-- เฉลย: เปรียบเทียบรายเดือน
WITH jan_apr AS (
    SELECT TO_CHAR(order_date, 'YYYY-MM') AS month,
           SUM(total_amount) AS revenue
    FROM orders
    WHERE order_date < '2024-05-01' AND status = 'completed'
    GROUP BY month
),
may_sep AS (
    SELECT TO_CHAR(order_date, 'YYYY-MM') AS month,
           SUM(total_amount) AS revenue
    FROM orders
    WHERE order_date >= '2024-05-01' AND status = 'completed'
    GROUP BY month
)
SELECT 
    COALESCE(ja.month, ms.month) AS period,
    COALESCE(ja.revenue, 0) AS first_half_revenue,
    COALESCE(ms.revenue, 0) AS second_half_revenue
FROM jan_apr ja
FULL OUTER JOIN may_sep ms ON ja.month = ms.month
ORDER BY period;
```

**ข้อ 8:** Data Quality Check ด้วย FULL OUTER JOIN

```sql
-- เฉลย: ตรวจสอบ data integrity
SELECT 
    'orders without items' AS issue_type,
    COUNT(*) AS count
FROM orders o
LEFT JOIN order_items oi ON o.order_id = oi.order_id
WHERE oi.item_id IS NULL AND o.status != 'cancelled'

UNION ALL

SELECT 
    'order_items without valid order',
    COUNT(*)
FROM order_items oi
LEFT JOIN orders o ON oi.order_id = o.order_id
WHERE o.order_id IS NULL

UNION ALL

SELECT 
    'items with invalid product',
    COUNT(*)
FROM order_items oi
LEFT JOIN products p ON oi.product_id = p.product_id
WHERE p.product_id IS NULL;
```

**ข้อ 9:** ใช้ FULL OUTER JOIN เพื่อหา employees ที่ขาด department หรือ departments ที่ขาด employees

```sql
-- เฉลย
SELECT 
    d.dept_id,
    d.dept_name,
    e.emp_id,
    e.first_name || ' ' || e.last_name AS employee,
    CASE 
        WHEN e.emp_id IS NULL THEN 'Department has NO employees'
        WHEN d.dept_id IS NULL THEN 'Employee has NO department'
    END AS issue
FROM departments d
FULL OUTER JOIN employees e ON d.dept_id = e.dept_id
WHERE e.emp_id IS NULL OR d.dept_id IS NULL
ORDER BY issue, d.dept_id NULLS LAST;
```

**ข้อ 10:** เขียน FULL OUTER JOIN ที่รวม 3 ตาราง

```sql
-- เฉลย: customers + orders + products (ทุกอย่าง)
WITH customer_orders AS (
    SELECT c.customer_id, c.first_name, o.order_id, o.total_amount
    FROM customers c
    FULL OUTER JOIN orders o ON c.customer_id = o.customer_id
),
product_order_items AS (
    SELECT p.product_id, p.product_name, oi.order_id
    FROM products p
    FULL OUTER JOIN order_items oi ON p.product_id = oi.product_id
)
SELECT 
    COALESCE(co.customer_id::TEXT, 'N/A') AS customer_id,
    COALESCE(co.first_name, 'Unknown') AS customer,
    COALESCE(co.order_id::TEXT, 'N/A') AS order_id,
    COALESCE(co.total_amount::TEXT, '0') AS order_amount,
    COALESCE(poi.product_name, 'No Product') AS product
FROM customer_orders co
FULL OUTER JOIN product_order_items poi ON co.order_id = poi.order_id
LIMIT 50;
```

---

## สรุปภาค 24

1. **FULL OUTER JOIN** — รวมทุกแถวจากทั้งสองตาราง ไม่ว่าจะ match หรือไม่
2. **MySQL workaround** — ใช้ `LEFT JOIN UNION ALL RIGHT JOIN WHERE left.id IS NULL`
3. **ใช้เมื่อ** — Reconciliation, Data comparison, Finding mismatches
4. **COALESCE** — จำเป็นมากใน FULL JOIN เพื่อจัดการ NULL ทั้งสองฝั่ง
5. **Symmetric Difference** — `WHERE left.id IS NULL OR right.id IS NULL`

**ในภาคถัดไป** จะเรียน CROSS JOIN ที่สร้าง Cartesian Product!
