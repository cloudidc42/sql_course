# Part 43: Multi-row Subqueries with IN and NOT IN

## 43.1 Multi-row Subquery คืออะไร?

**Multi-row Subquery** คือ subquery ที่คืนหลายแถว ไม่สามารถใช้กับ `=`, `>`, `<` ได้โดยตรง ต้องใช้กับ operators พิเศษ เช่น `IN`, `NOT IN`, `ANY`, `ALL`, `EXISTS`

```
Multi-row Operators:
├── IN       - ค่าต้องอยู่ในรายการที่คืนมา
├── NOT IN   - ค่าต้องไม่อยู่ในรายการที่คืนมา  ⚠️ NULL trap!
├── ANY      - ค่าต้องตรงกับอย่างน้อย 1 ค่า
├── ALL      - ค่าต้องตรงกับทุกค่า
└── EXISTS   - ตรวจว่ามีแถวคืนมาหรือไม่
```

---

## 43.2 IN กับ Subquery

### ตัวอย่างที่ 1: พื้นฐาน IN

```sql
-- ลูกค้าที่มีออเดอร์
SELECT customer_id, first_name, last_name, email
FROM   customers
WHERE  customer_id IN (
    SELECT DISTINCT customer_id
    FROM   orders
);
```

### ตัวอย่างที่ 2: IN กับ ค่าที่ต้องการ

```sql
-- สินค้าที่เคยถูกสั่งซื้อในออเดอร์ที่สำเร็จแล้ว
SELECT product_name, category, price
FROM   products
WHERE  product_id IN (
    SELECT DISTINCT oi.product_id
    FROM   order_items oi
    JOIN   orders o ON o.order_id = oi.order_id
    WHERE  o.status = 'completed'
)
ORDER BY category, price;
```

### ตัวอย่างที่ 3: IN กับ aggregate subquery

```sql
-- พนักงานที่อยู่ในแผนกที่มีเงินเดือนเฉลี่ยสูงกว่า 80,000
SELECT first_name, last_name, salary, department_id
FROM   employees
WHERE  department_id IN (
    SELECT department_id
    FROM   employees
    GROUP  BY department_id
    HAVING AVG(salary) > 80000
);
```

### ตัวอย่างที่ 4: IN กับหลายระดับ

```sql
-- ลูกค้าที่ซื้อสินค้า Electronics ราคา > 20,000
SELECT DISTINCT c.first_name, c.last_name, c.city
FROM   customers c
WHERE  c.customer_id IN (
    SELECT o.customer_id
    FROM   orders o
    WHERE  o.order_id IN (
        SELECT oi.order_id
        FROM   order_items oi
        WHERE  oi.product_id IN (
            SELECT product_id
            FROM   products
            WHERE  category = 'Electronics'
              AND  price > 20000
        )
    )
);
```

### ตัวอย่างที่ 5: IN กับ string values

```sql
-- พนักงานที่อยู่ในแผนกที่ตั้งอยู่ใน Bangkok
SELECT first_name, last_name, department_id
FROM   employees
WHERE  department_id IN (
    SELECT department_id
    FROM   departments
    WHERE  location = 'Bangkok'
);
```

### ตัวอย่างที่ 6: IN กับหลายคอลัมน์ (Row Subquery)

```sql
-- MySQL: IN กับ row subquery
SELECT order_id, customer_id, order_date
FROM   orders
WHERE  (customer_id, YEAR(order_date)) IN (
    SELECT customer_id, YEAR(MAX(order_date))
    FROM   orders
    GROUP  BY customer_id
);
```

### ตัวอย่างที่ 7: IN เทียบกับ JOIN

```sql
-- แบบ IN (subquery)
SELECT p.product_name, p.price
FROM   products p
WHERE  p.product_id IN (
    SELECT product_id FROM order_items
);

-- แบบ JOIN (มักเร็วกว่า)
SELECT DISTINCT p.product_name, p.price
FROM   products p
JOIN   order_items oi ON oi.product_id = p.product_id;

-- แบบ EXISTS (บางกรณีเร็วที่สุด)
SELECT p.product_name, p.price
FROM   products p
WHERE  EXISTS (
    SELECT 1 FROM order_items WHERE product_id = p.product_id
);
```

---

## 43.3 NOT IN กับ Subquery

### ตัวอย่างที่ 8: NOT IN พื้นฐาน

```sql
-- ลูกค้าที่ไม่เคยสั่งซื้อ
SELECT customer_id, first_name, last_name, email
FROM   customers
WHERE  customer_id NOT IN (
    SELECT DISTINCT customer_id FROM orders
);
```

### ตัวอย่างที่ 9: สินค้าที่ไม่เคยขาย

```sql
-- สินค้าที่ยังไม่เคยมีออเดอร์
SELECT product_name, category, price, stock_qty
FROM   products
WHERE  product_id NOT IN (
    SELECT DISTINCT product_id FROM order_items
);
```

### ตัวอย่างที่ 10: พนักงานที่ไม่ได้เป็นหัวหน้าใคร

```sql
-- พนักงานที่ไม่มีลูกน้อง (ไม่ปรากฏใน manager_id)
SELECT first_name, last_name, employee_id
FROM   employees
WHERE  employee_id NOT IN (
    SELECT DISTINCT manager_id
    FROM   employees
    WHERE  manager_id IS NOT NULL  -- ⚠️ ต้องกรอง NULL ออก!
);
```

### ตัวอย่างที่ 11: แผนกที่ไม่มีพนักงาน

```sql
-- แผนกที่ยังไม่มีพนักงาน
SELECT department_id, department_name
FROM   departments
WHERE  department_id NOT IN (
    SELECT DISTINCT department_id
    FROM   employees
    WHERE  department_id IS NOT NULL  -- ⚠️ จำเป็นมาก!
);
```

---

## 43.4 ⚠️ NULL Trap กับ NOT IN (แนวคิดสำคัญมาก!)

### ทำความเข้าใจ NULL ใน SQL

```sql
-- NULL ใน SQL: UNKNOWN ≠ FALSE ≠ TRUE
-- การเปรียบเทียบใดๆ กับ NULL = UNKNOWN (ไม่ใช่ TRUE หรือ FALSE)

SELECT NULL = NULL;    -- NULL (UNKNOWN)
SELECT NULL <> NULL;   -- NULL (UNKNOWN)
SELECT NULL = 1;       -- NULL (UNKNOWN)
SELECT 1 = 1;          -- TRUE
```

### ตัวอย่างที่ 12: NULL Trap - กรณีที่ทำให้ไม่ได้ผลลัพธ์

```sql
-- สร้างข้อมูลสำหรับทดสอบ NULL trap
-- สมมติ employees มี manager_id = NULL สำหรับหัวหน้าสูงสุด

-- ⚠️ DANGEROUS: NOT IN กับ subquery ที่มี NULL
SELECT first_name, last_name
FROM   employees
WHERE  employee_id NOT IN (
    SELECT manager_id       -- ← มี NULL อยู่ใน manager_id!
    FROM   employees
);
-- ผลลัพธ์: ไม่มีแถวเลย! แม้ควรจะมี
```

### ตัวอย่างที่ 13: อธิบาย NULL Trap ทีละขั้น

```sql
-- ลองเปรียบเทียบด้วยตาเปล่า:
-- manager_id = {1, 3, 5, NULL, NULL, ...}

-- NOT IN ทำงานแบบนี้:
-- employee_id NOT IN (1, 3, 5, NULL)
-- หมายความว่า: employee_id <> 1 AND employee_id <> 3 AND employee_id <> 5 AND employee_id <> NULL

-- แต่: employee_id <> NULL = UNKNOWN เสมอ!
-- AND UNKNOWN = UNKNOWN (ไม่ใช่ TRUE)
-- ดังนั้น: ไม่มีแถวไหนผ่านเงื่อนไขเลย!

-- ทดสอบ:
SELECT 1 NOT IN (2, 3, NULL);          -- NULL (ไม่ใช่ TRUE!)
SELECT 1 NOT IN (2, 3);               -- TRUE (ไม่มี NULL = ปลอดภัย)
SELECT 1 NOT IN (1, 2, NULL);         -- FALSE
```

### ตัวอย่างที่ 14: พิสูจน์ NULL Trap

```sql
-- ทดสอบกับข้อมูลจริง:

-- กี่แถวที่ manager_id เป็น NULL?
SELECT COUNT(*) FROM employees WHERE manager_id IS NULL;
-- ผลลัพธ์: 4 (Somchai, Prasert, Nattapol, Pimchanok, Theerawat)

-- ⚠️ Query นี้จะคืน 0 แถว เพราะมี NULL:
SELECT COUNT(*) FROM employees
WHERE employee_id NOT IN (
    SELECT manager_id FROM employees
);
-- ผลลัพธ์: 0 แถว ← ผิดพลาด!

-- ✓ Query ที่ถูกต้อง - กรอง NULL ออก:
SELECT COUNT(*) FROM employees
WHERE employee_id NOT IN (
    SELECT manager_id FROM employees WHERE manager_id IS NOT NULL
);
-- ผลลัพธ์: จำนวนที่ถูกต้อง
```

### ตัวอย่างที่ 15: การแก้ไข NULL Trap วิธีที่ 1 - WHERE IS NOT NULL

```sql
-- วิธีที่ 1: กรอง NULL ออกจาก subquery
SELECT first_name, last_name, employee_id
FROM   employees
WHERE  employee_id NOT IN (
    SELECT manager_id
    FROM   employees
    WHERE  manager_id IS NOT NULL  -- ← กรอง NULL ออก
);
```

### ตัวอย่างที่ 16: การแก้ไข NULL Trap วิธีที่ 2 - NOT EXISTS

```sql
-- วิธีที่ 2: ใช้ NOT EXISTS แทน NOT IN (ปลอดภัยกว่า)
SELECT first_name, last_name, employee_id
FROM   employees e1
WHERE  NOT EXISTS (
    SELECT 1
    FROM   employees e2
    WHERE  e2.manager_id = e1.employee_id
);
-- NOT EXISTS จัดการ NULL ได้อัตโนมัติ
```

### ตัวอย่างที่ 17: การแก้ไข NULL Trap วิธีที่ 3 - LEFT JOIN / IS NULL

```sql
-- วิธีที่ 3: LEFT JOIN + IS NULL
SELECT e1.first_name, e1.last_name, e1.employee_id
FROM   employees e1
LEFT   JOIN employees e2 ON e2.manager_id = e1.employee_id
WHERE  e2.manager_id IS NULL;
```

### ตัวอย่างที่ 18: NULL Trap กับตัวอย่าง customers

```sql
-- สมมติ customers บางคนมี city = NULL

-- ⚠️ อันตราย: ถ้า Bangkok customers มี NULL ใน customer_id (ไม่ควรเกิด แต่ถ้าเกิด...)
-- ใช้ NOT IN ระวัง

-- ปลอดภัย: ใช้ NOT EXISTS
SELECT c.first_name, c.last_name
FROM   customers c
WHERE  NOT EXISTS (
    SELECT 1
    FROM   orders o
    WHERE  o.customer_id = c.customer_id
);

-- เทียบกับ (อาจเป็น NULL trap ถ้า customer_id มี NULL):
SELECT first_name, last_name
FROM   customers
WHERE  customer_id NOT IN (
    SELECT customer_id FROM orders  -- ถ้า customer_id ใน orders มี NULL = ปัญหา!
);
```

---

## 43.5 ตัวอย่างเพิ่มเติมสำหรับ IN

### ตัวอย่างที่ 19: IN กับ subquery ที่มี DISTINCT

```sql
-- หมวดหมู่ที่มีสินค้าราคาแพง (> 10,000)
SELECT product_name, category, price
FROM   products
WHERE  category IN (
    SELECT DISTINCT category
    FROM   products
    WHERE  price > 10000
);
```

### ตัวอย่างที่ 20: IN กับ date range subquery

```sql
-- ลูกค้าที่ทำออเดอร์ในเดือนมกราคม 2024
SELECT first_name, last_name, email
FROM   customers
WHERE  customer_id IN (
    SELECT DISTINCT customer_id
    FROM   orders
    WHERE  order_date BETWEEN '2024-01-01' AND '2024-01-31'
);
```

### ตัวอย่างที่ 21: IN กับ HAVING subquery

```sql
-- ลูกค้าที่มีออเดอร์เฉลี่ยสูงกว่า 15,000
SELECT first_name, last_name
FROM   customers
WHERE  customer_id IN (
    SELECT customer_id
    FROM   orders
    GROUP  BY customer_id
    HAVING AVG(total_amount) > 15000
);
```

### ตัวอย่างที่ 22: IN กับ correlated subquery ที่ nested

```sql
-- พนักงานที่อยู่ในแผนกที่มีพนักงานเงินเดือนสูงกว่า 90,000
SELECT first_name, last_name, salary, department_id
FROM   employees
WHERE  department_id IN (
    SELECT DISTINCT department_id
    FROM   employees
    WHERE  salary > 90000
);
```

---

## 43.6 ตัวอย่างเพิ่มเติมสำหรับ NOT IN

### ตัวอย่างที่ 23: NOT IN สำหรับ Anti-join

```sql
-- สินค้าที่ไม่เคยอยู่ในออเดอร์ที่ pending
SELECT product_name, category, price
FROM   products
WHERE  product_id NOT IN (
    SELECT DISTINCT oi.product_id
    FROM   order_items oi
    JOIN   orders o ON o.order_id = oi.order_id
    WHERE  o.status = 'pending'
);
```

### ตัวอย่างที่ 24: NOT IN สำหรับ Gap Analysis

```sql
-- แผนกที่ไม่มีพนักงานเงินเดือนสูงกว่า 80,000
SELECT department_name
FROM   departments
WHERE  department_id NOT IN (
    SELECT DISTINCT department_id
    FROM   employees
    WHERE  salary > 80000
      AND  department_id IS NOT NULL
);
```

### ตัวอย่างที่ 25: NOT IN เพื่อหา New Records

```sql
-- ลูกค้าที่สมัครปีนี้แต่ยังไม่เคยสั่งซื้อ
SELECT first_name, last_name, created_at
FROM   customers
WHERE  YEAR(created_at) = 2023
  AND  customer_id NOT IN (
    SELECT DISTINCT customer_id
    FROM   orders
    WHERE  customer_id IS NOT NULL  -- ⚠️ กรอง NULL เสมอ
);
```

---

## 43.7 IN vs EXISTS Performance

### ตัวอย่างที่ 26: เปรียบเทียบ IN vs EXISTS

```sql
-- ทั้งสองให้ผลเหมือนกัน แต่ประสิทธิภาพต่างกัน

-- แบบ IN:
SELECT p.product_name
FROM   products p
WHERE  p.product_id IN (
    SELECT product_id FROM order_items
);

-- แบบ EXISTS:
SELECT p.product_name
FROM   products p
WHERE  EXISTS (
    SELECT 1
    FROM   order_items oi
    WHERE  oi.product_id = p.product_id
);

-- กฎทั่วไป:
-- IN  = ดีกว่าเมื่อ subquery เล็ก
-- EXISTS = ดีกว่าเมื่อ outer table เล็กกว่า subquery table
```

### ตัวอย่างที่ 27: NOT IN vs NOT EXISTS (NULL handling)

```sql
-- NOT IN: อันตรายเรื่อง NULL
-- NOT EXISTS: ปลอดภัยกว่า

-- ⚠️ NOT IN อาจให้ผลผิดถ้ามี NULL:
SELECT e1.first_name FROM employees e1
WHERE e1.employee_id NOT IN (
    SELECT manager_id FROM employees  -- มี NULL!
);

-- ✓ NOT EXISTS: ปลอดภัย
SELECT e1.first_name FROM employees e1
WHERE NOT EXISTS (
    SELECT 1 FROM employees e2
    WHERE e2.manager_id = e1.employee_id
);
```

---

## 43.8 Rewriting NOT IN เพื่อหลีกเลี่ยง NULL

### ตัวอย่างที่ 28: ทุกวิธีให้ผลเหมือนกัน

```sql
-- โจทย์: หาลูกค้าที่ไม่เคยสั่งซื้อ

-- วิธี 1: NOT IN + WHERE IS NOT NULL
SELECT customer_id, first_name, last_name
FROM   customers
WHERE  customer_id NOT IN (
    SELECT customer_id FROM orders WHERE customer_id IS NOT NULL
);

-- วิธี 2: NOT EXISTS (แนะนำ)
SELECT c.customer_id, c.first_name, c.last_name
FROM   customers c
WHERE  NOT EXISTS (
    SELECT 1 FROM orders o WHERE o.customer_id = c.customer_id
);

-- วิธี 3: LEFT JOIN / IS NULL (เร็วที่สุดในหลาย DB)
SELECT c.customer_id, c.first_name, c.last_name
FROM   customers c
LEFT   JOIN orders o ON o.customer_id = c.customer_id
WHERE  o.order_id IS NULL;
```

### ตัวอย่างที่ 29: เปรียบเทียบผลลัพธ์ NULL trap

```sql
-- สร้างข้อมูลทดสอบ
CREATE TEMPORARY TABLE test_a (val INT);
CREATE TEMPORARY TABLE test_b (val INT);
INSERT INTO test_a VALUES (1), (2), (3), (4), (5);
INSERT INTO test_b VALUES (1), (2), (NULL);

-- NOT IN กับ NULL:
SELECT val FROM test_a WHERE val NOT IN (SELECT val FROM test_b);
-- ผลลัพธ์: ไม่มีแถวเลย! (เพราะ NULL ใน test_b)

-- NOT IN + กรอง NULL:
SELECT val FROM test_a WHERE val NOT IN (SELECT val FROM test_b WHERE val IS NOT NULL);
-- ผลลัพธ์: 3, 4, 5

-- NOT EXISTS:
SELECT val FROM test_a a WHERE NOT EXISTS (SELECT 1 FROM test_b b WHERE b.val = a.val);
-- ผลลัพธ์: 3, 4, 5

DROP TEMPORARY TABLE test_a, test_b;
```

### ตัวอย่างที่ 30: Real-world Example พร้อม NULL protection

```sql
-- หาพนักงานที่ไม่ได้เป็น manager ของใคร (ปลอดภัยจาก NULL)
-- วิธีที่ดีที่สุด:
SELECT
    e.employee_id,
    CONCAT(e.first_name, ' ', e.last_name) AS employee_name,
    e.salary
FROM employees e
WHERE NOT EXISTS (
    SELECT 1
    FROM   employees sub
    WHERE  sub.manager_id = e.employee_id
)
ORDER BY e.employee_id;

-- ผลลัพธ์: พนักงานที่ไม่มีลูกน้อง
```

---

## 43.9 Dynamic IN Lists

### ตัวอย่างที่ 31: IN กับ Dynamic condition

```sql
-- หาสินค้าในหมวดหมู่ที่มียอดขายรวมสูงกว่า 30,000
SELECT product_name, category, price
FROM   products
WHERE  category IN (
    SELECT p.category
    FROM   products p
    JOIN   order_items oi ON oi.product_id = p.product_id
    GROUP  BY p.category
    HAVING SUM(oi.quantity * oi.unit_price) > 30000
)
ORDER BY category, price;
```

### ตัวอย่างที่ 32: ซ้อน IN หลายชั้น

```sql
-- ลูกค้าที่ซื้อสินค้าในหมวดหมู่ที่ขายดีสุด
SELECT DISTINCT c.first_name, c.last_name
FROM   customers c
WHERE  c.customer_id IN (
    -- ลูกค้าที่มีออเดอร์ที่มีสินค้าจาก top category
    SELECT DISTINCT o.customer_id
    FROM   orders o
    WHERE  o.order_id IN (
        -- ออเดอร์ที่มีสินค้าจาก top category
        SELECT DISTINCT oi.order_id
        FROM   order_items oi
        WHERE  oi.product_id IN (
            -- สินค้าจาก top category (Electronics)
            SELECT product_id
            FROM   products
            WHERE  category = 'Electronics'
        )
    )
);
```

---

## 43.10 สรุป Best Practices

```sql
-- ✓ ใช้ IN เมื่อ: subquery ไม่มี NULL, ต้องการอ่านง่าย
-- ✓ ใช้ NOT IN เมื่อ: มั่นใจว่าไม่มี NULL ใน subquery result
-- ✓ ใช้ NOT EXISTS แทน NOT IN เสมอเมื่อไม่แน่ใจเรื่อง NULL
-- ✓ เพิ่ม WHERE col IS NOT NULL ใน subquery ของ NOT IN เสมอ

-- สรุป NULL trap:
-- val NOT IN (1, 2, NULL)      = UNKNOWN (false-like)
-- val NOT IN (1, 2)            = TRUE ถ้า val ≠ 1 และ ≠ 2
-- NOT EXISTS (... WHERE x = val) = ปลอดภัยจาก NULL
```

---

## แบบฝึกหัดบทที่ 43

**ข้อ 1:** ใช้ IN หาสินค้าที่ถูกสั่งซื้อในเดือนมีนาคม 2024

```sql
-- เฉลย:
SELECT product_name, category, price
FROM   products
WHERE  product_id IN (
    SELECT DISTINCT oi.product_id
    FROM   order_items oi
    JOIN   orders o ON o.order_id = oi.order_id
    WHERE  YEAR(o.order_date) = 2024
      AND  MONTH(o.order_date) = 3
);
```

**ข้อ 2:** ใช้ NOT IN หาสินค้าที่ไม่เคยขายได้ (ระวัง NULL)

```sql
-- เฉลย:
SELECT product_name, category, price, stock_qty
FROM   products
WHERE  product_id NOT IN (
    SELECT DISTINCT product_id
    FROM   order_items
    WHERE  product_id IS NOT NULL
);
```

**ข้อ 3:** อธิบาย NULL trap และแก้ไข query ที่มีปัญหา

```sql
-- Query ที่มีปัญหา:
SELECT first_name FROM employees
WHERE employee_id NOT IN (SELECT manager_id FROM employees);

-- เฉลย (แก้ไข):
-- ปัญหา: manager_id มีค่า NULL
-- วิธีแก้ 1:
SELECT first_name FROM employees
WHERE employee_id NOT IN (
    SELECT manager_id FROM employees WHERE manager_id IS NOT NULL
);
-- วิธีแก้ 2 (แนะนำ):
SELECT e1.first_name FROM employees e1
WHERE NOT EXISTS (
    SELECT 1 FROM employees e2
    WHERE e2.manager_id = e1.employee_id
);
```

**ข้อ 4:** ใช้ IN หาลูกค้าในเมือง Bangkok ที่เคยซื้อ Furniture

```sql
-- เฉลย:
SELECT first_name, last_name, city
FROM   customers
WHERE  city = 'Bangkok'
  AND  customer_id IN (
    SELECT DISTINCT o.customer_id
    FROM   orders o
    JOIN   order_items oi ON oi.order_id = o.order_id
    JOIN   products p     ON p.product_id = oi.product_id
    WHERE  p.category = 'Furniture'
);
```

**ข้อ 5:** ใช้ NOT IN หาแผนกที่ไม่มีพนักงานเงินเดือนสูงกว่า 85,000

```sql
-- เฉลย:
SELECT department_name, budget
FROM   departments
WHERE  department_id NOT IN (
    SELECT DISTINCT department_id
    FROM   employees
    WHERE  salary > 85000
      AND  department_id IS NOT NULL
);
```

**ข้อ 6:** เปรียบเทียบ NOT IN vs NOT EXISTS สำหรับลูกค้าที่ไม่มีออเดอร์

```sql
-- เฉลย NOT IN:
SELECT first_name, last_name FROM customers
WHERE customer_id NOT IN (
    SELECT customer_id FROM orders WHERE customer_id IS NOT NULL
);

-- เฉลย NOT EXISTS (แนะนำ):
SELECT c.first_name, c.last_name FROM customers c
WHERE NOT EXISTS (
    SELECT 1 FROM orders o WHERE o.customer_id = c.customer_id
);
```

**ข้อ 7:** ใช้ IN แบบซ้อนหลายชั้น หาลูกค้าที่เคยซื้อสินค้าราคาแพงกว่า 20,000

```sql
-- เฉลย:
SELECT first_name, last_name
FROM   customers
WHERE  customer_id IN (
    SELECT DISTINCT customer_id
    FROM   orders
    WHERE  order_id IN (
        SELECT DISTINCT order_id
        FROM   order_items
        WHERE  unit_price > 20000
    )
);
```

**ข้อ 8:** หาพนักงานที่อยู่ในแผนกที่มีงบประมาณสูงกว่าค่าเฉลี่ยงบประมาณทุกแผนก

```sql
-- เฉลย:
SELECT first_name, last_name, department_id
FROM   employees
WHERE  department_id IN (
    SELECT department_id
    FROM   departments
    WHERE  budget > (SELECT AVG(budget) FROM departments)
);
```

**ข้อ 9:** ทำไม query ต่อไปนี้ถึงไม่คืนผลลัพธ์? และแก้ไขอย่างไร?

```sql
-- Query:
SELECT first_name FROM employees
WHERE employee_id NOT IN (SELECT manager_id FROM employees);
```

```sql
-- เฉลย:
-- เหตุผล: manager_id ใน employees มีค่า NULL สำหรับหัวหน้าสูงสุด
-- เมื่อ NOT IN กับ list ที่มี NULL, ผลลัพธ์จะเป็น UNKNOWN สำหรับทุกแถว

-- แก้ไข:
SELECT first_name FROM employees
WHERE employee_id NOT IN (
    SELECT manager_id
    FROM   employees
    WHERE  manager_id IS NOT NULL
);
```

**ข้อ 10:** เขียน query เปรียบเทียบผลลัพธ์ของ NOT IN (มี NULL) vs NOT IN (กรอง NULL) vs NOT EXISTS

```sql
-- เฉลย:
-- 1. NOT IN ที่มี NULL (อันตราย)
SELECT COUNT(*) AS not_in_with_null
FROM employees
WHERE employee_id NOT IN (SELECT manager_id FROM employees);
-- ผล: 0 (ผิด!)

-- 2. NOT IN กรอง NULL (ถูก)
SELECT COUNT(*) AS not_in_no_null
FROM employees
WHERE employee_id NOT IN (
    SELECT manager_id FROM employees WHERE manager_id IS NOT NULL
);
-- ผล: จำนวนที่ถูกต้อง

-- 3. NOT EXISTS (ถูกและปลอดภัย)
SELECT COUNT(*) AS not_exists_result
FROM employees e1
WHERE NOT EXISTS (
    SELECT 1 FROM employees e2 WHERE e2.manager_id = e1.employee_id
);
-- ผล: จำนวนที่ถูกต้อง (เหมือนข้อ 2)
```

---

*จบบทที่ 43: Multi-row Subqueries with IN and NOT IN*
*บทถัดไป: Part 44 - Derived Tables - Subqueries in FROM*
