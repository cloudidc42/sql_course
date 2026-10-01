# Part 46: EXISTS and NOT EXISTS

## 46.1 EXISTS คืออะไร?

`EXISTS` ตรวจสอบว่า subquery คืนแถวอย่างน้อย 1 แถวหรือไม่
- ถ้าคืนแถวใดก็ตาม → `EXISTS` = TRUE
- ถ้าไม่คืนแถว → `EXISTS` = FALSE

```sql
SELECT ...
FROM   table1 t1
WHERE  EXISTS (
    SELECT 1               -- ← ค่าที่ SELECT ไม่สำคัญ, SELECT 1 นิยมใช้
    FROM   table2 t2
    WHERE  t2.key = t1.key -- ← มักเป็น correlated
);
```

---

## 46.2 EXISTS พื้นฐาน

### ตัวอย่างที่ 1: ลูกค้าที่มีออเดอร์

```sql
-- ลูกค้าที่เคยสั่งซื้ออย่างน้อยหนึ่งออเดอร์
SELECT c.customer_id, c.first_name, c.last_name
FROM   customers c
WHERE  EXISTS (
    SELECT 1
    FROM   orders o
    WHERE  o.customer_id = c.customer_id
);
```

### ตัวอย่างที่ 2: สินค้าที่มีคนสั่งซื้อ

```sql
-- สินค้าที่เคยถูกสั่งซื้อ
SELECT product_id, product_name, category, price
FROM   products p
WHERE  EXISTS (
    SELECT 1
    FROM   order_items oi
    WHERE  oi.product_id = p.product_id
);
```

### ตัวอย่างที่ 3: EXISTS กับเงื่อนไขใน subquery

```sql
-- ลูกค้าที่มีออเดอร์ที่สำเร็จแล้ว (status = 'completed')
SELECT first_name, last_name, city
FROM   customers c
WHERE  EXISTS (
    SELECT 1
    FROM   orders o
    WHERE  o.customer_id = c.customer_id
      AND  o.status = 'completed'
);
```

### ตัวอย่างที่ 4: EXISTS กับ amount threshold

```sql
-- ลูกค้าที่มีออเดอร์ >= 20,000 บาทอย่างน้อย 1 ออเดอร์
SELECT first_name, last_name
FROM   customers c
WHERE  EXISTS (
    SELECT 1
    FROM   orders o
    WHERE  o.customer_id = c.customer_id
      AND  o.total_amount >= 20000
);
```

---

## 46.3 NOT EXISTS

### ตัวอย่างที่ 5: ลูกค้าที่ไม่มีออเดอร์

```sql
-- ลูกค้าที่ยังไม่เคยสั่งซื้อ
SELECT c.customer_id, c.first_name, c.last_name, c.email
FROM   customers c
WHERE  NOT EXISTS (
    SELECT 1
    FROM   orders o
    WHERE  o.customer_id = c.customer_id
);
```

### ตัวอย่างที่ 6: สินค้าที่ไม่เคยขาย

```sql
-- สินค้าที่ยังไม่เคยมีการสั่งซื้อ
SELECT product_name, category, price, stock_qty
FROM   products p
WHERE  NOT EXISTS (
    SELECT 1
    FROM   order_items oi
    WHERE  oi.product_id = p.product_id
);
```

### ตัวอย่างที่ 7: NOT EXISTS กับ date

```sql
-- ลูกค้าที่ไม่มีออเดอร์ใน 3 เดือนล่าสุด (Inactive Customers)
SELECT first_name, last_name, email
FROM   customers c
WHERE  NOT EXISTS (
    SELECT 1
    FROM   orders o
    WHERE  o.customer_id = c.customer_id
      AND  o.order_date >= DATE_SUB(CURDATE(), INTERVAL 3 MONTH)
);
```

### ตัวอย่างที่ 8: NOT EXISTS กับ status

```sql
-- พนักงานที่ไม่มีลูกน้อง
SELECT first_name, last_name, employee_id
FROM   employees e
WHERE  NOT EXISTS (
    SELECT 1
    FROM   employees subordinate
    WHERE  subordinate.manager_id = e.employee_id
);
```

---

## 46.4 EXISTS vs IN: Performance Analysis

### ตัวอย่างที่ 9: ให้ผลเหมือนกัน

```sql
-- EXISTS
SELECT product_name FROM products p
WHERE EXISTS (SELECT 1 FROM order_items WHERE product_id = p.product_id);

-- IN
SELECT product_name FROM products p
WHERE product_id IN (SELECT DISTINCT product_id FROM order_items);
```

### ตัวอย่างที่ 10: EXISTS ดีกว่าเมื่อ subquery table ใหญ่

```sql
-- ถ้า order_items มีล้านแถว:
-- EXISTS short-circuits เมื่อเจอ match แรก
-- IN ต้องดึงทุก product_id มาเปรียบเทียบ

-- สำหรับตาราง products เล็ก, order_items ใหญ่:
-- EXISTS มักเร็วกว่า
SELECT p.product_name FROM products p
WHERE EXISTS (
    SELECT 1 FROM order_items oi
    WHERE oi.product_id = p.product_id
    -- Short circuit: หยุดทันทีเมื่อเจอแถวแรก
);
```

### ตัวอย่างที่ 11: IN ดีกว่าเมื่อ subquery เล็ก

```sql
-- ถ้า subquery คืนค่าน้อยมาก (เช่น 3-5 ค่า):
-- IN อาจเร็วกว่าเพราะ MySQL สามารถ optimize ได้ดีกว่า

SELECT product_name FROM products
WHERE category IN (SELECT DISTINCT category FROM products WHERE price > 10000);
-- categories มีไม่กี่ค่า → IN ดีพอ
```

---

## 46.5 NOT EXISTS vs NOT IN: NULL Handling

### ตัวอย่างที่ 12: NULL trap กับ NOT IN

```sql
-- ⚠️ NOT IN + NULL = อันตราย!
-- สมมติ orders มี customer_id = NULL (theoretical)

-- อันตราย:
SELECT first_name FROM customers
WHERE customer_id NOT IN (SELECT customer_id FROM orders);
-- ถ้า customer_id ใน orders มี NULL → ไม่คืนผลลัพธ์!

-- ปลอดภัย:
SELECT first_name FROM customers c
WHERE NOT EXISTS (
    SELECT 1 FROM orders o WHERE o.customer_id = c.customer_id
);
-- NOT EXISTS จัดการ NULL ได้ถูกต้องเสมอ
```

### ตัวอย่างที่ 13: พิสูจน์ด้วย Test

```sql
-- สร้างข้อมูลทดสอบ
CREATE TEMPORARY TABLE tmp_has_null (val INT);
INSERT INTO tmp_has_null VALUES (1), (2), (NULL);

CREATE TEMPORARY TABLE tmp_test (val INT);
INSERT INTO tmp_test VALUES (1), (2), (3), (4);

-- NOT IN + NULL → ไม่คืนผล
SELECT val FROM tmp_test WHERE val NOT IN (SELECT val FROM tmp_has_null);
-- ผลลัพธ์: ว่างเปล่า! (เพราะ NULL)

-- NOT EXISTS → คืนผลถูกต้อง
SELECT t.val FROM tmp_test t
WHERE NOT EXISTS (SELECT 1 FROM tmp_has_null n WHERE n.val = t.val);
-- ผลลัพธ์: 3, 4

DROP TEMPORARY TABLE tmp_has_null, tmp_test;
```

### ตัวอย่างที่ 14: Recommendation

```sql
-- กฎ: ใช้ NOT EXISTS แทน NOT IN เสมอเมื่อ
-- 1. ไม่แน่ใจว่า subquery มี NULL หรือไม่
-- 2. subquery มาจาก column ที่อาจเป็น NULL
-- 3. ต้องการความปลอดภัยสูงสุด

-- ✓ แนะนำ:
SELECT c.first_name FROM customers c
WHERE NOT EXISTS (SELECT 1 FROM orders WHERE customer_id = c.customer_id);

-- ✓ ถ้าต้องใช้ NOT IN ต้องกรอง NULL:
SELECT c.first_name FROM customers c
WHERE c.customer_id NOT IN (
    SELECT customer_id FROM orders WHERE customer_id IS NOT NULL
);
```

---

## 46.6 Semi-join กับ EXISTS

### ตัวอย่างที่ 15: Semi-join แนวคิด

```sql
-- Semi-join: คืนแถวจาก outer table ที่มี match ใน inner table
-- ไม่ duplicate แม้ inner table มีหลายแถว match

-- แบบ EXISTS (Semi-join ที่ชัดเจน):
SELECT c.first_name, c.last_name
FROM   customers c
WHERE  EXISTS (
    SELECT 1 FROM orders WHERE customer_id = c.customer_id
);
-- ลูกค้าแต่ละคนปรากฏครั้งเดียว แม้จะมีหลายออเดอร์

-- แบบ JOIN (ต้อง DISTINCT เพื่อหลีกเลี่ยง duplicate):
SELECT DISTINCT c.first_name, c.last_name
FROM   customers c
JOIN   orders o ON o.customer_id = c.customer_id;
-- ต้อง DISTINCT!
```

### ตัวอย่างที่ 16: Anti-join กับ NOT EXISTS

```sql
-- Anti-join: คืนแถวจาก outer table ที่ไม่มี match ใน inner table
SELECT c.first_name, c.last_name
FROM   customers c
WHERE  NOT EXISTS (
    SELECT 1 FROM orders WHERE customer_id = c.customer_id
);
-- ลูกค้าที่ไม่มีออเดอร์ (Anti-join)
```

---

## 46.7 EXISTS กับ Multiple Conditions

### ตัวอย่างที่ 17: Multiple Conditions ใน Subquery

```sql
-- ลูกค้าที่มีออเดอร์ Electronics ราคาแพงมากกว่า 30,000
SELECT c.first_name, c.last_name
FROM   customers c
WHERE  EXISTS (
    SELECT 1
    FROM   orders o
    JOIN   order_items oi ON oi.order_id = o.order_id
    JOIN   products p     ON p.product_id = oi.product_id
    WHERE  o.customer_id = c.customer_id
      AND  p.category = 'Electronics'
      AND  oi.unit_price > 30000
);
```

### ตัวอย่างที่ 18: EXISTS with GROUP BY ใน Subquery

```sql
-- แผนกที่มีพนักงานเงินเดือนเฉลี่ยสูงกว่า 80,000
SELECT d.department_name
FROM   departments d
WHERE  EXISTS (
    SELECT 1
    FROM   employees e
    WHERE  e.department_id = d.department_id
    GROUP  BY e.department_id
    HAVING AVG(e.salary) > 80000
);
```

### ตัวอย่างที่ 19: Nested EXISTS

```sql
-- ลูกค้าที่มีออเดอร์ที่มีสินค้าที่ stock เหลือน้อยกว่า 30 ชิ้น
SELECT c.first_name, c.last_name
FROM   customers c
WHERE  EXISTS (
    SELECT 1
    FROM   orders o
    WHERE  o.customer_id = c.customer_id
      AND  EXISTS (
          SELECT 1
          FROM   order_items oi
          JOIN   products p ON p.product_id = oi.product_id
          WHERE  oi.order_id = o.order_id
            AND  p.stock_qty < 30
      )
);
```

### ตัวอย่างที่ 20: EXISTS กับ Correlated Date

```sql
-- ลูกค้าที่มีออเดอร์ในทุกเดือนของปี 2024 ที่มีข้อมูล
SELECT c.first_name, c.last_name
FROM   customers c
WHERE  (SELECT COUNT(DISTINCT MONTH(order_date))
        FROM   orders
        WHERE  customer_id = c.customer_id
          AND  YEAR(order_date) = 2024)
       = (SELECT COUNT(DISTINCT MONTH(order_date))
          FROM   orders
          WHERE  YEAR(order_date) = 2024);
```

---

## 46.8 Real-world Use Cases

### ตัวอย่างที่ 21: Customer Re-engagement

```sql
-- ลูกค้าที่เคยซื้อแต่ไม่ได้ซื้อใน 60 วันล่าสุด
SELECT c.first_name, c.last_name, c.email,
       (SELECT MAX(order_date) FROM orders WHERE customer_id = c.customer_id) AS last_order
FROM   customers c
WHERE  EXISTS (
    SELECT 1 FROM orders WHERE customer_id = c.customer_id
)
  AND  NOT EXISTS (
    SELECT 1
    FROM   orders o
    WHERE  o.customer_id = c.customer_id
      AND  o.order_date >= DATE_SUB(CURDATE(), INTERVAL 60 DAY)
);
```

### ตัวอย่างที่ 22: Product Stock Alert

```sql
-- สินค้าที่มี stock ต่ำและยังคงขายอยู่
SELECT p.product_name, p.category, p.stock_qty
FROM   products p
WHERE  p.stock_qty < 50
  AND  EXISTS (
    SELECT 1
    FROM   order_items oi
    JOIN   orders o ON o.order_id = oi.order_id
    WHERE  oi.product_id = p.product_id
      AND  o.order_date >= DATE_SUB(CURDATE(), INTERVAL 30 DAY)
  );
```

### ตัวอย่างที่ 23: Department with All Salary Ranges

```sql
-- แผนกที่มีพนักงานทั้งที่เงินเดือนสูงและต่ำ
SELECT d.department_name
FROM   departments d
WHERE  EXISTS (
    SELECT 1 FROM employees
    WHERE department_id = d.department_id AND salary > 85000
)
  AND  EXISTS (
    SELECT 1 FROM employees
    WHERE department_id = d.department_id AND salary < 75000
);
```

### ตัวอย่างที่ 24: Orphan Records Detection

```sql
-- ตรวจหา orphan records: order_items ที่ไม่มี order อ้างอิง
SELECT oi.item_id, oi.order_id, oi.product_id
FROM   order_items oi
WHERE  NOT EXISTS (
    SELECT 1 FROM orders o WHERE o.order_id = oi.order_id
);
```

### ตัวอย่างที่ 25: Products Sold in Every Category

```sql
-- ลูกค้าที่ซื้อสินค้าครบทุก category ที่มี
SELECT c.first_name, c.last_name
FROM   customers c
WHERE  NOT EXISTS (
    -- หา category ที่ลูกค้ายังไม่เคยซื้อ
    SELECT DISTINCT p.category
    FROM   products p
    WHERE  NOT EXISTS (
        SELECT 1
        FROM   orders o
        JOIN   order_items oi ON oi.order_id = o.order_id
        JOIN   products p2    ON p2.product_id = oi.product_id
        WHERE  o.customer_id = c.customer_id
          AND  p2.category = p.category
    )
);
```

---

## 46.9 EXISTS กับ DML (UPDATE, DELETE)

### ตัวอย่างที่ 26: UPDATE กับ EXISTS

```sql
-- เพิ่มส่วนลด 10% สำหรับสินค้าที่ขายดี
UPDATE products p
SET    price = price * 0.9
WHERE  EXISTS (
    SELECT 1
    FROM   order_items oi
    WHERE  oi.product_id = p.product_id
    GROUP  BY oi.product_id
    HAVING SUM(oi.quantity) > 3
);
```

### ตัวอย่างที่ 27: DELETE กับ NOT EXISTS

```sql
-- ลบออเดอร์ที่ไม่มี order_items
DELETE FROM orders
WHERE NOT EXISTS (
    SELECT 1
    FROM   order_items
    WHERE  order_id = orders.order_id
);
```

---

## 46.10 Performance Analysis

### ตัวอย่างที่ 28: Short-circuit Evaluation

```sql
-- EXISTS หยุดทันทีเมื่อเจอ match แรก
-- สำคัญมากเมื่อตาราง inner มีข้อมูลมาก

-- ตัวอย่าง: ถ้า order_items มี 10 ล้านแถว
-- EXISTS: ดึงแค่แถวแรกที่ match แล้วหยุด
-- IN: ดึงทุก product_id (อาจเป็น 100,000 ค่า)

-- EXISTS เหมาะกับ:
-- "มีอยู่หรือไม่?" - ไม่ต้องการ count หรือ value
```

### ตัวอย่างที่ 29: EXPLAIN เพื่อเปรียบเทียบ

```sql
-- Plan A: EXISTS
EXPLAIN SELECT c.customer_id
FROM customers c
WHERE EXISTS (SELECT 1 FROM orders WHERE customer_id = c.customer_id);

-- Plan B: IN
EXPLAIN SELECT c.customer_id
FROM customers c
WHERE customer_id IN (SELECT DISTINCT customer_id FROM orders);

-- Plan C: JOIN
EXPLAIN SELECT DISTINCT c.customer_id
FROM customers c
JOIN orders o ON o.customer_id = c.customer_id;
-- ดู type, possible_keys, rows ใน EXPLAIN
```

### ตัวอย่างที่ 30: EXISTS กับ Index

```sql
-- EXISTS ใช้ประโยชน์จาก index อย่างมีประสิทธิภาพ
-- สร้าง index บน foreign key
CREATE INDEX idx_orders_customer ON orders(customer_id);
CREATE INDEX idx_oi_product ON order_items(product_id);

-- EXISTS จะใช้ index เหล่านี้ได้ดี:
SELECT p.product_name FROM products p
WHERE EXISTS (
    SELECT 1 FROM order_items oi  -- ← idx_oi_product ถูกใช้
    WHERE oi.product_id = p.product_id
);
```

---

## แบบฝึกหัดบทที่ 46

**ข้อ 1:** ใช้ EXISTS หาสินค้าที่เคยถูกสั่งซื้อในปี 2024

```sql
-- เฉลย:
SELECT product_name, category, price
FROM   products p
WHERE  EXISTS (
    SELECT 1
    FROM   order_items oi
    JOIN   orders o ON o.order_id = oi.order_id
    WHERE  oi.product_id = p.product_id
      AND  YEAR(o.order_date) = 2024
)
ORDER BY category, product_name;
```

**ข้อ 2:** ใช้ NOT EXISTS หาลูกค้าที่ไม่มีออเดอร์ที่ status = 'completed'

```sql
-- เฉลย:
SELECT first_name, last_name, email
FROM   customers c
WHERE  NOT EXISTS (
    SELECT 1
    FROM   orders o
    WHERE  o.customer_id = c.customer_id
      AND  o.status = 'completed'
);
```

**ข้อ 3:** อธิบายและแก้ไขว่าทำไม NOT IN ถึงอาจให้ผลผิด และ NOT EXISTS ถูกต้องกว่า

```sql
-- เฉลย:
-- NOT IN ปัญหา: ถ้า subquery มี NULL, ผลลัพธ์ทั้งหมดจะ empty
-- NOT EXISTS: จัดการ NULL ได้ถูกต้องเพราะ correlated check

-- Demonstrate:
-- ถ้า orders มี customer_id = NULL:
-- NOT IN → empty (NULL trap)
-- NOT EXISTS → ถูกต้อง

SELECT c.first_name FROM customers c
WHERE NOT EXISTS (
    SELECT 1 FROM orders o WHERE o.customer_id = c.customer_id
);
```

**ข้อ 4:** ใช้ nested EXISTS หาลูกค้าที่มีออเดอร์ที่มีสินค้า Electronics

```sql
-- เฉลย:
SELECT c.first_name, c.last_name
FROM   customers c
WHERE  EXISTS (
    SELECT 1
    FROM   orders o
    WHERE  o.customer_id = c.customer_id
      AND  EXISTS (
          SELECT 1
          FROM   order_items oi
          JOIN   products p ON p.product_id = oi.product_id
          WHERE  oi.order_id = o.order_id
            AND  p.category = 'Electronics'
      )
);
```

**ข้อ 5:** เปรียบเทียบ EXISTS vs IN สำหรับการหาสินค้าที่มีคนสั่ง

```sql
-- เฉลย:

-- EXISTS:
SELECT product_name FROM products p
WHERE EXISTS (SELECT 1 FROM order_items WHERE product_id = p.product_id);

-- IN:
SELECT product_name FROM products
WHERE product_id IN (SELECT DISTINCT product_id FROM order_items);

-- ทั้งคู่ให้ผลเหมือนกัน
-- EXISTS: เร็วกว่าถ้า order_items ใหญ่มาก (short-circuit)
-- IN: เร็วกว่าถ้า subquery เล็ก
```

**ข้อ 6:** ใช้ EXISTS หาแผนกที่มีพนักงานเงินเดือนสูงกว่า 90,000

```sql
-- เฉลย:
SELECT d.department_name, d.budget
FROM   departments d
WHERE  EXISTS (
    SELECT 1
    FROM   employees e
    WHERE  e.department_id = d.department_id
      AND  e.salary > 90000
);
```

**ข้อ 7:** ใช้ NOT EXISTS หา products ที่ไม่เคยขายในเดือนพฤษภาคม 2024

```sql
-- เฉลย:
SELECT product_name, category, price
FROM   products p
WHERE  NOT EXISTS (
    SELECT 1
    FROM   order_items oi
    JOIN   orders o ON o.order_id = oi.order_id
    WHERE  oi.product_id = p.product_id
      AND  YEAR(o.order_date) = 2024
      AND  MONTH(o.order_date) = 5
);
```

**ข้อ 8:** ใช้ EXISTS และ NOT EXISTS ร่วมกัน หาลูกค้าที่เคยซื้อแต่ไม่ได้ซื้อ 2 เดือนล่าสุด

```sql
-- เฉลย:
SELECT c.first_name, c.last_name, c.email
FROM   customers c
WHERE  EXISTS (
    SELECT 1 FROM orders WHERE customer_id = c.customer_id
)
  AND  NOT EXISTS (
    SELECT 1
    FROM   orders o
    WHERE  o.customer_id = c.customer_id
      AND  o.order_date >= DATE_SUB(CURDATE(), INTERVAL 60 DAY)
);
```

**ข้อ 9:** ใช้ EXISTS กับ UPDATE เพื่ออัพเดต stock ของสินค้าที่ยังขายได้

```sql
-- เฉลย:
-- สินค้าที่เคยขายได้ใน 30 วันล่าสุด เพิ่ม stock 10 หน่วย
UPDATE products p
SET    stock_qty = stock_qty + 10
WHERE  EXISTS (
    SELECT 1
    FROM   order_items oi
    JOIN   orders o ON o.order_id = oi.order_id
    WHERE  oi.product_id = p.product_id
      AND  o.order_date >= DATE_SUB(CURDATE(), INTERVAL 30 DAY)
);
```

**ข้อ 10:** เขียน query ซับซ้อน: หาลูกค้าที่ซื้อสินค้าครบทุกหมวดหมู่

```sql
-- เฉลย (Relational Division):
SELECT c.first_name, c.last_name
FROM   customers c
WHERE  NOT EXISTS (
    -- หา category ที่ลูกค้ายังไม่ได้ซื้อ
    SELECT DISTINCT p.category
    FROM   products p
    WHERE  NOT EXISTS (
        -- ตรวจว่าลูกค้าเคยซื้อ category นี้หรือไม่
        SELECT 1
        FROM   orders o
        JOIN   order_items oi ON oi.order_id = o.order_id
        JOIN   products p2    ON p2.product_id = oi.product_id
        WHERE  o.customer_id = c.customer_id
          AND  p2.category = p.category
    )
);
```

---

*จบบทที่ 46: EXISTS and NOT EXISTS*
*บทถัดไป: Part 47 - ALL and ANY / SOME Operators*
