# Part 038: Window Functions Preview - Running Totals

## บทนำ (Introduction)

**Window Functions** (หรือ Analytic Functions) เป็นหนึ่งในฟีเจอร์ที่ทรงพลังที่สุดใน SQL สมัยใหม่ ต่างจาก Aggregate Functions ตรงที่:

- **Aggregate Functions** รวบรัดข้อมูลหลายแถวเป็น **1 แถว**
- **Window Functions** คำนวณบนกลุ่มแถว แต่ **ยังคงทุกแถวไว้**

ในบทนี้จะเรียน Window Functions เบื้องต้น (การศึกษาเชิงลึกอยู่ใน Part 91-95) เพื่อให้เข้าใจหลักการก่อน

---

## การตั้งค่าฐานข้อมูล

```sql
USE ecommerce_db;

-- ข้อมูลพร้อมใช้จาก Part 031-037
-- Window Functions ต้องการ MySQL 8.0+, PostgreSQL, SQL Server

-- ตรวจสอบ version (MySQL)
SELECT VERSION();
-- ต้องการ MySQL 8.0 ขึ้นไป หรือ MariaDB 10.2 ขึ้นไป
```

---

## 1. OVER() Clause พื้นฐาน

`OVER()` คือหัวใจของ Window Functions - กำหนด "หน้าต่าง" ของข้อมูลที่จะคำนวณ

```sql
-- รูปแบบ:
function_name() OVER (
    PARTITION BY column1, column2  -- แบ่งกลุ่ม (optional)
    ORDER BY column3               -- เรียงลำดับ (optional)
    ROWS BETWEEN ... AND ...       -- กำหนดขอบเขต (optional)
)
```

```sql
-- ตัวอย่างที่ 1: OVER() ง่ายๆ - Aggregate ที่ยังคงทุกแถว
SELECT 
    order_id,
    customer_id,
    total_amount,
    -- COUNT(*) ปกติ จะคืนค่าเดียว
    COUNT(*) OVER () AS total_order_count,
    -- SUM ปกติ จะคืนค่าเดียว
    SUM(total_amount) OVER () AS grand_total,
    -- แต่ละแถวยังคงอยู่!
    ROUND(total_amount * 100.0 / SUM(total_amount) OVER (), 2) AS pct_of_total
FROM orders
WHERE status NOT IN ('cancelled')
ORDER BY order_id;
```

```sql
-- ตัวอย่างที่ 2: เปรียบเทียบ Regular Aggregate vs Window Function
-- Regular Aggregate: คืน 1 แถว
SELECT COUNT(*), SUM(total_amount) FROM orders WHERE status != 'cancelled';

-- Window Function: คืนทุกแถว พร้อม aggregate ของทั้งหมด
SELECT 
    order_id,
    total_amount,
    COUNT(*) OVER () AS total_orders,
    SUM(total_amount) OVER () AS total_revenue
FROM orders
WHERE status != 'cancelled';
```

---

## 2. PARTITION BY

`PARTITION BY` แบ่งข้อมูลเป็นกลุ่มๆ คล้าย GROUP BY แต่ยังคงทุกแถว

```sql
-- ตัวอย่างที่ 3: PARTITION BY - คำนวณต่อกลุ่ม
SELECT 
    customer_id,
    order_id,
    order_date,
    total_amount,
    -- ยอดรวมของลูกค้าคนนั้น (ไม่ใช่ทั้งหมด)
    SUM(total_amount) OVER (PARTITION BY customer_id) AS customer_total,
    -- เปอร์เซ็นต์ของออเดอร์นี้ต่อยอดรวมของลูกค้าคนนั้น
    ROUND(total_amount * 100.0 / SUM(total_amount) OVER (PARTITION BY customer_id), 2) AS pct_of_customer_total
FROM orders
WHERE status NOT IN ('cancelled')
ORDER BY customer_id, order_date;
```

```sql
-- ตัวอย่างที่ 4: PARTITION BY กับ COUNT
SELECT 
    product_id,
    order_id,
    quantity,
    unit_price,
    -- จำนวนครั้งที่สินค้านี้ถูกสั่งซื้อ
    COUNT(*) OVER (PARTITION BY product_id) AS times_ordered,
    -- ยอดขายรวมของสินค้านี้
    SUM(quantity * unit_price) OVER (PARTITION BY product_id) AS product_revenue
FROM order_items
ORDER BY product_id, order_id;
```

```sql
-- ตัวอย่างที่ 5: PARTITION BY กับ AVG (เปรียบเทียบกับค่าเฉลี่ย)
SELECT 
    e.employee_id,
    CONCAT(e.first_name, ' ', e.last_name) AS employee_name,
    d.department_name,
    e.salary,
    -- เงินเดือนเฉลี่ยของแผนก
    ROUND(AVG(e.salary) OVER (PARTITION BY e.department_id), 0) AS dept_avg_salary,
    -- ส่วนต่างจากค่าเฉลี่ยแผนก
    e.salary - ROUND(AVG(e.salary) OVER (PARTITION BY e.department_id), 0) AS diff_from_avg,
    -- เปรียบเทียบกับค่าเฉลี่ยทั้งบริษัท
    ROUND(AVG(e.salary) OVER (), 0) AS company_avg_salary
FROM employees e
JOIN departments d ON e.department_id = d.department_id
WHERE e.salary IS NOT NULL
ORDER BY e.department_id, e.salary DESC;
```

---

## 3. ORDER BY ใน Window Functions

`ORDER BY` ใน OVER() ช่วยสร้าง Running Calculations

```sql
-- ตัวอย่างที่ 6: Running Total ด้วย SUM OVER ORDER BY
SELECT 
    order_id,
    order_date,
    total_amount,
    -- Running Total: ยอดสะสมตามลำดับเวลา
    SUM(total_amount) OVER (ORDER BY order_date, order_id) AS running_total
FROM orders
WHERE status NOT IN ('cancelled')
ORDER BY order_date, order_id;
```

```sql
-- ตัวอย่างที่ 7: Running Count
SELECT 
    order_id,
    order_date,
    customer_id,
    total_amount,
    -- นับออเดอร์สะสม
    COUNT(*) OVER (ORDER BY order_date, order_id) AS running_order_count
FROM orders
WHERE status NOT IN ('cancelled')
ORDER BY order_date;
```

---

## 4. Running Total ด้วย SUM OVER

```sql
-- ตัวอย่างที่ 8: Running Total รายวัน
SELECT 
    DATE(order_date) AS order_day,
    COUNT(*) AS daily_orders,
    SUM(total_amount) AS daily_revenue,
    -- Running Total ของยอดขาย
    SUM(SUM(total_amount)) OVER (ORDER BY DATE(order_date)) AS running_revenue,
    -- Running Total ของจำนวนออเดอร์
    SUM(COUNT(*)) OVER (ORDER BY DATE(order_date)) AS running_order_count
FROM orders
WHERE status NOT IN ('cancelled')
GROUP BY DATE(order_date)
ORDER BY order_day;
```

```sql
-- ตัวอย่างที่ 9: Running Total แยกตาม Customer
SELECT 
    customer_id,
    order_id,
    order_date,
    total_amount,
    -- Running total สำหรับลูกค้าแต่ละคน
    SUM(total_amount) OVER (
        PARTITION BY customer_id 
        ORDER BY order_date, order_id
    ) AS customer_running_total,
    -- Running order count ต่อลูกค้า
    COUNT(*) OVER (
        PARTITION BY customer_id 
        ORDER BY order_date, order_id
    ) AS customer_order_number
FROM orders
WHERE status NOT IN ('cancelled')
ORDER BY customer_id, order_date;
```

```sql
-- ตัวอย่างที่ 10: Monthly Running Total
SELECT 
    DATE_FORMAT(order_date, '%Y-%m') AS month,
    SUM(total_amount) AS monthly_revenue,
    SUM(SUM(total_amount)) OVER (
        ORDER BY DATE_FORMAT(order_date, '%Y-%m')
    ) AS cumulative_revenue,
    -- เปอร์เซ็นต์ของ Grand Total
    ROUND(
        SUM(SUM(total_amount)) OVER (
            ORDER BY DATE_FORMAT(order_date, '%Y-%m')
        ) * 100.0 / SUM(SUM(total_amount)) OVER (),
        2
    ) AS pct_of_total
FROM orders
WHERE status NOT IN ('cancelled')
GROUP BY DATE_FORMAT(order_date, '%Y-%m')
ORDER BY month;
```

---

## 5. Moving Average

Moving Average (ค่าเฉลี่ยเคลื่อนที่) ช่วย Smooth ข้อมูลที่ volatile

```sql
-- ตัวอย่างที่ 11: 3-Month Moving Average
SELECT 
    DATE_FORMAT(order_date, '%Y-%m') AS month,
    SUM(total_amount) AS monthly_revenue,
    -- 3-month moving average
    ROUND(
        AVG(SUM(total_amount)) OVER (
            ORDER BY DATE_FORMAT(order_date, '%Y-%m')
            ROWS BETWEEN 2 PRECEDING AND CURRENT ROW  -- 3 months window
        ),
        0
    ) AS moving_avg_3m
FROM orders
WHERE status NOT IN ('cancelled')
GROUP BY DATE_FORMAT(order_date, '%Y-%m')
ORDER BY month;
```

```sql
-- ตัวอย่างที่ 12: Centered Moving Average
SELECT 
    DATE_FORMAT(order_date, '%Y-%m') AS month,
    SUM(total_amount) AS monthly_revenue,
    -- Centered 3-month moving average (1 before, current, 1 after)
    ROUND(
        AVG(SUM(total_amount)) OVER (
            ORDER BY DATE_FORMAT(order_date, '%Y-%m')
            ROWS BETWEEN 1 PRECEDING AND 1 FOLLOWING
        ),
        0
    ) AS centered_avg_3m
FROM orders
WHERE status NOT IN ('cancelled')
GROUP BY DATE_FORMAT(order_date, '%Y-%m')
ORDER BY month;
```

---

## 6. Ranking Functions เบื้องต้น

```sql
-- ตัวอย่างที่ 13: ROW_NUMBER - เลขแถวต่อเนื่อง
SELECT 
    ROW_NUMBER() OVER (ORDER BY total_amount DESC) AS row_num,
    order_id,
    customer_id,
    total_amount,
    status
FROM orders
WHERE status NOT IN ('cancelled')
ORDER BY total_amount DESC;
```

```sql
-- ตัวอย่างที่ 14: RANK - จัดอันดับ (มี gap เมื่อ tie)
SELECT 
    RANK() OVER (ORDER BY salary DESC) AS salary_rank,
    employee_id,
    first_name,
    last_name,
    salary
FROM employees
WHERE salary IS NOT NULL
ORDER BY salary DESC;
```

```sql
-- ตัวอย่างที่ 15: DENSE_RANK - จัดอันดับ (ไม่มี gap)
SELECT 
    DENSE_RANK() OVER (ORDER BY salary DESC) AS salary_rank,
    employee_id,
    first_name,
    last_name,
    department_id,
    salary
FROM employees
WHERE salary IS NOT NULL
ORDER BY salary DESC;
```

```sql
-- ตัวอย่างที่ 16: เปรียบเทียบ ROW_NUMBER, RANK, DENSE_RANK
SELECT 
    salary,
    ROW_NUMBER() OVER (ORDER BY salary DESC) AS row_number,
    RANK()       OVER (ORDER BY salary DESC) AS rank_with_gap,
    DENSE_RANK() OVER (ORDER BY salary DESC) AS dense_rank
FROM employees
WHERE salary IS NOT NULL
ORDER BY salary DESC;
```

```sql
-- ตัวอย่างที่ 17: RANK ต่อ Partition
-- จัดอันดับพนักงานในแต่ละแผนก
SELECT 
    d.department_name,
    e.first_name,
    e.last_name,
    e.salary,
    RANK() OVER (
        PARTITION BY e.department_id 
        ORDER BY e.salary DESC
    ) AS dept_rank
FROM employees e
JOIN departments d ON e.department_id = d.department_id
WHERE e.salary IS NOT NULL
ORDER BY e.department_id, dept_rank;
```

---

## 7. Window Function กับ Aggregate ร่วมกัน

```sql
-- ตัวอย่างที่ 18: รวม GROUP BY กับ Window Function
SELECT 
    DATE_FORMAT(order_date, '%Y-%m') AS month,
    COUNT(*) AS monthly_orders,
    SUM(total_amount) AS monthly_revenue,
    -- Running total ข้ามเดือน
    SUM(SUM(total_amount)) OVER (
        ORDER BY DATE_FORMAT(order_date, '%Y-%m')
    ) AS ytd_revenue,
    -- Month-over-Month Growth
    SUM(total_amount) - LAG(SUM(total_amount)) OVER (
        ORDER BY DATE_FORMAT(order_date, '%Y-%m')
    ) AS mom_growth
FROM orders
WHERE status NOT IN ('cancelled')
GROUP BY DATE_FORMAT(order_date, '%Y-%m')
ORDER BY month;
```

```sql
-- ตัวอย่างที่ 19: Top N per Group ด้วย Window Function
-- หา Top 2 สินค้าขายดีของแต่ละ Category
SELECT *
FROM (
    SELECT 
        p.category,
        p.product_name,
        SUM(oi.quantity * oi.unit_price) AS revenue,
        RANK() OVER (
            PARTITION BY p.category 
            ORDER BY SUM(oi.quantity * oi.unit_price) DESC
        ) AS category_rank
    FROM products p
    JOIN order_items oi ON p.product_id = oi.product_id
    JOIN orders o ON oi.order_id = o.order_id
    WHERE o.status NOT IN ('cancelled')
    GROUP BY p.category, p.product_id, p.product_name
) AS ranked_products
WHERE category_rank <= 2
ORDER BY category, category_rank;
```

```sql
-- ตัวอย่างที่ 20: Percentile Ranking ด้วย PERCENT_RANK
SELECT 
    order_id,
    customer_id,
    total_amount,
    ROUND(PERCENT_RANK() OVER (ORDER BY total_amount) * 100, 2) AS percentile,
    NTILE(4) OVER (ORDER BY total_amount) AS quartile
FROM orders
WHERE status NOT IN ('cancelled')
ORDER BY total_amount DESC;
```

---

## 8. Window Frame Specification

```sql
-- ตัวอย่างที่ 21: ROWS vs RANGE
-- ROWS BETWEEN: นับตามจำนวนแถว
-- RANGE BETWEEN: นับตามช่วงค่า

SELECT 
    order_id,
    order_date,
    total_amount,
    -- Running Total (default: RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW)
    SUM(total_amount) OVER (ORDER BY order_date) AS running_total_default,
    -- ROWS: นับตามแถว
    SUM(total_amount) OVER (
        ORDER BY order_date
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    ) AS running_total_rows
FROM orders
WHERE status NOT IN ('cancelled')
ORDER BY order_date;
```

---

## สรุปบทที่ 38 - Window Functions Cheat Sheet

```sql
-- รูปแบบทั่วไปของ Window Functions:

-- 1. Grand Total Comparison
SELECT col, SUM(col) OVER () AS grand_total FROM table;

-- 2. Running Total
SELECT col, SUM(col) OVER (ORDER BY date_col) AS running_total FROM table;

-- 3. Partition Running Total
SELECT col, SUM(col) OVER (PARTITION BY grp ORDER BY date_col) AS grp_running_total FROM table;

-- 4. Moving Average (3 periods)
SELECT col, AVG(col) OVER (ORDER BY date_col ROWS BETWEEN 2 PRECEDING AND CURRENT ROW) AS ma3 FROM table;

-- 5. Ranking
SELECT col, RANK() OVER (ORDER BY col DESC) AS rnk FROM table;
SELECT col, RANK() OVER (PARTITION BY grp ORDER BY col DESC) AS grp_rnk FROM table;

-- 6. Row Number
SELECT col, ROW_NUMBER() OVER (ORDER BY col) AS rn FROM table;

-- 7. Percent Rank
SELECT col, PERCENT_RANK() OVER (ORDER BY col) * 100 AS pct_rank FROM table;
```

---

## แบบฝึกหัดท้ายบท

### แบบฝึกหัดที่ 1
แสดงทุกออเดอร์พร้อมยอดรวมทั้งหมดและเปอร์เซ็นต์ของแต่ละออเดอร์

```sql
-- เฉลย:
SELECT 
    order_id,
    total_amount,
    SUM(total_amount) OVER () AS grand_total,
    ROUND(total_amount * 100.0 / SUM(total_amount) OVER (), 2) AS pct
FROM orders
WHERE status NOT IN ('cancelled')
ORDER BY total_amount DESC;
```

### แบบฝึกหัดที่ 2
คำนวณ Running Total ของยอดขายเรียงตาม order_date

```sql
-- เฉลย:
SELECT 
    order_id,
    order_date,
    total_amount,
    SUM(total_amount) OVER (ORDER BY order_date, order_id) AS running_total
FROM orders
WHERE status NOT IN ('cancelled')
ORDER BY order_date;
```

### แบบฝึกหัดที่ 3
แสดงเงินเดือนพนักงานพร้อมค่าเฉลี่ยของแผนก และความแตกต่าง

```sql
-- เฉลย:
SELECT 
    first_name,
    last_name,
    department_id,
    salary,
    ROUND(AVG(salary) OVER (PARTITION BY department_id), 0) AS dept_avg,
    salary - ROUND(AVG(salary) OVER (PARTITION BY department_id), 0) AS diff
FROM employees
WHERE salary IS NOT NULL
ORDER BY department_id, salary DESC;
```

### แบบฝึกหัดที่ 4
จัดอันดับพนักงานตามเงินเดือนโดยรวม (ใช้ RANK)

```sql
-- เฉลย:
SELECT 
    RANK() OVER (ORDER BY salary DESC) AS salary_rank,
    first_name, last_name, salary
FROM employees
WHERE salary IS NOT NULL
ORDER BY salary DESC;
```

### แบบฝึกหัดที่ 5
จัดอันดับออเดอร์ในแต่ละ customer (ใช้ ROW_NUMBER ต่อ Partition)

```sql
-- เฉลย:
SELECT 
    customer_id,
    order_id,
    order_date,
    total_amount,
    ROW_NUMBER() OVER (
        PARTITION BY customer_id 
        ORDER BY order_date
    ) AS purchase_number
FROM orders
WHERE status NOT IN ('cancelled')
ORDER BY customer_id, order_date;
```

### แบบฝึกหัดที่ 6
คำนวณ 3-month Moving Average ของยอดขาย

```sql
-- เฉลย:
SELECT 
    DATE_FORMAT(order_date, '%Y-%m') AS month,
    SUM(total_amount) AS revenue,
    ROUND(AVG(SUM(total_amount)) OVER (
        ORDER BY DATE_FORMAT(order_date, '%Y-%m')
        ROWS BETWEEN 2 PRECEDING AND CURRENT ROW
    ), 0) AS moving_avg_3m
FROM orders
WHERE status NOT IN ('cancelled')
GROUP BY DATE_FORMAT(order_date, '%Y-%m')
ORDER BY month;
```

### แบบฝึกหัดที่ 7
หา Top 1 ลูกค้าที่ใช้จ่ายมากที่สุดในแต่ละจังหวัด

```sql
-- เฉลย:
SELECT province, customer_name, total_spent
FROM (
    SELECT 
        c.province,
        CONCAT(c.first_name, ' ', c.last_name) AS customer_name,
        SUM(o.total_amount) AS total_spent,
        RANK() OVER (
            PARTITION BY c.province 
            ORDER BY SUM(o.total_amount) DESC
        ) AS province_rank
    FROM customers c
    JOIN orders o ON c.customer_id = o.customer_id
    WHERE o.status != 'cancelled'
    GROUP BY c.customer_id, c.province, c.first_name, c.last_name
) AS ranked
WHERE province_rank = 1
ORDER BY total_spent DESC;
```

### แบบฝึกหัดที่ 8
แสดงยอดขายรายเดือนพร้อม Month-over-Month Growth

```sql
-- เฉลย:
SELECT 
    DATE_FORMAT(order_date, '%Y-%m') AS month,
    SUM(total_amount) AS revenue,
    LAG(SUM(total_amount)) OVER (ORDER BY DATE_FORMAT(order_date, '%Y-%m')) AS prev_month,
    SUM(total_amount) - LAG(SUM(total_amount)) OVER (ORDER BY DATE_FORMAT(order_date, '%Y-%m')) AS mom_change,
    ROUND(
        (SUM(total_amount) - LAG(SUM(total_amount)) OVER (ORDER BY DATE_FORMAT(order_date, '%Y-%m'))) * 100.0 /
        NULLIF(LAG(SUM(total_amount)) OVER (ORDER BY DATE_FORMAT(order_date, '%Y-%m')), 0),
        2
    ) AS mom_growth_pct
FROM orders
WHERE status NOT IN ('cancelled')
GROUP BY DATE_FORMAT(order_date, '%Y-%m')
ORDER BY month;
```

### แบบฝึกหัดที่ 9
แสดงสินค้าแต่ละตัวพร้อมอันดับใน Category ตามราคา

```sql
-- เฉลย:
SELECT 
    category,
    product_name,
    price,
    RANK() OVER (PARTITION BY category ORDER BY price DESC) AS price_rank_in_category,
    ROUND(price * 100.0 / SUM(price) OVER (PARTITION BY category), 2) AS pct_of_category_total
FROM products
WHERE is_active = TRUE
ORDER BY category, price DESC;
```

### แบบฝึกหัดที่ 10
สร้าง Complete Sales Report ด้วย Window Functions หลายตัว

```sql
-- เฉลย:
SELECT 
    DATE_FORMAT(order_date, '%Y-%m') AS month,
    COUNT(*) AS orders,
    SUM(total_amount) AS monthly_revenue,
    -- Running Total
    SUM(SUM(total_amount)) OVER (ORDER BY DATE_FORMAT(order_date, '%Y-%m')) AS ytd_revenue,
    -- Moving Average
    ROUND(AVG(SUM(total_amount)) OVER (
        ORDER BY DATE_FORMAT(order_date, '%Y-%m')
        ROWS BETWEEN 2 PRECEDING AND CURRENT ROW
    ), 0) AS ma3,
    -- Month Rank
    RANK() OVER (ORDER BY SUM(total_amount) DESC) AS revenue_rank,
    -- % of Total
    ROUND(SUM(total_amount) * 100.0 / SUM(SUM(total_amount)) OVER (), 2) AS pct_of_total
FROM orders
WHERE status NOT IN ('cancelled')
GROUP BY DATE_FORMAT(order_date, '%Y-%m')
ORDER BY month;
```

---

*จบบทที่ 038 - Window Functions Preview: Running Totals*

*บทถัดไป: Part 039 - Statistical Aggregations*
