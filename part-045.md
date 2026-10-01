# Part 45: Correlated Subqueries (Correlated Subqueries)

## 45.1 Correlated Subquery คืออะไร?

**Correlated Subquery** คือ subquery ที่อ้างอิงคอลัมน์จาก outer query ทำให้ต้องรันใหม่ทุกครั้งสำหรับแต่ละแถวของ outer query

```
Uncorrelated Subquery (รันครั้งเดียว):
SELECT * FROM employees
WHERE salary > (SELECT AVG(salary) FROM employees);
               ↑ ไม่ขึ้นอยู่กับแถวใน outer query

Correlated Subquery (รันทุกแถว):
SELECT * FROM employees e1
WHERE salary > (SELECT AVG(salary) FROM employees e2
                WHERE e2.department_id = e1.department_id);
                                         ↑ อ้างอิง e1 จาก outer query!
```

### วิธีการทำงาน (Row by Row)

```
1. Outer query ดึงแถวแรกของ employees: e1 = {emp_id=1, dept=1, salary=85000}
2. Subquery รัน: SELECT AVG(salary) FROM employees WHERE department_id = 1
   → ผลลัพธ์: 79666.67
3. เปรียบเทียบ: 85000 > 79666.67 → TRUE → ดึงแถวนี้
4. ทำซ้ำสำหรับทุกแถวใน outer query
```

---

## 45.2 Correlated Subquery ใน WHERE

### ตัวอย่างที่ 1: พื้นฐาน Correlated

```sql
-- พนักงานที่มีเงินเดือนสูงกว่าค่าเฉลี่ยของแผนกตัวเอง
SELECT first_name, last_name, salary, department_id
FROM   employees e1
WHERE  salary > (
    SELECT AVG(salary)
    FROM   employees e2
    WHERE  e2.department_id = e1.department_id  -- ← correlated!
)
ORDER BY department_id, salary DESC;
```

### ตัวอย่างที่ 2: ต่างแถวต่างผล

```sql
-- เงินเดือนสูงสุดในแผนก (correlated)
SELECT first_name, last_name, salary, department_id
FROM   employees e1
WHERE  salary = (
    SELECT MAX(salary)
    FROM   employees e2
    WHERE  e2.department_id = e1.department_id
);
-- แต่ละแถวใน outer query จะเปรียบเทียบกับ MAX ของแผนกตัวเอง
```

### ตัวอย่างที่ 3: Correlated กับ date

```sql
-- ออเดอร์แรกของแต่ละลูกค้า
SELECT order_id, customer_id, order_date, total_amount
FROM   orders o1
WHERE  order_date = (
    SELECT MIN(order_date)
    FROM   orders o2
    WHERE  o2.customer_id = o1.customer_id
)
ORDER BY customer_id;
```

### ตัวอย่างที่ 4: ออเดอร์ล่าสุดของแต่ละลูกค้า

```sql
-- ออเดอร์ล่าสุดของแต่ละลูกค้า
SELECT order_id, customer_id, order_date, total_amount, status
FROM   orders o1
WHERE  order_date = (
    SELECT MAX(order_date)
    FROM   orders o2
    WHERE  o2.customer_id = o1.customer_id
)
ORDER BY customer_id;
```

### ตัวอย่างที่ 5: Correlated กับ COUNT

```sql
-- ลูกค้าที่มีออเดอร์มากกว่า 1 ออเดอร์
SELECT first_name, last_name
FROM   customers c
WHERE  (
    SELECT COUNT(*)
    FROM   orders o
    WHERE  o.customer_id = c.customer_id
) > 1;
```

### ตัวอย่างที่ 6: Correlated กับ SUM

```sql
-- สินค้าที่ขายได้มากกว่าค่าเฉลี่ยของ category เดียวกัน
SELECT product_name, category, price
FROM   products p
WHERE  (
    SELECT COALESCE(SUM(oi.quantity), 0)
    FROM   order_items oi
    WHERE  oi.product_id = p.product_id
) > (
    SELECT AVG(cat_qty)
    FROM (
        SELECT p2.product_id,
               COALESCE(SUM(oi2.quantity), 0) AS cat_qty
        FROM   products p2
        LEFT   JOIN order_items oi2 ON oi2.product_id = p2.product_id
        WHERE  p2.category = p.category  -- ← correlated!
        GROUP  BY p2.product_id
    ) AS cat_totals
);
```

---

## 45.3 Correlated Subquery ใน SELECT

### ตัวอย่างที่ 7: Computed Column ที่ขึ้นกับแถว

```sql
-- แสดงข้อมูลพนักงานพร้อมค่าเฉลี่ยเงินเดือนของแผนกตัวเอง
SELECT
    e.first_name,
    e.last_name,
    e.salary,
    e.department_id,
    (SELECT ROUND(AVG(salary), 2)
     FROM   employees
     WHERE  department_id = e.department_id) AS dept_avg_salary,
    (SELECT COUNT(*)
     FROM   employees
     WHERE  department_id = e.department_id) AS dept_size
FROM employees e;
```

### ตัวอย่างที่ 8: Rank ใน SELECT

```sql
-- Rank เงินเดือนในแผนก (โดยไม่ใช้ RANK() window function)
SELECT
    first_name,
    last_name,
    salary,
    department_id,
    (SELECT COUNT(*) + 1
     FROM   employees e2
     WHERE  e2.department_id = e1.department_id
       AND  e2.salary > e1.salary) AS dept_salary_rank
FROM employees e1
ORDER BY department_id, dept_salary_rank;
```

### ตัวอย่างที่ 9: Previous/Next Value

```sql
-- ออเดอร์แต่ละออเดอร์พร้อม order ก่อนหน้าของลูกค้าคนเดียวกัน
SELECT
    o1.order_id,
    o1.customer_id,
    o1.order_date,
    o1.total_amount,
    (SELECT o2.total_amount
     FROM   orders o2
     WHERE  o2.customer_id = o1.customer_id
       AND  o2.order_date < o1.order_date
     ORDER  BY o2.order_date DESC
     LIMIT  1) AS prev_order_amount
FROM orders o1
ORDER BY o1.customer_id, o1.order_date;
```

### ตัวอย่างที่ 10: Cumulative Count

```sql
-- จำนวนออเดอร์สะสมของแต่ละลูกค้า ณ วันที่ทำออเดอร์นั้น
SELECT
    o1.order_id,
    o1.customer_id,
    o1.order_date,
    o1.total_amount,
    (SELECT COUNT(*)
     FROM   orders o2
     WHERE  o2.customer_id = o1.customer_id
       AND  o2.order_date <= o1.order_date) AS cumulative_orders
FROM orders o1
ORDER BY o1.customer_id, o1.order_date;
```

---

## 45.4 Correlated Subquery ใน HAVING

### ตัวอย่างที่ 11: Group ที่ดีกว่าค่าเฉลี่ยของกลุ่มเดียวกัน

```sql
-- หมวดหมู่สินค้าที่มีราคาเฉลี่ยสูงกว่าค่าเฉลี่ยของทุก category รวม
SELECT category, AVG(price) AS avg_price
FROM   products p_outer
GROUP  BY category
HAVING AVG(price) > (
    SELECT AVG(price)
    FROM   products
);
```

### ตัวอย่างที่ 12: แผนกที่มีเงินเดือนรวมสูงกว่าค่าเฉลี่ย

```sql
-- แผนกที่จ่ายเงินเดือนรวมสูงกว่าค่าเฉลี่ยเงินเดือนรวมต่อแผนก
SELECT department_id, SUM(salary) AS total_salary
FROM   employees e
GROUP  BY department_id
HAVING SUM(salary) > (
    SELECT AVG(dept_total)
    FROM (
        SELECT department_id, SUM(salary) AS dept_total
        FROM   employees
        GROUP  BY department_id
    ) AS dept_totals
);
```

---

## 45.5 การ Convert Correlated Subquery เป็น JOIN

### ตัวอย่างที่ 13: Correlated → JOIN

```sql
-- แบบ Correlated (ช้ากว่า)
SELECT first_name, last_name, salary, department_id
FROM   employees e1
WHERE  salary = (
    SELECT MAX(salary)
    FROM   employees e2
    WHERE  e2.department_id = e1.department_id
);

-- แบบ JOIN (เร็วกว่า)
SELECT e.first_name, e.last_name, e.salary, e.department_id
FROM   employees e
JOIN (
    SELECT department_id, MAX(salary) AS max_salary
    FROM   employees
    GROUP  BY department_id
) AS dept_max ON dept_max.department_id = e.department_id
             AND dept_max.max_salary = e.salary;
```

### ตัวอย่างที่ 14: Correlated COUNT → JOIN COUNT

```sql
-- แบบ Correlated (ช้า):
SELECT c.first_name, c.last_name
FROM   customers c
WHERE  (SELECT COUNT(*) FROM orders WHERE customer_id = c.customer_id) > 1;

-- แบบ JOIN (เร็ว):
SELECT c.first_name, c.last_name
FROM   customers c
JOIN (
    SELECT customer_id, COUNT(*) AS order_count
    FROM   orders
    GROUP  BY customer_id
    HAVING COUNT(*) > 1
) AS multi_orders ON multi_orders.customer_id = c.customer_id;
```

### ตัวอย่างที่ 15: Correlated SUM → JOIN SUM

```sql
-- แบบ Correlated:
SELECT customer_id, first_name
FROM   customers c
WHERE  (
    SELECT SUM(total_amount)
    FROM   orders
    WHERE  customer_id = c.customer_id
) > 20000;

-- แบบ JOIN (เร็วกว่า):
SELECT c.customer_id, c.first_name
FROM   customers c
JOIN (
    SELECT customer_id, SUM(total_amount) AS total_spent
    FROM   orders
    GROUP  BY customer_id
    HAVING SUM(total_amount) > 20000
) AS big_spenders ON big_spenders.customer_id = c.customer_id;
```

---

## 45.6 Performance ของ Correlated Subquery

### ตัวอย่างที่ 16: EXPLAIN เพื่อดู Execution

```sql
-- Correlated subquery
EXPLAIN SELECT first_name, last_name
FROM   customers c
WHERE  (SELECT COUNT(*) FROM orders WHERE customer_id = c.customer_id) > 1;
-- จะเห็น "DEPENDENT SUBQUERY" ใน EXPLAIN

-- JOIN alternative
EXPLAIN SELECT c.first_name, c.last_name
FROM   customers c
JOIN (
    SELECT customer_id FROM orders GROUP BY customer_id HAVING COUNT(*) > 1
) AS multi ON multi.customer_id = c.customer_id;
-- จะเห็นไม่มี DEPENDENT SUBQUERY
```

### ตัวอย่างที่ 17: DEPENDENT SUBQUERY คืออะไร?

```sql
-- MySQL's EXPLAIN จะแสดง:
-- select_type = DEPENDENT SUBQUERY เมื่อ subquery ขึ้นอยู่กับ outer query

EXPLAIN
SELECT e.*,
       (SELECT COUNT(*) FROM employees e2 WHERE e2.department_id = e.department_id) AS dept_size
FROM employees e;

-- ถ้า dept_size ถูกเรียกใช้บ่อย ควรเปลี่ยนเป็น JOIN:
EXPLAIN
SELECT e.*, dept_counts.emp_count AS dept_size
FROM employees e
JOIN (
    SELECT department_id, COUNT(*) AS emp_count
    FROM   employees
    GROUP  BY department_id
) AS dept_counts ON dept_counts.department_id = e.department_id;
```

### ตัวอย่างที่ 18: เมื่อ Correlated ยังเหมาะอยู่

```sql
-- Correlated เหมาะสำหรับ EXISTS/NOT EXISTS
-- เหมาะสำหรับ queries ที่ต้องการ row-by-row comparison
-- เหมาะสำหรับ queries ที่ logic ซับซ้อนและต้องการอ่านง่าย

-- ตัวอย่าง: หา First Order ของแต่ละลูกค้า
SELECT order_id, customer_id, order_date, total_amount
FROM   orders o1
WHERE  NOT EXISTS (
    SELECT 1
    FROM   orders o2
    WHERE  o2.customer_id = o1.customer_id
      AND  o2.order_date  < o1.order_date
);
-- อ่านง่ายกว่า JOIN version
```

---

## 45.7 ตัวอย่าง Correlated เพิ่มเติม

### ตัวอย่างที่ 19: Correlated กับ Multiple Conditions

```sql
-- พนักงานที่มีเงินเดือนอยู่ใน top 50% ของแผนก
SELECT first_name, last_name, salary, department_id
FROM   employees e1
WHERE  salary >= (
    SELECT PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY salary)
    FROM   employees e2
    WHERE  e2.department_id = e1.department_id
);
-- MySQL ไม่มี PERCENTILE_CONT, ใช้ NTILE แทน:
SELECT first_name, last_name, salary, department_id
FROM   employees e1
WHERE  salary >= (
    SELECT MIN(salary)
    FROM (
        SELECT salary,
               NTILE(2) OVER (
                   PARTITION BY department_id ORDER BY salary
               ) AS half
        FROM employees
        WHERE department_id = e1.department_id
    ) AS halves
    WHERE half = 2
);
```

### ตัวอย่างที่ 20: Correlated กับ String

```sql
-- ลูกค้าที่มีออเดอร์ล่าสุด status = 'completed'
SELECT first_name, last_name
FROM   customers c
WHERE  (
    SELECT status
    FROM   orders o
    WHERE  o.customer_id = c.customer_id
    ORDER  BY order_date DESC
    LIMIT  1
) = 'completed';
```

### ตัวอย่างที่ 21: Correlated ข้ามหลายตาราง

```sql
-- สินค้าที่มียอดขายใน Q1 2024 สูงกว่าค่าเฉลี่ยยอดขาย Q1 ของ category เดียวกัน
SELECT p.product_name, p.category,
       (SELECT SUM(oi.quantity)
        FROM   order_items oi
        JOIN   orders o ON o.order_id = oi.order_id
        WHERE  oi.product_id = p.product_id
          AND  o.order_date BETWEEN '2024-01-01' AND '2024-03-31') AS q1_qty
FROM products p
WHERE (
    SELECT SUM(oi.quantity)
    FROM   order_items oi
    JOIN   orders o ON o.order_id = oi.order_id
    WHERE  oi.product_id = p.product_id
      AND  o.order_date BETWEEN '2024-01-01' AND '2024-03-31'
) > (
    SELECT AVG(cat_qty)
    FROM (
        SELECT p2.product_id, COALESCE(SUM(oi2.quantity), 0) AS cat_qty
        FROM   products p2
        LEFT   JOIN order_items oi2 ON oi2.product_id = p2.product_id
        LEFT   JOIN orders o2 ON o2.order_id = oi2.order_id
                               AND o2.order_date BETWEEN '2024-01-01' AND '2024-03-31'
        WHERE  p2.category = p.category
        GROUP  BY p2.product_id
    ) AS cat_sales
);
```

### ตัวอย่างที่ 22: N-th Row ด้วย Correlated

```sql
-- ออเดอร์ที่ 2 ของแต่ละลูกค้า (ลำดับที่ 2 ตาม order_date)
SELECT order_id, customer_id, order_date, total_amount
FROM   orders o1
WHERE  (
    SELECT COUNT(*)
    FROM   orders o2
    WHERE  o2.customer_id = o1.customer_id
      AND  o2.order_date <= o1.order_date
) = 2  -- ลำดับที่ 2
ORDER BY customer_id;
```

### ตัวอย่างที่ 23: Correlated กับ NULL check

```sql
-- พนักงานที่มีเงินเดือนสูงกว่าพนักงานทุกคนในแผนกที่เล็กกว่า
SELECT e1.first_name, e1.last_name, e1.salary, e1.department_id
FROM   employees e1
WHERE  e1.salary > ALL (
    SELECT e2.salary
    FROM   employees e2
    WHERE  e2.department_id <> e1.department_id
      AND  (SELECT COUNT(*) FROM employees e3
            WHERE e3.department_id = e2.department_id)
          < (SELECT COUNT(*) FROM employees e4
             WHERE e4.department_id = e1.department_id)
);
```

---

## 45.8 เปรียบเทียบ Correlated vs Non-correlated

### ตัวอย่างที่ 24: Non-correlated (รันครั้งเดียว)

```sql
-- Non-correlated: subquery รันครั้งเดียว
EXPLAIN SELECT * FROM employees
WHERE salary > (SELECT AVG(salary) FROM employees);
-- select_type = SUBQUERY (รันครั้งเดียว)
```

### ตัวอย่างที่ 25: Correlated (รันทุกแถว)

```sql
-- Correlated: subquery รันทุกแถว
EXPLAIN SELECT * FROM employees e1
WHERE salary > (SELECT AVG(salary) FROM employees e2
                WHERE e2.department_id = e1.department_id);
-- select_type = DEPENDENT SUBQUERY (รันทุกแถว)
```

---

## 45.9 Best Practices

### ตัวอย่างที่ 26: ใช้ index ช่วย correlated subquery

```sql
-- เพิ่ม index บน column ที่ใช้ใน correlated subquery
CREATE INDEX idx_emp_dept ON employees(department_id);
-- ทำให้ correlated subquery เร็วขึ้น

-- Correlated query ที่ใช้ indexed column
SELECT first_name, last_name, salary
FROM   employees e1
WHERE  salary > (
    SELECT AVG(salary)
    FROM   employees e2
    WHERE  e2.department_id = e1.department_id  -- ← ใช้ indexed column
);
```

### ตัวอย่างที่ 27: Materialization แก้ปัญหา Performance

```sql
-- แทนที่จะใช้ correlated subquery ซ้ำๆ
-- ใช้ CTE หรือ Derived Table เพื่อ materialize ผลลัพธ์ก่อน

-- ช้า (correlated ซ้ำ 2 ครั้ง):
SELECT
    first_name,
    salary,
    (SELECT AVG(salary) FROM employees WHERE department_id = e.department_id) AS dept_avg,
    (SELECT MAX(salary) FROM employees WHERE department_id = e.department_id) AS dept_max
FROM employees e;

-- เร็วกว่า (JOIN กับ derived table):
SELECT
    e.first_name,
    e.salary,
    ds.avg_salary AS dept_avg,
    ds.max_salary AS dept_max
FROM employees e
JOIN (
    SELECT department_id,
           AVG(salary) AS avg_salary,
           MAX(salary) AS max_salary
    FROM   employees
    GROUP  BY department_id
) AS ds ON ds.department_id = e.department_id;
```

### ตัวอย่างที่ 28: กรณีที่ Correlated ดีกว่า

```sql
-- Correlated EXISTS มักเร็วกว่า IN สำหรับ large tables
-- เพราะ short-circuit เมื่อเจอแถวแรก

-- ช้ากว่า (IN):
SELECT * FROM customers
WHERE customer_id IN (SELECT customer_id FROM orders);

-- เร็วกว่า (EXISTS):
SELECT * FROM customers c
WHERE EXISTS (SELECT 1 FROM orders WHERE customer_id = c.customer_id);
```

---

## 45.10 Real-world Examples

### ตัวอย่างที่ 29: Sales Person vs Team Average

```sql
-- แสดงพนักงาน Sales พร้อมยอดขายของตัวเองเทียบกับ team average
-- (สมมติว่า employees บางคนเป็น sales staff ที่รับผิดชอบ orders)
SELECT
    e.first_name,
    e.last_name,
    d.department_name,
    (SELECT COUNT(*)
     FROM   orders o
     WHERE  o.customer_id IN (
         SELECT customer_id FROM customers
         WHERE  city = (SELECT city FROM departments WHERE department_id = e.department_id)
     )) AS regional_orders
FROM employees e
JOIN departments d ON d.department_id = e.department_id
WHERE d.department_name = 'Sales';
```

### ตัวอย่างที่ 30: Customer Purchase Frequency

```sql
-- หาลูกค้าที่ซื้อบ่อยกว่าค่าเฉลี่ยของเมืองเดียวกัน
SELECT c.first_name, c.last_name, c.city,
       (SELECT COUNT(*)
        FROM   orders
        WHERE  customer_id = c.customer_id) AS order_count
FROM   customers c
WHERE  (SELECT COUNT(*) FROM orders WHERE customer_id = c.customer_id)
       > (
           SELECT AVG(city_orders)
           FROM (
               SELECT c2.customer_id,
                      COUNT(o.order_id) AS city_orders
               FROM   customers c2
               LEFT   JOIN orders o ON o.customer_id = c2.customer_id
               WHERE  c2.city = c.city  -- ← correlated!
               GROUP  BY c2.customer_id
           ) AS city_counts
       );
```

---

## แบบฝึกหัดบทที่ 45

**ข้อ 1:** ใช้ correlated subquery หาสินค้าที่มีราคาสูงกว่าราคาเฉลี่ยของ category เดียวกัน

```sql
-- เฉลย:
SELECT product_name, category, price
FROM   products p1
WHERE  price > (
    SELECT AVG(price)
    FROM   products p2
    WHERE  p2.category = p1.category
)
ORDER BY category, price DESC;
```

**ข้อ 2:** หาออเดอร์แรกของแต่ละลูกค้า (correlated subquery)

```sql
-- เฉลย:
SELECT order_id, customer_id, order_date, total_amount
FROM   orders o1
WHERE  order_date = (
    SELECT MIN(order_date)
    FROM   orders o2
    WHERE  o2.customer_id = o1.customer_id
)
ORDER BY customer_id;
```

**ข้อ 3:** ใช้ correlated subquery ใน SELECT แสดงสินค้าพร้อม rank ในหมวดหมู่ตามราคา

```sql
-- เฉลย:
SELECT
    product_name,
    category,
    price,
    (SELECT COUNT(*) + 1
     FROM   products p2
     WHERE  p2.category = p1.category
       AND  p2.price > p1.price) AS price_rank_in_category
FROM products p1
ORDER BY category, price_rank_in_category;
```

**ข้อ 4:** แปลง correlated subquery ต่อไปนี้เป็น JOIN

```sql
-- Original:
SELECT c.first_name, c.last_name
FROM   customers c
WHERE  (SELECT COUNT(*) FROM orders WHERE customer_id = c.customer_id) >= 2;

-- เฉลย (แปลงเป็น JOIN):
SELECT c.first_name, c.last_name
FROM   customers c
JOIN (
    SELECT customer_id, COUNT(*) AS order_count
    FROM   orders
    GROUP  BY customer_id
    HAVING COUNT(*) >= 2
) AS frequent_buyers ON frequent_buyers.customer_id = c.customer_id;
```

**ข้อ 5:** ใช้ correlated subquery หาพนักงานที่มีเงินเดือนต่ำสุดในแผนกตัวเอง

```sql
-- เฉลย:
SELECT first_name, last_name, salary, department_id
FROM   employees e1
WHERE  salary = (
    SELECT MIN(salary)
    FROM   employees e2
    WHERE  e2.department_id = e1.department_id
);
```

**ข้อ 6:** ใช้ correlated subquery ใน SELECT แสดงออเดอร์พร้อม running total ของลูกค้า

```sql
-- เฉลย:
SELECT
    o1.order_id,
    o1.customer_id,
    o1.order_date,
    o1.total_amount,
    (SELECT SUM(total_amount)
     FROM   orders o2
     WHERE  o2.customer_id = o1.customer_id
       AND  o2.order_date <= o1.order_date) AS running_total
FROM orders o1
ORDER BY o1.customer_id, o1.order_date;
```

**ข้อ 7:** หาสินค้าที่ขายได้มากที่สุดในแต่ละ category (correlated)

```sql
-- เฉลย:
SELECT p.product_name, p.category,
       COALESCE(SUM(oi.quantity), 0) AS total_qty
FROM   products p
LEFT   JOIN order_items oi ON oi.product_id = p.product_id
GROUP  BY p.product_id, p.product_name, p.category
HAVING COALESCE(SUM(oi.quantity), 0) = (
    SELECT MAX(cat_qty)
    FROM (
        SELECT p2.product_id,
               COALESCE(SUM(oi2.quantity), 0) AS cat_qty
        FROM   products p2
        LEFT   JOIN order_items oi2 ON oi2.product_id = p2.product_id
        WHERE  p2.category = p.category
        GROUP  BY p2.product_id
    ) AS cat_totals
)
ORDER BY p.category;
```

**ข้อ 8:** ใช้ correlated subquery ใน HAVING หาเดือนที่มียอดขายสูงกว่าค่าเฉลี่ยของทุกเดือน

```sql
-- เฉลย:
SELECT DATE_FORMAT(order_date, '%Y-%m') AS month_label,
       SUM(total_amount)                AS monthly_revenue
FROM   orders
GROUP  BY DATE_FORMAT(order_date, '%Y-%m')
HAVING SUM(total_amount) > (
    SELECT AVG(monthly_total)
    FROM (
        SELECT SUM(total_amount) AS monthly_total
        FROM   orders
        GROUP  BY DATE_FORMAT(order_date, '%Y-%m')
    ) AS all_months
);
```

**ข้อ 9:** เปรียบเทียบ performance ระหว่าง correlated subquery และ JOIN โดยใช้ EXPLAIN

```sql
-- เฉลย:
-- Query 1: Correlated
EXPLAIN
SELECT c.first_name, c.last_name
FROM   customers c
WHERE  (SELECT SUM(total_amount) FROM orders WHERE customer_id = c.customer_id) > 30000;

-- Query 2: JOIN
EXPLAIN
SELECT c.first_name, c.last_name
FROM   customers c
JOIN (
    SELECT customer_id, SUM(total_amount) AS total_spent
    FROM   orders
    GROUP  BY customer_id
    HAVING SUM(total_amount) > 30000
) AS big_spenders ON big_spenders.customer_id = c.customer_id;

-- สังเกต: Query 1 มี DEPENDENT SUBQUERY, Query 2 ไม่มี
```

**ข้อ 10:** เขียน correlated subquery ที่ซับซ้อน: หาลูกค้าที่ทุกออเดอร์มียอดสูงกว่า 5,000 บาท

```sql
-- เฉลย:
SELECT first_name, last_name
FROM   customers c
WHERE  NOT EXISTS (
    SELECT 1
    FROM   orders o
    WHERE  o.customer_id = c.customer_id
      AND  o.total_amount <= 5000
)
  AND EXISTS (
    SELECT 1 FROM orders WHERE customer_id = c.customer_id
);
-- หรือแบบ correlated aggregate:
SELECT first_name, last_name
FROM   customers c
WHERE  EXISTS (SELECT 1 FROM orders WHERE customer_id = c.customer_id)
  AND  (SELECT MIN(total_amount) FROM orders WHERE customer_id = c.customer_id) > 5000;
```

---

*จบบทที่ 45: Correlated Subqueries*
*บทถัดไป: Part 46 - EXISTS and NOT EXISTS*
