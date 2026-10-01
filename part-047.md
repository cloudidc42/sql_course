# Part 47: ALL and ANY / SOME Operators

## 47.1 ALL และ ANY คืออะไร?

`ALL` และ `ANY` (หรือ `SOME`) ใช้ร่วมกับ comparison operator เพื่อเปรียบเทียบค่ากับทุกค่า (ALL) หรืออย่างน้อยหนึ่งค่า (ANY) ที่ subquery คืนมา

```
Syntax:
value operator ALL (subquery)
value operator ANY (subquery)
value operator SOME (subquery)  -- SOME เหมือนกับ ANY

Operators ที่ใช้ได้: =, <>, >, <, >=, <=
```

---

## 47.2 ALL Operator

### ตัวอย่างที่ 1: > ALL พื้นฐาน

```sql
-- salary > ALL (list) หมายความว่า salary > ทุกค่าในรายการ
-- เหมือนกับ salary > MAX(list)

-- หาพนักงานที่มีเงินเดือนสูงกว่าทุกคนในแผนก Sales
SELECT first_name, last_name, salary
FROM   employees
WHERE  salary > ALL (
    SELECT salary
    FROM   employees
    WHERE  department_id = 1  -- Sales
);
```

### ตัวอย่างที่ 2: ALL เทียบกับ MAX

```sql
-- ALL เทียบเท่ากับ MAX:
-- salary > ALL (subquery)  =  salary > MAX(subquery)

-- แบบ ALL:
SELECT first_name, salary FROM employees
WHERE salary > ALL (SELECT salary FROM employees WHERE department_id = 1);

-- แบบ MAX (equivalent):
SELECT first_name, salary FROM employees
WHERE salary > (SELECT MAX(salary) FROM employees WHERE department_id = 1);
```

### ตัวอย่างที่ 3: < ALL (น้อยกว่าทุกค่า)

```sql
-- สินค้าที่มีราคาต่ำกว่าสินค้าทุกชิ้นใน Electronics
SELECT product_name, category, price
FROM   products
WHERE  price < ALL (
    SELECT price FROM products WHERE category = 'Electronics'
)
ORDER BY price;
-- เทียบเท่า: price < MIN(price ของ Electronics)
```

### ตัวอย่างที่ 4: >= ALL (สูงกว่าหรือเท่ากับทุกค่า = MAX)

```sql
-- ราคาสูงสุดในทุกหมวดหมู่
SELECT product_name, category, price
FROM   products p
WHERE  price >= ALL (
    SELECT price FROM products WHERE category = p.category
);
-- เทียบเท่า: หาสินค้าที่มีราคาสูงสุดในแต่ละ category
```

### ตัวอย่างที่ 5: ALL กับ NULL

```sql
-- ⚠️ ถ้า subquery คืน NULL, ALL จะทำงานผิดพลาด
-- เหมือนกับ NOT IN + NULL trap

-- ปัญหา:
SELECT first_name FROM employees
WHERE salary > ALL (
    SELECT salary FROM employees WHERE department_id = 999
    -- ถ้าไม่มีข้อมูล → empty set → ALL = TRUE (special case)
);
-- ถ้า list ว่างเปล่า → TRUE เสมอ!
-- ถ้า list มี NULL → UNKNOWN

-- ตรวจสอบก่อนใช้:
SELECT first_name FROM employees
WHERE salary > ALL (
    SELECT salary FROM employees
    WHERE department_id = 1 AND salary IS NOT NULL
);
```

---

## 47.3 ANY / SOME Operator

### ตัวอย่างที่ 6: > ANY พื้นฐาน

```sql
-- salary > ANY (list) หมายความว่า salary > อย่างน้อย 1 ค่าในรายการ
-- เหมือนกับ salary > MIN(list)

SELECT first_name, last_name, salary
FROM   employees
WHERE  salary > ANY (
    SELECT salary FROM employees WHERE department_id = 3  -- Marketing
);
-- เทียบเท่า: salary > MIN(salary ของ Marketing)
```

### ตัวอย่างที่ 7: ANY เทียบกับ MIN

```sql
-- ANY เทียบเท่ากับ MIN:
-- salary > ANY (subquery)  =  salary > MIN(subquery)

-- แบบ ANY:
SELECT first_name FROM employees
WHERE salary > ANY (SELECT salary FROM employees WHERE department_id = 3);

-- แบบ MIN (equivalent):
SELECT first_name FROM employees
WHERE salary > (SELECT MIN(salary) FROM employees WHERE department_id = 3);
```

### ตัวอย่างที่ 8: = ANY (เหมือนกับ IN)

```sql
-- = ANY เหมือนกับ IN
SELECT product_name, category
FROM   products
WHERE  category = ANY (
    SELECT DISTINCT category FROM products WHERE price > 5000
);
-- เทียบเท่า:
SELECT product_name, category FROM products
WHERE category IN (SELECT DISTINCT category FROM products WHERE price > 5000);
```

### ตัวอย่างที่ 9: <> ANY (ต่างจากอย่างน้อยหนึ่ง)

```sql
-- <> ANY หมายความว่าต่างจากอย่างน้อยหนึ่งค่า (เกือบทุกแถวจะผ่าน!)
SELECT first_name, salary
FROM   employees
WHERE  salary <> ANY (
    SELECT salary FROM employees WHERE department_id = 1
);
-- เกือบทุกแถวจะผ่าน เพราะมีแถวใน dept 1 ที่ salary ต่างออกไปเสมอ
-- ใช้ไม่ค่อยมีประโยชน์ในทางปฏิบัติ
```

### ตัวอย่างที่ 10: SOME (เหมือน ANY)

```sql
-- SOME เหมือนกับ ANY ทุกอย่าง
SELECT product_name, price
FROM   products
WHERE  price > SOME (
    SELECT price FROM products WHERE category = 'Stationery'
);
-- เทียบเท่า:
SELECT product_name, price FROM products
WHERE price > ANY (SELECT price FROM products WHERE category = 'Stationery');
```

---

## 47.4 ALL vs MAX/MIN

### ตัวอย่างที่ 11: ความเหมือนและต่าง

```sql
-- ทั้งหมดนี้เหมือนกัน:

-- 1. > ALL
SELECT * FROM employees
WHERE salary > ALL (SELECT salary FROM employees WHERE department_id = 2);

-- 2. > MAX
SELECT * FROM employees
WHERE salary > (SELECT MAX(salary) FROM employees WHERE department_id = 2);

-- แนะนำ: ใช้ MAX/MIN เพราะ:
-- 1. อ่านเข้าใจง่ายกว่า
-- 2. MySQL optimize ได้ดีกว่า
-- 3. ปลอดภัยกว่าเรื่อง NULL
```

### ตัวอย่างที่ 12: กรณีที่ ALL empty set

```sql
-- กรณีพิเศษ: ถ้า subquery ไม่คืนแถว
-- ALL กับ empty set = TRUE (vacuously true)

SELECT first_name FROM employees
WHERE salary > ALL (
    SELECT salary FROM employees WHERE department_id = 999  -- ไม่มีแผนกนี้
);
-- ผลลัพธ์: ทุกแถว! (เพราะ > ALL empty set = TRUE)

-- MAX กับ empty set = NULL
SELECT first_name FROM employees
WHERE salary > (SELECT MAX(salary) FROM employees WHERE department_id = 999);
-- ผลลัพธ์: ไม่มีแถว (NULL > NULL = UNKNOWN = FALSE)
-- ⚠️ ผลต่างกัน!
```

---

## 47.5 ANY vs EXISTS

### ตัวอย่างที่ 13: = ANY เหมือน EXISTS

```sql
-- = ANY เทียบเท่า EXISTS สำหรับการตรวจสอบ membership

-- แบบ = ANY:
SELECT * FROM customers
WHERE customer_id = ANY (SELECT customer_id FROM orders);

-- แบบ EXISTS:
SELECT * FROM customers c
WHERE EXISTS (SELECT 1 FROM orders WHERE customer_id = c.customer_id);

-- แบบ IN:
SELECT * FROM customers
WHERE customer_id IN (SELECT customer_id FROM orders);
```

### ตัวอย่างที่ 14: ความต่างเรื่อง NULL

```sql
-- ANY: มีปัญหากับ NULL คล้าย IN
-- EXISTS: ปลอดภัยกว่า

-- ⚠️ = ANY กับ NULL:
SELECT 1 WHERE 5 = ANY (SELECT NULL);
-- ผลลัพธ์: NULL (ไม่คืนแถว)

-- ✓ EXISTS กับ NULL:
SELECT 1 WHERE EXISTS (SELECT NULL);
-- ผลลัพธ์: 1 (คืนแถว เพราะ subquery คืน 1 แถว)
```

---

## 47.6 Practical Use Cases

### ตัวอย่างที่ 15: พนักงานที่ได้เงินเดือนสูงกว่าทุกคนใน HR

```sql
SELECT first_name, last_name, salary, department_id
FROM   employees
WHERE  salary > ALL (
    SELECT e.salary
    FROM   employees e
    JOIN   departments d ON d.department_id = e.department_id
    WHERE  d.department_name = 'HR'
)
ORDER BY salary;
```

### ตัวอย่างที่ 16: สินค้าที่แพงกว่าสินค้าทุกชิ้นใน Stationery

```sql
SELECT product_name, category, price
FROM   products
WHERE  price > ALL (
    SELECT price FROM products WHERE category = 'Stationery'
)
ORDER BY price;
```

### ตัวอย่างที่ 17: ออเดอร์ที่ต่ำกว่าออเดอร์ทุกออเดอร์ของลูกค้า 1

```sql
SELECT order_id, customer_id, total_amount
FROM   orders
WHERE  customer_id <> 1
  AND  total_amount < ALL (
    SELECT total_amount FROM orders WHERE customer_id = 1
  );
```

### ตัวอย่างที่ 18: ราคาสินค้าที่สูงกว่าราคาอย่างน้อยหนึ่งชิ้นใน Furniture

```sql
SELECT product_name, category, price
FROM   products
WHERE  price > ANY (
    SELECT price FROM products WHERE category = 'Furniture'
)
  AND  category <> 'Furniture'
ORDER BY price;
```

### ตัวอย่างที่ 19: แผนกที่มีเงินเดือนเฉลี่ยสูงกว่าทุกแผนกที่มีงบ < 4,000,000

```sql
SELECT d.department_name, AVG(e.salary) AS avg_salary
FROM   employees e
JOIN   departments d ON d.department_id = e.department_id
GROUP  BY d.department_id, d.department_name
HAVING AVG(e.salary) > ALL (
    SELECT AVG(e2.salary)
    FROM   employees e2
    JOIN   departments d2 ON d2.department_id = e2.department_id
    WHERE  d2.budget < 4000000
    GROUP  BY e2.department_id
);
```

---

## 47.7 Cross-database Compatibility

### ตัวอย่างที่ 20: MySQL Support

```sql
-- MySQL รองรับ ALL, ANY, SOME
-- ตัวอย่าง MySQL:
SELECT product_name FROM products
WHERE price > ANY (SELECT price FROM products WHERE category = 'Electronics');
```

```sql
-- PostgreSQL รองรับเหมือนกัน:
SELECT product_name FROM products
WHERE price > ANY (SELECT price FROM products WHERE category = 'Electronics');
```

```sql
-- SQL Server:
SELECT product_name FROM products
WHERE price > ANY (SELECT price FROM products WHERE category = 'Electronics');
-- SQL Server ใช้ SOME แทน ANY ได้ด้วย
```

### ตัวอย่างที่ 21: แนะนำ: ใช้ MAX/MIN แทน ALL/ANY

```sql
-- ทำงานได้ทุก Database และเข้าใจง่ายกว่า:

-- แทน > ALL: ใช้ > MAX
-- แทน < ALL: ใช้ < MIN
-- แทน > ANY: ใช้ > MIN
-- แทน < ANY: ใช้ < MAX
-- แทน = ANY: ใช้ IN

-- ตัวอย่าง:
-- แบบ ALL:
SELECT * FROM employees WHERE salary > ALL (SELECT salary FROM employees WHERE department_id = 1);
-- แบบ MAX (แนะนำ):
SELECT * FROM employees WHERE salary > (SELECT MAX(salary) FROM employees WHERE department_id = 1);
```

---

## 47.8 สรุปตาราง

```
┌─────────────────────────────────────────────────────────────────┐
│ Operator    │ เทียบเท่า │ ตัวอย่าง                            │
├─────────────┼───────────┼─────────────────────────────────────┤
│ > ALL       │ > MAX()   │ salary > ALL (dept salaries)        │
│ < ALL       │ < MIN()   │ price < ALL (category prices)       │
│ >= ALL      │ >= MAX()  │ price >= ALL (= MAX in category)    │
│ <= ALL      │ <= MIN()  │ salary <= ALL (= MIN in dept)       │
│ = ALL       │ (rare)    │ ทุกค่าต้องเท่ากัน (ใช้น้อย)       │
│ <> ALL      │ NOT IN    │ salary <> ALL = NOT IN              │
│─────────────┼───────────┼─────────────────────────────────────│
│ > ANY       │ > MIN()   │ salary > ANY (dept salaries)        │
│ < ANY       │ < MAX()   │ price < ANY (category prices)       │
│ = ANY       │ IN        │ category = ANY (subquery)           │
│ <> ANY      │ (tricky)  │ ต่างจากอย่างน้อย 1 ค่า            │
└─────────────┴───────────┴─────────────────────────────────────┘
```

---

## แบบฝึกหัดบทที่ 47

**ข้อ 1:** ใช้ > ALL หาพนักงานที่มีเงินเดือนสูงกว่าทุกคนในแผนก Marketing

```sql
-- เฉลย:
SELECT first_name, last_name, salary, department_id
FROM   employees
WHERE  salary > ALL (
    SELECT salary
    FROM   employees e
    JOIN   departments d ON d.department_id = e.department_id
    WHERE  d.department_name = 'Marketing'
);
-- เทียบเท่า:
SELECT first_name, last_name, salary FROM employees
WHERE salary > (SELECT MAX(e.salary) FROM employees e
                JOIN departments d ON d.department_id = e.department_id
                WHERE d.department_name = 'Marketing');
```

**ข้อ 2:** ใช้ > ANY หาสินค้าที่ราคาสูงกว่าสินค้าอย่างน้อยหนึ่งชิ้นใน Stationery

```sql
-- เฉลย:
SELECT product_name, category, price
FROM   products
WHERE  price > ANY (
    SELECT price FROM products WHERE category = 'Stationery'
)
  AND  category <> 'Stationery'
ORDER BY price;
```

**ข้อ 3:** อธิบายความต่างระหว่าง > ALL กับ > MAX

```sql
-- เฉลย:
-- ทำงานเหมือนกันเมื่อ subquery ไม่ว่างและไม่มี NULL
-- ต่างกันเมื่อ subquery ว่างเปล่า:
-- > ALL empty set = TRUE (vacuously true)
-- > MAX empty set = > NULL = UNKNOWN (FALSE)

-- ตัวอย่าง:
SELECT 'ALL' AS test, COUNT(*) AS rows FROM employees
WHERE salary > ALL (SELECT salary FROM employees WHERE department_id = 999);
-- ผล: นับทุกพนักงาน (TRUE)

SELECT 'MAX' AS test, COUNT(*) AS rows FROM employees
WHERE salary > (SELECT MAX(salary) FROM employees WHERE department_id = 999);
-- ผล: 0 (NULL comparison)
```

**ข้อ 4:** ใช้ = ANY หาลูกค้าที่อยู่ใน city เดียวกับลูกค้าที่ใช้จ่ายมากกว่า 30,000

```sql
-- เฉลย:
SELECT first_name, last_name, city
FROM   customers
WHERE  city = ANY (
    SELECT DISTINCT c2.city
    FROM   customers c2
    JOIN   orders o ON o.customer_id = c2.customer_id
    GROUP  BY c2.customer_id, c2.city
    HAVING SUM(o.total_amount) > 30000
)
ORDER BY city, first_name;
```

**ข้อ 5:** ใช้ < ALL หาออเดอร์ที่มียอดต่ำกว่าทุกออเดอร์ที่ทำในเดือนมกราคม

```sql
-- เฉลย:
SELECT order_id, customer_id, order_date, total_amount
FROM   orders
WHERE  total_amount < ALL (
    SELECT total_amount
    FROM   orders
    WHERE  YEAR(order_date) = 2024
      AND  MONTH(order_date) = 1
)
ORDER BY total_amount;
```

**ข้อ 6:** เปรียบเทียบ = ANY กับ IN สำหรับลูกค้าที่มีออเดอร์

```sql
-- เฉลย:
-- แบบ = ANY:
SELECT customer_id, first_name FROM customers
WHERE customer_id = ANY (SELECT DISTINCT customer_id FROM orders);

-- แบบ IN:
SELECT customer_id, first_name FROM customers
WHERE customer_id IN (SELECT DISTINCT customer_id FROM orders);

-- ทั้งคู่ให้ผลเหมือนกัน
-- IN อ่านง่ายและ optimize ได้ดีกว่าใน MySQL
```

**ข้อ 7:** ใช้ ALL หาสินค้าที่มีราคาสูงสุดในแต่ละ category

```sql
-- เฉลย:
SELECT p1.product_name, p1.category, p1.price
FROM   products p1
WHERE  p1.price >= ALL (
    SELECT p2.price
    FROM   products p2
    WHERE  p2.category = p1.category
)
ORDER BY p1.category;
```

**ข้อ 8:** ใช้ > ANY หาแผนกที่มีเงินเดือนเฉลี่ยสูงกว่าเงินเดือนเฉลี่ยของแผนกอื่นอย่างน้อยหนึ่งแผนก

```sql
-- เฉลย:
SELECT d.department_name, AVG(e.salary) AS avg_salary
FROM   employees e
JOIN   departments d ON d.department_id = e.department_id
GROUP  BY d.department_id, d.department_name
HAVING AVG(e.salary) > ANY (
    SELECT AVG(e2.salary)
    FROM   employees e2
    WHERE  e2.department_id <> e.department_id
    GROUP  BY e2.department_id
);
```

**ข้อ 9:** อธิบายว่า SOME กับ ANY ต่างกันอย่างไร พร้อมตัวอย่าง

```sql
-- เฉลย:
-- SOME และ ANY ทำงานเหมือนกันทุกประการ
-- SOME เป็น alias ของ ANY ใน SQL standard
-- MySQL, PostgreSQL, SQL Server รองรับทั้งคู่

-- ตัวอย่าง SOME:
SELECT product_name FROM products
WHERE price > SOME (SELECT price FROM products WHERE category = 'Stationery');

-- ตัวอย่าง ANY (เหมือนกัน):
SELECT product_name FROM products
WHERE price > ANY (SELECT price FROM products WHERE category = 'Stationery');

-- ผลลัพธ์เหมือนกัน 100%
-- ปฏิบัติ: ใช้ ANY มากกว่า SOME เพราะคุ้นเคยกว่า
```

**ข้อ 10:** เขียน query เปรียบเทียบ: > ALL, > ANY, > MAX, > MIN สำหรับข้อมูลเดียวกัน

```sql
-- เฉลย:
-- ข้อมูลเปรียบเทียบ: เงินเดือนพนักงาน dept 1 = {85000, 72000, 82000}

-- 1. > ALL → > 85000 (MAX)
SELECT COUNT(*) AS count_gt_all FROM employees
WHERE salary > ALL (SELECT salary FROM employees WHERE department_id = 1);

-- 2. > MAX → > 85000 (เหมือน > ALL)
SELECT COUNT(*) AS count_gt_max FROM employees
WHERE salary > (SELECT MAX(salary) FROM employees WHERE department_id = 1);

-- 3. > ANY → > 72000 (MIN)
SELECT COUNT(*) AS count_gt_any FROM employees
WHERE salary > ANY (SELECT salary FROM employees WHERE department_id = 1);

-- 4. > MIN → > 72000 (เหมือน > ANY)
SELECT COUNT(*) AS count_gt_min FROM employees
WHERE salary > (SELECT MIN(salary) FROM employees WHERE department_id = 1);

-- ผล: count_gt_all <= count_gt_max <= count_gt_min <= count_gt_any
```

---

*จบบทที่ 47: ALL and ANY / SOME Operators*
*บทถัดไป: Part 48 - Subqueries in UPDATE, DELETE, and INSERT*
