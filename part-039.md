# Part 039: Statistical Aggregations

## บทนำ (Introduction)

**Statistical Aggregations** ช่วยให้เราทำการวิเคราะห์เชิงสถิติด้วย SQL ซึ่งมีประโยชน์มากสำหรับ:

- **Data Science** - วิเคราะห์ distribution ของข้อมูล
- **Business Analytics** - ประเมินความเสี่ยงและความแปรปรวน
- **Quality Control** - หา outliers และ anomalies

ฟังก์ชันสถิติที่จะเรียนในบทนี้:
- `STDDEV` / `STDDEV_POP` / `STDDEV_SAMP`
- `VARIANCE` / `VAR_POP` / `VAR_SAMP`
- `PERCENTILE_CONT` / `PERCENTILE_DISC`
- `MEDIAN` / `MODE`
- Correlation Analysis

---

## การตั้งค่าฐานข้อมูล

```sql
USE ecommerce_db;

-- เพิ่มข้อมูล salary ให้สมบูรณ์สำหรับการวิเคราะห์สถิติ
UPDATE employees SET salary = 38000 WHERE employee_id = 20;

-- ตรวจสอบข้อมูล
SELECT 
    MIN(salary) AS min_sal,
    MAX(salary) AS max_sal,
    AVG(salary) AS avg_sal,
    COUNT(salary) AS count_sal
FROM employees;
```

---

## 1. STDDEV - Standard Deviation

Standard Deviation วัดการกระจายตัวของข้อมูลรอบค่าเฉลี่ย

### ชนิดของ STDDEV

| Function | ความหมาย | ใช้เมื่อ |
|----------|---------|---------|
| `STDDEV()` | Population StdDev (alias) | |
| `STDDEV_POP()` | Population Standard Deviation | มีข้อมูลทั้งหมด (ทั้ง Population) |
| `STDDEV_SAMP()` | Sample Standard Deviation | มีข้อมูลเป็นตัวอย่าง (Sample) |
| `STD()` | Alias ของ STDDEV_POP() | |

```sql
-- ตัวอย่างที่ 1: STDDEV พื้นฐาน
SELECT 
    COUNT(salary)           AS n,
    ROUND(AVG(salary), 0)   AS mean_salary,
    ROUND(STDDEV(salary), 0) AS stddev_salary,
    ROUND(STDDEV_POP(salary), 0) AS stddev_pop,
    ROUND(STDDEV_SAMP(salary), 0) AS stddev_samp
FROM employees
WHERE salary IS NOT NULL;
```

```sql
-- ตัวอย่างที่ 2: STDDEV แยกตามแผนก
SELECT 
    d.department_name,
    COUNT(e.employee_id) AS n,
    ROUND(AVG(e.salary), 0) AS avg_salary,
    ROUND(STDDEV(e.salary), 0) AS salary_stddev,
    ROUND(MIN(e.salary), 0) AS min_salary,
    ROUND(MAX(e.salary), 0) AS max_salary
FROM employees e
JOIN departments d ON e.department_id = d.department_id
WHERE e.salary IS NOT NULL
GROUP BY d.department_id, d.department_name
ORDER BY salary_stddev DESC;
```

```sql
-- ตัวอย่างที่ 3: Coefficient of Variation (CV) - ความแปรปรวนสัมพัทธ์
-- CV = StdDev / Mean * 100%
-- ยิ่งสูง = ข้อมูลกระจายมาก
SELECT 
    d.department_name,
    ROUND(AVG(e.salary), 0) AS avg_salary,
    ROUND(STDDEV(e.salary), 0) AS stddev_salary,
    ROUND(STDDEV(e.salary) / AVG(e.salary) * 100, 2) AS cv_pct
FROM employees e
JOIN departments d ON e.department_id = d.department_id
WHERE e.salary IS NOT NULL
GROUP BY d.department_id, d.department_name
ORDER BY cv_pct DESC;
```

```sql
-- ตัวอย่างที่ 4: STDDEV ของยอดขาย
SELECT 
    DATE_FORMAT(order_date, '%Y-%m') AS month,
    COUNT(*) AS n,
    ROUND(AVG(total_amount), 0) AS avg_order,
    ROUND(STDDEV(total_amount), 0) AS stddev_order,
    MIN(total_amount) AS min_order,
    MAX(total_amount) AS max_order,
    -- 95% Confidence Interval: mean ± 2σ
    ROUND(AVG(total_amount) - 2 * STDDEV(total_amount), 0) AS lower_95ci,
    ROUND(AVG(total_amount) + 2 * STDDEV(total_amount), 0) AS upper_95ci
FROM orders
WHERE status NOT IN ('cancelled')
GROUP BY DATE_FORMAT(order_date, '%Y-%m')
ORDER BY month;
```

---

## 2. VARIANCE - ความแปรปรวน

Variance = StdDev² (ค่า StdDev ยกกำลังสอง)

```sql
-- ตัวอย่างที่ 5: VARIANCE พื้นฐาน
SELECT 
    ROUND(AVG(salary), 0) AS mean_salary,
    ROUND(VARIANCE(salary), 0) AS variance_salary,
    ROUND(VAR_POP(salary), 0) AS var_pop,
    ROUND(VAR_SAMP(salary), 0) AS var_samp,
    -- Verify: variance = stddev²
    ROUND(STDDEV(salary) * STDDEV(salary), 0) AS stddev_squared
FROM employees
WHERE salary IS NOT NULL;
```

```sql
-- ตัวอย่างที่ 6: VARIANCE ต่อ Category (ราคาสินค้า)
SELECT 
    category,
    COUNT(*) AS n,
    ROUND(AVG(price), 0) AS avg_price,
    ROUND(STDDEV(price), 0) AS price_stddev,
    ROUND(VARIANCE(price), 0) AS price_variance,
    MIN(price) AS min_price,
    MAX(price) AS max_price
FROM products
WHERE is_active = TRUE
GROUP BY category
ORDER BY price_variance DESC;
```

```sql
-- ตัวอย่างที่ 7: เปรียบเทียบ Variance ระหว่างปี
SELECT 
    YEAR(order_date) AS year,
    COUNT(*) AS order_count,
    ROUND(AVG(total_amount), 0) AS mean,
    ROUND(STDDEV(total_amount), 0) AS stddev,
    ROUND(VARIANCE(total_amount), 0) AS variance,
    MIN(total_amount) AS min_val,
    MAX(total_amount) AS max_val
FROM orders
WHERE status NOT IN ('cancelled')
GROUP BY YEAR(order_date)
ORDER BY year;
```

---

## 3. Percentiles - เปอร์เซ็นไทล์

เปอร์เซ็นไทล์ช่วยให้เข้าใจ Distribution ของข้อมูล

### MySQL: ใช้ Subquery สำหรับ Percentile

```sql
-- ตัวอย่างที่ 8: หา Percentiles ของเงินเดือน (MySQL)
SELECT 
    MIN(salary) AS p0,
    -- 25th percentile (Q1)
    (SELECT salary FROM (
        SELECT salary, ROW_NUMBER() OVER (ORDER BY salary) AS rn,
               COUNT(*) OVER () AS total
        FROM employees WHERE salary IS NOT NULL
    ) AS t WHERE rn = CEIL(total * 0.25)) AS p25,
    -- 50th percentile (Median)
    (SELECT salary FROM (
        SELECT salary, ROW_NUMBER() OVER (ORDER BY salary) AS rn,
               COUNT(*) OVER () AS total
        FROM employees WHERE salary IS NOT NULL
    ) AS t WHERE rn = CEIL(total * 0.50)) AS p50_median,
    -- 75th percentile (Q3)
    (SELECT salary FROM (
        SELECT salary, ROW_NUMBER() OVER (ORDER BY salary) AS rn,
               COUNT(*) OVER () AS total
        FROM employees WHERE salary IS NOT NULL
    ) AS t WHERE rn = CEIL(total * 0.75)) AS p75,
    MAX(salary) AS p100
FROM employees
WHERE salary IS NOT NULL;
```

### PostgreSQL: ใช้ PERCENTILE_CONT / PERCENTILE_DISC

```sql
-- ตัวอย่างที่ 9: PERCENTILE_CONT (PostgreSQL)
-- PERCENTILE_CONT = ค่าที่ interpolate (ระหว่างค่า)
-- PERCENTILE_DISC = ค่าที่มีอยู่จริงในข้อมูล

-- PostgreSQL syntax:
-- SELECT 
--     PERCENTILE_CONT(0.25) WITHIN GROUP (ORDER BY salary) AS p25,
--     PERCENTILE_CONT(0.50) WITHIN GROUP (ORDER BY salary) AS median,
--     PERCENTILE_CONT(0.75) WITHIN GROUP (ORDER BY salary) AS p75,
--     PERCENTILE_DISC(0.50) WITHIN GROUP (ORDER BY salary) AS median_disc
-- FROM employees WHERE salary IS NOT NULL;

-- MySQL equivalent ด้วย Window Functions:
SELECT DISTINCT
    PERCENTILE_CONT(0.25) WITHIN GROUP (ORDER BY salary) 
        OVER () AS p25,
    PERCENTILE_CONT(0.50) WITHIN GROUP (ORDER BY salary) 
        OVER () AS median
FROM employees
WHERE salary IS NOT NULL;
-- หมายเหตุ: MySQL 8.0 ยังไม่รองรับ PERCENTILE_CONT เต็มรูปแบบ
```

```sql
-- ตัวอย่างที่ 10: Distribution Analysis ด้วย NTILE
SELECT 
    salary,
    first_name,
    last_name,
    NTILE(4) OVER (ORDER BY salary) AS quartile,
    NTILE(10) OVER (ORDER BY salary) AS decile
FROM employees
WHERE salary IS NOT NULL
ORDER BY salary;
```

---

## 4. MEDIAN - ค่ากลาง

MEDIAN = ค่ากลางของข้อมูล (ค่าที่ 50th percentile)

```sql
-- ตัวอย่างที่ 11: คำนวณ MEDIAN ใน MySQL
-- MySQL ไม่มี MEDIAN function โดยตรง ต้องคำนวณเอง

-- วิธีที่ 1: ใช้ Window Functions
WITH ranked AS (
    SELECT 
        salary,
        ROW_NUMBER() OVER (ORDER BY salary) AS rn,
        COUNT(*) OVER () AS total
    FROM employees
    WHERE salary IS NOT NULL
)
SELECT AVG(salary) AS median_salary
FROM ranked
WHERE rn IN (FLOOR((total + 1) / 2), CEIL((total + 1) / 2));
```

```sql
-- ตัวอย่างที่ 12: Median vs Mean - ดูความแตกต่าง
-- Mean สูงกว่า Median = ข้อมูลเบ้ขวา (มีค่าสูงมากๆ บางส่วน)
-- Mean ต่ำกว่า Median = ข้อมูลเบ้ซ้าย (มีค่าต่ำมากๆ บางส่วน)

WITH ranked AS (
    SELECT 
        total_amount,
        ROW_NUMBER() OVER (ORDER BY total_amount) AS rn,
        COUNT(*) OVER () AS total
    FROM orders
    WHERE status NOT IN ('cancelled')
)
SELECT 
    (SELECT AVG(total_amount) FROM orders WHERE status NOT IN ('cancelled')) AS mean_order,
    (SELECT AVG(total_amount) FROM ranked WHERE rn IN (FLOOR((total+1)/2), CEIL((total+1)/2))) AS median_order,
    (SELECT MIN(total_amount) FROM orders WHERE status NOT IN ('cancelled')) AS min_order,
    (SELECT MAX(total_amount) FROM orders WHERE status NOT IN ('cancelled')) AS max_order;
```

---

## 5. MODE - ฐานนิยม (ค่าที่พบบ่อยที่สุด)

```sql
-- ตัวอย่างที่ 13: MODE ของ payment_method
SELECT 
    payment_method,
    COUNT(*) AS frequency
FROM orders
GROUP BY payment_method
ORDER BY frequency DESC
LIMIT 1;  -- MODE = ค่าที่พบบ่อยที่สุด
```

```sql
-- ตัวอย่างที่ 14: MODE ของสินค้าที่ถูกสั่งซื้อ
SELECT 
    p.product_name,
    COUNT(*) AS frequency
FROM order_items oi
JOIN products p ON oi.product_id = p.product_id
GROUP BY oi.product_id, p.product_name
ORDER BY frequency DESC
LIMIT 1;
```

```sql
-- ตัวอย่างที่ 15: Multiple MODE Values
-- หาทุกค่าที่มีความถี่สูงสุด (อาจมีหลาย mode)
SELECT payment_method, order_count
FROM (
    SELECT 
        payment_method,
        COUNT(*) AS order_count,
        RANK() OVER (ORDER BY COUNT(*) DESC) AS freq_rank
    FROM orders
    GROUP BY payment_method
) AS freq_data
WHERE freq_rank = 1;
```

---

## 6. Box Plot Statistics (5-Number Summary)

```sql
-- ตัวอย่างที่ 16: 5-Number Summary สำหรับ salary
WITH salary_ranked AS (
    SELECT 
        salary,
        ROW_NUMBER() OVER (ORDER BY salary) AS rn,
        COUNT(*) OVER () AS n
    FROM employees
    WHERE salary IS NOT NULL
)
SELECT 
    MIN(salary) AS minimum,
    AVG(CASE WHEN rn IN (FLOOR((n+1)/4), CEIL((n+1)/4)) THEN salary END) AS Q1,
    AVG(CASE WHEN rn IN (FLOOR((n+1)/2), CEIL((n+1)/2)) THEN salary END) AS median_Q2,
    AVG(CASE WHEN rn IN (FLOOR(3*(n+1)/4), CEIL(3*(n+1)/4)) THEN salary END) AS Q3,
    MAX(salary) AS maximum
FROM salary_ranked;
```

```sql
-- ตัวอย่างที่ 17: IQR (Interquartile Range) - หา Outliers
WITH salary_stats AS (
    WITH ranked AS (
        SELECT salary, ROW_NUMBER() OVER (ORDER BY salary) AS rn, COUNT(*) OVER () AS n
        FROM employees WHERE salary IS NOT NULL
    )
    SELECT 
        AVG(CASE WHEN rn IN (FLOOR((n+1)/4), CEIL((n+1)/4)) THEN salary END) AS q1,
        AVG(CASE WHEN rn IN (FLOOR(3*(n+1)/4), CEIL(3*(n+1)/4)) THEN salary END) AS q3
    FROM ranked
)
SELECT 
    e.employee_id,
    e.first_name,
    e.last_name,
    e.salary,
    s.q1,
    s.q3,
    s.q3 - s.q1 AS iqr,
    s.q1 - 1.5 * (s.q3 - s.q1) AS lower_fence,
    s.q3 + 1.5 * (s.q3 - s.q1) AS upper_fence,
    CASE 
        WHEN e.salary < s.q1 - 1.5 * (s.q3 - s.q1) THEN 'Low Outlier'
        WHEN e.salary > s.q3 + 1.5 * (s.q3 - s.q1) THEN 'High Outlier'
        ELSE 'Normal'
    END AS outlier_status
FROM employees e
CROSS JOIN salary_stats s
WHERE e.salary IS NOT NULL
ORDER BY e.salary;
```

---

## 7. Z-Score Analysis

Z-Score บอกว่าค่านั้นอยู่ห่างจากค่าเฉลี่ยกี่ Standard Deviation

```sql
-- ตัวอย่างที่ 18: Z-Score ของเงินเดือน
SELECT 
    e.first_name,
    e.last_name,
    e.salary,
    stats.mean_salary,
    stats.stddev_salary,
    ROUND((e.salary - stats.mean_salary) / NULLIF(stats.stddev_salary, 0), 2) AS z_score,
    CASE 
        WHEN ABS((e.salary - stats.mean_salary) / NULLIF(stats.stddev_salary, 0)) > 2 
            THEN 'Statistical Outlier'
        WHEN ABS((e.salary - stats.mean_salary) / NULLIF(stats.stddev_salary, 0)) > 1
            THEN 'Above/Below 1σ'
        ELSE 'Within 1σ (Normal)'
    END AS deviation_category
FROM employees e
CROSS JOIN (
    SELECT 
        AVG(salary) AS mean_salary,
        STDDEV(salary) AS stddev_salary
    FROM employees 
    WHERE salary IS NOT NULL
) AS stats
WHERE e.salary IS NOT NULL
ORDER BY ABS((e.salary - stats.mean_salary) / NULLIF(stats.stddev_salary, 0)) DESC;
```

```sql
-- ตัวอย่างที่ 19: Z-Score สำหรับยอดสั่งซื้อ
SELECT 
    o.order_id,
    o.customer_id,
    o.total_amount,
    ROUND((o.total_amount - stats.mean_amount) / NULLIF(stats.stddev_amount, 0), 2) AS z_score,
    CASE 
        WHEN ABS((o.total_amount - stats.mean_amount) / NULLIF(stats.stddev_amount, 0)) > 2 
            THEN 'Unusual Order'
        ELSE 'Normal Order'
    END AS order_classification
FROM orders o
CROSS JOIN (
    SELECT 
        AVG(total_amount) AS mean_amount,
        STDDEV(total_amount) AS stddev_amount
    FROM orders WHERE status NOT IN ('cancelled')
) AS stats
WHERE o.status NOT IN ('cancelled')
ORDER BY ABS((o.total_amount - stats.mean_amount) / NULLIF(stats.stddev_amount, 0)) DESC;
```

---

## 8. Correlation Analysis

```sql
-- ตัวอย่างที่ 20: Pearson Correlation ระหว่างตัวแปรสองตัว
-- MySQL ไม่มี CORR() ต้องคำนวณเอง
-- Formula: corr(x,y) = (Σ(x-x̄)(y-ȳ)) / (n * σx * σy)

-- Correlation ระหว่าง loyalty_points กับจำนวนออเดอร์
SELECT 
    (COUNT(*) * SUM(lp * oc) - SUM(lp) * SUM(oc)) /
    SQRT(
        (COUNT(*) * SUM(lp * lp) - SUM(lp) * SUM(lp)) *
        (COUNT(*) * SUM(oc * oc) - SUM(oc) * SUM(oc))
    ) AS correlation_points_orders
FROM (
    SELECT 
        c.loyalty_points AS lp,
        COUNT(o.order_id) AS oc
    FROM customers c
    LEFT JOIN orders o ON c.customer_id = o.customer_id
        AND o.status != 'cancelled'
    GROUP BY c.customer_id, c.loyalty_points
) AS corr_data;
```

```sql
-- ตัวอย่างที่ 21: สรุป Descriptive Statistics ครบถ้วน
SELECT 
    'salary' AS variable,
    COUNT(salary) AS n,
    ROUND(MIN(salary), 0) AS min_val,
    ROUND(MAX(salary), 0) AS max_val,
    ROUND(AVG(salary), 0) AS mean,
    ROUND(STDDEV(salary), 0) AS std_dev,
    ROUND(VARIANCE(salary), 0) AS variance,
    ROUND(STDDEV(salary) / AVG(salary) * 100, 2) AS cv_pct,
    MAX(salary) - MIN(salary) AS range_val
FROM employees
WHERE salary IS NOT NULL;
```

---

## 9. Business Statistics Reports

```sql
-- ตัวอย่างที่ 22: Revenue Distribution Analysis
SELECT 
    'Q1' AS quartile,
    MIN(total_amount) AS min_val,
    MAX(total_amount) AS max_val,
    COUNT(*) AS order_count
FROM (
    SELECT 
        total_amount,
        NTILE(4) OVER (ORDER BY total_amount) AS q
    FROM orders
    WHERE status NOT IN ('cancelled')
) AS q_data
WHERE q = 1

UNION ALL

SELECT 'Q2', MIN(total_amount), MAX(total_amount), COUNT(*)
FROM (SELECT total_amount, NTILE(4) OVER (ORDER BY total_amount) AS q 
      FROM orders WHERE status NOT IN ('cancelled')) AS q2 WHERE q = 2

UNION ALL

SELECT 'Q3', MIN(total_amount), MAX(total_amount), COUNT(*)
FROM (SELECT total_amount, NTILE(4) OVER (ORDER BY total_amount) AS q 
      FROM orders WHERE status NOT IN ('cancelled')) AS q3 WHERE q = 3

UNION ALL

SELECT 'Q4', MIN(total_amount), MAX(total_amount), COUNT(*)
FROM (SELECT total_amount, NTILE(4) OVER (ORDER BY total_amount) AS q 
      FROM orders WHERE status NOT IN ('cancelled')) AS q4 WHERE q = 4;
```

```sql
-- ตัวอย่างที่ 23: Price Distribution ต่อ Category
SELECT 
    category,
    COUNT(*) AS n,
    MIN(price) AS min_price,
    ROUND(AVG(price), 0) AS mean_price,
    MAX(price) AS max_price,
    ROUND(STDDEV(price), 0) AS std_dev,
    ROUND(STDDEV(price) / AVG(price) * 100, 1) AS cv_pct,
    -- ถ้า CV สูง = ราคาไม่สม่ำเสมอ, ต่ำ = ราคาสม่ำเสมอ
    CASE 
        WHEN STDDEV(price) / AVG(price) * 100 < 20 THEN 'ราคาสม่ำเสมอ'
        WHEN STDDEV(price) / AVG(price) * 100 < 50 THEN 'ราคาปานกลาง'
        ELSE 'ราคาผันแปรมาก'
    END AS price_consistency
FROM products
WHERE is_active = TRUE
GROUP BY category
ORDER BY cv_pct DESC;
```

```sql
-- ตัวอย่างที่ 24: Customer Spending Statistics
SELECT 
    c.province,
    COUNT(DISTINCT c.customer_id) AS customers,
    ROUND(AVG(customer_stats.total_spent), 0) AS avg_spending,
    ROUND(STDDEV(customer_stats.total_spent), 0) AS stddev_spending,
    ROUND(MIN(customer_stats.total_spent), 0) AS min_spending,
    ROUND(MAX(customer_stats.total_spent), 0) AS max_spending,
    -- High spenders = avg + 1σ
    ROUND(AVG(customer_stats.total_spent) + STDDEV(customer_stats.total_spent), 0) AS high_spender_threshold
FROM customers c
JOIN (
    SELECT customer_id, SUM(total_amount) AS total_spent
    FROM orders
    WHERE status != 'cancelled'
    GROUP BY customer_id
) AS customer_stats ON c.customer_id = customer_stats.customer_id
GROUP BY c.province
ORDER BY avg_spending DESC;
```

```sql
-- ตัวอย่างที่ 25: Statistical Summary Dashboard
SELECT 
    -- Order Value Statistics
    COUNT(*) AS order_count,
    ROUND(AVG(total_amount), 0) AS mean_order,
    ROUND(STDDEV(total_amount), 0) AS stddev_order,
    ROUND(MIN(total_amount), 0) AS min_order,
    ROUND(MAX(total_amount), 0) AS max_order,
    -- Derived Statistics
    ROUND(STDDEV(total_amount) / AVG(total_amount) * 100, 1) AS cv_pct,
    ROUND(AVG(total_amount) - 2 * STDDEV(total_amount), 0) AS lower_2sigma,
    ROUND(AVG(total_amount) + 2 * STDDEV(total_amount), 0) AS upper_2sigma,
    -- Count within 2σ
    COUNT(CASE WHEN total_amount BETWEEN 
        AVG(total_amount) OVER () - 2 * STDDEV(total_amount) OVER () AND
        AVG(total_amount) OVER () + 2 * STDDEV(total_amount) OVER ()
        THEN 1 END) AS within_2sigma_count
FROM orders
WHERE status NOT IN ('cancelled');
```

---

## สรุปบทที่ 39

### Functions สรุป

| Function | MySQL | PostgreSQL | SQL Server |
|---------|-------|-----------|-----------|
| `STDDEV()` | ✅ | `STDDEV_POP()` | `STDEV()` |
| `STDDEV_POP()` | ✅ | ✅ | `STDEVP()` |
| `STDDEV_SAMP()` | ✅ | ✅ | `STDEV()` |
| `VARIANCE()` | ✅ | `VAR_POP()` | `VAR()` |
| `VAR_POP()` | ✅ | ✅ | `VARP()` |
| `VAR_SAMP()` | ✅ | ✅ | `VAR()` |
| `PERCENTILE_CONT()` | ❌ (ใช้ NTILE) | ✅ | ✅ |
| `MEDIAN()` | ❌ (คำนวณเอง) | `PERCENTILE_CONT(0.5)` | `PERCENTILE_CONT(0.5)` |
| `CORR()` | ❌ (คำนวณเอง) | ✅ | ❌ |

---

## แบบฝึกหัดท้ายบท

### แบบฝึกหัดที่ 1
คำนวณ Standard Deviation ของราคาสินค้าแต่ละ category

```sql
-- เฉลย:
SELECT 
    category,
    COUNT(*) AS products,
    ROUND(AVG(price), 0) AS avg_price,
    ROUND(STDDEV(price), 0) AS price_stddev
FROM products
WHERE is_active = TRUE
GROUP BY category
ORDER BY price_stddev DESC;
```

### แบบฝึกหัดที่ 2
คำนวณ Variance ของยอดสั่งซื้อแต่ละเดือน

```sql
-- เฉลย:
SELECT 
    DATE_FORMAT(order_date, '%Y-%m') AS month,
    COUNT(*) AS orders,
    ROUND(AVG(total_amount), 0) AS mean,
    ROUND(VARIANCE(total_amount), 0) AS variance,
    ROUND(STDDEV(total_amount), 0) AS stddev
FROM orders
WHERE status NOT IN ('cancelled')
GROUP BY DATE_FORMAT(order_date, '%Y-%m')
ORDER BY month;
```

### แบบฝึกหัดที่ 3
แบ่งเงินเดือนออกเป็น 4 Quartile ด้วย NTILE

```sql
-- เฉลย:
SELECT 
    first_name,
    last_name,
    salary,
    NTILE(4) OVER (ORDER BY salary) AS quartile
FROM employees
WHERE salary IS NOT NULL
ORDER BY salary;
```

### แบบฝึกหัดที่ 4
หา Outliers ในยอดสั่งซื้อโดยใช้ Z-Score > 2

```sql
-- เฉลย:
SELECT 
    order_id,
    customer_id,
    total_amount,
    ROUND((total_amount - stats.mean_amt) / NULLIF(stats.std_amt, 0), 2) AS z_score
FROM orders
CROSS JOIN (
    SELECT AVG(total_amount) AS mean_amt, STDDEV(total_amount) AS std_amt
    FROM orders WHERE status NOT IN ('cancelled')
) AS stats
WHERE status NOT IN ('cancelled')
    AND ABS((total_amount - stats.mean_amt) / NULLIF(stats.std_amt, 0)) > 2
ORDER BY ABS((total_amount - stats.mean_amt) / NULLIF(stats.std_amt, 0)) DESC;
```

### แบบฝึกหัดที่ 5
คำนวณ Coefficient of Variation ของเงินเดือนในแต่ละแผนก

```sql
-- เฉลย:
SELECT 
    d.department_name,
    COUNT(*) AS headcount,
    ROUND(AVG(e.salary), 0) AS avg_salary,
    ROUND(STDDEV(e.salary), 0) AS stddev,
    ROUND(STDDEV(e.salary) / NULLIF(AVG(e.salary), 0) * 100, 2) AS cv_pct
FROM employees e
JOIN departments d ON e.department_id = d.department_id
WHERE e.salary IS NOT NULL
GROUP BY d.department_id, d.department_name
ORDER BY cv_pct DESC;
```

### แบบฝึกหัดที่ 6
หา MODE ของหมวดหมู่สินค้าที่ถูกสั่งซื้อบ่อยที่สุด

```sql
-- เฉลย:
SELECT 
    p.category,
    COUNT(*) AS order_frequency
FROM order_items oi
JOIN products p ON oi.product_id = p.product_id
GROUP BY p.category
ORDER BY order_frequency DESC
LIMIT 1;
```

### แบบฝึกหัดที่ 7
สร้าง 5-Number Summary สำหรับ total_amount ของออเดอร์

```sql
-- เฉลย:
WITH ranked AS (
    SELECT total_amount,
           ROW_NUMBER() OVER (ORDER BY total_amount) AS rn,
           COUNT(*) OVER () AS n
    FROM orders WHERE status NOT IN ('cancelled')
)
SELECT 
    MIN(total_amount) AS minimum,
    AVG(CASE WHEN rn IN (FLOOR((n+1)/4), CEIL((n+1)/4)) THEN total_amount END) AS Q1,
    AVG(CASE WHEN rn IN (FLOOR((n+1)/2), CEIL((n+1)/2)) THEN total_amount END) AS median,
    AVG(CASE WHEN rn IN (FLOOR(3*(n+1)/4), CEIL(3*(n+1)/4)) THEN total_amount END) AS Q3,
    MAX(total_amount) AS maximum
FROM ranked;
```

### แบบฝึกหัดที่ 8
เปรียบเทียบ Mean vs Median ของยอดสั่งซื้อ

```sql
-- เฉลย:
SELECT 
    ROUND(AVG(total_amount), 0) AS mean_order_value,
    (SELECT AVG(total_amount) 
     FROM (
         SELECT total_amount, ROW_NUMBER() OVER (ORDER BY total_amount) AS rn, COUNT(*) OVER () AS n
         FROM orders WHERE status NOT IN ('cancelled')
     ) AS t
     WHERE rn IN (FLOOR((n+1)/2), CEIL((n+1)/2))
    ) AS median_order_value
FROM orders
WHERE status NOT IN ('cancelled');
```

### แบบฝึกหัดที่ 9
วิเคราะห์การกระจายของ loyalty_points ลูกค้า

```sql
-- เฉลย:
SELECT 
    COUNT(*) AS customers,
    MIN(loyalty_points) AS min_points,
    MAX(loyalty_points) AS max_points,
    ROUND(AVG(loyalty_points), 0) AS mean_points,
    ROUND(STDDEV(loyalty_points), 0) AS stddev_points,
    ROUND(VARIANCE(loyalty_points), 0) AS variance_points,
    ROUND(STDDEV(loyalty_points) / NULLIF(AVG(loyalty_points), 0) * 100, 1) AS cv_pct
FROM customers;
```

### แบบฝึกหัดที่ 10
สร้าง Complete Statistical Dashboard

```sql
-- เฉลย:
SELECT 
    'Orders' AS metric,
    COUNT(*) AS n,
    ROUND(MIN(total_amount), 0) AS min_val,
    ROUND(AVG(total_amount), 0) AS mean_val,
    ROUND(MAX(total_amount), 0) AS max_val,
    ROUND(STDDEV(total_amount), 0) AS std_dev,
    ROUND(VARIANCE(total_amount), 0) AS variance,
    ROUND(STDDEV(total_amount) / AVG(total_amount) * 100, 1) AS cv_pct
FROM orders WHERE status NOT IN ('cancelled')

UNION ALL

SELECT 
    'Products',
    COUNT(*),
    ROUND(MIN(price), 0),
    ROUND(AVG(price), 0),
    ROUND(MAX(price), 0),
    ROUND(STDDEV(price), 0),
    ROUND(VARIANCE(price), 0),
    ROUND(STDDEV(price) / AVG(price) * 100, 1)
FROM products WHERE is_active = TRUE

UNION ALL

SELECT 
    'Salaries',
    COUNT(salary),
    ROUND(MIN(salary), 0),
    ROUND(AVG(salary), 0),
    ROUND(MAX(salary), 0),
    ROUND(STDDEV(salary), 0),
    ROUND(VARIANCE(salary), 0),
    ROUND(STDDEV(salary) / AVG(salary) * 100, 1)
FROM employees WHERE salary IS NOT NULL;
```

---

*จบบทที่ 039 - Statistical Aggregations*

*บทถัดไป: Part 040 - Real-world Aggregation Projects*
