# ภาค 30: Practical JOIN Projects

## บทสรุปการ JOIN ทั้งหมด

ในภาคนี้จะนำทุกอย่างที่เรียนมาประยุกต์ใช้จริงผ่าน Business Reports และ Projects ขนาดใหญ่

```
สิ่งที่ได้เรียนมาใน Parts 21-29:
Part 21: Table Relationships (One-to-One, One-to-Many, Many-to-Many)
Part 22: INNER JOIN (40+ examples)
Part 23: LEFT JOIN / RIGHT JOIN + Anti-join
Part 24: FULL OUTER JOIN + UNION workaround
Part 25: CROSS JOIN + Cartesian product
Part 26: Self JOIN + Recursive CTE + Hierarchy
Part 27: Multiple Table JOINs (4-6 tables)
Part 28: Advanced Techniques (non-equi, LATERAL, CTEs, Window Functions)
Part 29: Performance Optimization (EXPLAIN, indexes, query rewriting)

ภาค 30: รวมทุกอย่างมาสร้าง 10 Business Reports จริง
```

---

## Database Setup (ใช้ข้อมูลจาก Part 21)

```sql
-- หมายเหตุ: ใช้ตารางและข้อมูลที่สร้างใน Part 021
-- departments, employees, customers, products, orders, order_items
-- ถ้ายังไม่มีข้อมูล ให้รัน DDL และ INSERT จาก Part 021 ก่อน

-- ตรวจสอบข้อมูล
SELECT 'departments' AS table_name, COUNT(*) AS rows FROM departments
UNION ALL SELECT 'employees', COUNT(*) FROM employees
UNION ALL SELECT 'customers', COUNT(*) FROM customers
UNION ALL SELECT 'products', COUNT(*) FROM products
UNION ALL SELECT 'orders', COUNT(*) FROM orders
UNION ALL SELECT 'order_items', COUNT(*) FROM order_items;
```

---

## Project 1: Executive Sales Dashboard

**รายงานสรุปยอดขายรายเดือน สำหรับผู้บริหาร**

```sql
-- Project 1: Monthly Sales Executive Dashboard
-- จุดประสงค์: สรุปยอดขาย, จำนวนออเดอร์, ลูกค้าใหม่ รายเดือน
-- ใช้: INNER JOIN, LEFT JOIN, Aggregation, Window Functions, CTE

WITH monthly_orders AS (
    -- ดึง completed orders รายเดือน
    SELECT
        DATE_TRUNC('month', o.order_date) AS order_month,
        COUNT(DISTINCT o.order_id)         AS total_orders,
        COUNT(DISTINCT o.customer_id)      AS active_customers,
        SUM(oi.quantity * oi.unit_price - oi.discount) AS gross_revenue,
        SUM(oi.discount)                               AS total_discount,
        SUM(oi.quantity * oi.unit_price)               AS gross_sales
    FROM orders o
    JOIN order_items oi ON o.order_id = oi.order_id
    WHERE o.status IN ('completed', 'shipped')
    GROUP BY DATE_TRUNC('month', o.order_date)
),
monthly_new_customers AS (
    -- ลูกค้าที่สั่งครั้งแรกในแต่ละเดือน
    SELECT
        DATE_TRUNC('month', MIN(o.order_date)) AS first_order_month,
        COUNT(DISTINCT o.customer_id)          AS new_customers
    FROM orders o
    WHERE o.status IN ('completed', 'shipped')
    GROUP BY o.customer_id
    -- MIN(order_date) per customer = first order month
),
new_cust_agg AS (
    SELECT
        first_order_month,
        SUM(new_customers) AS new_customers
    FROM monthly_new_customers
    GROUP BY first_order_month
)
SELECT
    TO_CHAR(mo.order_month, 'YYYY-MM')          AS month,
    mo.total_orders,
    mo.active_customers,
    COALESCE(nc.new_customers, 0)               AS new_customers,
    mo.active_customers - COALESCE(nc.new_customers, 0) AS returning_customers,
    ROUND(mo.gross_revenue, 2)                  AS net_revenue,
    ROUND(mo.gross_sales, 2)                    AS gross_sales,
    ROUND(mo.total_discount, 2)                 AS total_discounts,
    ROUND(mo.gross_revenue / mo.total_orders, 2) AS avg_order_value,
    -- Month-over-month growth
    ROUND(
        (mo.gross_revenue - LAG(mo.gross_revenue) OVER (ORDER BY mo.order_month))
        / NULLIF(LAG(mo.gross_revenue) OVER (ORDER BY mo.order_month), 0) * 100
    , 1) AS mom_growth_pct
FROM monthly_orders mo
LEFT JOIN new_cust_agg nc ON mo.order_month = nc.first_order_month
ORDER BY mo.order_month;
```

---

## Project 2: Customer Lifetime Value (CLV) Report

**รายงาน Customer Lifetime Value และการจัดกลุ่มลูกค้า**

```sql
-- Project 2: Customer Lifetime Value and Segmentation
-- จุดประสงค์: คำนวณ CLV, จัดกลุ่ม VIP/Regular/Churn risk
-- ใช้: LEFT JOIN, Aggregation, Window Functions, CASE, NTILE

WITH customer_metrics AS (
    -- คำนวณ metrics ต่าง ๆ สำหรับแต่ละลูกค้า
    SELECT
        c.customer_id,
        c.first_name || ' ' || c.last_name     AS full_name,
        c.city,
        c.email,
        MIN(o.order_date)                       AS first_order_date,
        MAX(o.order_date)                       AS last_order_date,
        COUNT(DISTINCT o.order_id)              AS total_orders,
        COUNT(DISTINCT oi.item_id)              AS total_items_ordered,
        COALESCE(SUM(oi.quantity * oi.unit_price - oi.discount), 0) AS total_spend,
        COALESCE(AVG(oi.quantity * oi.unit_price - oi.discount), 0) AS avg_order_value,
        -- Days since last order
        CURRENT_DATE - MAX(o.order_date)        AS days_since_last_order,
        -- Customer lifespan in days
        MAX(o.order_date) - MIN(o.order_date)   AS lifespan_days
    FROM customers c
    LEFT JOIN orders o ON c.customer_id = o.customer_id
                       AND o.status IN ('completed', 'shipped')
    LEFT JOIN order_items oi ON o.order_id = oi.order_id
    GROUP BY c.customer_id, c.first_name, c.last_name, c.city, c.email
),
customer_ranked AS (
    SELECT
        *,
        -- CLV Score: total_spend ปรับตาม frequency และ recency
        ROUND(
            total_spend
            * (1 + LN(GREATEST(total_orders, 1)))
            * EXP(-0.001 * COALESCE(days_since_last_order, 999))
        , 2) AS clv_score,
        -- Percentile rank by spending
        NTILE(4) OVER (ORDER BY total_spend DESC)     AS spend_quartile,
        RANK() OVER (ORDER BY total_spend DESC)        AS spend_rank
    FROM customer_metrics
)
SELECT
    customer_id,
    full_name,
    city,
    total_orders,
    ROUND(total_spend, 2)              AS lifetime_value,
    ROUND(avg_order_value, 2)          AS avg_order_value,
    days_since_last_order,
    lifespan_days,
    clv_score,
    spend_rank,
    -- Customer Segment
    CASE
        WHEN total_orders = 0                                THEN 'Never Ordered'
        WHEN days_since_last_order <= 30 AND spend_quartile = 1 THEN 'VIP Active'
        WHEN days_since_last_order <= 60 AND spend_quartile <= 2 THEN 'Loyal Customer'
        WHEN days_since_last_order > 180                        THEN 'Churned Risk'
        WHEN days_since_last_order > 90                         THEN 'At Risk'
        ELSE 'Regular'
    END AS customer_segment
FROM customer_ranked
ORDER BY clv_score DESC NULLS LAST;
```

---

## Project 3: Product Performance Report

**รายงานประสิทธิภาพสินค้า — ขายดี, กำไร, สต็อก**

```sql
-- Project 3: Product Performance Analysis
-- จุดประสงค์: วิเคราะห์สินค้าขายดี, กำไร, ABC classification
-- ใช้: INNER JOIN, LEFT JOIN, GROUP BY, Window Functions, CASE

WITH product_sales AS (
    SELECT
        p.product_id,
        p.product_name,
        p.category,
        p.unit_price        AS current_price,
        p.cost_price,
        p.stock_quantity,
        -- Sales metrics
        COUNT(DISTINCT o.order_id)                              AS times_ordered,
        SUM(oi.quantity)                                        AS total_qty_sold,
        SUM(oi.quantity * oi.unit_price)                        AS gross_revenue,
        SUM(oi.quantity * oi.unit_price - oi.discount)          AS net_revenue,
        SUM(oi.quantity * oi.unit_price - oi.discount
            - oi.quantity * p.cost_price)                       AS gross_profit,
        -- Average selling price (may differ from current_price)
        ROUND(AVG(oi.unit_price), 2)                            AS avg_sell_price,
        -- Date range
        MIN(o.order_date)                                       AS first_sold_date,
        MAX(o.order_date)                                       AS last_sold_date
    FROM products p
    LEFT JOIN order_items oi ON p.product_id = oi.product_id
    LEFT JOIN orders o ON oi.order_id = o.order_id
                       AND o.status IN ('completed', 'shipped')
    GROUP BY p.product_id, p.product_name, p.category,
             p.unit_price, p.cost_price, p.stock_quantity
),
product_totals AS (
    SELECT SUM(net_revenue) AS total_revenue FROM product_sales WHERE net_revenue > 0
),
product_abc AS (
    SELECT
        ps.*,
        -- Cumulative revenue % for ABC analysis
        SUM(ps.net_revenue) OVER (ORDER BY ps.net_revenue DESC
            ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW)    AS cumulative_revenue,
        pt.total_revenue,
        ROUND(ps.net_revenue / NULLIF(pt.total_revenue, 0) * 100, 2)  AS revenue_share_pct,
        ROUND(ps.gross_profit / NULLIF(ps.net_revenue, 0) * 100, 2)   AS margin_pct,
        -- Rank within category
        RANK() OVER (PARTITION BY ps.category ORDER BY ps.net_revenue DESC) AS rank_in_category
    FROM product_sales ps
    CROSS JOIN product_totals pt
)
SELECT
    product_id,
    product_name,
    category,
    rank_in_category,
    current_price,
    ROUND(cost_price, 2)        AS cost_price,
    stock_quantity,
    total_qty_sold,
    times_ordered,
    ROUND(net_revenue, 2)       AS net_revenue,
    revenue_share_pct,
    ROUND(gross_profit, 2)      AS gross_profit,
    margin_pct,
    -- ABC Classification
    CASE
        WHEN cumulative_revenue / NULLIF(total_revenue, 0) <= 0.70 THEN 'A - Top Sellers'
        WHEN cumulative_revenue / NULLIF(total_revenue, 0) <= 0.90 THEN 'B - Mid Sellers'
        ELSE 'C - Low Sellers'
    END AS abc_class,
    -- Stock status
    CASE
        WHEN stock_quantity = 0              THEN 'Out of Stock'
        WHEN stock_quantity < total_qty_sold / NULLIF(
            GREATEST(EXTRACT(EPOCH FROM (MAX(last_sold_date) OVER ()) 
                     - MIN(first_sold_date) OVER ()) / 86400.0 / 30, 1)
        , 0) * 2 THEN 'Low Stock'
        ELSE 'In Stock'
    END AS stock_status
FROM product_abc
ORDER BY net_revenue DESC NULLS LAST;
```

---

## Project 4: Employee Hierarchy Report

**รายงานโครงสร้างองค์กรและประสิทธิภาพทีม**

```sql
-- Project 4: Employee Hierarchy and Team Performance
-- จุดประสงค์: แสดงโครงสร้าง manager-subordinate, เงินเดือนเปรียบเทียบ
-- ใช้: Self JOIN, Recursive CTE, Window Functions

WITH RECURSIVE org_tree AS (
    -- Base: CEO (no manager)
    SELECT
        emp_id,
        first_name || ' ' || last_name  AS full_name,
        job_title,
        dept_id,
        salary,
        manager_id,
        0                               AS depth,
        ARRAY[emp_id]                   AS path,
        (first_name || ' ' || last_name)::TEXT AS path_names
    FROM employees
    WHERE manager_id IS NULL

    UNION ALL

    -- Recursive: employees with managers
    SELECT
        e.emp_id,
        e.first_name || ' ' || e.last_name,
        e.job_title,
        e.dept_id,
        e.salary,
        e.manager_id,
        ot.depth + 1,
        ot.path || e.emp_id,
        ot.path_names || ' > ' || e.first_name || ' ' || e.last_name
    FROM employees e
    JOIN org_tree ot ON e.manager_id = ot.emp_id
),
dept_stats AS (
    -- Statistics per department
    SELECT
        e.dept_id,
        d.dept_name,
        COUNT(e.emp_id)           AS headcount,
        ROUND(AVG(e.salary), 2)   AS avg_salary,
        SUM(e.salary)             AS total_salary_budget,
        MAX(e.salary)             AS max_salary,
        MIN(e.salary)             AS min_salary
    FROM employees e
    JOIN departments d ON e.dept_id = d.dept_id
    GROUP BY e.dept_id, d.dept_name
),
direct_reports AS (
    -- Count direct reports per manager
    SELECT
        manager_id,
        COUNT(*) AS direct_report_count
    FROM employees
    WHERE manager_id IS NOT NULL
    GROUP BY manager_id
)
SELECT
    REPEAT('  ', ot.depth) || ot.full_name  AS org_chart,
    ot.full_name,
    ot.job_title,
    ds.dept_name,
    ot.depth                                AS hierarchy_level,
    ot.salary,
    -- Salary vs dept average
    ROUND(ot.salary - ds.avg_salary, 2)     AS salary_vs_dept_avg,
    ROUND(ot.salary / NULLIF(ds.avg_salary, 0) * 100 - 100, 1) AS pct_above_dept_avg,
    -- Rank within department
    RANK() OVER (PARTITION BY ot.dept_id ORDER BY ot.salary DESC) AS salary_rank_in_dept,
    -- Direct reports
    COALESCE(dr.direct_report_count, 0)     AS direct_reports,
    -- Manager name
    mgr.first_name || ' ' || mgr.last_name  AS manager_name,
    -- Full path
    ot.path_names                           AS reporting_chain
FROM org_tree ot
LEFT JOIN departments d ON ot.dept_id = d.dept_id  -- for dept_name in select
JOIN dept_stats ds ON ot.dept_id = ds.dept_id
LEFT JOIN direct_reports dr ON ot.emp_id = dr.manager_id
LEFT JOIN employees mgr ON ot.manager_id = mgr.emp_id
ORDER BY ot.path;
```

---

## Project 5: Inventory Reorder Report

**รายงานสินค้าที่ต้องสั่งซื้อเพิ่ม**

```sql
-- Project 5: Inventory Reorder Alert Report
-- จุดประสงค์: ระบุสินค้าที่ stock ต่ำ คำนวณยอดสั่งซื้อที่แนะนำ
-- ใช้: LEFT JOIN, Aggregation, CASE, Date arithmetic

WITH sales_velocity AS (
    -- คำนวณ average daily sales ในช่วง 30 วันที่ผ่านมา
    SELECT
        p.product_id,
        p.product_name,
        p.category,
        p.stock_quantity             AS current_stock,
        p.unit_price,
        p.cost_price,
        COALESCE(SUM(oi.quantity), 0) AS qty_sold_30d,
        -- Daily sales rate
        ROUND(COALESCE(SUM(oi.quantity), 0) / 30.0, 2) AS daily_sales_rate
    FROM products p
    LEFT JOIN order_items oi ON p.product_id = oi.product_id
    LEFT JOIN orders o ON oi.order_id = o.order_id
                       AND o.status IN ('completed', 'shipped')
                       AND o.order_date >= CURRENT_DATE - INTERVAL '30 days'
    GROUP BY p.product_id, p.product_name, p.category,
             p.stock_quantity, p.unit_price, p.cost_price
),
reorder_calc AS (
    SELECT
        *,
        -- Days of stock remaining
        CASE
            WHEN daily_sales_rate > 0
            THEN ROUND(current_stock / daily_sales_rate, 0)
            ELSE 9999
        END AS days_of_stock,
        -- Recommended reorder quantity (30-day supply + safety stock)
        GREATEST(
            ROUND(daily_sales_rate * 45, 0) - current_stock,  -- 45-day target
            0
        ) AS recommended_reorder_qty,
        -- Reorder point (14-day lead time)
        ROUND(daily_sales_rate * 14, 0) AS reorder_point
    FROM sales_velocity
)
SELECT
    rc.product_id,
    rc.product_name,
    rc.category,
    rc.current_stock,
    rc.reorder_point,
    rc.daily_sales_rate,
    rc.days_of_stock,
    rc.recommended_reorder_qty,
    ROUND(rc.recommended_reorder_qty * rc.cost_price, 2) AS estimated_reorder_cost,
    -- Priority
    CASE
        WHEN rc.current_stock = 0              THEN '🔴 OUT OF STOCK — Order Now'
        WHEN rc.days_of_stock <= 7             THEN '🟠 CRITICAL — < 7 days'
        WHEN rc.current_stock <= rc.reorder_point THEN '🟡 REORDER — Below Reorder Point'
        WHEN rc.days_of_stock <= 30            THEN '🔵 WATCH — 30 days or less'
        ELSE '✅ OK'
    END AS reorder_status
FROM reorder_calc rc
WHERE rc.days_of_stock <= 30 OR rc.current_stock = 0
ORDER BY
    CASE WHEN rc.current_stock = 0 THEN 0
         WHEN rc.days_of_stock <= 7 THEN 1
         WHEN rc.current_stock <= rc.reorder_point THEN 2
         ELSE 3 END,
    rc.days_of_stock;
```

---

## Project 6: Sales by Region Report

**รายงานยอดขายตามภูมิภาค/เมือง**

```sql
-- Project 6: Geographic Sales Analysis
-- จุดประสงค์: วิเคราะห์ยอดขายตาม city/region
-- ใช้: INNER JOIN, LEFT JOIN, GROUP BY ROLLUP, Window Functions

WITH city_sales AS (
    SELECT
        c.city,
        COUNT(DISTINCT c.customer_id)                               AS customer_count,
        COUNT(DISTINCT o.order_id)                                  AS order_count,
        SUM(oi.quantity * oi.unit_price - oi.discount)              AS net_revenue,
        SUM(oi.quantity)                                            AS units_sold,
        ROUND(AVG(oi.quantity * oi.unit_price - oi.discount), 2)    AS avg_order_value,
        -- Best selling category per city
        MODE() WITHIN GROUP (ORDER BY p.category)                   AS top_category
    FROM customers c
    JOIN orders o ON c.customer_id = o.customer_id
                  AND o.status IN ('completed', 'shipped')
    JOIN order_items oi ON o.order_id = oi.order_id
    JOIN products p ON oi.product_id = p.product_id
    GROUP BY c.city
)
SELECT
    COALESCE(city, 'GRAND TOTAL')           AS city,
    customer_count,
    order_count,
    ROUND(net_revenue, 2)                   AS net_revenue,
    units_sold,
    avg_order_value,
    top_category,
    -- Market share
    ROUND(
        net_revenue / SUM(net_revenue) OVER () * 100
    , 2)                                    AS market_share_pct,
    -- Rank
    RANK() OVER (ORDER BY net_revenue DESC) AS revenue_rank
FROM city_sales
ORDER BY net_revenue DESC;
```

---

## Project 7: Customer Purchase History

**รายงานประวัติการซื้อสินค้าของลูกค้าแต่ละราย**

```sql
-- Project 7: Customer Purchase History Timeline
-- จุดประสงค์: แสดง purchase history ครบถ้วน พร้อม pattern analysis
-- ใช้: Multiple JOINs, Window Functions (LAG, LEAD), CTE

WITH customer_orders AS (
    -- ทุก order พร้อมรายละเอียด
    SELECT
        c.customer_id,
        c.first_name || ' ' || c.last_name  AS customer_name,
        c.city,
        o.order_id,
        o.order_date,
        o.status,
        COUNT(oi.item_id)                   AS item_count,
        SUM(oi.quantity)                    AS total_qty,
        ROUND(SUM(oi.quantity * oi.unit_price - oi.discount), 2) AS order_total,
        -- Order number for this customer (1st, 2nd, ...)
        ROW_NUMBER() OVER (
            PARTITION BY c.customer_id
            ORDER BY o.order_date
        )                                   AS order_number,
        -- Days since previous order
        o.order_date - LAG(o.order_date) OVER (
            PARTITION BY c.customer_id
            ORDER BY o.order_date
        )                                   AS days_since_prev_order,
        -- Running total
        SUM(SUM(oi.quantity * oi.unit_price - oi.discount)) OVER (
            PARTITION BY c.customer_id
            ORDER BY o.order_date
            ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
        )                                   AS running_lifetime_value
    FROM customers c
    JOIN orders o ON c.customer_id = o.customer_id
    JOIN order_items oi ON o.order_id = oi.order_id
    WHERE o.status IN ('completed', 'shipped', 'processing')
    GROUP BY c.customer_id, c.first_name, c.last_name, c.city,
             o.order_id, o.order_date, o.status
),
order_products AS (
    -- สินค้าในแต่ละ order
    SELECT
        oi.order_id,
        STRING_AGG(p.product_name, ', ' ORDER BY p.product_name) AS products_bought
    FROM order_items oi
    JOIN products p ON oi.product_id = p.product_id
    GROUP BY oi.order_id
)
SELECT
    co.customer_id,
    co.customer_name,
    co.city,
    co.order_id,
    co.order_date,
    co.status,
    co.order_number,
    co.item_count,
    co.total_qty,
    co.order_total,
    COALESCE(co.days_since_prev_order::TEXT, 'First Order') AS days_between_orders,
    ROUND(co.running_lifetime_value, 2)  AS cumulative_spend,
    op.products_bought
FROM customer_orders co
JOIN order_products op ON co.order_id = op.order_id
ORDER BY co.customer_id, co.order_date;
```

---

## Project 8: Category Cross-Sell Analysis

**วิเคราะห์สินค้าที่ลูกค้ามักซื้อร่วมกัน (Market Basket)**

```sql
-- Project 8: Product Category Cross-Sell Opportunities
-- จุดประสงค์: หาสินค้าที่ถูกซื้อร่วมกันบ่อย (market basket analysis)
-- ใช้: Self JOIN บน order_items, Aggregation

-- Cross-sell: category pairs ที่ถูกซื้อใน order เดียวกัน
WITH order_categories AS (
    SELECT DISTINCT
        oi.order_id,
        p.category
    FROM order_items oi
    JOIN products p ON oi.product_id = p.product_id
),
category_pairs AS (
    -- Self JOIN: pair categories ที่อยู่ใน order เดียวกัน
    SELECT
        oc1.category    AS category_a,
        oc2.category    AS category_b,
        COUNT(DISTINCT oc1.order_id) AS co_occurrence_count
    FROM order_categories oc1
    JOIN order_categories oc2
        ON oc1.order_id = oc2.order_id
        AND oc1.category < oc2.category  -- avoid duplicates and self-pairs
    GROUP BY oc1.category, oc2.category
),
category_totals AS (
    -- Total orders per category
    SELECT
        category,
        COUNT(DISTINCT order_id) AS total_orders
    FROM order_categories
    GROUP BY category
)
SELECT
    cp.category_a,
    cp.category_b,
    cp.co_occurrence_count,
    ct_a.total_orders AS orders_with_a,
    ct_b.total_orders AS orders_with_b,
    -- Support: proportion of all orders with both categories
    ROUND(cp.co_occurrence_count::NUMERIC / (
        SELECT COUNT(DISTINCT order_id) FROM orders
        WHERE status IN ('completed','shipped')
    ) * 100, 2) AS support_pct,
    -- Confidence: given A, probability of buying B
    ROUND(cp.co_occurrence_count::NUMERIC / ct_a.total_orders * 100, 2) AS confidence_a_to_b_pct,
    -- Lift: how much more likely than random
    ROUND(
        cp.co_occurrence_count::NUMERIC
        * (SELECT COUNT(DISTINCT order_id) FROM orders WHERE status IN ('completed','shipped'))
        / (ct_a.total_orders::NUMERIC * ct_b.total_orders)
    , 3) AS lift
FROM category_pairs cp
JOIN category_totals ct_a ON cp.category_a = ct_a.category
JOIN category_totals ct_b ON cp.category_b = ct_b.category
WHERE cp.co_occurrence_count >= 2  -- minimum support
ORDER BY cp.co_occurrence_count DESC, lift DESC;
```

---

## Project 9: Sales Team Performance Report

**รายงานประสิทธิภาพทีมขาย — ยอดขายรายบุคคลและทีม**

```sql
-- Project 9: Sales Team and Department Performance KPIs
-- จุดประสงค์: KPI report สำหรับ HR/Management
-- ใช้: Multiple JOINs, Self JOIN (manager), Window Functions

WITH dept_sales AS (
    -- ยอดขายของแผนก (สมมติ employees ขาย = พนักงานในแผนก Sales)
    SELECT
        d.dept_id,
        d.dept_name,
        d.budget          AS dept_budget,
        COUNT(DISTINCT e.emp_id)        AS employee_count,
        SUM(e.salary)                   AS total_salary_expense,
        -- หมายเหตุ: ใน scenario นี้ไม่มี sales rep-to-order mapping
        -- จึงใช้ order stats ทั้งหมดเป็น benchmark
        d.budget / NULLIF(COUNT(DISTINCT e.emp_id), 0) AS budget_per_head
    FROM departments d
    LEFT JOIN employees e ON d.dept_id = e.dept_id
    GROUP BY d.dept_id, d.dept_name, d.budget
),
employee_details AS (
    SELECT
        e.emp_id,
        e.first_name || ' ' || e.last_name  AS emp_name,
        e.job_title,
        e.salary,
        e.hire_date,
        e.dept_id,
        d.dept_name,
        -- Manager name
        mgr.first_name || ' ' || mgr.last_name AS manager_name,
        -- Years at company
        EXTRACT(YEAR FROM AGE(CURRENT_DATE, e.hire_date)) AS years_at_company,
        -- Salary rank in department
        RANK() OVER (PARTITION BY e.dept_id ORDER BY e.salary DESC) AS salary_rank_dept,
        -- Salary vs dept average
        ROUND(e.salary - AVG(e.salary) OVER (PARTITION BY e.dept_id), 2) AS salary_vs_dept_avg,
        -- Salary percentile
        PERCENT_RANK() OVER (ORDER BY e.salary) * 100 AS salary_percentile
    FROM employees e
    JOIN departments d ON e.dept_id = d.dept_id
    LEFT JOIN employees mgr ON e.manager_id = mgr.emp_id
)
SELECT
    ed.dept_name,
    ed.emp_id,
    ed.emp_name,
    ed.job_title,
    ed.manager_name,
    ed.hire_date,
    ed.years_at_company,
    ed.salary,
    ed.salary_rank_dept,
    ROUND(ed.salary_vs_dept_avg, 2)     AS salary_vs_dept_avg,
    ROUND(ed.salary_percentile, 1)      AS salary_percentile,
    ds.employee_count                   AS dept_headcount,
    ds.total_salary_expense             AS dept_salary_cost,
    ds.dept_budget,
    ROUND(ds.total_salary_expense / NULLIF(ds.dept_budget, 0) * 100, 1) AS salary_to_budget_pct,
    -- Performance grade (simplified by salary percentile)
    CASE
        WHEN ed.salary_percentile >= 80 THEN 'Top Performer'
        WHEN ed.salary_percentile >= 50 THEN 'Good Performer'
        WHEN ed.salary_percentile >= 20 THEN 'Average'
        ELSE 'Needs Improvement'
    END AS performance_grade
FROM employee_details ed
JOIN dept_sales ds ON ed.dept_id = ds.dept_id
ORDER BY ed.dept_name, ed.salary DESC;
```

---

## Project 10: Complete E-Commerce Business Report

**รายงานธุรกิจ E-Commerce ฉบับสมบูรณ์ — Executive Summary**

```sql
-- Project 10: Complete E-Commerce Business Summary
-- จุดประสงค์: รวม KPIs ทั้งหมดในรายงานเดียว
-- ใช้: CTEs, Multiple JOINs, Window Functions, CASE

WITH
-- === SECTION 1: Revenue Overview ===
revenue_overview AS (
    SELECT
        COUNT(DISTINCT o.order_id)                                 AS total_orders,
        COUNT(DISTINCT CASE WHEN o.status = 'completed' THEN o.order_id END) AS completed_orders,
        COUNT(DISTINCT CASE WHEN o.status = 'cancelled' THEN o.order_id END) AS cancelled_orders,
        COUNT(DISTINCT o.customer_id)                              AS unique_customers,
        SUM(CASE WHEN o.status IN ('completed','shipped')
            THEN oi.quantity * oi.unit_price - oi.discount END)    AS total_revenue,
        AVG(CASE WHEN o.status IN ('completed','shipped')
            THEN oi.quantity * oi.unit_price - oi.discount END)    AS avg_order_value
    FROM orders o
    LEFT JOIN order_items oi ON o.order_id = oi.order_id
),

-- === SECTION 2: Top 5 Products ===
top_products AS (
    SELECT
        p.product_name,
        SUM(oi.quantity)                                    AS units_sold,
        ROUND(SUM(oi.quantity * oi.unit_price - oi.discount), 2) AS revenue,
        ROW_NUMBER() OVER (ORDER BY SUM(oi.quantity * oi.unit_price - oi.discount) DESC) AS rn
    FROM order_items oi
    JOIN products p ON oi.product_id = p.product_id
    JOIN orders o ON oi.order_id = o.order_id
    WHERE o.status IN ('completed', 'shipped')
    GROUP BY p.product_id, p.product_name
    LIMIT 5
),

-- === SECTION 3: Top 5 Customers ===
top_customers AS (
    SELECT
        c.first_name || ' ' || c.last_name                         AS customer_name,
        c.city,
        COUNT(DISTINCT o.order_id)                                  AS orders,
        ROUND(SUM(oi.quantity * oi.unit_price - oi.discount), 2)    AS lifetime_value,
        ROW_NUMBER() OVER (ORDER BY SUM(oi.quantity * oi.unit_price - oi.discount) DESC) AS rn
    FROM customers c
    JOIN orders o ON c.customer_id = o.customer_id
    JOIN order_items oi ON o.order_id = oi.order_id
    WHERE o.status IN ('completed', 'shipped')
    GROUP BY c.customer_id, c.first_name, c.last_name, c.city
    LIMIT 5
),

-- === SECTION 4: Category Performance ===
category_perf AS (
    SELECT
        p.category,
        COUNT(DISTINCT p.product_id)                                AS product_count,
        SUM(oi.quantity)                                            AS units_sold,
        ROUND(SUM(oi.quantity * oi.unit_price - oi.discount), 2)    AS revenue,
        ROUND(SUM(oi.quantity * oi.unit_price - oi.discount
                  - oi.quantity * p.cost_price), 2)                  AS gross_profit
    FROM products p
    LEFT JOIN order_items oi ON p.product_id = oi.product_id
    LEFT JOIN orders o ON oi.order_id = o.order_id
                       AND o.status IN ('completed', 'shipped')
    GROUP BY p.category
),

-- === SECTION 5: Stock Alerts ===
stock_alerts AS (
    SELECT COUNT(*) AS low_stock_products
    FROM products
    WHERE stock_quantity < 10
)

-- === FINAL ASSEMBLY ===
-- Revenue Overview
SELECT 'REVENUE OVERVIEW' AS report_section, NULL::TEXT AS detail_1, NULL::TEXT AS detail_2,
       total_orders::TEXT AS metric_1, total_revenue::TEXT AS metric_2
FROM revenue_overview

UNION ALL SELECT '--- Total Orders', NULL, NULL, total_orders::TEXT, NULL FROM revenue_overview
UNION ALL SELECT '--- Completed Orders', NULL, NULL, completed_orders::TEXT, NULL FROM revenue_overview
UNION ALL SELECT '--- Total Revenue (THB)', NULL, NULL, ROUND(total_revenue::NUMERIC,2)::TEXT, NULL FROM revenue_overview
UNION ALL SELECT '--- Avg Order Value', NULL, NULL, ROUND(avg_order_value::NUMERIC,2)::TEXT, NULL FROM revenue_overview
UNION ALL SELECT '--- Unique Customers', NULL, NULL, unique_customers::TEXT, NULL FROM revenue_overview

UNION ALL SELECT '', NULL, NULL, NULL, NULL  -- spacer

UNION ALL SELECT 'TOP 5 PRODUCTS', 'Product', 'Units Sold', 'Revenue', NULL FROM (SELECT 1) t
UNION ALL SELECT '  #' || rn, product_name, units_sold::TEXT, revenue::TEXT, NULL FROM top_products

UNION ALL SELECT '', NULL, NULL, NULL, NULL

UNION ALL SELECT 'TOP 5 CUSTOMERS', 'Customer', 'City', 'Orders', 'Lifetime Value'  FROM (SELECT 1) t
UNION ALL SELECT '  #' || rn, customer_name, city, orders::TEXT, lifetime_value::TEXT FROM top_customers

UNION ALL SELECT '', NULL, NULL, NULL, NULL

UNION ALL SELECT 'CATEGORY PERFORMANCE', 'Category', 'Products', 'Revenue', 'Gross Profit' FROM (SELECT 1) t
UNION ALL SELECT '  ' || category, category, product_count::TEXT, revenue::TEXT, gross_profit::TEXT FROM category_perf

UNION ALL SELECT '', NULL, NULL, NULL, NULL

UNION ALL SELECT 'STOCK ALERTS', 'Products with stock < 10', low_stock_products::TEXT, NULL, NULL
FROM stock_alerts;
```

---

## แบบฝึกหัดภาค 30

**ข้อ 1:** เขียน query แสดง Monthly Revenue พร้อม Month-over-Month Growth %

```sql
-- เฉลย
WITH monthly_revenue AS (
    SELECT
        DATE_TRUNC('month', o.order_date)           AS month,
        ROUND(SUM(oi.quantity * oi.unit_price - oi.discount), 2) AS revenue
    FROM orders o
    JOIN order_items oi ON o.order_id = oi.order_id
    WHERE o.status IN ('completed', 'shipped')
    GROUP BY DATE_TRUNC('month', o.order_date)
)
SELECT
    TO_CHAR(month, 'YYYY-MM')                       AS month,
    revenue,
    LAG(revenue) OVER (ORDER BY month)              AS prev_month_revenue,
    ROUND(
        (revenue - LAG(revenue) OVER (ORDER BY month))
        / NULLIF(LAG(revenue) OVER (ORDER BY month), 0) * 100
    , 1)                                            AS mom_growth_pct
FROM monthly_revenue
ORDER BY month;
```

**ข้อ 2:** เขียน query หา Top 3 Products ในแต่ละ Category

```sql
-- เฉลย
WITH product_sales AS (
    SELECT
        p.category,
        p.product_name,
        SUM(oi.quantity * oi.unit_price - oi.discount)  AS revenue,
        RANK() OVER (
            PARTITION BY p.category
            ORDER BY SUM(oi.quantity * oi.unit_price - oi.discount) DESC
        ) AS rank_in_cat
    FROM products p
    JOIN order_items oi ON p.product_id = oi.product_id
    JOIN orders o ON oi.order_id = o.order_id
    WHERE o.status IN ('completed', 'shipped')
    GROUP BY p.category, p.product_id, p.product_name
)
SELECT category, rank_in_cat, product_name, ROUND(revenue, 2) AS revenue
FROM product_sales
WHERE rank_in_cat <= 3
ORDER BY category, rank_in_cat;
```

**ข้อ 3:** เขียน query แสดง Customers ที่ไม่ได้สั่งซื้อใน 60 วันที่ผ่านมา

```sql
-- เฉลย (Anti-join + Date filter)
SELECT
    c.customer_id,
    c.first_name || ' ' || c.last_name  AS customer_name,
    c.city,
    c.email,
    MAX(o.order_date)                   AS last_order_date,
    CURRENT_DATE - MAX(o.order_date)    AS days_inactive
FROM customers c
LEFT JOIN orders o ON c.customer_id = o.customer_id
                   AND o.status IN ('completed', 'shipped')
GROUP BY c.customer_id, c.first_name, c.last_name, c.city, c.email
HAVING MAX(o.order_date) < CURRENT_DATE - INTERVAL '60 days'
    OR MAX(o.order_date) IS NULL  -- ไม่เคยสั่งเลย
ORDER BY days_inactive DESC NULLS FIRST;
```

**ข้อ 4:** เขียน query แสดง Employee ที่มีเงินเดือนสูงกว่า Manager

```sql
-- เฉลย
SELECT
    e.first_name || ' ' || e.last_name  AS employee_name,
    e.job_title,
    e.salary                            AS emp_salary,
    mgr.first_name || ' ' || mgr.last_name AS manager_name,
    mgr.job_title                       AS mgr_title,
    mgr.salary                          AS mgr_salary,
    e.salary - mgr.salary               AS salary_diff
FROM employees e
JOIN employees mgr ON e.manager_id = mgr.emp_id
WHERE e.salary > mgr.salary
ORDER BY salary_diff DESC;
```

**ข้อ 5:** เขียน query แสดง Products ที่ไม่เคยถูกสั่งซื้อ

```sql
-- เฉลย (Anti-join)
SELECT
    p.product_id,
    p.product_name,
    p.category,
    p.unit_price,
    p.stock_quantity
FROM products p
LEFT JOIN order_items oi ON p.product_id = oi.product_id
WHERE oi.item_id IS NULL
ORDER BY p.category, p.product_name;
```

**ข้อ 6:** เขียน query แสดง Average Order Value ตาม Customer City

```sql
-- เฉลย
SELECT
    c.city,
    COUNT(DISTINCT c.customer_id)                               AS customer_count,
    COUNT(DISTINCT o.order_id)                                  AS total_orders,
    ROUND(SUM(oi.quantity * oi.unit_price - oi.discount), 2)    AS total_revenue,
    ROUND(
        SUM(oi.quantity * oi.unit_price - oi.discount)
        / NULLIF(COUNT(DISTINCT o.order_id), 0)
    , 2)                                                        AS avg_order_value
FROM customers c
JOIN orders o ON c.customer_id = o.customer_id
             AND o.status IN ('completed', 'shipped')
JOIN order_items oi ON o.order_id = oi.order_id
GROUP BY c.city
ORDER BY avg_order_value DESC;
```

**ข้อ 7:** เขียน query แสดงลำดับ Employee ตาม Hierarchy (CEO → Manager → Staff)

```sql
-- เฉลย (Recursive CTE)
WITH RECURSIVE hierarchy AS (
    SELECT emp_id, first_name || ' ' || last_name AS name,
           job_title, manager_id, 0 AS level,
           first_name || ' ' || last_name AS chain
    FROM employees WHERE manager_id IS NULL
    UNION ALL
    SELECT e.emp_id, e.first_name || ' ' || e.last_name,
           e.job_title, e.manager_id, h.level + 1,
           h.chain || ' > ' || e.first_name || ' ' || e.last_name
    FROM employees e JOIN hierarchy h ON e.manager_id = h.emp_id
)
SELECT
    REPEAT('  ', level) || name   AS org_chart,
    job_title, level, chain
FROM hierarchy
ORDER BY chain;
```

**ข้อ 8:** เขียน query แสดง Customers ที่ซื้อสินค้าทุก Category

```sql
-- เฉลย
SELECT
    c.customer_id,
    c.first_name || ' ' || c.last_name   AS customer_name,
    COUNT(DISTINCT p.category)            AS categories_bought
FROM customers c
JOIN orders o ON c.customer_id = o.customer_id
             AND o.status IN ('completed', 'shipped')
JOIN order_items oi ON o.order_id = oi.order_id
JOIN products p ON oi.product_id = p.product_id
GROUP BY c.customer_id, c.first_name, c.last_name
HAVING COUNT(DISTINCT p.category) = (SELECT COUNT(DISTINCT category) FROM products)
ORDER BY categories_bought DESC;
```

**ข้อ 9:** เขียน query หา Product ที่ขายดีที่สุดในแต่ละเดือน

```sql
-- เฉลย
WITH monthly_product_sales AS (
    SELECT
        DATE_TRUNC('month', o.order_date)           AS month,
        p.product_id,
        p.product_name,
        SUM(oi.quantity * oi.unit_price - oi.discount) AS revenue,
        RANK() OVER (
            PARTITION BY DATE_TRUNC('month', o.order_date)
            ORDER BY SUM(oi.quantity * oi.unit_price - oi.discount) DESC
        ) AS rank_in_month
    FROM orders o
    JOIN order_items oi ON o.order_id = oi.order_id
    JOIN products p ON oi.product_id = p.product_id
    WHERE o.status IN ('completed', 'shipped')
    GROUP BY DATE_TRUNC('month', o.order_date), p.product_id, p.product_name
)
SELECT
    TO_CHAR(month, 'YYYY-MM')   AS month,
    product_name                AS best_seller,
    ROUND(revenue, 2)           AS monthly_revenue
FROM monthly_product_sales
WHERE rank_in_month = 1
ORDER BY month;
```

**ข้อ 10:** เขียน Complete Business Health Check Query

```sql
-- เฉลย: Business Health Scorecard
WITH
revenue AS (
    SELECT ROUND(SUM(oi.quantity * oi.unit_price - oi.discount), 2) AS total
    FROM orders o JOIN order_items oi ON o.order_id = oi.order_id
    WHERE o.status IN ('completed','shipped')
),
prev_revenue AS (
    SELECT ROUND(SUM(oi.quantity * oi.unit_price - oi.discount), 2) AS total
    FROM orders o JOIN order_items oi ON o.order_id = oi.order_id
    WHERE o.status IN ('completed','shipped')
      AND o.order_date < DATE_TRUNC('month', CURRENT_DATE)
),
customer_stats AS (
    SELECT
        COUNT(DISTINCT customer_id) AS total,
        COUNT(DISTINCT CASE WHEN order_date >= CURRENT_DATE - 30 THEN customer_id END) AS active_30d
    FROM orders WHERE status IN ('completed','shipped')
),
order_stats AS (
    SELECT
        COUNT(*) AS total,
        COUNT(CASE WHEN status = 'cancelled' THEN 1 END) AS cancelled,
        ROUND(COUNT(CASE WHEN status = 'cancelled' THEN 1 END)::NUMERIC / NULLIF(COUNT(*),0)*100,1) AS cancel_rate
    FROM orders
),
stock_stat AS (
    SELECT COUNT(*) AS out_of_stock FROM products WHERE stock_quantity = 0
)
SELECT 'Total Revenue' AS metric, r.total::TEXT AS value, NULL AS note FROM revenue r
UNION ALL SELECT 'Active Customers (30d)', cs.active_30d::TEXT, cs.total::TEXT || ' total' FROM customer_stats cs
UNION ALL SELECT 'Total Orders', os.total::TEXT, 'Cancel Rate: ' || os.cancel_rate || '%' FROM order_stats os
UNION ALL SELECT 'Out of Stock Products', ss.out_of_stock::TEXT, 'Needs restock' FROM stock_stat ss;
```

---

## สรุปภาค 30 และ Parts 21-30 ทั้งหมด

### สิ่งที่ได้เรียนใน Parts 21-30

| Part | หัวข้อ | เนื้อหาสำคัญ |
|------|--------|-------------|
| 21 | Table Relationships | One-to-One, One-to-Many, Many-to-Many, ERD |
| 22 | INNER JOIN | 40+ examples, syntax, USING |
| 23 | LEFT/RIGHT JOIN | Anti-join, ON vs WHERE |
| 24 | FULL OUTER JOIN | UNION workaround, symmetric difference |
| 25 | CROSS JOIN | Cartesian product, date generation |
| 26 | Self JOIN | Hierarchy, Recursive CTE |
| 27 | Multiple Table JOINs | 4-6 tables, aliasing |
| 28 | Advanced Techniques | Non-equi, LATERAL, Window Functions |
| 29 | Performance | EXPLAIN, indexes, query rewriting |
| 30 | Projects | 10 business reports |

### JOIN Quick Reference

```sql
-- INNER JOIN: แถวที่ match ทั้งคู่
SELECT * FROM A INNER JOIN B ON A.id = B.a_id;

-- LEFT JOIN: แถวทั้งหมดจาก A + match จาก B (NULL ถ้าไม่ match)
SELECT * FROM A LEFT JOIN B ON A.id = B.a_id;

-- RIGHT JOIN: แถวทั้งหมดจาก B + match จาก A
SELECT * FROM A RIGHT JOIN B ON A.id = B.a_id;

-- FULL OUTER JOIN: แถวทั้งหมดจากทั้งสองตาราง
SELECT * FROM A FULL OUTER JOIN B ON A.id = B.a_id;
-- MySQL: LEFT JOIN UNION ALL RIGHT JOIN WHERE A.id IS NULL

-- CROSS JOIN: Cartesian product
SELECT * FROM A CROSS JOIN B;

-- Self JOIN: join ตารางกับตัวเอง
SELECT e.name, m.name AS manager FROM employees e LEFT JOIN employees m ON e.manager_id = m.emp_id;

-- Anti-join: แถวใน A ที่ไม่มีคู่ใน B
SELECT * FROM A LEFT JOIN B ON A.id = B.a_id WHERE B.a_id IS NULL;

-- Non-equi JOIN: JOIN ด้วย range condition
SELECT * FROM employees e JOIN salary_grades sg ON e.salary BETWEEN sg.min_salary AND sg.max_salary;
```

**ยินดีด้วย! คุณได้เรียน SQL JOINs ครบทุกรูปแบบแล้ว**
**ในส่วนถัดไป (Parts 31+) จะเรียน Subqueries, Window Functions, CTEs เชิงลึก และ Advanced SQL**
