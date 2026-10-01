# ภาค 26: Self JOIN — การ JOIN ตารางกับตัวเอง

## Self JOIN คืออะไร?

**Self JOIN** คือการ JOIN ตารางกับตัวเอง โดยใช้ **Alias** เพื่อทำให้ SQL engine มองว่าเป็นสองตารางที่ต่างกัน

```
Concept ของ Self JOIN:

employees (original)      employees (aliased as manager)
┌──────┬──────────┬────────────┐   ┌──────┬──────────┐
│empid │ name     │ manager_id │   │empid │ name     │
├──────┼──────────┼────────────┤   ├──────┼──────────┤
│ 1    │ Somchai  │ NULL       │   │ 1    │ Somchai  │
│ 2    │ Wanchai  │ 1          │   │ 2    │ Wanchai  │
│ 4    │ Narong   │ 2          │   │ 4    │ Narong   │
│ 5    │ Siriporn │ 4          │   │ 5    │ Siriporn │
└──────┴──────────┴────────────┘   └──────┴──────────┘
         JOIN ON
         employees.manager_id = managers.emp_id
```

### เมื่อใดใช้ Self JOIN

1. **Hierarchical data** — พนักงาน-ผู้จัดการ, หมวดหมู่-หมวดหมู่ย่อย
2. **Sequential data** — หาแถวก่อนหน้า/ถัดไป
3. **Comparing rows** — เปรียบเทียบแถวในตารางเดียวกัน
4. **Finding duplicates** — หาข้อมูลซ้ำ
5. **Adjacency list** — โครงสร้างข้อมูลแบบ tree

---

## Employee Hierarchy ด้วย Self JOIN

### ตัวอย่างที่ 1: พื้นฐาน — พนักงานกับผู้จัดการ

```sql
-- แสดงพนักงานพร้อมชื่อผู้จัดการ
SELECT 
    e.emp_id,
    CONCAT(e.first_name, ' ', e.last_name) AS employee_name,
    e.job_title,
    CONCAT(m.first_name, ' ', m.last_name) AS manager_name,
    m.job_title AS manager_title
FROM employees e
INNER JOIN employees m ON e.manager_id = m.emp_id
ORDER BY m.emp_id, e.emp_id;

-- หมายเหตุ: CEO (manager_id = NULL) จะไม่ปรากฏเพราะ INNER JOIN
```

### ตัวอย่างที่ 2: LEFT JOIN เพื่อรวม CEO ด้วย

```sql
-- แสดงพนักงานทุกคน รวม CEO ที่ไม่มีผู้จัดการ
SELECT 
    e.emp_id,
    CONCAT(e.first_name, ' ', e.last_name) AS employee,
    e.job_title,
    COALESCE(
        CONCAT(m.first_name, ' ', m.last_name),
        'No Manager (Top Level)'
    ) AS reports_to,
    COALESCE(m.job_title, 'N/A') AS manager_title
FROM employees e
LEFT JOIN employees m ON e.manager_id = m.emp_id
ORDER BY m.emp_id NULLS FIRST, e.emp_id;
```

### ตัวอย่างที่ 3: แสดง Chain of Command (3 ระดับ)

```sql
-- แสดงสายบังคับบัญชา 3 ระดับ: employee → manager → grand_manager
SELECT 
    e.emp_id,
    CONCAT(e.first_name, ' ', e.last_name) AS employee,
    e.job_title,
    CONCAT(m.first_name, ' ', m.last_name) AS manager,
    m.job_title AS manager_title,
    COALESCE(
        CONCAT(gm.first_name, ' ', gm.last_name),
        'Top Level'
    ) AS grand_manager,
    COALESCE(gm.job_title, 'N/A') AS grand_manager_title
FROM employees e
JOIN employees m ON e.manager_id = m.emp_id
LEFT JOIN employees m2 ON m.manager_id = m2.emp_id  -- manager's manager
LEFT JOIN employees gm ON m2.emp_id = gm.emp_id
ORDER BY e.dept_id, e.emp_id;
```

### ตัวอย่างที่ 4: นับจำนวน Direct Reports

```sql
-- นับจำนวนพนักงานที่รายงานตรงต่อแต่ละผู้จัดการ
SELECT 
    m.emp_id AS manager_id,
    CONCAT(m.first_name, ' ', m.last_name) AS manager_name,
    m.job_title,
    COUNT(e.emp_id) AS direct_reports,
    GROUP_CONCAT(CONCAT(e.first_name, ' ', e.last_name) SEPARATOR ', ') AS team_members
FROM employees m
INNER JOIN employees e ON m.emp_id = e.manager_id
GROUP BY m.emp_id, m.first_name, m.last_name, m.job_title
ORDER BY direct_reports DESC;

-- PostgreSQL version (ใช้ STRING_AGG)
SELECT 
    m.emp_id AS manager_id,
    CONCAT(m.first_name, ' ', m.last_name) AS manager_name,
    m.job_title,
    COUNT(e.emp_id) AS direct_reports,
    STRING_AGG(CONCAT(e.first_name, ' ', e.last_name), ', ') AS team_members
FROM employees m
INNER JOIN employees e ON m.emp_id = e.manager_id
GROUP BY m.emp_id, m.first_name, m.last_name, m.job_title
ORDER BY direct_reports DESC;
```

### ตัวอย่างที่ 5: Hierarchy Tree ด้วย ASCII

```sql
-- สร้าง text tree ของ hierarchy
WITH RECURSIVE emp_tree AS (
    -- Anchor: CEO (top level)
    SELECT 
        emp_id,
        first_name || ' ' || last_name AS name,
        job_title,
        manager_id,
        0 AS level,
        CAST(first_name || ' ' || last_name AS VARCHAR(500)) AS path
    FROM employees
    WHERE manager_id IS NULL
    
    UNION ALL
    
    -- Recursive: employees ที่มี manager
    SELECT 
        e.emp_id,
        e.first_name || ' ' || e.last_name,
        e.job_title,
        e.manager_id,
        et.level + 1,
        et.path || ' > ' || e.first_name || ' ' || e.last_name
    FROM employees e
    JOIN emp_tree et ON e.manager_id = et.emp_id
)
SELECT 
    REPEAT('  ', level) || CASE level WHEN 0 THEN '' ELSE '└─ ' END || name AS org_chart,
    job_title,
    level AS hierarchy_level,
    path
FROM emp_tree
ORDER BY path;
```

---

## Finding Duplicates ด้วย Self JOIN

### ตัวอย่างที่ 6: หา Duplicate Email

```sql
-- หา customers ที่มี email ซ้ำกัน (สมมติ constraint ไม่ strict)
SELECT 
    a.customer_id AS id1,
    a.email,
    a.first_name AS name1,
    b.customer_id AS id2,
    b.first_name AS name2
FROM customers a
INNER JOIN customers b ON a.email = b.email
    AND a.customer_id < b.customer_id  -- หลีกเลี่ยง duplicate pairs
ORDER BY a.email;
```

### ตัวอย่างที่ 7: หา Duplicate Name

```sql
-- หาพนักงานที่มีชื่อ-สกุลเหมือนกัน
SELECT 
    e1.emp_id AS emp_id_1,
    CONCAT(e1.first_name, ' ', e1.last_name) AS name,
    e1.email AS email_1,
    e2.emp_id AS emp_id_2,
    e2.email AS email_2
FROM employees e1
INNER JOIN employees e2 
    ON e1.first_name = e2.first_name 
    AND e1.last_name = e2.last_name
    AND e1.emp_id < e2.emp_id  -- ป้องกัน duplicate pairs
ORDER BY name;
```

### ตัวอย่างที่ 8: หา Products ในราคาใกล้เคียง

```sql
-- หาสินค้าที่ราคาต่างกันไม่เกิน 10% (เปรียบเทียบตัวเอง)
SELECT 
    p1.product_name AS product_1,
    p1.price AS price_1,
    p2.product_name AS product_2,
    p2.price AS price_2,
    ABS(p1.price - p2.price) AS price_diff,
    ROUND(ABS(p1.price - p2.price) / GREATEST(p1.price, p2.price) * 100, 1) AS pct_diff
FROM products p1
INNER JOIN products p2 
    ON p1.category = p2.category          -- หมวดเดียวกัน
    AND p1.product_id < p2.product_id     -- หลีกเลี่ยง duplicates
    AND ABS(p1.price - p2.price) / GREATEST(p1.price, p2.price) <= 0.10  -- ต่างกัน <= 10%
ORDER BY p1.category, price_diff;
```

---

## Comparing Sequential Rows

### ตัวอย่างที่ 9: เปรียบเทียบ Orders ต่อเนื่องของลูกค้า

```sql
-- เปรียบเทียบ order ปัจจุบันกับ order ก่อนหน้าของลูกค้าเดียวกัน
SELECT 
    o1.customer_id,
    c.first_name || ' ' || c.last_name AS customer,
    o1.order_id AS current_order,
    o1.order_date AS current_date,
    o1.total_amount AS current_amount,
    o2.order_id AS previous_order,
    o2.order_date AS previous_date,
    o2.total_amount AS previous_amount,
    o1.total_amount - o2.total_amount AS amount_change,
    o1.order_date - o2.order_date AS days_between
FROM orders o1
JOIN orders o2 ON o1.customer_id = o2.customer_id
    AND o1.order_date > o2.order_date  -- o1 มาหลัง o2
    AND NOT EXISTS (
        -- ไม่มี order อื่นที่อยู่ระหว่าง o2 และ o1
        SELECT 1 FROM orders o3
        WHERE o3.customer_id = o1.customer_id
          AND o3.order_date > o2.order_date
          AND o3.order_date < o1.order_date
    )
JOIN customers c ON o1.customer_id = c.customer_id
ORDER BY o1.customer_id, o1.order_date;
```

### ตัวอย่างที่ 10: หา employees ที่มีเงินเดือนมากกว่าผู้จัดการ

```sql
-- พนักงานที่ได้เงินมากกว่าหัวหน้าตัวเอง (unusual case)
SELECT 
    e.emp_id,
    CONCAT(e.first_name, ' ', e.last_name) AS employee,
    e.job_title,
    e.salary AS emp_salary,
    CONCAT(m.first_name, ' ', m.last_name) AS manager,
    m.job_title AS manager_title,
    m.salary AS manager_salary,
    e.salary - m.salary AS salary_difference
FROM employees e
JOIN employees m ON e.manager_id = m.emp_id
WHERE e.salary > m.salary
ORDER BY salary_difference DESC;
```

### ตัวอย่างที่ 11: เปรียบเทียบเงินเดือนภายในแผนก

```sql
-- เปรียบเทียบเงินเดือนทุก pair ในแผนกเดียวกัน
SELECT 
    e1.dept_id,
    d.dept_name,
    CONCAT(e1.first_name, ' ', e1.last_name) AS employee_1,
    e1.salary AS salary_1,
    CONCAT(e2.first_name, ' ', e2.last_name) AS employee_2,
    e2.salary AS salary_2,
    e1.salary - e2.salary AS salary_gap
FROM employees e1
JOIN employees e2 ON e1.dept_id = e2.dept_id
    AND e1.emp_id < e2.emp_id  -- หลีกเลี่ยง duplicates
JOIN departments d ON e1.dept_id = d.dept_id
WHERE ABS(e1.salary - e2.salary) > 30000  -- ห่างกันมาก
ORDER BY salary_gap DESC;
```

---

## Adjacency List Model

ตาราง `employees` ใช้ **Adjacency List** model:
- แต่ละ node เก็บ `parent_id` ของตัวเอง
- ง่ายต่อการ insert/update
- แต่ต้องใช้ recursive query เพื่อ traverse tree ลึกๆ

### ตัวอย่างที่ 12: ดู Subtree ของ VP Engineering

```sql
-- หาพนักงานทุกคนที่อยู่ใต้ VP Engineering (emp_id = 4)
WITH RECURSIVE subordinates AS (
    -- Anchor: VP Engineering
    SELECT emp_id, first_name, last_name, job_title, manager_id, 0 AS depth
    FROM employees
    WHERE emp_id = 4
    
    UNION ALL
    
    -- Recursive: ทุกคนที่รายงานต่อ subtree
    SELECT e.emp_id, e.first_name, e.last_name, e.job_title, e.manager_id, sub.depth + 1
    FROM employees e
    JOIN subordinates sub ON e.manager_id = sub.emp_id
)
SELECT 
    depth,
    REPEAT('  ', depth) || first_name || ' ' || last_name AS name,
    job_title
FROM subordinates
ORDER BY depth, last_name;
```

### ตัวอย่างที่ 13: หา Path จาก Employee ถึง CEO

```sql
-- ติดตาม path จากพนักงานขึ้นไปจนถึง CEO
WITH RECURSIVE chain AS (
    -- Anchor: พนักงานที่ต้องการ
    SELECT emp_id, first_name || ' ' || last_name AS name, 
           job_title, manager_id, 0 AS level
    FROM employees
    WHERE emp_id = 7  -- ตัวอย่าง: Malee (Engineer)
    
    UNION ALL
    
    -- Recursive: ขึ้นไปหาผู้จัดการ
    SELECT e.emp_id, e.first_name || ' ' || e.last_name,
           e.job_title, e.manager_id, c.level + 1
    FROM employees e
    JOIN chain c ON e.emp_id = c.manager_id
)
SELECT level, name, job_title
FROM chain
ORDER BY level DESC;  -- เริ่มจาก CEO ลงมา
```

### ตัวอย่างที่ 14: Count Total Subordinates (ทุกระดับ)

```sql
-- นับพนักงานทั้งหมดใต้ผู้จัดการแต่ละคน (ทุกระดับ)
WITH RECURSIVE all_subs AS (
    SELECT manager_id, emp_id, 1 AS depth
    FROM employees
    WHERE manager_id IS NOT NULL
    
    UNION ALL
    
    SELECT e.manager_id, sub.emp_id, sub.depth + 1
    FROM employees e
    JOIN all_subs sub ON e.emp_id = sub.manager_id
    WHERE e.manager_id IS NOT NULL
)
SELECT 
    m.emp_id,
    CONCAT(m.first_name, ' ', m.last_name) AS manager,
    m.job_title,
    COUNT(DISTINCT sub.emp_id) AS total_subordinates
FROM employees m
LEFT JOIN all_subs sub ON m.emp_id = sub.manager_id
GROUP BY m.emp_id, m.first_name, m.last_name, m.job_title
ORDER BY total_subordinates DESC;
```

---

## ตัวอย่าง Self JOIN ขั้นสูง (15-25)

### ตัวอย่างที่ 15: Peer Salary Comparison

```sql
-- เปรียบเทียบเงินเดือนกับพนักงานที่มี title เดียวกัน
SELECT 
    e1.emp_id,
    CONCAT(e1.first_name, ' ', e1.last_name) AS employee,
    e1.job_title,
    e1.salary,
    AVG(e2.salary) OVER (PARTITION BY e1.job_title) AS avg_peer_salary,
    e1.salary - AVG(e2.salary) OVER (PARTITION BY e1.job_title) AS vs_peers,
    COUNT(e2.emp_id) OVER (PARTITION BY e1.job_title) AS peer_count
FROM employees e1
JOIN employees e2 ON e1.job_title = e2.job_title
    AND e1.emp_id != e2.emp_id
ORDER BY e1.job_title, e1.salary DESC;
```

### ตัวอย่างที่ 16: หา Employee ที่ Hire ในวันเดียวกัน

```sql
-- หาพนักงานที่เข้างานในวันเดียวกัน
SELECT 
    e1.emp_id AS emp1_id,
    CONCAT(e1.first_name, ' ', e1.last_name) AS employee_1,
    e2.emp_id AS emp2_id,
    CONCAT(e2.first_name, ' ', e2.last_name) AS employee_2,
    e1.hire_date AS same_hire_date
FROM employees e1
JOIN employees e2 ON e1.hire_date = e2.hire_date
    AND e1.emp_id < e2.emp_id
ORDER BY e1.hire_date, e1.emp_id;
```

### ตัวอย่างที่ 17: Category Hierarchy (สมมติมี parent_category)

```sql
-- สมมติ products มี parent_category field
CREATE TEMP TABLE categories_hierarchy (
    cat_id INT PRIMARY KEY,
    cat_name VARCHAR(50),
    parent_cat_id INT
);

INSERT INTO categories_hierarchy VALUES
(1, 'All Products', NULL),
(2, 'Electronics', 1),
(3, 'Furniture', 1),
(4, 'Accessories', 1),
(5, 'Computers', 2),
(6, 'Mobile', 2),
(7, 'Office Furniture', 3),
(8, 'Home Furniture', 3),
(9, 'Laptop', 5),
(10, 'Desktop', 5);

-- Self JOIN เพื่อแสดง hierarchy
SELECT 
    c.cat_id,
    c.cat_name,
    p.cat_name AS parent_category,
    COALESCE(p.cat_name || ' > ', '') || c.cat_name AS full_path
FROM categories_hierarchy c
LEFT JOIN categories_hierarchy p ON c.parent_cat_id = p.cat_id
ORDER BY full_path;
```

### ตัวอย่างที่ 18: หา Employees ที่ Join ก่อน Manager ของตัวเอง

```sql
-- พนักงานที่ทำงานนานกว่าผู้จัดการตัวเอง (Senior แต่ rank ต่ำกว่า)
SELECT 
    e.emp_id,
    CONCAT(e.first_name, ' ', e.last_name) AS employee,
    e.hire_date AS emp_hire_date,
    e.job_title,
    CONCAT(m.first_name, ' ', m.last_name) AS manager,
    m.hire_date AS manager_hire_date,
    m.job_title AS manager_title,
    m.hire_date - e.hire_date AS days_longer_than_manager
FROM employees e
JOIN employees m ON e.manager_id = m.emp_id
WHERE e.hire_date < m.hire_date  -- พนักงาน join ก่อน manager
ORDER BY days_longer_than_manager DESC;
```

### ตัวอย่างที่ 19: Multi-Level Bonus Calculation

```sql
-- คำนวณ bonus โดยคำนึงถึง hierarchy
-- Manager ได้ bonus เพิ่มตาม performance ของทีม
SELECT 
    m.emp_id AS manager_id,
    CONCAT(m.first_name, ' ', m.last_name) AS manager,
    m.salary AS manager_salary,
    COUNT(e.emp_id) AS team_size,
    AVG(e.salary) AS avg_team_salary,
    -- Manager bonus: 5% ของ salary รวมทีม
    ROUND(SUM(e.salary) * 0.05, 2) AS team_bonus_contribution,
    ROUND(m.salary * 0.10, 2) AS base_bonus,
    ROUND(m.salary * 0.10 + SUM(e.salary) * 0.05, 2) AS total_bonus
FROM employees m
JOIN employees e ON e.manager_id = m.emp_id
GROUP BY m.emp_id, m.first_name, m.last_name, m.salary
ORDER BY total_bonus DESC;
```

### ตัวอย่างที่ 20: Finding Sibling Nodes (พนักงาน Level เดียวกัน)

```sql
-- หา "siblings" — พนักงานที่รายงานต่อผู้จัดการเดียวกัน
SELECT 
    e1.emp_id AS emp1_id,
    CONCAT(e1.first_name, ' ', e1.last_name) AS employee_1,
    e1.job_title AS title_1,
    e2.emp_id AS emp2_id,
    CONCAT(e2.first_name, ' ', e2.last_name) AS employee_2,
    e2.job_title AS title_2,
    CONCAT(m.first_name, ' ', m.last_name) AS shared_manager,
    ABS(e1.salary - e2.salary) AS salary_difference
FROM employees e1
JOIN employees e2 ON e1.manager_id = e2.manager_id
    AND e1.emp_id < e2.emp_id
JOIN employees m ON e1.manager_id = m.emp_id
ORDER BY shared_manager, e1.emp_id;
```

### ตัวอย่างที่ 21: Manager Span of Control Analysis

```sql
-- วิเคราะห์ span of control ของแต่ละ manager
SELECT 
    m.emp_id,
    CONCAT(m.first_name, ' ', m.last_name) AS manager,
    m.job_title,
    d.dept_name,
    COUNT(e.emp_id) AS direct_reports,
    MIN(e.salary) AS team_min_salary,
    MAX(e.salary) AS team_max_salary,
    AVG(e.salary) AS team_avg_salary,
    MAX(e.salary) - MIN(e.salary) AS salary_spread,
    CASE 
        WHEN COUNT(e.emp_id) = 0 THEN 'No Direct Reports'
        WHEN COUNT(e.emp_id) <= 3 THEN 'Small Team'
        WHEN COUNT(e.emp_id) <= 6 THEN 'Medium Team'
        ELSE 'Large Team'
    END AS team_size_category
FROM employees m
LEFT JOIN employees e ON m.emp_id = e.manager_id
JOIN departments d ON m.dept_id = d.dept_id
GROUP BY m.emp_id, m.first_name, m.last_name, m.job_title, d.dept_name
ORDER BY direct_reports DESC;
```

### ตัวอย่างที่ 22: Organizational Distance

```sql
-- หาระยะห่างระหว่างพนักงานสองคนใน org chart
-- (ระดับในองค์กร)
WITH RECURSIVE emp_levels AS (
    SELECT emp_id, first_name || ' ' || last_name AS name, 
           manager_id, 1 AS level
    FROM employees
    WHERE manager_id IS NULL  -- CEO = level 1
    
    UNION ALL
    
    SELECT e.emp_id, e.first_name || ' ' || e.last_name,
           e.manager_id, el.level + 1
    FROM employees e
    JOIN emp_levels el ON e.manager_id = el.emp_id
)
SELECT 
    e1.name AS employee_1,
    e1.level AS level_1,
    e2.name AS employee_2,
    e2.level AS level_2,
    ABS(e1.level - e2.level) AS level_distance
FROM emp_levels e1
CROSS JOIN emp_levels e2
WHERE e1.emp_id < e2.emp_id
  AND ABS(e1.level - e2.level) <= 1  -- อยู่ระดับใกล้กัน
ORDER BY level_distance, e1.name;
```

### ตัวอย่างที่ 23: Succession Planning

```sql
-- หา potential successors: คนที่มี tenure ยาวและ salary สูงใน dept เดียวกัน
SELECT 
    m.emp_id AS current_manager_id,
    CONCAT(m.first_name, ' ', m.last_name) AS current_manager,
    m.job_title,
    m.hire_date AS manager_hire_date,
    e.emp_id AS potential_successor_id,
    CONCAT(e.first_name, ' ', e.last_name) AS potential_successor,
    e.job_title AS current_title,
    e.hire_date AS successor_hire_date,
    e.salary,
    EXTRACT(YEAR FROM AGE(CURRENT_DATE, e.hire_date)) AS years_experience,
    RANK() OVER (PARTITION BY m.emp_id ORDER BY e.salary DESC, e.hire_date ASC) AS succession_rank
FROM employees m
JOIN employees e ON m.dept_id = e.dept_id
    AND m.emp_id != e.emp_id
    AND e.manager_id = m.emp_id  -- direct reports
WHERE m.job_title LIKE '%Manager%' OR m.job_title LIKE '%Director%'
ORDER BY m.emp_id, succession_rank;
```

### ตัวอย่างที่ 24: Department Comparison

```sql
-- เปรียบเทียบเงินเดือนระหว่างแผนก (ใช้ Self JOIN บน departments)
SELECT 
    d1.dept_name AS dept_1,
    ROUND(AVG(e1.salary), 2) AS avg_salary_1,
    d2.dept_name AS dept_2,
    ROUND(AVG(e2.salary), 2) AS avg_salary_2,
    ROUND(AVG(e1.salary) - AVG(e2.salary), 2) AS salary_difference
FROM employees e1
JOIN departments d1 ON e1.dept_id = d1.dept_id
JOIN employees e2 ON e2.dept_id != e1.dept_id
JOIN departments d2 ON e2.dept_id = d2.dept_id
WHERE d1.dept_id < d2.dept_id  -- หลีกเลี่ยง duplicate pairs
GROUP BY d1.dept_id, d1.dept_name, d2.dept_id, d2.dept_name
ORDER BY ABS(AVG(e1.salary) - AVG(e2.salary)) DESC
LIMIT 10;
```

### ตัวอย่างที่ 25: Customer Purchase Pattern (Self JOIN บน orders)

```sql
-- หาลูกค้าที่สั่งสินค้าสองครั้งใน 30 วัน
SELECT 
    c.customer_id,
    c.first_name || ' ' || c.last_name AS customer,
    o1.order_id AS first_order,
    o1.order_date AS first_date,
    o1.total_amount AS first_amount,
    o2.order_id AS second_order,
    o2.order_date AS second_date,
    o2.total_amount AS second_amount,
    o2.order_date - o1.order_date AS days_between
FROM orders o1
JOIN orders o2 ON o1.customer_id = o2.customer_id
    AND o2.order_date > o1.order_date  -- o2 มาหลัง
    AND o2.order_date <= o1.order_date + INTERVAL '30 days'  -- ใน 30 วัน
JOIN customers c ON o1.customer_id = c.customer_id
ORDER BY days_between ASC, o1.customer_id;
```

---

## แบบฝึกหัดภาค 26

**ข้อ 1:** แสดงพนักงานทุกคนพร้อมชื่อผู้จัดการ (รวม CEO ที่ไม่มี manager)

```sql
-- เฉลย
SELECT 
    e.emp_id,
    CONCAT(e.first_name, ' ', e.last_name) AS employee,
    e.job_title,
    e.salary,
    COALESCE(CONCAT(m.first_name, ' ', m.last_name), 'TOP LEVEL') AS manager,
    COALESCE(m.job_title, '-') AS manager_title
FROM employees e
LEFT JOIN employees m ON e.manager_id = m.emp_id
ORDER BY m.emp_id NULLS FIRST, e.emp_id;
```

**ข้อ 2:** หาผู้จัดการที่มี direct reports มากที่สุด 3 อันดับ

```sql
-- เฉลย
SELECT 
    m.emp_id,
    CONCAT(m.first_name, ' ', m.last_name) AS manager,
    m.job_title,
    COUNT(e.emp_id) AS direct_reports
FROM employees m
INNER JOIN employees e ON m.emp_id = e.manager_id
GROUP BY m.emp_id, m.first_name, m.last_name, m.job_title
ORDER BY direct_reports DESC
LIMIT 3;
```

**ข้อ 3:** หา employees ที่ได้เงินเดือนมากกว่าผู้จัดการตัวเอง

```sql
-- เฉลย
SELECT 
    e.emp_id,
    CONCAT(e.first_name, ' ', e.last_name) AS employee,
    e.salary AS emp_salary,
    CONCAT(m.first_name, ' ', m.last_name) AS manager,
    m.salary AS manager_salary,
    e.salary - m.salary AS excess
FROM employees e
JOIN employees m ON e.manager_id = m.emp_id
WHERE e.salary > m.salary
ORDER BY excess DESC;
```

**ข้อ 4:** แสดง 3 ระดับ hierarchy: employee → manager → VP

```sql
-- เฉลย
SELECT 
    e.emp_id,
    CONCAT(e.first_name, ' ', e.last_name) AS employee,
    e.job_title AS emp_title,
    CONCAT(m.first_name, ' ', m.last_name) AS direct_manager,
    m.job_title AS mgr_title,
    COALESCE(CONCAT(gm.first_name, ' ', gm.last_name), 'Top') AS grand_manager,
    COALESCE(gm.job_title, '-') AS gm_title
FROM employees e
LEFT JOIN employees m ON e.manager_id = m.emp_id
LEFT JOIN employees gm ON m.manager_id = gm.emp_id
ORDER BY e.dept_id, e.emp_id;
```

**ข้อ 5:** หาพนักงานในแผนกเดียวกันที่เข้างานในปีเดียวกัน

```sql
-- เฉลย
SELECT 
    e1.emp_id AS emp1,
    CONCAT(e1.first_name, ' ', e1.last_name) AS employee_1,
    e2.emp_id AS emp2,
    CONCAT(e2.first_name, ' ', e2.last_name) AS employee_2,
    d.dept_name,
    EXTRACT(YEAR FROM e1.hire_date) AS hire_year
FROM employees e1
JOIN employees e2 ON e1.dept_id = e2.dept_id
    AND EXTRACT(YEAR FROM e1.hire_date) = EXTRACT(YEAR FROM e2.hire_date)
    AND e1.emp_id < e2.emp_id
JOIN departments d ON e1.dept_id = d.dept_id
ORDER BY d.dept_name, hire_year;
```

**ข้อ 6:** หาพนักงานที่มีเงินเดือนสูงกว่าค่าเฉลี่ยของ "siblings" (คนที่มี manager เดียวกัน)

```sql
-- เฉลย
WITH team_avg AS (
    SELECT manager_id, AVG(salary) AS avg_team_salary
    FROM employees
    WHERE manager_id IS NOT NULL
    GROUP BY manager_id
)
SELECT 
    e.emp_id,
    CONCAT(e.first_name, ' ', e.last_name) AS employee,
    e.salary,
    ta.avg_team_salary,
    e.salary - ta.avg_team_salary AS above_team_avg,
    CONCAT(m.first_name, ' ', m.last_name) AS manager
FROM employees e
JOIN team_avg ta ON e.manager_id = ta.manager_id
JOIN employees m ON e.manager_id = m.emp_id
WHERE e.salary > ta.avg_team_salary
ORDER BY above_team_avg DESC;
```

**ข้อ 7:** นับจำนวน subordinates ทุกระดับของ CEO

```sql
-- เฉลย: ใช้ Recursive CTE
WITH RECURSIVE all_under_ceo AS (
    SELECT emp_id, manager_id, 0 AS depth
    FROM employees WHERE manager_id IS NULL  -- CEO
    
    UNION ALL
    
    SELECT e.emp_id, e.manager_id, auc.depth + 1
    FROM employees e
    JOIN all_under_ceo auc ON e.manager_id = auc.emp_id
)
SELECT 
    depth AS hierarchy_level,
    COUNT(*) - CASE WHEN depth = 0 THEN 1 ELSE 0 END AS employees_at_level
FROM all_under_ceo
GROUP BY depth
ORDER BY depth;
```

**ข้อ 8:** หาคู่พนักงานที่ hire_date ห่างกันน้อยกว่า 30 วัน

```sql
-- เฉลย
SELECT 
    e1.emp_id AS emp1,
    CONCAT(e1.first_name, ' ', e1.last_name) AS employee_1,
    e1.hire_date AS hire_1,
    e2.emp_id AS emp2,
    CONCAT(e2.first_name, ' ', e2.last_name) AS employee_2,
    e2.hire_date AS hire_2,
    ABS(e1.hire_date - e2.hire_date) AS days_apart
FROM employees e1
JOIN employees e2 ON e1.emp_id < e2.emp_id
    AND ABS(e1.hire_date - e2.hire_date) < 30
ORDER BY days_apart, e1.emp_id;
```

**ข้อ 9:** แสดง org chart level สำหรับพนักงานทุกคน

```sql
-- เฉลย
WITH RECURSIVE levels AS (
    SELECT emp_id, first_name || ' ' || last_name AS name,
           job_title, manager_id, 1 AS level
    FROM employees WHERE manager_id IS NULL
    
    UNION ALL
    
    SELECT e.emp_id, e.first_name || ' ' || e.last_name,
           e.job_title, e.manager_id, l.level + 1
    FROM employees e
    JOIN levels l ON e.manager_id = l.emp_id
)
SELECT 
    level,
    COUNT(*) AS employees_at_this_level,
    STRING_AGG(name, ', ' ORDER BY name) AS names
FROM levels
GROUP BY level
ORDER BY level;
```

**ข้อ 10:** สร้าง Mentor-Mentee pairs: Senior (5+ ปี) กับ Junior (< 2 ปี) ในแผนกเดียวกัน

```sql
-- เฉลย
SELECT 
    s.emp_id AS mentor_id,
    CONCAT(s.first_name, ' ', s.last_name) AS mentor,
    s.job_title AS mentor_title,
    EXTRACT(YEAR FROM AGE(CURRENT_DATE, s.hire_date)) AS mentor_years,
    j.emp_id AS mentee_id,
    CONCAT(j.first_name, ' ', j.last_name) AS mentee,
    j.job_title AS mentee_title,
    EXTRACT(YEAR FROM AGE(CURRENT_DATE, j.hire_date)) AS mentee_years,
    d.dept_name
FROM employees s  -- Senior
JOIN employees j ON s.dept_id = j.dept_id  -- แผนกเดียวกัน
    AND s.emp_id != j.emp_id
JOIN departments d ON s.dept_id = d.dept_id
WHERE EXTRACT(YEAR FROM AGE(CURRENT_DATE, s.hire_date)) >= 5
  AND EXTRACT(YEAR FROM AGE(CURRENT_DATE, j.hire_date)) < 2
ORDER BY d.dept_name, mentor, mentee;
```

---

## สรุปภาค 26

1. **Self JOIN** — JOIN ตารางกับตัวเอง โดยใช้ Alias ต่างกัน
2. **ใช้เพื่อ** — Hierarchy, Duplicate finding, Sequential comparison, Adjacency list
3. **LEFT JOIN** — ใช้เมื่อต้องการรวม root node (ไม่มี parent)
4. **INNER JOIN** — ใช้เมื่อต้องการเฉพาะ nodes ที่มี parent
5. **Recursive CTE** — สำหรับ traverse tree ลึกๆ ไม่จำกัดระดับ
6. **ป้องกัน duplicates** — ใช้ `e1.id < e2.id` เมื่อเปรียบเทียบ pair

**ในภาคถัดไป** จะเรียนการ JOIN 4+ ตารางพร้อมกัน!
