# Part 42: Scalar Subqueries in SELECT and WHERE (Scalar Subqueries)

## 42.1 Scalar Subquery คืออะไร?

**Scalar Subquery** คือ subquery ที่คืนค่าเพียง **1 แถว และ 1 คอลัมน์** เท่านั้น สามารถใช้แทนค่าคงที่ (literal value) ได้ทุกที่ใน query

```
Scalar Subquery → คืนค่าเดียว (scalar value)

ตัวอย่าง:
(SELECT AVG(salary) FROM employees)   → 80200.00
(SELECT MAX(price) FROM products)     → 45000.00
(SELECT COUNT(*) FROM orders)         → 15
```

---

## 42.2 Scalar Subquery ใน SELECT clause

### ตัวอย่างที่ 1: แสดงค่าเฉลี่ยรวมในทุกแถว

```sql
-- แสดงเงินเดือนแต่ละคนพร้อมค่าเฉลี่ยบริษัท
SELECT
    first_name,
    last_name,
    salary,
    (SELECT AVG(salary) FROM employees)  AS company_avg_salary
FROM employees;
```

ผลลัพธ์:
```
first_name  | last_name   | salary   | company_avg_salary
------------|-------------|----------|-------------------
Somchai     | Jaidee      | 85000.00 | 80200.00
Wanida      | Kaewkla     | 72000.00 | 80200.00
Prasert     | Thongdee    | 95000.00 | 80200.00
...
```

### ตัวอย่างที่ 2: คำนวณความต่างจากค่าเฉลี่ย

```sql
SELECT
    first_name,
    last_name,
    salary,
    ROUND(salary - (SELECT AVG(salary) FROM employees), 2) AS diff_from_avg,
    CASE
        WHEN salary > (SELECT AVG(salary) FROM employees) THEN 'Above Average'
        WHEN salary < (SELECT AVG(salary) FROM employees) THEN 'Below Average'
        ELSE 'At Average'
    END AS salary_status
FROM employees
ORDER BY salary DESC;
```

### ตัวอย่างที่ 3: เปอร์เซ็นต์เทียบกับสูงสุด

```sql
SELECT
    product_name,
    price,
    (SELECT MAX(price) FROM products)            AS max_price,
    ROUND(price / (SELECT MAX(price) FROM products) * 100, 1) AS pct_of_max
FROM products
ORDER BY price DESC;
```

### ตัวอย่างที่ 4: Scalar subquery อ้างอิงตารางอื่น

```sql
-- แสดงชื่อแผนกในตาราง employees (แทนที่จะต้อง JOIN)
SELECT
    e.first_name,
    e.last_name,
    e.salary,
    (SELECT d.department_name
     FROM   departments d
     WHERE  d.department_id = e.department_id) AS department_name
FROM employees e
ORDER BY department_name;
```

### ตัวอย่างที่ 5: COUNT ด้วย subquery

```sql
-- แสดงจำนวนออเดอร์ของแต่ละลูกค้า
SELECT
    c.customer_id,
    CONCAT(c.first_name, ' ', c.last_name)   AS customer_name,
    c.city,
    (SELECT COUNT(*)
     FROM   orders o
     WHERE  o.customer_id = c.customer_id)   AS total_orders,
    (SELECT COALESCE(SUM(total_amount), 0)
     FROM   orders o
     WHERE  o.customer_id = c.customer_id)   AS total_spent
FROM customers c
ORDER BY total_spent DESC;
```

### ตัวอย่างที่ 6: ข้อมูลวันที่ล่าสุด

```sql
-- แสดงออเดอร์ล่าสุดของแต่ละลูกค้า
SELECT
    c.customer_id,
    c.first_name,
    c.last_name,
    (SELECT MAX(order_date)
     FROM   orders o
     WHERE  o.customer_id = c.customer_id) AS last_order_date,
    (SELECT status
     FROM   orders o
     WHERE  o.customer_id = c.customer_id
       AND  o.order_date = (
                SELECT MAX(order_date)
                FROM   orders
                WHERE  customer_id = c.customer_id
            )
     LIMIT 1) AS last_order_status
FROM customers c;
```

### ตัวอย่างที่ 7: Ranking ด้วย scalar subquery

```sql
-- Rank ของเงินเดือนในบริษัท (ก่อน MySQL 8 มี RANK())
SELECT
    first_name,
    last_name,
    salary,
    (SELECT COUNT(*) + 1
     FROM   employees e2
     WHERE  e2.salary > e1.salary) AS salary_rank
FROM employees e1
ORDER BY salary DESC;
```

### ตัวอย่างที่ 8: Scalar subquery กับ NULL handling

```sql
-- แสดงชื่อ Manager (ถ้าไม่มีแสดงว่า Top Level)
SELECT
    e.first_name,
    e.last_name,
    COALESCE(
        (SELECT CONCAT(m.first_name, ' ', m.last_name)
         FROM   employees m
         WHERE  m.employee_id = e.manager_id),
        'Top Level'
    ) AS manager_name
FROM employees e;
```

---

## 42.3 Scalar Subquery ใน WHERE clause

### ตัวอย่างที่ 9: เปรียบเทียบกับ AVG

```sql
-- พนักงานที่มีเงินเดือนสูงกว่าค่าเฉลี่ย
SELECT first_name, last_name, salary
FROM   employees
WHERE  salary > (SELECT AVG(salary) FROM employees)
ORDER  BY salary DESC;
```

### ตัวอย่างที่ 10: เปรียบเทียบกับ MAX

```sql
-- สินค้าที่มีราคาต่ำกว่าราคาสูงสุดของ Electronics 50%
SELECT product_name, category, price
FROM   products
WHERE  price < (SELECT MAX(price) FROM products WHERE category = 'Electronics') * 0.5;
```

### ตัวอย่างที่ 11: เปรียบเทียบกับ MIN

```sql
-- ออเดอร์ที่มียอดสูงกว่าออเดอร์น้อยสุด 3 เท่า
SELECT order_id, customer_id, total_amount, order_date
FROM   orders
WHERE  total_amount > (SELECT MIN(total_amount) FROM orders) * 3
ORDER  BY total_amount DESC;
```

### ตัวอย่างที่ 12: เปรียบเทียบ >= และ <=

```sql
-- พนักงานที่มีเงินเดือนอยู่ระหว่าง MIN และ AVG
SELECT first_name, last_name, salary
FROM   employees
WHERE  salary >= (SELECT MIN(salary) FROM employees)
  AND  salary <= (SELECT AVG(salary) FROM employees)
ORDER  BY salary;
```

### ตัวอย่างที่ 13: ใช้กับ <> (ไม่เท่ากับ)

```sql
-- ออเดอร์ที่ไม่ใช่ออเดอร์ล่าสุด
SELECT order_id, customer_id, order_date, total_amount
FROM   orders
WHERE  order_date <> (SELECT MAX(order_date) FROM orders);
```

### ตัวอย่างที่ 14: Scalar subquery กับ BETWEEN

```sql
-- สินค้าที่มีราคาอยู่ระหว่างราคาต่ำสุดกับราคาเฉลี่ย
SELECT product_name, category, price
FROM   products
WHERE  price BETWEEN
    (SELECT MIN(price) FROM products)
    AND
    (SELECT AVG(price) FROM products)
ORDER  BY price;
```

### ตัวอย่างที่ 15: ใช้กับ Date functions

```sql
-- ออเดอร์ที่เกิดขึ้นก่อนออเดอร์แรกของลูกค้าคนที่ 2
SELECT order_id, customer_id, order_date, total_amount
FROM   orders
WHERE  order_date < (
    SELECT MIN(order_date)
    FROM   orders
    WHERE  customer_id = 2
);
```

---

## 42.4 Scalar Subquery กับ Aggregate Functions

### ตัวอย่างที่ 16: ใช้ AVG ใน subquery

```sql
-- แสดงสินค้าที่มีราคาสูงกว่าค่าเฉลี่ยของแต่ละหมวดหมู่
SELECT p1.product_name, p1.category, p1.price,
       ROUND((SELECT AVG(p2.price)
              FROM   products p2
              WHERE  p2.category = p1.category), 2) AS category_avg
FROM   products p1
WHERE  p1.price > (
    SELECT AVG(p2.price)
    FROM   products p2
    WHERE  p2.category = p1.category
)
ORDER  BY p1.category, p1.price DESC;
```

### ตัวอย่างที่ 17: ใช้ SUM ใน subquery

```sql
-- แสดงออเดอร์ที่มียอดสูงกว่าค่าเฉลี่ยรายเดือน
SELECT order_id, order_date, total_amount
FROM   orders
WHERE  total_amount > (
    SELECT AVG(monthly_total)
    FROM (
        SELECT YEAR(order_date)  AS yr,
               MONTH(order_date) AS mo,
               SUM(total_amount) AS monthly_total
        FROM   orders
        GROUP  BY YEAR(order_date), MONTH(order_date)
    ) AS monthly_totals
);
```

### ตัวอย่างที่ 18: ใช้ COUNT ใน subquery

```sql
-- พนักงานในแผนกที่มีสมาชิกมากกว่า 2 คน
SELECT first_name, last_name, department_id
FROM   employees
WHERE  department_id IN (
    SELECT department_id
    FROM   employees
    GROUP  BY department_id
    HAVING COUNT(*) > 2
);
-- หมายเหตุ: ตรงนี้ใช้ IN กับ subquery ที่คืนหลายค่า ไม่ใช่ scalar
-- ตัวอย่าง scalar จริงๆ:

-- แผนกที่มีพนักงานมากที่สุด
SELECT first_name, last_name, department_id
FROM   employees
WHERE  department_id = (
    SELECT department_id
    FROM   employees
    GROUP  BY department_id
    ORDER  BY COUNT(*) DESC
    LIMIT  1
);
```

### ตัวอย่างที่ 19: ใช้ MAX กับ subquery ซ้อน

```sql
-- สินค้าที่มีราคาสูงสุดในหมวดหมู่ที่มีสินค้าหลายชิ้น
SELECT product_name, category, price
FROM   products p
WHERE  price = (
    SELECT MAX(price)
    FROM   products
    WHERE  category = p.category
)
  AND  category IN (
    SELECT category
    FROM   products
    GROUP  BY category
    HAVING COUNT(*) > 1
);
```

---

## 42.5 Pattern: Aggregates ใน WHERE

### ตัวอย่างที่ 20: หา top N records

```sql
-- ลูกค้าที่ใช้จ่ายมากกว่าค่าใช้จ่ายเฉลี่ยของ Top 3 ลูกค้า
SELECT c.first_name, c.last_name,
       SUM(o.total_amount) AS total_spent
FROM   customers c
JOIN   orders o ON o.customer_id = c.customer_id
GROUP  BY c.customer_id, c.first_name, c.last_name
HAVING SUM(o.total_amount) > (
    SELECT AVG(top3_total)
    FROM (
        SELECT SUM(total_amount) AS top3_total
        FROM   orders
        GROUP  BY customer_id
        ORDER  BY top3_total DESC
        LIMIT  3
    ) AS top3
);
```

### ตัวอย่างที่ 21: เปรียบเทียบกับ Benchmark

```sql
-- สินค้าที่มีราคาต่ำกว่าสินค้าที่ขายดีที่สุด (สินค้า id = 1)
SELECT product_name, price
FROM   products
WHERE  price < (
    SELECT price FROM products WHERE product_id = 1
)
ORDER  BY price DESC;
```

### ตัวอย่างที่ 22: ใช้ Subquery ในการกรองวันที่

```sql
-- ออเดอร์ที่ทำหลังจากออเดอร์แรกของระบบ 30 วัน
SELECT order_id, customer_id, order_date, total_amount
FROM   orders
WHERE  order_date > DATE_ADD(
    (SELECT MIN(order_date) FROM orders),
    INTERVAL 30 DAY
);
```

---

## 42.6 Error: Subquery Returns More Than One Row

### ตัวอย่างที่ 23: ข้อผิดพลาดที่พบบ่อย

```sql
-- ERROR: กรณีนี้จะเกิด error เพราะ subquery คืนหลายแถว
SELECT first_name, last_name
FROM   employees
WHERE  salary = (
    SELECT salary
    FROM   employees
    WHERE  department_id = 1
    -- คืน: 85000, 72000, 82000 (3 แถว!)
);
-- ERROR 1242: Subquery returns more than 1 row

-- วิธีแก้ไข 1: ใช้ MAX/MIN
SELECT first_name, last_name
FROM   employees
WHERE  salary = (
    SELECT MAX(salary)
    FROM   employees
    WHERE  department_id = 1
);

-- วิธีแก้ไข 2: ใช้ ANY/IN
SELECT first_name, last_name
FROM   employees
WHERE  salary = ANY (
    SELECT salary
    FROM   employees
    WHERE  department_id = 1
);
-- หรือ
SELECT first_name, last_name
FROM   employees
WHERE  salary IN (
    SELECT salary
    FROM   employees
    WHERE  department_id = 1
);

-- วิธีแก้ไข 3: ใช้ LIMIT 1 (ระวัง: ผลอาจไม่ถูกต้องเสมอไป)
SELECT first_name, last_name
FROM   employees
WHERE  salary = (
    SELECT salary
    FROM   employees
    WHERE  department_id = 1
    ORDER  BY salary DESC
    LIMIT  1
);
```

### ตัวอย่างที่ 24: ตรวจสอบก่อนใช้

```sql
-- ตรวจสอบว่า subquery คืนค่าเดียวหรือไม่
SELECT COUNT(*)
FROM (
    SELECT salary
    FROM   employees
    WHERE  department_id = 1
) AS check_count;
-- ถ้า > 1 ต้องปรับ query

-- วิธีปลอดภัย: ใช้ aggregate เสมอ
SELECT first_name, last_name
FROM   employees
WHERE  salary > (
    SELECT AVG(salary)      -- AVG คืนค่าเดียวเสมอ
    FROM   employees
    WHERE  department_id = 1
);
```

---

## 42.7 Scalar Subquery กับ NULL

### ตัวอย่างที่ 25: Subquery ที่ไม่คืนแถว

```sql
-- ถ้า subquery ไม่มีข้อมูล จะคืน NULL
SELECT first_name, salary
FROM   employees
WHERE  salary > (
    SELECT MAX(salary)
    FROM   employees
    WHERE  department_id = 999  -- ไม่มีแผนกนี้
);
-- ผลลัพธ์: ไม่มีแถว (NULL > anything = UNKNOWN = FALSE)

-- ป้องกันด้วย COALESCE
SELECT first_name, salary
FROM   employees
WHERE  salary > COALESCE(
    (SELECT MAX(salary) FROM employees WHERE department_id = 999),
    0
);
-- ผลลัพธ์: ทุกแถว (เพราะ salary > 0)
```

### ตัวอย่างที่ 26: NULL ใน aggregate

```sql
-- AVG ของชุดข้อมูลว่างคือ NULL ไม่ใช่ 0
SELECT
    product_name,
    price,
    (SELECT AVG(unit_price)
     FROM   order_items oi
     WHERE  oi.product_id = p.product_id) AS avg_sold_price
FROM products p;
-- สินค้าที่ไม่เคยขาย → avg_sold_price = NULL
```

### ตัวอย่างที่ 27: COALESCE กับ scalar subquery

```sql
SELECT
    c.first_name,
    c.last_name,
    COALESCE(
        (SELECT SUM(total_amount)
         FROM   orders
         WHERE  customer_id = c.customer_id),
        0
    ) AS total_spent,
    COALESCE(
        (SELECT MAX(order_date)
         FROM   orders
         WHERE  customer_id = c.customer_id),
        'No orders yet'
    ) AS last_order
FROM customers c;
```

---

## 42.8 Single-row Subquery Patterns

### ตัวอย่างที่ 28: Pattern - "ดีกว่าค่าเฉลี่ย"

```sql
-- Products ที่ขายดีกว่าค่าเฉลี่ย
SELECT p.product_name, SUM(oi.quantity) AS total_sold
FROM   products p
JOIN   order_items oi ON oi.product_id = p.product_id
GROUP  BY p.product_id, p.product_name
HAVING SUM(oi.quantity) > (
    SELECT AVG(item_qty)
    FROM (
        SELECT product_id, SUM(quantity) AS item_qty
        FROM   order_items
        GROUP  BY product_id
    ) AS t
);
```

### ตัวอย่างที่ 29: Pattern - "เทียบกับ Record ล่าสุด"

```sql
-- ออเดอร์ในวันที่มีออเดอร์ล่าสุด
SELECT order_id, customer_id, order_date, total_amount
FROM   orders
WHERE  order_date = (SELECT MAX(order_date) FROM orders);
```

### ตัวอย่างที่ 30: Pattern - "Top 1"

```sql
-- ลูกค้าที่ใช้จ่ายมากที่สุด
SELECT first_name, last_name
FROM   customers
WHERE  customer_id = (
    SELECT customer_id
    FROM   orders
    GROUP  BY customer_id
    ORDER  BY SUM(total_amount) DESC
    LIMIT  1
);
```

### ตัวอย่างที่ 31: Pattern - "Percentile"

```sql
-- พนักงานที่มีเงินเดือนอยู่ใน 25% บนสุด
SELECT first_name, last_name, salary
FROM   employees
WHERE  salary >= (
    SELECT salary
    FROM (
        SELECT salary,
               NTILE(4) OVER (ORDER BY salary) AS quartile
        FROM   employees
    ) AS quartiled
    WHERE  quartile = 4
    ORDER  BY salary
    LIMIT  1
);
```

---

## 42.9 ตัวอย่างจากธุรกิจจริง

### ตัวอย่างที่ 32: Dashboard Report

```sql
-- สรุปข้อมูลธุรกิจ (ใช้ scalar subquery ใน SELECT)
SELECT
    'Total Revenue'                                           AS metric,
    (SELECT SUM(total_amount) FROM orders WHERE status = 'completed') AS value
UNION ALL
SELECT
    'Total Customers',
    (SELECT COUNT(*) FROM customers)
UNION ALL
SELECT
    'Average Order Value',
    (SELECT AVG(total_amount) FROM orders)
UNION ALL
SELECT
    'Top Product Revenue',
    (SELECT SUM(oi.quantity * oi.unit_price)
     FROM   order_items oi
     WHERE  oi.product_id = (
         SELECT product_id
         FROM   order_items
         GROUP  BY product_id
         ORDER  BY SUM(quantity) DESC
         LIMIT  1
     ));
```

### ตัวอย่างที่ 33: Inventory Alert

```sql
-- สินค้าที่มี stock ต่ำกว่าค่าเฉลี่ย stock
SELECT
    product_name,
    category,
    stock_qty,
    (SELECT AVG(stock_qty) FROM products) AS avg_stock,
    ROUND(
        (SELECT AVG(stock_qty) FROM products) - stock_qty
    , 0) AS below_avg_by
FROM   products
WHERE  stock_qty < (SELECT AVG(stock_qty) FROM products)
ORDER  BY stock_qty ASC;
```

### ตัวอย่างที่ 34: Customer Segment Analysis

```sql
-- แบ่ง segment ลูกค้าตามยอดซื้อเทียบกับค่าเฉลี่ย
SELECT
    c.first_name,
    c.last_name,
    COALESCE(SUM(o.total_amount), 0) AS total_spent,
    CASE
        WHEN COALESCE(SUM(o.total_amount), 0) >
             (SELECT AVG(customer_total) * 2
              FROM (SELECT customer_id, SUM(total_amount) AS customer_total
                    FROM orders GROUP BY customer_id) AS t)
        THEN 'VIP'
        WHEN COALESCE(SUM(o.total_amount), 0) >
             (SELECT AVG(customer_total)
              FROM (SELECT customer_id, SUM(total_amount) AS customer_total
                    FROM orders GROUP BY customer_id) AS t)
        THEN 'Regular'
        ELSE 'New/Inactive'
    END AS segment
FROM customers c
LEFT JOIN orders o ON o.customer_id = c.customer_id
GROUP BY c.customer_id, c.first_name, c.last_name
ORDER BY total_spent DESC;
```

### ตัวอย่างที่ 35: Price Comparison

```sql
-- เปรียบเทียบราคาสินค้ากับ Market Benchmark
SELECT
    product_name,
    category,
    price AS our_price,
    (SELECT AVG(price) FROM products WHERE category = p.category) AS category_avg,
    ROUND(
        (price - (SELECT AVG(price) FROM products WHERE category = p.category))
        / (SELECT AVG(price) FROM products WHERE category = p.category) * 100
    , 1) AS pct_diff_from_avg
FROM products p
ORDER BY ABS(
    (price - (SELECT AVG(price) FROM products WHERE category = p.category))
    / (SELECT AVG(price) FROM products WHERE category = p.category) * 100
) DESC;
```

---

## 42.10 Performance Note

```sql
-- Scalar subquery ใน SELECT ทำงานซ้ำทุกแถว
-- ถ้ามีข้อมูลมาก อาจช้า
-- แนะนำ: ใช้ JOIN หรือ CTE แทน

-- ช้า (scalar subquery ใน SELECT):
SELECT
    c.customer_id,
    c.first_name,
    (SELECT COUNT(*) FROM orders WHERE customer_id = c.customer_id) AS order_count
FROM customers c;

-- เร็วกว่า (JOIN):
SELECT
    c.customer_id,
    c.first_name,
    COUNT(o.order_id) AS order_count
FROM customers c
LEFT JOIN orders o ON o.customer_id = c.customer_id
GROUP BY c.customer_id, c.first_name;
```

---

## แบบฝึกหัดบทที่ 42

**ข้อ 1:** ใช้ scalar subquery ใน SELECT แสดงรายการสินค้าพร้อมราคา และเปอร์เซ็นต์ราคาเทียบกับราคาเฉลี่ยทั้งหมด

```sql
-- เฉลย:
SELECT
    product_name,
    category,
    price,
    ROUND((SELECT AVG(price) FROM products), 2) AS overall_avg,
    ROUND(price / (SELECT AVG(price) FROM products) * 100, 1) AS pct_of_avg
FROM products
ORDER BY pct_of_avg DESC;
```

**ข้อ 2:** หาพนักงานที่มีเงินเดือนอยู่ระหว่างเงินเดือนต่ำสุดและเงินเดือนเฉลี่ย

```sql
-- เฉลย:
SELECT first_name, last_name, salary
FROM   employees
WHERE  salary BETWEEN (SELECT MIN(salary) FROM employees)
              AND     (SELECT AVG(salary) FROM employees)
ORDER  BY salary;
```

**ข้อ 3:** แสดงชื่อลูกค้าพร้อมกับชื่อเมืองและจำนวนลูกค้าในเมืองเดียวกัน

```sql
-- เฉลย:
SELECT
    first_name,
    last_name,
    city,
    (SELECT COUNT(*)
     FROM   customers c2
     WHERE  c2.city = c.city) AS customers_in_same_city
FROM customers c
ORDER BY city, first_name;
```

**ข้อ 4:** หาออเดอร์ที่มียอดสูงกว่าออเดอร์ล่าสุด (highest order_date) ทั้งหมด

```sql
-- เฉลย:
SELECT order_id, customer_id, order_date, total_amount
FROM   orders
WHERE  total_amount > (
    SELECT total_amount
    FROM   orders
    ORDER  BY order_date DESC
    LIMIT  1
);
```

**ข้อ 5:** แสดงสินค้าพร้อมยอดขายในหน่วยนับ และเปรียบเทียบว่าขายได้กี่เปอร์เซ็นต์ของยอดขายรวม

```sql
-- เฉลย:
SELECT
    p.product_name,
    COALESCE(SUM(oi.quantity), 0) AS units_sold,
    (SELECT SUM(quantity) FROM order_items) AS total_units,
    ROUND(
        COALESCE(SUM(oi.quantity), 0) * 100.0
        / (SELECT SUM(quantity) FROM order_items),
        1
    ) AS pct_of_total
FROM products p
LEFT JOIN order_items oi ON oi.product_id = p.product_id
GROUP BY p.product_id, p.product_name
ORDER BY units_sold DESC;
```

**ข้อ 6:** หาแผนกที่มีเงินเดือนสูงสุดใน SELECT clause พร้อมชื่อพนักงานและเงินเดือน

```sql
-- เฉลย:
SELECT
    e.first_name,
    e.last_name,
    e.salary,
    d.department_name,
    (SELECT MAX(e2.salary)
     FROM   employees e2
     WHERE  e2.department_id = e.department_id) AS dept_max_salary,
    CASE WHEN e.salary = (SELECT MAX(e2.salary) FROM employees e2
                          WHERE e2.department_id = e.department_id)
         THEN 'Highest in Dept'
         ELSE ''
    END AS note
FROM employees e
JOIN departments d ON d.department_id = e.department_id
ORDER BY d.department_name, e.salary DESC;
```

**ข้อ 7:** ใช้ scalar subquery หา customers ที่มียอดซื้อสูงกว่าค่าเฉลี่ยยอดซื้อต่อคน

```sql
-- เฉลย:
SELECT c.first_name, c.last_name,
       SUM(o.total_amount) AS total_spent
FROM   customers c
JOIN   orders o ON o.customer_id = c.customer_id
GROUP  BY c.customer_id, c.first_name, c.last_name
HAVING SUM(o.total_amount) > (
    SELECT AVG(customer_total)
    FROM (
        SELECT customer_id, SUM(total_amount) AS customer_total
        FROM   orders
        GROUP  BY customer_id
    ) AS t
);
```

**ข้อ 8:** แสดงสินค้าในหมวด Electronics ที่มีราคาต่ำกว่าราคาเฉลี่ยของ Electronics ทั้งหมด

```sql
-- เฉลย:
SELECT product_name, price
FROM   products
WHERE  category = 'Electronics'
  AND  price < (
    SELECT AVG(price)
    FROM   products
    WHERE  category = 'Electronics'
)
ORDER BY price DESC;
```

**ข้อ 9:** ใช้ scalar subquery แสดง order พร้อมค่า deviation จาก average ของ customer นั้น

```sql
-- เฉลย:
SELECT
    o.order_id,
    o.customer_id,
    o.order_date,
    o.total_amount,
    ROUND(
        (SELECT AVG(total_amount) FROM orders WHERE customer_id = o.customer_id),
        2
    ) AS customer_avg,
    ROUND(
        o.total_amount - (SELECT AVG(total_amount) FROM orders WHERE customer_id = o.customer_id),
        2
    ) AS deviation
FROM orders o
ORDER BY o.customer_id, o.order_date;
```

**ข้อ 10:** สร้าง report แสดงสถิติรวม: จำนวนลูกค้า, จำนวนสินค้า, จำนวนออเดอร์, ยอดขายรวม ในแถวเดียว

```sql
-- เฉลย:
SELECT
    (SELECT COUNT(*) FROM customers)              AS total_customers,
    (SELECT COUNT(*) FROM products)               AS total_products,
    (SELECT COUNT(*) FROM orders)                 AS total_orders,
    (SELECT SUM(total_amount) FROM orders)        AS total_revenue,
    (SELECT AVG(total_amount) FROM orders)        AS avg_order_value,
    (SELECT MAX(total_amount) FROM orders)        AS max_order,
    (SELECT MIN(total_amount) FROM orders)        AS min_order;
```

---

*จบบทที่ 42: Scalar Subqueries in SELECT and WHERE*
*บทถัดไป: Part 43 - Multi-row Subqueries with IN and NOT IN*
