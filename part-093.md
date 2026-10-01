# ส่วนที่ 93: Window Functions ตอนที่ 1 - Ranking Functions

## บทนำ

Window Functions เป็นหนึ่งในฟีเจอร์ที่ทรงพลังที่สุดใน SQL สมัยใหม่ แตกต่างจาก aggregate functions ที่ "ยุบ" หลายแถวเป็นแถวเดียว Window Functions ทำงานกับ "หน้าต่าง" ของแถวที่เกี่ยวข้องกับแถวปัจจุบัน และ return ผลลัพธ์สำหรับ **ทุกแถว** โดยไม่ลดจำนวนแถว

---

## 93.1 OVER() Clause - หัวใจของ Window Functions

### ไวยากรณ์พื้นฐาน

```sql
function_name(args) OVER (
    [PARTITION BY column1, column2, ...]
    [ORDER BY column3 [ASC|DESC], ...]
    [frame_specification]
)
```

### ความแตกต่าง: Window Function vs GROUP BY

```sql
-- GROUP BY: ลดจำนวนแถว
SELECT department, AVG(salary) AS avg_sal
FROM employees
GROUP BY department;
-- ผลลัพธ์: 3 แถว (เท่ากับจำนวน department)

-- Window Function: คงจำนวนแถว
SELECT 
    name, 
    department, 
    salary,
    AVG(salary) OVER (PARTITION BY department) AS dept_avg
FROM employees;
-- ผลลัพธ์: 10 แถว (คงจำนวนเดิม) พร้อมค่าเฉลี่ยแผนกในทุกแถว
```

---

## 93.2 PARTITION BY - แบ่งกลุ่มสำหรับการคำนวณ

### ตัวอย่างที่ 1: OVER() ไม่มี PARTITION

```sql
-- สร้างข้อมูล
CREATE TABLE employees (
    emp_id      INT PRIMARY KEY,
    name        VARCHAR(100),
    department  VARCHAR(50),
    salary      DECIMAL(10,2),
    hire_date   DATE
);

INSERT INTO employees VALUES
(1,  'สมชาย ใจดี',      'IT',      85000,  '2020-01-15'),
(2,  'สมหญิง รักงาน',   'HR',      65000,  '2019-03-20'),
(3,  'วิชัย เก่งกาจ',   'IT',      90000,  '2021-06-01'),
(4,  'มานี มีทรัพย์',   'Finance', 75000,  '2018-11-10'),
(5,  'ประเสริฐ ดีมาก',  'IT',      70000,  '2022-02-14'),
(6,  'อรุณี สวยงาม',    'HR',      60000,  '2020-07-22'),
(7,  'บุญมี มากทรัพย์', 'Finance', 80000,  '2017-05-30'),
(8,  'จรัส เจริญรุ่ง',  'IT',      95000,  '2016-09-15'),
(9,  'ลัดดา ดาวเรือง',  'HR',      55000,  '2023-01-08'),
(10, 'ไพศาล ใหญ่โต',   'Finance', 72000,  '2021-04-19');

-- ROW_NUMBER ทั่วทั้งตาราง (ไม่มี partition)
SELECT 
    emp_id,
    name,
    salary,
    ROW_NUMBER() OVER (ORDER BY salary DESC) AS salary_rank
FROM employees;
```

### ตัวอย่างที่ 2: OVER() พร้อม PARTITION BY

```sql
-- ROW_NUMBER แยกตามแผนก
SELECT 
    emp_id,
    name,
    department,
    salary,
    ROW_NUMBER() OVER (PARTITION BY department ORDER BY salary DESC) AS dept_rank
FROM employees;
-- แต่ละแผนกเริ่มนับจาก 1 ใหม่
```

---

## 93.3 ROW_NUMBER() - ตัวเลขลำดับที่ไม่ซ้ำ

ROW_NUMBER() กำหนดหมายเลขลำดับที่ unique ให้แต่ละแถว ถึงแม้มีค่าเท่ากัน ก็จะได้เลขที่ต่างกัน

### ตัวอย่างที่ 3: ROW_NUMBER พื้นฐาน

```sql
-- ลำดับพนักงานตามเงินเดือน
SELECT 
    name,
    department,
    salary,
    ROW_NUMBER() OVER (ORDER BY salary DESC) AS overall_rank,
    ROW_NUMBER() OVER (PARTITION BY department ORDER BY salary DESC) AS dept_rank
FROM employees;
```

### ตัวอย่างที่ 4: ROW_NUMBER สำหรับ Deduplication

```sql
-- ลบ duplicates: เก็บแถวที่มี hire_date เร็วที่สุดสำหรับแต่ละ employee
CREATE TABLE emp_duplicates AS SELECT * FROM employees;
INSERT INTO emp_duplicates SELECT emp_id, name, department, salary, hire_date + 30 FROM employees WHERE emp_id <= 3;

WITH ranked AS (
    SELECT *,
           ROW_NUMBER() OVER (PARTITION BY emp_id ORDER BY hire_date) AS rn
    FROM emp_duplicates
)
SELECT emp_id, name, department, salary, hire_date
FROM ranked
WHERE rn = 1;
```

### ตัวอย่างที่ 5: ROW_NUMBER สำหรับ Pagination

```sql
-- แสดงหน้าที่ 2 (แถวที่ 4-6)
WITH paged AS (
    SELECT 
        emp_id, name, department, salary,
        ROW_NUMBER() OVER (ORDER BY emp_id) AS rn
    FROM employees
)
SELECT *
FROM paged
WHERE rn BETWEEN 4 AND 6;
```

### ตัวอย่างที่ 6: Top-N per Group ด้วย ROW_NUMBER

```sql
-- Top 2 พนักงานเงินเดือนสูงสุดในแต่ละแผนก
WITH ranked AS (
    SELECT 
        emp_id, name, department, salary,
        ROW_NUMBER() OVER (PARTITION BY department ORDER BY salary DESC) AS rn
    FROM employees
)
SELECT emp_id, name, department, salary
FROM ranked
WHERE rn <= 2;
```

### ตัวอย่างที่ 7: ROW_NUMBER สำหรับ Assigning Sequential IDs

```sql
-- สร้าง sequential ID สำหรับ import data
SELECT 
    ROW_NUMBER() OVER (ORDER BY hire_date, emp_id) AS new_id,
    name,
    department,
    salary
FROM employees;
```

---

## 93.4 RANK() - อันดับพร้อม Gaps

RANK() กำหนดอันดับ แต่ถ้ามีค่าเท่ากัน จะได้อันดับเดียวกัน และข้ามอันดับถัดไป

### ตัวอย่างที่ 8: RANK vs ROW_NUMBER เมื่อมีค่าเท่ากัน

```sql
-- สร้างข้อมูลที่มี salary เท่ากัน
CREATE TABLE sales_data (
    rep_id  INT,
    name    VARCHAR(100),
    region  VARCHAR(50),
    sales   DECIMAL(10,2)
);

INSERT INTO sales_data VALUES
(1, 'สมชาย', 'เหนือ', 150000),
(2, 'สมหญิง','ใต้',   120000),
(3, 'วิชัย', 'กลาง',  150000),  -- เท่ากับ สมชาย
(4, 'มานี',  'ออก',   130000),
(5, 'ประเสริฐ','เหนือ',120000), -- เท่ากับ สมหญิง
(6, 'อรุณี', 'ใต้',   95000);

SELECT 
    name,
    sales,
    ROW_NUMBER() OVER (ORDER BY sales DESC) AS row_num,  -- 1,2,3,4,5,6
    RANK()       OVER (ORDER BY sales DESC) AS rank_with_gaps,    -- 1,1,3,4,4,6
    DENSE_RANK() OVER (ORDER BY sales DESC) AS dense_rank         -- 1,1,2,3,3,4
FROM sales_data;
```

### ตัวอย่างที่ 9: RANK สำหรับ Competition Results

```sql
-- ผลการแข่งขัน (เหมาะกับ RANK เพราะมีลำดับที่ข้ามได้)
CREATE TABLE competition_scores (
    contestant VARCHAR(100),
    event      VARCHAR(50),
    score      INT
);

INSERT INTO competition_scores VALUES
('Alice', 'Swimming', 95),
('Bob',   'Swimming', 95),
('Carol', 'Swimming', 88),
('Dave',  'Swimming', 85),
('Eve',   'Swimming', 85),
('Frank', 'Swimming', 72);

SELECT 
    contestant,
    score,
    RANK() OVER (ORDER BY score DESC) AS rank
FROM competition_scores
WHERE event = 'Swimming'
ORDER BY rank;
-- Alice และ Bob ได้อันดับ 1 ร่วม
-- Carol ได้อันดับ 3 (ข้าม 2)
-- Dave และ Eve ได้อันดับ 4 ร่วม
-- Frank ได้อันดับ 6 (ข้าม 5)
```

### ตัวอย่างที่ 10: RANK ใน Sales Leaderboard

```sql
WITH sales_summary AS (
    SELECT 
        salesperson,
        SUM(quantity * unit_price) AS total_sales
    FROM sales
    GROUP BY salesperson
)
SELECT 
    salesperson,
    total_sales,
    RANK() OVER (ORDER BY total_sales DESC) AS rank
FROM sales_summary;
```

---

## 93.5 DENSE_RANK() - อันดับต่อเนื่องไม่มีช่องว่าง

### ตัวอย่างที่ 11: DENSE_RANK สำหรับ Grade/Level Assignment

```sql
-- กำหนด grade level ตามเงินเดือน
SELECT 
    name,
    salary,
    DENSE_RANK() OVER (ORDER BY salary DESC) AS salary_tier,
    CASE DENSE_RANK() OVER (ORDER BY salary DESC)
        WHEN 1 THEN 'Level A'
        WHEN 2 THEN 'Level B'
        WHEN 3 THEN 'Level C'
        ELSE 'Level D'
    END AS salary_level
FROM employees;
```

### ตัวอย่างที่ 12: DENSE_RANK สำหรับ Version Numbering

```sql
-- กำหนดเวอร์ชันจากวันที่ update
CREATE TABLE product_versions (
    product_id  INT,
    updated_at  TIMESTAMP,
    version_data TEXT
);

SELECT 
    product_id,
    updated_at,
    DENSE_RANK() OVER (PARTITION BY product_id ORDER BY updated_at) AS version_number
FROM product_versions;
```

### ตัวอย่างที่ 13: ความแตกต่างระหว่าง RANK และ DENSE_RANK

```sql
-- เปรียบเทียบชัดเจน
WITH scores AS (
    SELECT * FROM (VALUES
        ('A', 100), ('B', 100), ('C', 90), ('D', 85), ('E', 85), ('F', 70)
    ) AS t(name, score)
)
SELECT 
    name, score,
    RANK()       OVER (ORDER BY score DESC) AS rank,
    DENSE_RANK() OVER (ORDER BY score DESC) AS dense_rank
FROM scores;
-- RANK:       1, 1, 3, 4, 4, 6
-- DENSE_RANK: 1, 1, 2, 3, 3, 4
```

---

## 93.6 NTILE(n) - แบ่งข้อมูลเป็น n กลุ่มเท่าๆ กัน

### ตัวอย่างที่ 14: NTILE สำหรับ Quartile Analysis

```sql
-- แบ่งพนักงานเป็น 4 กลุ่มตามเงินเดือน
SELECT 
    name,
    salary,
    NTILE(4) OVER (ORDER BY salary) AS quartile,
    CASE NTILE(4) OVER (ORDER BY salary)
        WHEN 1 THEN 'Q1 - ต่ำสุด 25%'
        WHEN 2 THEN 'Q2 - 25-50%'
        WHEN 3 THEN 'Q3 - 50-75%'
        WHEN 4 THEN 'Q4 - สูงสุด 25%'
    END AS quartile_label
FROM employees
ORDER BY salary;
```

### ตัวอย่างที่ 15: NTILE สำหรับ AB Testing

```sql
-- แบ่ง users เป็น 3 กลุ่มสำหรับ A/B/C test
CREATE TABLE users (
    user_id INT PRIMARY KEY,
    name    VARCHAR(100),
    email   VARCHAR(200)
);

SELECT 
    user_id,
    name,
    NTILE(3) OVER (ORDER BY user_id) AS test_group,
    CASE NTILE(3) OVER (ORDER BY user_id)
        WHEN 1 THEN 'Control'
        WHEN 2 THEN 'Variant A'
        WHEN 3 THEN 'Variant B'
    END AS group_name
FROM users
ORDER BY test_group, user_id;
```

### ตัวอย่างที่ 16: NTILE สำหรับ Performance Buckets

```sql
-- แบ่งพนักงานเป็น 5 กลุ่มตาม performance
CREATE TABLE performance AS
SELECT emp_id, name, 
       (RANDOM() * 100)::INT AS performance_score
FROM employees;

SELECT 
    name,
    performance_score,
    NTILE(5) OVER (ORDER BY performance_score) AS performance_bucket,
    CASE NTILE(5) OVER (ORDER BY performance_score)
        WHEN 5 THEN 'Top Performer'
        WHEN 4 THEN 'Above Average'
        WHEN 3 THEN 'Average'
        WHEN 2 THEN 'Below Average'
        WHEN 1 THEN 'Needs Improvement'
    END AS bucket_label
FROM performance
ORDER BY performance_score DESC;
```

### ตัวอย่างที่ 17: NTILE กับ uneven distribution

```sql
-- เมื่อจำนวนแถวหารด้วย n ไม่ลงตัว
-- กลุ่มแรกๆ จะมีสมาชิกมากกว่า 1 คน
WITH data AS (
    SELECT generate_series(1, 7) AS n
)
SELECT 
    n,
    NTILE(3) OVER (ORDER BY n) AS bucket
FROM data;
-- 7 แถว / 3 กลุ่ม = กลุ่ม 1 มี 3 คน, กลุ่ม 2 มี 2 คน, กลุ่ม 3 มี 2 คน
```

---

## 93.7 Ranking with Ties Handling

### ตัวอย่างที่ 18: จัดการ Ties แตกต่างกัน

```sql
-- 3 วิธีจัดการ ties
SELECT 
    name,
    score,
    -- วิธี 1: แบ่งอันดับเท่ากัน (random)
    ROW_NUMBER() OVER (ORDER BY score DESC) AS rn_no_ties,
    -- วิธี 2: อันดับเดียวกัน ข้ามอันดับ
    RANK() OVER (ORDER BY score DESC) AS rank_with_gaps,
    -- วิธี 3: อันดับเดียวกัน ไม่ข้าม
    DENSE_RANK() OVER (ORDER BY score DESC) AS dense_no_gaps
FROM competition_scores
ORDER BY score DESC;
```

### ตัวอย่างที่ 19: Tie-breaking ด้วย secondary sort

```sql
-- แก้ tie ด้วย secondary criterion (hire_date เก่ากว่า = อันดับสูงกว่า)
SELECT 
    name,
    salary,
    hire_date,
    ROW_NUMBER() OVER (ORDER BY salary DESC, hire_date ASC) AS rank
FROM employees;
-- เงินเดือนเท่ากัน → คนที่ทำงานนานกว่าได้อันดับสูงกว่า
```

### ตัวอย่างที่ 20: สร้าง Leaderboard แบบ real world

```sql
-- Leaderboard พร้อม medal
WITH leaderboard AS (
    SELECT 
        contestant,
        SUM(score) AS total_score,
        RANK() OVER (ORDER BY SUM(score) DESC) AS position
    FROM competition_scores
    GROUP BY contestant
)
SELECT 
    position,
    contestant,
    total_score,
    CASE 
        WHEN position = 1 THEN '🥇 Gold'
        WHEN position = 2 THEN '🥈 Silver'
        WHEN position = 3 THEN '🥉 Bronze'
        ELSE 'Participant'
    END AS medal
FROM leaderboard
ORDER BY position;
```

---

## 93.8 Window Function vs GROUP BY

### ตัวอย่างที่ 21: ความแตกต่างสำคัญ

```sql
-- GROUP BY: ข้อมูลระดับแผนก เท่านั้น
SELECT department, AVG(salary), MAX(salary)
FROM employees
GROUP BY department;
-- ผล: 3 แถว - ไม่รู้ว่าพนักงานแต่ละคนเทียบกับ avg แผนกยังไง

-- Window Function: ข้อมูลระดับบุคคล + ข้อมูลระดับแผนก
SELECT 
    name,
    department,
    salary,
    AVG(salary) OVER (PARTITION BY department) AS dept_avg,
    MAX(salary) OVER (PARTITION BY department) AS dept_max,
    salary - AVG(salary) OVER (PARTITION BY department) AS diff_from_avg
FROM employees;
-- ผล: 10 แถว - เห็นทั้งข้อมูลบุคคลและ context ของแผนก
```

### ตัวอย่างที่ 22: รวม GROUP BY และ Window Function

```sql
-- ใช้ทั้งสองอย่างพร้อมกัน
WITH monthly_sales AS (
    SELECT 
        salesperson,
        DATE_TRUNC('month', sale_date) AS month,
        SUM(quantity * unit_price) AS monthly_revenue
    FROM sales
    GROUP BY salesperson, DATE_TRUNC('month', sale_date)
)
SELECT 
    salesperson,
    month,
    monthly_revenue,
    RANK() OVER (PARTITION BY month ORDER BY monthly_revenue DESC) AS rank_in_month,
    AVG(monthly_revenue) OVER (PARTITION BY salesperson) AS avg_monthly_revenue
FROM monthly_sales
ORDER BY month, rank_in_month;
```

---

## 93.9 Real-world Patterns

### ตัวอย่างที่ 23: Employee Salary Percentile

```sql
-- หาว่าแต่ละคนอยู่ที่ percentile ไหน
SELECT 
    name,
    department,
    salary,
    PERCENT_RANK() OVER (ORDER BY salary) AS percentile_overall,
    PERCENT_RANK() OVER (PARTITION BY department ORDER BY salary) AS percentile_in_dept,
    CUME_DIST() OVER (ORDER BY salary) AS cumulative_distribution
FROM employees;
```

### ตัวอย่างที่ 24: Top Products per Category

```sql
-- Top 3 สินค้าขายดีในแต่ละ category
CREATE TABLE product_sales (
    product_id  INT,
    product_name VARCHAR(100),
    category    VARCHAR(50),
    units_sold  INT
);

INSERT INTO product_sales VALUES
(1, 'Laptop A',  'Electronics', 500),
(2, 'Phone B',   'Electronics', 1200),
(3, 'Tablet C',  'Electronics', 300),
(4, 'TV D',      'Electronics', 150),
(5, 'Shirt E',   'Clothing',    800),
(6, 'Pants F',   'Clothing',    600),
(7, 'Shoes G',   'Clothing',    450),
(8, 'Hat H',     'Clothing',    200),
(9, 'Sofa I',    'Furniture',   50),
(10,'Desk J',    'Furniture',   80),
(11,'Chair K',   'Furniture',   120);

WITH ranked_products AS (
    SELECT 
        product_id,
        product_name,
        category,
        units_sold,
        RANK() OVER (PARTITION BY category ORDER BY units_sold DESC) AS rnk
    FROM product_sales
)
SELECT *
FROM ranked_products
WHERE rnk <= 3
ORDER BY category, rnk;
```

### ตัวอย่างที่ 25: Customer Purchase Rank

```sql
-- จัดอันดับลูกค้าตามยอดซื้อ
WITH customer_totals AS (
    SELECT 
        customer_id,
        SUM(total_amount) AS total_spent,
        COUNT(*) AS num_orders
    FROM orders
    GROUP BY customer_id
),
ranked_customers AS (
    SELECT 
        ct.*,
        DENSE_RANK() OVER (ORDER BY total_spent DESC) AS spending_rank,
        NTILE(10) OVER (ORDER BY total_spent DESC) AS decile  -- Top 10%, 20%, etc.
    FROM customer_totals ct
)
SELECT 
    customer_id,
    total_spent,
    num_orders,
    spending_rank,
    CASE decile
        WHEN 1 THEN 'Top 10%'
        WHEN 2 THEN 'Top 20%'
        WHEN 3 THEN 'Top 30%'
        ELSE 'Bottom ' || (decile * 10) || '%'
    END AS customer_tier
FROM ranked_customers
ORDER BY spending_rank
LIMIT 20;
```

### ตัวอย่างที่ 26: Latest Record per Entity

```sql
-- ดึง record ล่าสุดของแต่ละ account
CREATE TABLE account_events (
    event_id    INT,
    account_id  INT,
    event_type  VARCHAR(50),
    event_time  TIMESTAMP,
    details     TEXT
);

WITH latest_events AS (
    SELECT *,
           ROW_NUMBER() OVER (
               PARTITION BY account_id 
               ORDER BY event_time DESC
           ) AS rn
    FROM account_events
)
SELECT account_id, event_type, event_time, details
FROM latest_events
WHERE rn = 1;
```

### ตัวอย่างที่ 27: Score Distribution Analysis

```sql
-- วิเคราะห์การกระจายคะแนน
CREATE TABLE exam_scores (
    student_id  INT,
    subject     VARCHAR(50),
    score       INT
);

INSERT INTO exam_scores VALUES
(1, 'Math', 85), (2, 'Math', 92), (3, 'Math', 78),
(4, 'Math', 65), (5, 'Math', 88), (6, 'Math', 92),
(7, 'Math', 55), (8, 'Math', 70), (9, 'Math', 88),
(10,'Math', 95);

SELECT 
    student_id,
    subject,
    score,
    RANK() OVER (PARTITION BY subject ORDER BY score DESC) AS rank,
    DENSE_RANK() OVER (PARTITION BY subject ORDER BY score DESC) AS dense_rank,
    PERCENT_RANK() OVER (PARTITION BY subject ORDER BY score) AS pct_rank,
    NTILE(4) OVER (PARTITION BY subject ORDER BY score) AS quartile
FROM exam_scores
ORDER BY rank;
```

---

## 93.10 Advanced Ranking Patterns

### ตัวอย่างที่ 28: Conditional Ranking

```sql
-- จัดอันดับเฉพาะในเงื่อนไขที่กำหนด
SELECT 
    name,
    department,
    salary,
    -- จัดอันดับเฉพาะคนที่ทำงานมาก่อน 2021
    CASE WHEN hire_date < '2021-01-01' THEN
        RANK() OVER (
            PARTITION BY department
            ORDER BY CASE WHEN hire_date < '2021-01-01' THEN salary END DESC NULLS LAST
        )
    END AS veteran_rank
FROM employees;
```

### ตัวอย่างที่ 29: Ranking with NULL Handling

```sql
-- จัดการ NULL ใน ORDER BY
CREATE TABLE scores_with_nulls (
    player  VARCHAR(50),
    score   INT     -- อาจเป็น NULL (ไม่ได้เล่น)
);

INSERT INTO scores_with_nulls VALUES
('Alice', 95), ('Bob', NULL), ('Carol', 88), 
('Dave', NULL), ('Eve', 92);

SELECT 
    player,
    score,
    -- NULL อยู่ท้ายสุด (NULLS LAST)
    RANK() OVER (ORDER BY score DESC NULLS LAST) AS rank_nulls_last,
    -- NULL อยู่ต้น (NULLS FIRST)  
    RANK() OVER (ORDER BY score DESC NULLS FIRST) AS rank_nulls_first
FROM scores_with_nulls;
```

### ตัวอย่างที่ 30: Multi-level Ranking

```sql
-- จัดอันดับหลายระดับ
SELECT 
    name,
    department,
    salary,
    -- อันดับในบริษัท
    DENSE_RANK() OVER (ORDER BY salary DESC) AS company_rank,
    -- อันดับในแผนก
    DENSE_RANK() OVER (PARTITION BY department ORDER BY salary DESC) AS dept_rank,
    -- กลุ่มเงินเดือน (แบ่ง 3 กลุ่ม)
    NTILE(3) OVER (ORDER BY salary) AS salary_tier
FROM employees
ORDER BY department, salary DESC;
```

### ตัวอย่างที่ 31: Ranking with Subquery Filter

```sql
-- เฉพาะพนักงานที่สมัครงานหลัง 2020 และจัดอันดับใน group
WITH recent_hires AS (
    SELECT *
    FROM employees
    WHERE hire_date >= '2020-01-01'
),
ranked_recent AS (
    SELECT *,
           RANK() OVER (PARTITION BY department ORDER BY salary DESC) AS rank_in_dept
    FROM recent_hires
)
SELECT 
    name, department, salary, hire_date, rank_in_dept
FROM ranked_recent
WHERE rank_in_dept <= 2
ORDER BY department, rank_in_dept;
```

### ตัวอย่างที่ 32: Ranking for Report Generation

```sql
-- สร้าง report สรุปการจัดอันดับ
WITH 
dept_rankings AS (
    SELECT 
        department,
        name,
        salary,
        RANK() OVER (PARTITION BY department ORDER BY salary DESC) AS dept_rank,
        COUNT(*) OVER (PARTITION BY department) AS dept_size
    FROM employees
),
dept_stats AS (
    SELECT 
        department,
        SUM(salary) AS total_salary,
        AVG(salary) AS avg_salary,
        RANK() OVER (ORDER BY SUM(salary) DESC) AS dept_total_rank
    FROM employees
    GROUP BY department
)
SELECT 
    dr.department,
    ds.dept_total_rank AS dept_rank_by_payroll,
    dr.name,
    dr.dept_rank AS employee_rank,
    dr.dept_size,
    dr.salary,
    ROUND(ds.avg_salary, 2) AS dept_avg
FROM dept_rankings dr
JOIN dept_stats ds ON dr.department = ds.department
WHERE dr.dept_rank = 1  -- เฉพาะ top earner ในแต่ละแผนก
ORDER BY ds.dept_total_rank;
```

### ตัวอย่างที่ 33: Percentile Ranking

```sql
-- วิเคราะห์ percentile distribution ของเงินเดือน
SELECT 
    name,
    department,
    salary,
    ROUND(PERCENT_RANK() OVER (ORDER BY salary) * 100, 1) AS pct_rank,
    ROUND(CUME_DIST() OVER (ORDER BY salary) * 100, 1) AS cumulative_dist
FROM employees
ORDER BY salary;
-- PERCENT_RANK: (rank - 1) / (total - 1)
-- CUME_DIST:    rank / total
```

### ตัวอย่างที่ 34: Rolling Ranking (ตาม time window)

```sql
-- จัดอันดับยอดขายใน 3 เดือนที่ผ่านมาสำหรับแต่ละ salesperson
WITH monthly_rev AS (
    SELECT 
        salesperson,
        DATE_TRUNC('month', sale_date) AS month,
        SUM(quantity * unit_price) AS revenue
    FROM sales
    GROUP BY salesperson, DATE_TRUNC('month', sale_date)
),
rolling_3m AS (
    SELECT 
        salesperson,
        month,
        revenue,
        SUM(revenue) OVER (
            PARTITION BY salesperson
            ORDER BY month
            ROWS BETWEEN 2 PRECEDING AND CURRENT ROW
        ) AS rolling_3m_revenue
    FROM monthly_rev
),
ranked AS (
    SELECT *,
           RANK() OVER (PARTITION BY month ORDER BY rolling_3m_revenue DESC) AS rank_by_rolling
    FROM rolling_3m
)
SELECT *
FROM ranked
ORDER BY month, rank_by_rolling;
```

### ตัวอย่างที่ 35: Department Rank vs Company Rank

```sql
-- เปรียบเทียบอันดับระดับแผนกและระดับบริษัท
SELECT 
    name,
    department,
    salary,
    DENSE_RANK() OVER (ORDER BY salary DESC) AS company_rank,
    DENSE_RANK() OVER (PARTITION BY department ORDER BY salary DESC) AS dept_rank,
    -- ถ้า dept_rank < company_rank แสดงว่าเป็น "big fish in small pond"
    CASE 
        WHEN DENSE_RANK() OVER (PARTITION BY department ORDER BY salary DESC) <
             DENSE_RANK() OVER (ORDER BY salary DESC)
        THEN 'Big fish'
        WHEN DENSE_RANK() OVER (PARTITION BY department ORDER BY salary DESC) >
             DENSE_RANK() OVER (ORDER BY salary DESC)
        THEN 'Small fish'
        ELSE 'Consistent'
    END AS relative_standing
FROM employees
ORDER BY salary DESC;
```

---

## 93.11 ตัวอย่างเพิ่มเติม

### ตัวอย่างที่ 36: Employee of the Month

```sql
-- หา Employee of the Month แต่ละเดือน
CREATE TABLE monthly_performance (
    emp_id      INT,
    month       DATE,
    sales_count INT,
    revenue     DECIMAL(12,2),
    rating      DECIMAL(3,1)
);

WITH monthly_scores AS (
    SELECT 
        emp_id,
        month,
        (sales_count * 0.3 + revenue/1000 * 0.5 + rating * 10 * 0.2) AS composite_score
    FROM monthly_performance
),
ranked AS (
    SELECT 
        ms.*,
        RANK() OVER (PARTITION BY month ORDER BY composite_score DESC) AS rank
    FROM monthly_scores ms
)
SELECT 
    r.month,
    e.name AS employee_of_month,
    ROUND(r.composite_score, 2) AS score
FROM ranked r
JOIN employees e ON r.emp_id = e.emp_id
WHERE r.rank = 1
ORDER BY r.month;
```

### ตัวอย่างที่ 37: Class Rank for Report Card

```sql
-- สร้าง report card พร้อมอันดับ
WITH subject_ranks AS (
    SELECT 
        student_id,
        subject,
        score,
        RANK() OVER (PARTITION BY subject ORDER BY score DESC) AS subject_rank
    FROM exam_scores
),
overall_ranks AS (
    SELECT 
        student_id,
        AVG(score) AS avg_score,
        RANK() OVER (ORDER BY AVG(score) DESC) AS class_rank
    FROM exam_scores
    GROUP BY student_id
)
SELECT 
    or2.student_id,
    or2.avg_score,
    or2.class_rank,
    MAX(CASE WHEN sr.subject = 'Math' THEN sr.subject_rank END) AS math_rank,
    MAX(CASE WHEN sr.subject = 'Science' THEN sr.subject_rank END) AS science_rank
FROM overall_ranks or2
LEFT JOIN subject_ranks sr ON or2.student_id = sr.student_id
GROUP BY or2.student_id, or2.avg_score, or2.class_rank
ORDER BY or2.class_rank;
```

### ตัวอย่างที่ 38: Store Performance Ranking

```sql
-- จัดอันดับสาขาร้าน
CREATE TABLE store_metrics (
    store_id    INT,
    region      VARCHAR(50),
    metric      VARCHAR(50),
    value       DECIMAL(12,2)
);

WITH store_scores AS (
    SELECT 
        store_id,
        region,
        SUM(CASE WHEN metric = 'revenue' THEN value ELSE 0 END) AS revenue,
        SUM(CASE WHEN metric = 'customer_count' THEN value ELSE 0 END) AS customers,
        SUM(CASE WHEN metric = 'satisfaction_score' THEN value ELSE 0 END) AS satisfaction
    FROM store_metrics
    GROUP BY store_id, region
)
SELECT 
    store_id,
    region,
    revenue,
    customers,
    satisfaction,
    RANK() OVER (PARTITION BY region ORDER BY revenue DESC) AS revenue_rank_in_region,
    RANK() OVER (ORDER BY revenue DESC) AS overall_revenue_rank,
    DENSE_RANK() OVER (ORDER BY satisfaction DESC) AS satisfaction_rank
FROM store_scores
ORDER BY overall_revenue_rank;
```

### ตัวอย่างที่ 39: Rank Changes Over Time

```sql
-- ติดตามการเปลี่ยนแปลงอันดับ
WITH monthly_ranks AS (
    SELECT 
        product_id,
        DATE_TRUNC('month', sale_date) AS month,
        SUM(quantity) AS units_sold,
        RANK() OVER (
            PARTITION BY DATE_TRUNC('month', sale_date)
            ORDER BY SUM(quantity) DESC
        ) AS monthly_rank
    FROM sales s
    GROUP BY product_id, DATE_TRUNC('month', sale_date)
),
rank_changes AS (
    SELECT 
        product_id,
        month,
        units_sold,
        monthly_rank,
        LAG(monthly_rank) OVER (PARTITION BY product_id ORDER BY month) AS prev_rank
    FROM monthly_ranks
)
SELECT 
    product_id,
    month,
    units_sold,
    monthly_rank,
    prev_rank,
    CASE 
        WHEN prev_rank IS NULL THEN 'New Entry'
        WHEN monthly_rank < prev_rank THEN '▲ Up ' || (prev_rank - monthly_rank)
        WHEN monthly_rank > prev_rank THEN '▼ Down ' || (monthly_rank - prev_rank)
        ELSE '→ Same'
    END AS rank_change
FROM rank_changes
ORDER BY month, monthly_rank;
```

### ตัวอย่างที่ 40: Percentile Groups for Salary Bands

```sql
-- สร้าง salary bands ตาม percentile
WITH salary_stats AS (
    SELECT 
        name,
        department,
        salary,
        NTILE(10) OVER (ORDER BY salary) AS decile,
        NTILE(4)  OVER (ORDER BY salary) AS quartile,
        PERCENT_RANK() OVER (ORDER BY salary) * 100 AS percentile
    FROM employees
)
SELECT 
    name,
    department,
    salary,
    decile,
    quartile,
    ROUND(percentile, 1) AS percentile,
    CASE 
        WHEN percentile >= 90 THEN 'P90+ (Elite)'
        WHEN percentile >= 75 THEN 'P75-90 (High)'
        WHEN percentile >= 50 THEN 'P50-75 (Above Average)'
        WHEN percentile >= 25 THEN 'P25-50 (Below Average)'
        ELSE 'P0-25 (Low)'
    END AS salary_band
FROM salary_stats
ORDER BY salary DESC;
```

---

## แบบฝึกหัดท้ายบท

### แบบฝึกหัดที่ 1
เขียน query ที่แสดงพนักงาน พร้อมอันดับเงินเดือนทั้งในบริษัทและในแผนก โดยใช้ DENSE_RANK

**คำตอบ:**
```sql
SELECT 
    name,
    department,
    salary,
    DENSE_RANK() OVER (ORDER BY salary DESC) AS company_rank,
    DENSE_RANK() OVER (PARTITION BY department ORDER BY salary DESC) AS dept_rank
FROM employees
ORDER BY company_rank, department;
```

### แบบฝึกหัดที่ 2
หา top 3 ผู้ขายในแต่ละภูมิภาคตามยอดขายรวม

**คำตอบ:**
```sql
WITH regional_sales AS (
    SELECT 
        region,
        salesperson,
        SUM(quantity * unit_price) AS total_sales,
        RANK() OVER (PARTITION BY region ORDER BY SUM(quantity * unit_price) DESC) AS rank
    FROM sales
    GROUP BY region, salesperson
)
SELECT region, salesperson, total_sales, rank
FROM regional_sales
WHERE rank <= 3
ORDER BY region, rank;
```

### แบบฝึกหัดที่ 3
แบ่งนักเรียนออกเป็น 4 กลุ่มตามคะแนนสอบ (quartiles) และแสดงสถิติของแต่ละกลุ่ม

**คำตอบ:**
```sql
WITH quartiled AS (
    SELECT 
        student_id,
        subject,
        score,
        NTILE(4) OVER (PARTITION BY subject ORDER BY score) AS quartile
    FROM exam_scores
)
SELECT 
    subject,
    quartile,
    COUNT(*) AS student_count,
    MIN(score) AS min_score,
    MAX(score) AS max_score,
    ROUND(AVG(score), 2) AS avg_score
FROM quartiled
GROUP BY subject, quartile
ORDER BY subject, quartile;
```

### แบบฝึกหัดที่ 4
หาพนักงานที่มีอันดับเงินเดือนในแผนกสูงกว่าอันดับเงินเดือนในบริษัท (คน outstanding ในแผนกตัวเอง)

**คำตอบ:**
```sql
WITH rankings AS (
    SELECT 
        name,
        department,
        salary,
        DENSE_RANK() OVER (PARTITION BY department ORDER BY salary DESC) AS dept_rank,
        DENSE_RANK() OVER (ORDER BY salary DESC) AS company_rank
    FROM employees
)
SELECT name, department, salary, dept_rank, company_rank
FROM rankings
WHERE dept_rank < company_rank  -- อันดับในแผนกดีกว่าในบริษัท
ORDER BY dept_rank;
```

### แบบฝึกหัดที่ 5
สร้าง report แสดง ROW_NUMBER, RANK, DENSE_RANK ของยอดขายสินค้า พร้อม tie-breaking ที่ชัดเจน

**คำตอบ:**
```sql
SELECT 
    product_name,
    category,
    units_sold,
    ROW_NUMBER() OVER (ORDER BY units_sold DESC, product_name) AS row_num,
    RANK()       OVER (ORDER BY units_sold DESC) AS rank_with_gaps,
    DENSE_RANK() OVER (ORDER BY units_sold DESC) AS dense_rank,
    ROW_NUMBER() OVER (PARTITION BY category ORDER BY units_sold DESC) AS rank_in_category
FROM product_sales
ORDER BY units_sold DESC;
```

### แบบฝึกหัดที่ 6
แสดงพนักงานพร้อม percentile rank ของเงินเดือน และระบุว่าอยู่ใน "Top 25%" หรือไม่

**คำตอบ:**
```sql
SELECT 
    name,
    department,
    salary,
    ROUND(PERCENT_RANK() OVER (ORDER BY salary) * 100, 2) AS percentile,
    CASE 
        WHEN PERCENT_RANK() OVER (ORDER BY salary) >= 0.75 THEN 'YES - Top 25%'
        ELSE 'NO'
    END AS is_top_25_percent
FROM employees
ORDER BY salary DESC;
```

### แบบฝึกหัดที่ 7
หาสินค้าที่ขายดีที่สุดในแต่ละ category และแสดง % ของยอดขายใน category นั้น

**คำตอบ:**
```sql
WITH category_sales AS (
    SELECT 
        product_name,
        category,
        units_sold,
        RANK() OVER (PARTITION BY category ORDER BY units_sold DESC) AS rank,
        SUM(units_sold) OVER (PARTITION BY category) AS category_total
    FROM product_sales
)
SELECT 
    category,
    product_name,
    units_sold,
    ROUND(units_sold * 100.0 / category_total, 2) AS pct_of_category
FROM category_sales
WHERE rank = 1
ORDER BY category;
```

### แบบฝึกหัดที่ 8
สร้าง competition bracket: แสดงอันดับ 1-3 พร้อม medal สำหรับแต่ละ event

**คำตอบ:**
```sql
WITH ranked_contestants AS (
    SELECT 
        contestant,
        event,
        score,
        DENSE_RANK() OVER (PARTITION BY event ORDER BY score DESC) AS position
    FROM competition_scores
)
SELECT 
    event,
    position,
    contestant,
    score,
    CASE position
        WHEN 1 THEN 'Gold Medal'
        WHEN 2 THEN 'Silver Medal'
        WHEN 3 THEN 'Bronze Medal'
    END AS award
FROM ranked_contestants
WHERE position <= 3
ORDER BY event, position;
```

### แบบฝึกหัดที่ 9
หาพนักงานที่เงินเดือนสูงเป็นอันดับ 2 (second highest) ในแต่ละแผนก โดยใช้ DENSE_RANK

**คำตอบ:**
```sql
WITH salary_ranked AS (
    SELECT 
        name,
        department,
        salary,
        DENSE_RANK() OVER (PARTITION BY department ORDER BY salary DESC) AS rank
    FROM employees
)
SELECT name, department, salary
FROM salary_ranked
WHERE rank = 2
ORDER BY department;
```

### แบบฝึกหัดที่ 10
สร้าง quarterly ranking report: แสดงอันดับยอดขายรายไตรมาส และ rank change จากไตรมาสก่อน

**คำตอบ:**
```sql
WITH quarterly_sales AS (
    SELECT 
        salesperson,
        EXTRACT(YEAR FROM sale_date) AS yr,
        EXTRACT(QUARTER FROM sale_date) AS qtr,
        SUM(quantity * unit_price) AS revenue
    FROM sales
    GROUP BY salesperson, yr, qtr
),
with_ranks AS (
    SELECT 
        salesperson,
        yr,
        qtr,
        revenue,
        RANK() OVER (PARTITION BY yr, qtr ORDER BY revenue DESC) AS qtr_rank
    FROM quarterly_sales
),
with_prev_rank AS (
    SELECT *,
           LAG(qtr_rank) OVER (PARTITION BY salesperson ORDER BY yr, qtr) AS prev_qtr_rank
    FROM with_ranks
)
SELECT 
    salesperson,
    yr,
    qtr,
    revenue,
    qtr_rank,
    prev_qtr_rank,
    CASE 
        WHEN prev_qtr_rank IS NULL THEN 'First Quarter'
        WHEN qtr_rank < prev_qtr_rank THEN 'Moved Up ▲'
        WHEN qtr_rank > prev_qtr_rank THEN 'Moved Down ▼'
        ELSE 'No Change →'
    END AS rank_movement
FROM with_prev_rank
ORDER BY yr, qtr, qtr_rank;
```

---

## สรุปบทที่ 93

Ranking Functions ใน SQL มีหลายตัว แต่ละตัวเหมาะกับงานที่แตกต่างกัน:

| Function | Ties | Gaps | Use case |
|----------|------|------|----------|
| ROW_NUMBER() | ต่างกัน | ไม่มี | Pagination, deduplication |
| RANK() | เหมือนกัน | มี | Competition, leaderboard |
| DENSE_RANK() | เหมือนกัน | ไม่มี | Grade levels, version numbers |
| NTILE(n) | ตาม n | N/A | Buckets, percentiles |
| PERCENT_RANK() | - | - | Statistical percentile |
| CUME_DIST() | - | - | Cumulative distribution |

ในบทถัดไปเราจะเรียน Value Functions ได้แก่ LAG(), LEAD(), FIRST_VALUE(), LAST_VALUE() และ NTH_VALUE() ซึ่งใช้สำหรับวิเคราะห์ time series และการเปรียบเทียบ period-over-period
