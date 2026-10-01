# Part 007: Working with Column Aliases

> **หลักสูตร SQL ครบวงจร | Part 7 of 120**

---

## 🎯 สิ่งที่จะได้เรียนรู้ในบทนี้

- Column aliases ด้วย AS keyword
- Alias ไม่มี AS (shorthand)
- Aliases สำหรับ expressions
- Table aliases
- Aliases ใน ORDER BY
- Aliases ใน GROUP BY
- ข้อจำกัดของ aliases
- Best practices

**เวลาที่ใช้เรียน**: ประมาณ 1.5 ชั่วโมง

---

## 7.1 Column Aliases พื้นฐาน

```sql
-- ============================================================
-- EXAMPLE 1: AS keyword พื้นฐาน
-- ============================================================

-- ไม่มี alias
SELECT first_name, salary FROM employees;
-- column names = "first_name", "salary"

-- มี alias
SELECT first_name AS name, salary AS monthly_pay FROM employees;
-- column names = "name", "monthly_pay"

-- ============================================================
-- EXAMPLE 2: AS optional (ละได้)
-- ============================================================

-- มี AS
SELECT first_name AS fname FROM employees;

-- ไม่มี AS (ทำงานเหมือนกัน)
SELECT first_name fname FROM employees;

-- แนะนำ: ใช้ AS เสมอ เพื่อความชัดเจน

-- ============================================================
-- EXAMPLE 3: Aliases สำหรับ Expressions
-- ============================================================

SELECT 
    first_name,
    last_name,
    salary * 12 AS annual_salary,
    salary * 12 * 0.05 AS annual_provident_fund,
    ROUND(salary / 30, 2) AS daily_rate
FROM employees;

-- ผลลัพธ์:
-- first_name | last_name | annual_salary | annual_provident_fund | daily_rate
-- -----------|-----------|---------------|----------------------|----------
-- สมชาย      | นักเขียน  | 900000.00     | 45000.00             | 2500.00
-- สมหญิง     | ดีงาม     | 780000.00     | 39000.00             | 2166.67
-- ...

-- ============================================================
-- EXAMPLE 4: Aliases สำหรับ Concatenated Strings
-- ============================================================

SELECT 
    first_name || ' ' || last_name AS full_name,
    'Employee ID: ' || employee_id AS employee_code,
    email AS contact_email
FROM employees;

-- ============================================================
-- EXAMPLE 5: Aliases สำหรับ Functions
-- ============================================================

SELECT 
    UPPER(first_name) AS first_name_upper,
    LOWER(last_name) AS last_name_lower,
    LENGTH(email) AS email_length,
    ROUND(salary, -3) AS salary_rounded_k
FROM employees
LIMIT 5;
```

---

## 7.2 Double-Quoted Aliases

```sql
-- ============================================================
-- EXAMPLE 6: Aliases ที่มี spaces (ต้องใช้ quotes)
-- ============================================================

-- ใช้ double quotes สำหรับ alias ที่มี space
SELECT 
    first_name AS "First Name",
    last_name AS "Last Name",
    salary AS "Monthly Salary (THB)",
    salary * 12 AS "Annual Salary"
FROM employees;

-- หมายเหตุ: SQLite, PostgreSQL, SQL Server ใช้ double quotes
-- MySQL ใช้ backticks: `First Name` หรือ double quotes ก็ได้

-- ============================================================
-- EXAMPLE 7: Aliases ที่มี special characters
-- ============================================================

SELECT 
    unit_price AS "Price (THB)",
    unit_price * 1.07 AS "Price + VAT (7%)",
    units_in_stock AS "In Stock #"
FROM products;

-- ============================================================
-- EXAMPLE 8: Aliases ที่เป็น Reserved Words
-- ============================================================

-- "date", "name", "order" เป็น reserved words
-- ต้องใช้ quotes ถ้าจะใช้เป็น alias:
SELECT 
    order_date AS "date",    -- ✓ ใช้ quotes
    product_name AS "name",  -- ✓ ใช้ quotes
    order_id AS "order"      -- ✓ ใช้ quotes
FROM orders;

-- แนะนำ: หลีกเลี่ยงการใช้ reserved words เป็น alias
SELECT 
    order_date AS order_dt,      -- ✓ ดีกว่า
    product_name AS prod_name,
    order_id AS ord_id
FROM orders;
```

---

## 7.3 Table Aliases

```sql
-- ============================================================
-- EXAMPLE 9: Table Aliases พื้นฐาน
-- ============================================================

-- ไม่มี table alias
SELECT employees.first_name, employees.salary
FROM employees;

-- มี table alias
SELECT e.first_name, e.salary
FROM employees AS e;

-- ไม่มี AS (shorthand)
SELECT e.first_name, e.salary
FROM employees e;

-- ============================================================
-- EXAMPLE 10: Table Aliases ใน JOIN (สำคัญมาก)
-- ============================================================

-- ไม่ใช้ alias (ยาว อ่านยาก)
SELECT 
    employees.first_name,
    employees.last_name,
    departments.department_name
FROM employees
JOIN departments ON employees.department_id = departments.department_id;

-- ใช้ alias (สั้น อ่านง่าย)
SELECT 
    e.first_name,
    e.last_name,
    d.department_name
FROM employees AS e
JOIN departments AS d ON e.department_id = d.department_id;

-- ============================================================
-- EXAMPLE 11: Table Aliases ที่มีความหมาย
-- ============================================================

-- ❌ ไม่ดี: alias ไม่มีความหมาย
SELECT a.first_name, b.department_name
FROM employees a JOIN departments b ON a.department_id = b.department_id;

-- ✓ ดี: alias มีความหมาย
SELECT emp.first_name, dept.department_name
FROM employees emp JOIN departments dept ON emp.department_id = dept.department_id;

-- ============================================================
-- EXAMPLE 12: Self Join ต้องใช้ Table Alias
-- ============================================================

-- ดูพนักงานพร้อมชื่อ manager (self join)
SELECT 
    e.first_name AS employee_name,
    m.first_name AS manager_name
FROM employees AS e
LEFT JOIN employees AS m ON e.manager_id = m.employee_id;

-- ผลลัพธ์:
-- employee_name | manager_name
-- --------------|-------------
-- สมชาย         | NULL
-- สมหญิง        | NULL
-- ...
-- มาลี          | สมชาย
-- อนันต์        | สมชาย
-- จินตนา        | สมชาย
-- รัตนา         | สมหญิง
-- ...
```

---

## 7.4 Aliases ใน ORDER BY, GROUP BY, HAVING

```sql
-- ============================================================
-- EXAMPLE 13: Alias ใน ORDER BY (✓ ได้)
-- ============================================================

SELECT 
    first_name,
    salary * 12 AS annual_salary
FROM employees
ORDER BY annual_salary DESC;  -- ใช้ alias ใน ORDER BY ได้!

-- ✓ ทำงานได้เพราะ ORDER BY ประมวลผลหลัง SELECT

-- ============================================================
-- EXAMPLE 14: Alias ใน GROUP BY (บางครั้ง)
-- ============================================================

-- PostgreSQL / MySQL: สามารถใช้ alias ใน GROUP BY ได้
SELECT 
    YEAR(hire_date) AS hire_year,
    COUNT(*) AS employee_count
FROM employees
GROUP BY hire_year;   -- ใช้ alias ใน GROUP BY (MySQL/PostgreSQL)

-- SQLite / Standard SQL: ต้องใช้ expression ซ้ำ
SELECT 
    strftime('%Y', hire_date) AS hire_year,
    COUNT(*) AS employee_count
FROM employees
GROUP BY strftime('%Y', hire_date);  -- ซ้ำ expression ใน GROUP BY

-- ============================================================
-- EXAMPLE 15: Alias ใน WHERE (❌ ไม่ได้!)
-- ============================================================

-- ❌ ผิด! ใช้ alias ใน WHERE ไม่ได้
SELECT first_name, salary * 12 AS annual_salary
FROM employees
WHERE annual_salary > 700000;  -- ERROR: column "annual_salary" does not exist

-- ✓ ถูก: ใช้ expression ตรงๆ ใน WHERE
SELECT first_name, salary * 12 AS annual_salary
FROM employees
WHERE salary * 12 > 700000;  -- ซ้ำ expression ใน WHERE

-- ✓ หรือใช้ subquery / CTE
SELECT * FROM (
    SELECT first_name, salary * 12 AS annual_salary
    FROM employees
) sub
WHERE sub.annual_salary > 700000;

-- ============================================================
-- EXAMPLE 16: Alias ใน HAVING (❌ ขึ้นกับ database)
-- ============================================================

-- ❌ Standard SQL: alias ใน HAVING ไม่ได้
SELECT department_id, AVG(salary) AS avg_salary
FROM employees
GROUP BY department_id
HAVING avg_salary > 60000;  -- อาจ error ใน strict SQL

-- ✓ Standard: ซ้ำ function
SELECT department_id, AVG(salary) AS avg_salary
FROM employees
GROUP BY department_id
HAVING AVG(salary) > 60000;

-- MySQL, PostgreSQL รองรับ alias ใน HAVING แต่ standard ไม่รองรับ
```

---

## 7.5 Aliases ในสถานการณ์จริง

```sql
-- ============================================================
-- EXAMPLE 17: Report ที่ใช้ Aliases อย่างมืออาชีพ
-- ============================================================

-- Employee Salary Report
SELECT 
    e.employee_id                           AS "ID",
    e.first_name || ' ' || e.last_name     AS "Full Name",
    e.job_title                            AS "Position",
    d.department_name                      AS "Department",
    e.salary                               AS "Base Salary",
    e.salary * 12                          AS "Annual Salary",
    ROUND(e.salary * 12 * 0.05, 2)         AS "Provident Fund (5%)",
    e.salary * 12 - ROUND(e.salary * 12 * 0.05, 2) AS "Net Annual"
FROM employees e
JOIN departments d ON e.department_id = d.department_id
ORDER BY e.salary DESC;

-- ============================================================
-- EXAMPLE 18: Product Catalog ด้วย Aliases
-- ============================================================

SELECT 
    p.product_id                            AS "No.",
    p.product_name                         AS "Product",
    p.category                             AS "Category",
    p.unit_price                           AS "Price",
    ROUND(p.unit_price * 1.07, 2)          AS "Price incl. VAT",
    p.units_in_stock                       AS "Stock",
    p.unit_price * p.units_in_stock        AS "Stock Value",
    CASE 
        WHEN p.units_in_stock = 0  THEN 'Out of Stock'
        WHEN p.units_in_stock < 30 THEN 'Low Stock'
        ELSE 'In Stock'
    END                                    AS "Status"
FROM products p
WHERE p.discontinued = FALSE
ORDER BY p.category, p.product_name;

-- ============================================================
-- EXAMPLE 19: Summary Statistics ด้วย Aliases
-- ============================================================

SELECT 
    COUNT(*) AS total_employees,
    COUNT(manager_id) AS employees_with_manager,
    COUNT(*) - COUNT(manager_id) AS top_level_employees,
    ROUND(AVG(salary), 2) AS average_salary,
    MIN(salary) AS lowest_salary,
    MAX(salary) AS highest_salary,
    MAX(salary) - MIN(salary) AS salary_range,
    ROUND(MAX(salary) / MIN(salary), 2) AS max_to_min_ratio
FROM employees;

-- ผลลัพธ์:
-- total_employees | employees_with_manager | top_level_employees | average_salary | ...
-- ----------------|------------------------|---------------------|----------------|----
-- 15              | 8                      | 7                   | 64200.00       | ...

-- ============================================================
-- EXAMPLE 20: Business KPI Dashboard
-- ============================================================

SELECT 
    -- Order counts
    COUNT(*) AS total_orders,
    COUNT(CASE WHEN status = 'Delivered' THEN 1 END) AS delivered_orders,
    COUNT(CASE WHEN status = 'Cancelled' THEN 1 END) AS cancelled_orders,
    COUNT(CASE WHEN status IN ('Pending', 'Processing') THEN 1 END) AS active_orders,
    
    -- Revenue
    SUM(CASE WHEN status = 'Delivered' THEN total_amount ELSE 0 END) AS confirmed_revenue,
    SUM(CASE WHEN status <> 'Cancelled' THEN total_amount ELSE 0 END) AS potential_revenue,
    
    -- Averages
    ROUND(AVG(total_amount), 2) AS avg_order_value,
    MAX(total_amount) AS largest_order,
    MIN(total_amount) AS smallest_order
FROM orders;
```

---

## 7.6 Best Practices สำหรับ Aliases

```sql
-- ============================================================
-- EXAMPLE 21: Naming Convention
-- ============================================================

-- ✓ ดี: snake_case (lowercase + underscore)
SELECT salary * 12 AS annual_salary FROM employees;
SELECT first_name || ' ' || last_name AS full_name FROM employees;

-- ✓ ดี: lowercase words
SELECT COUNT(*) AS total_count FROM employees;

-- ✓ ดีสำหรับ reports: Title Case ใน quotes
SELECT salary AS "Monthly Salary" FROM employees;

-- ❌ ไม่ดี: ชื่อ alias ไม่สื่อความหมาย
SELECT salary * 12 AS x FROM employees;  -- x คืออะไร?
SELECT salary * 12 AS a1 FROM employees;

-- ❌ ไม่ดี: alias ซ้ำชื่อ column จริง
SELECT salary AS salary FROM employees;  -- ไม่มีประโยชน์

-- ============================================================
-- EXAMPLE 22: Consistent Style
-- ============================================================

-- ✓ สม่ำเสมอ: ใช้ AS ตลอด
SELECT 
    employee_id AS emp_id,
    first_name  AS first,
    last_name   AS last,
    salary      AS base_salary,
    salary * 12 AS annual_salary
FROM employees;

-- ❌ ไม่สม่ำเสมอ: บางทีมี AS บางทีไม่มี
SELECT 
    employee_id AS emp_id,
    first_name  first,     -- ไม่มี AS
    last_name   AS last,
    salary      base_salary
FROM employees;

-- ============================================================
-- EXAMPLE 23: Descriptive Aliases สำหรับ Complex Expressions
-- ============================================================

SELECT 
    product_name,
    unit_price,
    units_in_stock,
    
    -- ไม่มี alias - ไม่รู้ว่าคืออะไร
    unit_price * units_in_stock,
    
    -- มี alias - ชัดเจน
    unit_price * units_in_stock AS total_inventory_value,
    ROUND(unit_price * 1.07, 2) AS price_with_vat_7pct,
    ROUND(unit_price * 0.9, 2) AS price_with_10pct_discount
FROM products;
```

---

## 7.7 20+ ตัวอย่างครบถ้วน

```sql
-- 1. Basic alias
SELECT first_name AS name FROM employees;

-- 2. Multiple aliases
SELECT first_name AS fname, last_name AS lname, salary AS pay
FROM employees;

-- 3. Expression alias
SELECT product_name, unit_price * 1.07 AS price_with_vat FROM products;

-- 4. Concatenation alias
SELECT first_name || ' ' || last_name AS full_name FROM employees;

-- 5. Function alias
SELECT UPPER(product_name) AS product_upper FROM products;

-- 6. Quoted alias (with space)
SELECT salary AS "Monthly Salary" FROM employees;

-- 7. Table alias
SELECT e.first_name, e.salary FROM employees e;

-- 8. Table alias in expression
SELECT e.salary * 12 AS annual FROM employees e;

-- 9. Table alias in ORDER BY
SELECT e.first_name, e.salary AS pay FROM employees e ORDER BY pay DESC;

-- 10. Multiple tables with aliases
SELECT e.first_name, d.department_name
FROM employees e, departments d
WHERE e.department_id = d.department_id;

-- 11. Alias for aggregates
SELECT COUNT(*) AS total, AVG(salary) AS avg_sal, MAX(salary) AS max_sal
FROM employees;

-- 12. Alias in complex query
SELECT 
    dept_id,
    emp_count,
    avg_pay,
    ROUND(avg_pay * 12, 2) AS total_annual
FROM (
    SELECT department_id AS dept_id,
           COUNT(*) AS emp_count,
           AVG(salary) AS avg_pay
    FROM employees
    GROUP BY department_id
) dept_stats;

-- 13. CASE with alias
SELECT 
    first_name,
    salary,
    CASE WHEN salary > 70000 THEN 'High' ELSE 'Normal' END AS salary_level
FROM employees;

-- 14. Alias สำหรับ date expression
SELECT 
    order_id,
    order_date AS placed_on,
    shipped_date AS shipped_on
FROM orders;

-- 15. Boolean alias
SELECT product_name, discontinued AS is_discontinued FROM products;

-- 16. Math expression aliases
SELECT 
    product_name,
    unit_price AS original,
    ROUND(unit_price * 0.8, 2) AS after_20pct_off,
    ROUND(unit_price * 1.15, 2) AS with_15pct_markup
FROM products;

-- 17. COUNT DISTINCT alias
SELECT COUNT(DISTINCT category) AS unique_categories FROM products;

-- 18. NULL handling alias
SELECT 
    first_name,
    COALESCE(manager_id, 0) AS manager_or_zero
FROM employees;

-- 19. Alias กับ ROUND
SELECT 
    product_name,
    unit_price,
    ROUND(unit_price, -2) AS rounded_to_100
FROM products;

-- 20. Complex report
SELECT 
    e.employee_id AS "ID",
    e.first_name || ' ' || e.last_name AS "Name",
    e.job_title AS "Title",
    e.salary AS "Salary",
    d.department_name AS "Department"
FROM employees e
JOIN departments d ON e.department_id = d.department_id
ORDER BY "Salary" DESC;
```

---

## 7.8 สรุปบทที่ 7

```
✅ AS keyword ตั้งชื่อ column alias
✅ AS ละได้ แต่แนะนำให้ใส่เสมอ
✅ Double quotes สำหรับ alias ที่มี space
✅ Table aliases ทำให้ queries อ่านง่ายขึ้น
✅ Aliases ใช้ได้ใน ORDER BY (แต่ไม่ได้ใน WHERE)
✅ GROUP BY: ขึ้นกับ database
✅ HAVING: ใช้ expression ซ้ำ (safer)
✅ Best practices: ชัดเจน, สม่ำเสมอ, มีความหมาย
```

---

## 📝 แบบฝึกหัดท้ายบท

**คำถาม 1:** เขียน query แสดง `product_name` เป็น "Product Name", `unit_price` เป็น "Price (THB)", `unit_price * 1.07` เป็น "Price with VAT"

**คำถาม 2:** สร้าง employee report ด้วย aliases: "ID", "Full Name", "Position", "Monthly Pay", "Annual Pay"

**คำถาม 3:** ใช้ table alias `e` สำหรับ employees แล้วดู first_name, email, salary

**คำถาม 4:** สร้าง product summary ด้วย aliases ที่อ่านง่าย

**คำถาม 5:** เขียน query ที่ใช้ alias ใน ORDER BY (แสดงว่าทำได้)

**คำถาม 6:** เขียน query ที่แสดงให้เห็นว่า alias ใน WHERE ไม่ได้ แล้วแก้ไข

**คำถาม 7:** สร้าง KPI report ด้วย aliases ที่ชัดเจน สำหรับ products

**คำถาม 8:** ใช้ self join ด้วย table aliases เพื่อแสดง employee + manager name

**คำถาม 9:** สร้าง query ที่มี quoted aliases ที่มี spaces

**คำถาม 10:** สร้าง comprehensive employee directory ด้วย aliases ที่เป็น professional report format

---

## ✅ เฉลยแบบฝึกหัด

### เฉลยที่ 1:
```sql
SELECT 
    product_name AS "Product Name",
    unit_price AS "Price (THB)",
    ROUND(unit_price * 1.07, 2) AS "Price with VAT"
FROM products
ORDER BY "Price (THB)";
```

### เฉลยที่ 2:
```sql
SELECT 
    employee_id AS "ID",
    first_name || ' ' || last_name AS "Full Name",
    job_title AS "Position",
    salary AS "Monthly Pay",
    salary * 12 AS "Annual Pay"
FROM employees
ORDER BY salary DESC;
```

### เฉลยที่ 3:
```sql
SELECT e.first_name, e.email, e.salary
FROM employees AS e
ORDER BY e.salary DESC;
```

### เฉลยที่ 4:
```sql
SELECT 
    p.product_id AS id,
    p.product_name AS product,
    p.category AS cat,
    p.unit_price AS price,
    p.units_in_stock AS stock,
    p.unit_price * p.units_in_stock AS stock_value,
    CASE WHEN p.discontinued THEN 'Discontinued' ELSE 'Active' END AS status
FROM products p
ORDER BY cat, price;
```

### เฉลยที่ 5:
```sql
SELECT 
    product_name,
    unit_price * units_in_stock AS stock_value
FROM products
ORDER BY stock_value DESC;  -- ใช้ alias ใน ORDER BY ✓
```

### เฉลยที่ 6:
```sql
-- ❌ ผิด:
-- SELECT first_name, salary * 12 AS annual
-- FROM employees WHERE annual > 700000;

-- ✓ ถูก (วิธีที่ 1 - ซ้ำ expression):
SELECT first_name, salary * 12 AS annual
FROM employees 
WHERE salary * 12 > 700000;

-- ✓ ถูก (วิธีที่ 2 - subquery):
SELECT * FROM (
    SELECT first_name, salary * 12 AS annual FROM employees
) sub
WHERE sub.annual > 700000;
```

### เฉลยที่ 7:
```sql
SELECT 
    COUNT(*) AS total_products,
    COUNT(DISTINCT category) AS categories,
    ROUND(AVG(unit_price), 2) AS avg_price,
    MIN(unit_price) AS cheapest,
    MAX(unit_price) AS most_expensive,
    SUM(units_in_stock) AS total_stock,
    SUM(unit_price * units_in_stock) AS total_stock_value
FROM products
WHERE discontinued = FALSE;
```

### เฉลยที่ 8:
```sql
SELECT 
    e.first_name AS employee_name,
    e.job_title AS employee_title,
    m.first_name AS manager_name,
    m.job_title AS manager_title
FROM employees e
LEFT JOIN employees m ON e.manager_id = m.employee_id
ORDER BY e.first_name;
```

### เฉลยที่ 9:
```sql
SELECT 
    employee_id AS "Employee ID",
    first_name AS "First Name",
    last_name AS "Last Name",
    salary AS "Monthly Salary (THB)",
    salary * 12 AS "Annual Salary (THB)",
    hire_date AS "Date of Joining"
FROM employees
ORDER BY "Monthly Salary (THB)" DESC;
```

### เฉลยที่ 10:
```sql
SELECT 
    e.employee_id AS "Emp #",
    e.first_name || ' ' || e.last_name AS "Full Name",
    e.email AS "Email",
    e.phone AS "Phone",
    e.job_title AS "Title",
    d.department_name AS "Department",
    e.salary AS "Salary",
    CASE 
        WHEN e.salary >= 80000 THEN 'Executive'
        WHEN e.salary >= 60000 THEN 'Senior'
        WHEN e.salary >= 40000 THEN 'Staff'
        ELSE 'Junior'
    END AS "Grade",
    e.hire_date AS "Since"
FROM employees e
LEFT JOIN departments d ON e.department_id = d.department_id
WHERE e.is_active = TRUE
ORDER BY d.department_name, e.salary DESC;
```

---

## ➡️ บทถัดไป

**[Part 008: NULL Values - Understanding and Handling](part-008.md)**

ในบทถัดไปเราจะเรียนรู้เรื่อง NULL อย่างละเอียด ซึ่งเป็นเรื่องสำคัญและมักสร้างความสับสน!

---

*Part 007 of 120 | หลักสูตร SQL ครบวงจร*
