# Part 112: E-commerce Analytics Dashboard Queries

## บทนำ (Introduction)

Analytics Dashboard เป็นหัวใจสำคัญของธุรกิจ E-commerce ช่วยให้ผู้บริหารตัดสินใจได้อย่างมีข้อมูล บทนี้จะครอบคลุม 30+ Query สำหรับวิเคราะห์ธุรกิจในทุกมิติ

## ฐานข้อมูลที่ใช้

ใช้ Schema จาก Part 111 (E-commerce Database)

---

## SECTION 1: Revenue Reports

### Query 1: Daily Revenue Report (รายงานยอดขายรายวัน)

```sql
-- รายงานยอดขายรายวัน พร้อม Running Total และ % เปลี่ยนแปลง
WITH daily_revenue AS (
    SELECT 
        DATE(created_at) AS sale_date,
        COUNT(DISTINCT order_id) AS orders,
        COUNT(DISTINCT customer_id) AS unique_customers,
        SUM(total_amount) AS revenue,
        SUM(discount_amount) AS discounts_given,
        SUM(shipping_fee) AS shipping_revenue,
        SUM(tax_amount) AS tax_collected,
        AVG(total_amount) AS avg_order_value
    FROM orders
    WHERE payment_status = 'paid'
      AND status NOT IN ('cancelled', 'refunded')
      AND created_at >= DATE_SUB(CURDATE(), INTERVAL 30 DAY)
    GROUP BY DATE(created_at)
),
revenue_with_lag AS (
    SELECT 
        *,
        LAG(revenue) OVER (ORDER BY sale_date) AS prev_day_revenue,
        SUM(revenue) OVER (ORDER BY sale_date ROWS UNBOUNDED PRECEDING) AS cumulative_revenue
    FROM daily_revenue
)
SELECT 
    sale_date,
    orders,
    unique_customers,
    ROUND(revenue, 2) AS revenue,
    ROUND(avg_order_value, 2) AS avg_order_value,
    ROUND(cumulative_revenue, 2) AS cumulative_revenue,
    CASE 
        WHEN prev_day_revenue IS NOT NULL AND prev_day_revenue > 0
        THEN ROUND((revenue - prev_day_revenue) * 100.0 / prev_day_revenue, 1)
        ELSE NULL 
    END AS day_over_day_pct
FROM revenue_with_lag
ORDER BY sale_date DESC;
```

**ผลลัพธ์ที่คาดหวัง:**
```
sale_date   | orders | unique_customers | revenue    | avg_order_value | cumulative_revenue | day_over_day_pct
------------|--------|-----------------|------------|-----------------|-------------------|------------------
2024-03-20  |   15   |       14        |  45,230.00 |    3,015.33     |    1,234,567.00    |       +12.3
2024-03-19  |   13   |       12        |  40,290.00 |    3,099.23     |    1,189,337.00    |        -5.6
```

---

### Query 2: Weekly Revenue Comparison (เปรียบเทียบรายสัปดาห์)

```sql
SELECT 
    YEAR(created_at) AS year,
    WEEK(created_at, 1) AS week_number,
    MIN(DATE(created_at)) AS week_start,
    MAX(DATE(created_at)) AS week_end,
    COUNT(DISTINCT order_id) AS total_orders,
    SUM(total_amount) AS revenue,
    ROUND(SUM(total_amount) / COUNT(DISTINCT order_id), 2) AS aov,
    LAG(SUM(total_amount)) OVER (ORDER BY YEAR(created_at), WEEK(created_at, 1)) AS prev_week_revenue,
    ROUND(
        (SUM(total_amount) - LAG(SUM(total_amount)) OVER (ORDER BY YEAR(created_at), WEEK(created_at, 1))) 
        * 100.0 / NULLIF(LAG(SUM(total_amount)) OVER (ORDER BY YEAR(created_at), WEEK(created_at, 1)), 0),
        1
    ) AS week_over_week_pct
FROM orders
WHERE payment_status = 'paid'
  AND status NOT IN ('cancelled', 'refunded')
GROUP BY YEAR(created_at), WEEK(created_at, 1)
ORDER BY year DESC, week_number DESC
LIMIT 12;
```

---

### Query 3: Monthly Revenue with YoY Growth (เดือนต่อเดือน + ปีต่อปี)

```sql
WITH monthly_data AS (
    SELECT 
        YEAR(created_at) AS yr,
        MONTH(created_at) AS mo,
        DATE_FORMAT(created_at, '%Y-%m') AS period,
        COUNT(DISTINCT order_id) AS orders,
        COUNT(DISTINCT customer_id) AS customers,
        SUM(total_amount) AS revenue,
        AVG(total_amount) AS aov
    FROM orders
    WHERE payment_status = 'paid'
      AND status NOT IN ('cancelled', 'refunded')
    GROUP BY YEAR(created_at), MONTH(created_at), DATE_FORMAT(created_at, '%Y-%m')
)
SELECT 
    curr.period,
    curr.orders,
    curr.customers,
    ROUND(curr.revenue, 2) AS revenue,
    ROUND(curr.aov, 2) AS avg_order_value,
    -- MoM Growth
    ROUND(
        (curr.revenue - LAG(curr.revenue) OVER (ORDER BY curr.yr, curr.mo)) 
        * 100.0 / NULLIF(LAG(curr.revenue) OVER (ORDER BY curr.yr, curr.mo), 0), 
        1
    ) AS mom_growth_pct,
    -- YoY Growth
    ROUND(
        (curr.revenue - prev_year.revenue) 
        * 100.0 / NULLIF(prev_year.revenue, 0), 
        1
    ) AS yoy_growth_pct
FROM monthly_data curr
LEFT JOIN monthly_data prev_year 
    ON curr.yr - 1 = prev_year.yr 
    AND curr.mo = prev_year.mo
ORDER BY curr.yr DESC, curr.mo DESC;
```

---

### Query 4: Yearly Revenue Summary

```sql
SELECT 
    YEAR(created_at) AS year,
    COUNT(DISTINCT order_id) AS total_orders,
    COUNT(DISTINCT customer_id) AS unique_customers,
    SUM(CASE WHEN MONTH(created_at) = 1 THEN total_amount ELSE 0 END) AS jan,
    SUM(CASE WHEN MONTH(created_at) = 2 THEN total_amount ELSE 0 END) AS feb,
    SUM(CASE WHEN MONTH(created_at) = 3 THEN total_amount ELSE 0 END) AS mar,
    SUM(CASE WHEN MONTH(created_at) = 4 THEN total_amount ELSE 0 END) AS apr,
    SUM(CASE WHEN MONTH(created_at) = 5 THEN total_amount ELSE 0 END) AS may_rev,
    SUM(CASE WHEN MONTH(created_at) = 6 THEN total_amount ELSE 0 END) AS jun,
    SUM(CASE WHEN MONTH(created_at) = 7 THEN total_amount ELSE 0 END) AS jul,
    SUM(CASE WHEN MONTH(created_at) = 8 THEN total_amount ELSE 0 END) AS aug,
    SUM(CASE WHEN MONTH(created_at) = 9 THEN total_amount ELSE 0 END) AS sep,
    SUM(CASE WHEN MONTH(created_at) = 10 THEN total_amount ELSE 0 END) AS oct_rev,
    SUM(CASE WHEN MONTH(created_at) = 11 THEN total_amount ELSE 0 END) AS nov,
    SUM(CASE WHEN MONTH(created_at) = 12 THEN total_amount ELSE 0 END) AS dec_rev,
    SUM(total_amount) AS annual_total
FROM orders
WHERE payment_status = 'paid'
  AND status NOT IN ('cancelled', 'refunded')
GROUP BY YEAR(created_at)
ORDER BY year DESC;
```

---

## SECTION 2: Product Analytics

### Query 5: Best-Selling Products (สินค้าขายดีที่สุด)

```sql
SELECT 
    RANK() OVER (ORDER BY SUM(oi.quantity) DESC) AS rank_by_units,
    DENSE_RANK() OVER (ORDER BY SUM(oi.total_price) DESC) AS rank_by_revenue,
    p.product_id,
    p.name AS product_name,
    b.name AS brand,
    c.name AS category,
    SUM(oi.quantity) AS units_sold,
    SUM(oi.total_price) AS total_revenue,
    COUNT(DISTINCT oi.order_id) AS times_ordered,
    ROUND(AVG(oi.unit_price), 2) AS avg_selling_price,
    ROUND(SUM(oi.total_price) / SUM(oi.quantity), 2) AS revenue_per_unit,
    -- Market share within category
    ROUND(
        SUM(oi.total_price) * 100.0 / 
        SUM(SUM(oi.total_price)) OVER (PARTITION BY p.category_id),
        1
    ) AS category_revenue_share_pct
FROM order_items oi
JOIN orders o ON oi.order_id = o.order_id
JOIN product_variants pv ON oi.variant_id = pv.variant_id
JOIN products p ON pv.product_id = p.product_id
LEFT JOIN brands b ON p.brand_id = b.brand_id
JOIN categories c ON p.category_id = c.category_id
WHERE o.payment_status = 'paid'
  AND o.status NOT IN ('cancelled', 'refunded')
  AND o.created_at >= DATE_SUB(CURDATE(), INTERVAL 90 DAY)
GROUP BY p.product_id, p.name, b.name, c.name, p.category_id
ORDER BY units_sold DESC
LIMIT 20;
```

---

### Query 6: Product Performance Metrics (ประสิทธิภาพสินค้ารอบด้าน)

```sql
WITH product_sales AS (
    SELECT 
        pv.product_id,
        SUM(oi.quantity) AS units_sold,
        SUM(oi.total_price) AS revenue,
        COUNT(DISTINCT oi.order_id) AS orders
    FROM order_items oi
    JOIN orders o ON oi.order_id = o.order_id
    JOIN product_variants pv ON oi.variant_id = pv.variant_id
    WHERE o.payment_status = 'paid'
      AND o.status NOT IN ('cancelled', 'refunded')
      AND o.created_at >= DATE_SUB(CURDATE(), INTERVAL 30 DAY)
    GROUP BY pv.product_id
),
product_reviews_summary AS (
    SELECT 
        product_id,
        COUNT(*) AS review_count,
        AVG(rating) AS avg_rating,
        SUM(helpful_count) AS total_helpful
    FROM product_reviews
    WHERE is_approved = TRUE
    GROUP BY product_id
),
product_returns AS (
    SELECT 
        pv.product_id,
        COUNT(ri.return_item_id) AS return_count,
        SUM(ri.quantity) AS returned_units
    FROM return_items ri
    JOIN order_items oi ON ri.order_item_id = oi.order_item_id
    JOIN product_variants pv ON oi.variant_id = pv.variant_id
    GROUP BY pv.product_id
),
product_wishlist AS (
    SELECT 
        product_id,
        COUNT(*) AS wishlist_adds
    FROM wishlist_items
    GROUP BY product_id
)
SELECT 
    p.name,
    COALESCE(ps.units_sold, 0) AS units_sold_30d,
    COALESCE(ps.revenue, 0) AS revenue_30d,
    COALESCE(prs.avg_rating, 0) AS avg_rating,
    COALESCE(prs.review_count, 0) AS reviews,
    COALESCE(pret.return_count, 0) AS returns,
    CASE 
        WHEN COALESCE(ps.units_sold, 0) > 0 
        THEN ROUND(COALESCE(pret.returned_units, 0) * 100.0 / ps.units_sold, 1)
        ELSE 0 
    END AS return_rate_pct,
    COALESCE(pw.wishlist_adds, 0) AS wishlist_adds,
    p.view_count,
    CASE 
        WHEN p.view_count > 0 
        THEN ROUND(COALESCE(ps.orders, 0) * 100.0 / p.view_count, 2)
        ELSE 0 
    END AS conversion_rate_pct
FROM products p
LEFT JOIN product_sales ps ON p.product_id = ps.product_id
LEFT JOIN product_reviews_summary prs ON p.product_id = prs.product_id
LEFT JOIN product_returns pret ON p.product_id = pret.product_id
LEFT JOIN product_wishlist pw ON p.product_id = pw.product_id
WHERE p.status = 'active'
ORDER BY revenue_30d DESC;
```

---

## SECTION 3: Customer Analytics

### Query 7: Customer Acquisition Analysis (การหาลูกค้าใหม่)

```sql
SELECT 
    DATE_FORMAT(created_at, '%Y-%m') AS cohort_month,
    COUNT(*) AS new_customers,
    SUM(COUNT(*)) OVER (ORDER BY DATE_FORMAT(created_at, '%Y-%m')) AS cumulative_customers,
    -- First purchase within 30 days
    SUM(CASE 
        WHEN first_order_date IS NOT NULL 
        AND DATEDIFF(first_order_date, created_at) <= 30 
        THEN 1 ELSE 0 
    END) AS converted_in_30d,
    ROUND(
        SUM(CASE 
            WHEN first_order_date IS NOT NULL 
            AND DATEDIFF(first_order_date, created_at) <= 30 
            THEN 1 ELSE 0 
        END) * 100.0 / COUNT(*),
        1
    ) AS activation_rate_30d_pct
FROM customers c
LEFT JOIN (
    SELECT customer_id, MIN(created_at) AS first_order_date
    FROM orders
    WHERE payment_status = 'paid'
    GROUP BY customer_id
) fo ON c.customer_id = fo.customer_id
WHERE c.is_active = TRUE
GROUP BY DATE_FORMAT(created_at, '%Y-%m')
ORDER BY cohort_month DESC;
```

---

### Query 8: Customer Cohort Analysis (Cohort Retention)

```sql
-- Cohort Analysis: ลูกค้าที่สมัครในเดือน X กลับมาซื้อในเดือน Y กี่เปอร์เซ็นต์
WITH customer_cohorts AS (
    SELECT 
        customer_id,
        DATE_FORMAT(created_at, '%Y-%m') AS cohort_month,
        created_at AS join_date
    FROM customers
    WHERE is_active = TRUE
),
order_cohorts AS (
    SELECT 
        o.customer_id,
        DATE_FORMAT(o.created_at, '%Y-%m') AS order_month,
        o.created_at AS order_date
    FROM orders o
    WHERE o.payment_status = 'paid'
      AND o.status NOT IN ('cancelled')
),
cohort_data AS (
    SELECT 
        cc.cohort_month,
        TIMESTAMPDIFF(
            MONTH, 
            STR_TO_DATE(CONCAT(cc.cohort_month, '-01'), '%Y-%m-%d'),
            STR_TO_DATE(CONCAT(oc.order_month, '-01'), '%Y-%m-%d')
        ) AS months_since_join,
        COUNT(DISTINCT cc.customer_id) AS customer_count
    FROM customer_cohorts cc
    LEFT JOIN order_cohorts oc ON cc.customer_id = oc.customer_id
    GROUP BY cc.cohort_month, months_since_join
),
cohort_sizes AS (
    SELECT cohort_month, customer_count AS cohort_size
    FROM cohort_data
    WHERE months_since_join = 0
)
SELECT 
    cd.cohort_month,
    cs.cohort_size,
    MAX(CASE WHEN months_since_join = 0 THEN customer_count END) AS month_0,
    MAX(CASE WHEN months_since_join = 1 THEN customer_count END) AS month_1,
    MAX(CASE WHEN months_since_join = 2 THEN customer_count END) AS month_2,
    MAX(CASE WHEN months_since_join = 3 THEN customer_count END) AS month_3,
    ROUND(MAX(CASE WHEN months_since_join = 1 THEN customer_count END) * 100.0 / cs.cohort_size, 1) AS retention_m1_pct,
    ROUND(MAX(CASE WHEN months_since_join = 2 THEN customer_count END) * 100.0 / cs.cohort_size, 1) AS retention_m2_pct,
    ROUND(MAX(CASE WHEN months_since_join = 3 THEN customer_count END) * 100.0 / cs.cohort_size, 1) AS retention_m3_pct
FROM cohort_data cd
JOIN cohort_sizes cs ON cd.cohort_month = cs.cohort_month
GROUP BY cd.cohort_month, cs.cohort_size
ORDER BY cd.cohort_month DESC;
```

**อธิบาย:** Cohort Analysis ช่วยให้เห็นว่าลูกค้าที่สมัครในเดือนเดียวกัน (Cohort) มีพฤติกรรมการกลับมาซื้อซ้ำอย่างไร

---

### Query 9: Customer Lifetime Value (CLV) Analysis

```sql
WITH clv_data AS (
    SELECT 
        c.customer_id,
        CONCAT(c.first_name, ' ', c.last_name) AS name,
        c.created_at AS joined,
        DATEDIFF(CURDATE(), c.created_at) AS customer_age_days,
        COUNT(DISTINCT o.order_id) AS total_orders,
        SUM(o.total_amount) AS total_spent,
        MIN(o.created_at) AS first_order,
        MAX(o.created_at) AS last_order,
        DATEDIFF(MAX(o.created_at), MIN(o.created_at)) AS customer_lifespan_days,
        AVG(o.total_amount) AS avg_order_value
    FROM customers c
    LEFT JOIN orders o ON c.customer_id = o.customer_id
        AND o.payment_status = 'paid'
        AND o.status NOT IN ('cancelled', 'refunded')
    GROUP BY c.customer_id, c.first_name, c.last_name, c.created_at
)
SELECT 
    customer_id,
    name,
    total_orders,
    ROUND(total_spent, 2) AS total_spent,
    ROUND(avg_order_value, 2) AS avg_order_value,
    customer_age_days,
    -- Purchase Frequency (orders per month)
    ROUND(total_orders / GREATEST(customer_age_days / 30.0, 1), 2) AS orders_per_month,
    -- Projected Annual Value
    ROUND(avg_order_value * (total_orders / GREATEST(customer_age_days / 30.0, 1)) * 12, 2) AS projected_annual_value,
    -- CLV (Historical) 
    ROUND(total_spent, 2) AS historical_clv,
    -- RFM Scores
    NTILE(5) OVER (ORDER BY DATEDIFF(CURDATE(), last_order) ASC) AS recency_score,
    NTILE(5) OVER (ORDER BY total_orders DESC) AS frequency_score,
    NTILE(5) OVER (ORDER BY total_spent DESC) AS monetary_score
FROM clv_data
ORDER BY total_spent DESC NULLS LAST;
```

---

### Query 10: RFM Segmentation (R=Recency, F=Frequency, M=Monetary)

```sql
WITH rfm_base AS (
    SELECT 
        c.customer_id,
        CONCAT(c.first_name, ' ', c.last_name) AS customer_name,
        c.email,
        DATEDIFF(CURDATE(), MAX(o.created_at)) AS recency_days,
        COUNT(DISTINCT o.order_id) AS frequency,
        SUM(o.total_amount) AS monetary
    FROM customers c
    JOIN orders o ON c.customer_id = o.customer_id
    WHERE o.payment_status = 'paid'
      AND o.status NOT IN ('cancelled', 'refunded')
    GROUP BY c.customer_id, c.first_name, c.last_name, c.email
),
rfm_scores AS (
    SELECT *,
        NTILE(5) OVER (ORDER BY recency_days ASC) AS r_score,
        NTILE(5) OVER (ORDER BY frequency DESC) AS f_score,
        NTILE(5) OVER (ORDER BY monetary DESC) AS m_score
    FROM rfm_base
),
rfm_segments AS (
    SELECT *,
        CONCAT(r_score, f_score, m_score) AS rfm_code,
        (r_score + f_score + m_score) AS rfm_total
    FROM rfm_scores
)
SELECT 
    customer_id,
    customer_name,
    email,
    recency_days,
    frequency,
    ROUND(monetary, 2) AS monetary,
    r_score, f_score, m_score,
    rfm_code,
    CASE 
        WHEN r_score >= 4 AND f_score >= 4 AND m_score >= 4 THEN 'Champions'
        WHEN r_score >= 3 AND f_score >= 3 AND m_score >= 3 THEN 'Loyal Customers'
        WHEN r_score >= 4 AND f_score <= 2 THEN 'Recent Customers'
        WHEN r_score <= 2 AND f_score >= 4 AND m_score >= 4 THEN 'At Risk (High Value)'
        WHEN r_score <= 2 AND f_score >= 4 THEN 'Cant Lose Them'
        WHEN r_score <= 2 AND f_score <= 2 THEN 'Lost Customers'
        ELSE 'Potential Loyalists'
    END AS segment
FROM rfm_segments
ORDER BY rfm_total DESC;
```

---

## SECTION 4: Cart Analytics

### Query 11: Cart Abandonment Analysis (วิเคราะห์การละทิ้ง Cart)

```sql
-- วิเคราะห์ Cart Abandonment Rate
WITH cart_stats AS (
    SELECT 
        DATE(cart.created_at) AS date,
        COUNT(DISTINCT cart.cart_id) AS total_carts_created,
        COUNT(DISTINCT cart.customer_id) AS unique_shoppers,
        SUM(ci.quantity * ci.unit_price) AS total_cart_value
    FROM carts cart
    JOIN cart_items ci ON cart.cart_id = ci.cart_id
    GROUP BY DATE(cart.created_at)
),
checkout_stats AS (
    SELECT 
        DATE(o.created_at) AS date,
        COUNT(DISTINCT o.order_id) AS orders_placed,
        SUM(o.total_amount) AS checkout_value
    FROM orders o
    WHERE o.payment_status IN ('paid', 'pending')
    GROUP BY DATE(o.created_at)
)
SELECT 
    cs.date,
    cs.total_carts_created,
    cs.unique_shoppers,
    ROUND(cs.total_cart_value, 2) AS total_cart_value,
    COALESCE(cks.orders_placed, 0) AS orders_placed,
    ROUND(
        (cs.total_carts_created - COALESCE(cks.orders_placed, 0)) * 100.0 / cs.total_carts_created,
        1
    ) AS abandonment_rate_pct,
    ROUND(
        COALESCE(cks.orders_placed, 0) * 100.0 / cs.total_carts_created,
        1
    ) AS conversion_rate_pct,
    ROUND(cs.total_cart_value - COALESCE(cks.checkout_value, 0), 2) AS lost_revenue
FROM cart_stats cs
LEFT JOIN checkout_stats cks ON cs.date = cks.date
ORDER BY cs.date DESC;
```

---

### Query 12: Most Abandoned Products in Cart

```sql
-- สินค้าที่ถูกเพิ่มลง Cart แต่ไม่ได้ซื้อบ่อยที่สุด
SELECT 
    p.name AS product_name,
    b.name AS brand,
    COUNT(ci.cart_item_id) AS times_added_to_cart,
    SUM(ci.quantity) AS units_added,
    SUM(ci.quantity * ci.unit_price) AS total_cart_value,
    -- จำนวนที่ซื้อจริง
    COALESCE(sold.units_sold, 0) AS units_actually_sold,
    -- คำนวณ abandonment
    ROUND(
        (SUM(ci.quantity) - COALESCE(sold.units_sold, 0)) * 100.0 / SUM(ci.quantity),
        1
    ) AS cart_abandonment_rate_pct
FROM cart_items ci
JOIN product_variants pv ON ci.variant_id = pv.variant_id
JOIN products p ON pv.product_id = p.product_id
LEFT JOIN brands b ON p.brand_id = b.brand_id
LEFT JOIN (
    SELECT pv2.product_id, SUM(oi.quantity) AS units_sold
    FROM order_items oi
    JOIN orders o ON oi.order_id = o.order_id
    JOIN product_variants pv2 ON oi.variant_id = pv2.variant_id
    WHERE o.payment_status = 'paid'
    GROUP BY pv2.product_id
) sold ON p.product_id = sold.product_id
GROUP BY p.product_id, p.name, b.name, sold.units_sold
ORDER BY cart_abandonment_rate_pct DESC
LIMIT 15;
```

---

## SECTION 5: Category & Geographic Analysis

### Query 13: Revenue by Category (Hierarchical)

```sql
WITH RECURSIVE cat_hierarchy AS (
    SELECT 
        category_id,
        parent_id,
        name,
        CAST(name AS CHAR(500)) AS path,
        0 AS level
    FROM categories
    WHERE parent_id IS NULL
    
    UNION ALL
    
    SELECT 
        c.category_id,
        c.parent_id,
        c.name,
        CONCAT(ch.path, ' > ', c.name),
        ch.level + 1
    FROM categories c
    JOIN cat_hierarchy ch ON c.parent_id = ch.category_id
)
SELECT 
    ch.path AS category_path,
    ch.level,
    COUNT(DISTINCT o.order_id) AS orders,
    COUNT(DISTINCT o.customer_id) AS customers,
    SUM(oi.quantity) AS units_sold,
    ROUND(SUM(oi.total_price), 2) AS revenue,
    ROUND(AVG(oi.unit_price), 2) AS avg_item_price,
    ROUND(SUM(oi.total_price) * 100.0 / SUM(SUM(oi.total_price)) OVER (), 1) AS revenue_share_pct
FROM cat_hierarchy ch
JOIN products p ON ch.category_id = p.category_id
JOIN product_variants pv ON p.product_id = pv.product_id
JOIN order_items oi ON pv.variant_id = oi.variant_id
JOIN orders o ON oi.order_id = o.order_id
WHERE o.payment_status = 'paid'
  AND o.status NOT IN ('cancelled', 'refunded')
GROUP BY ch.category_id, ch.path, ch.level
ORDER BY revenue DESC;
```

---

### Query 14: Geographic Sales Analysis (ยอดขายตามภูมิภาค)

```sql
SELECT 
    o.shipping_city AS city,
    o.shipping_country AS country,
    COUNT(DISTINCT o.order_id) AS total_orders,
    COUNT(DISTINCT o.customer_id) AS unique_customers,
    SUM(oi.quantity) AS units_sold,
    ROUND(SUM(o.total_amount), 2) AS revenue,
    ROUND(AVG(o.total_amount), 2) AS avg_order_value,
    ROUND(SUM(o.total_amount) * 100.0 / SUM(SUM(o.total_amount)) OVER (), 1) AS revenue_share_pct,
    -- Top product in this city
    (
        SELECT p2.name 
        FROM order_items oi2
        JOIN orders o2 ON oi2.order_id = o2.order_id
        JOIN product_variants pv2 ON oi2.variant_id = pv2.variant_id
        JOIN products p2 ON pv2.product_id = p2.product_id
        WHERE o2.shipping_city = o.shipping_city
          AND o2.payment_status = 'paid'
        GROUP BY p2.product_id
        ORDER BY SUM(oi2.quantity) DESC
        LIMIT 1
    ) AS top_product
FROM orders o
JOIN order_items oi ON o.order_id = oi.order_id
WHERE o.payment_status = 'paid'
  AND o.status NOT IN ('cancelled', 'refunded')
GROUP BY o.shipping_city, o.shipping_country
ORDER BY revenue DESC;
```

---

## SECTION 6: Inventory Analytics

### Query 15: Inventory Turnover Analysis (อัตราการหมุนเวียนสต็อก)

```sql
-- Inventory Turnover = Cost of Goods Sold / Average Inventory Value
WITH sales_data AS (
    SELECT 
        pv.product_id,
        SUM(oi.quantity * p.cost_price) AS cogs,
        SUM(oi.quantity) AS units_sold
    FROM order_items oi
    JOIN orders o ON oi.order_id = o.order_id
    JOIN product_variants pv ON oi.variant_id = pv.variant_id
    JOIN products p ON pv.product_id = p.product_id
    WHERE o.created_at >= DATE_SUB(CURDATE(), INTERVAL 365 DAY)
      AND o.payment_status = 'paid'
      AND o.status NOT IN ('cancelled', 'refunded')
    GROUP BY pv.product_id
),
inventory_data AS (
    SELECT 
        pv.product_id,
        SUM(i.quantity) AS current_inventory,
        SUM(i.quantity * p.cost_price) AS inventory_value
    FROM inventory i
    JOIN product_variants pv ON i.variant_id = pv.variant_id
    JOIN products p ON pv.product_id = p.product_id
    GROUP BY pv.product_id
)
SELECT 
    p.name AS product_name,
    ROUND(COALESCE(sd.cogs, 0), 2) AS annual_cogs,
    ROUND(COALESCE(id.inventory_value, 0), 2) AS current_inventory_value,
    COALESCE(sd.units_sold, 0) AS units_sold_annual,
    COALESCE(id.current_inventory, 0) AS current_stock,
    ROUND(
        CASE WHEN id.inventory_value > 0 
        THEN COALESCE(sd.cogs, 0) / id.inventory_value 
        ELSE 0 END,
        2
    ) AS inventory_turnover_ratio,
    ROUND(
        CASE WHEN sd.units_sold > 0 
        THEN id.current_inventory / (sd.units_sold / 365.0) 
        ELSE NULL END,
        0
    ) AS days_of_stock_remaining,
    CASE 
        WHEN (COALESCE(sd.cogs, 0) / NULLIF(id.inventory_value, 0)) > 6 THEN 'HIGH TURNOVER'
        WHEN (COALESCE(sd.cogs, 0) / NULLIF(id.inventory_value, 0)) > 2 THEN 'NORMAL'
        WHEN (COALESCE(sd.cogs, 0) / NULLIF(id.inventory_value, 0)) > 0 THEN 'SLOW MOVING'
        ELSE 'NO SALES'
    END AS turnover_category
FROM products p
LEFT JOIN sales_data sd ON p.product_id = sd.product_id
LEFT JOIN inventory_data id ON p.product_id = id.product_id
WHERE p.status = 'active'
ORDER BY inventory_turnover_ratio DESC;
```

---

### Query 16: Dead Stock Analysis (สต็อกที่ไม่มีการเคลื่อนไหว)

```sql
SELECT 
    p.name AS product_name,
    pv.sku,
    pv.name AS variant_name,
    i.quantity AS current_stock,
    p.cost_price * i.quantity AS tied_up_capital,
    COALESCE(last_sale.last_sale_date, 'Never') AS last_sold,
    CASE 
        WHEN last_sale.last_sale_date IS NULL THEN 9999
        ELSE DATEDIFF(CURDATE(), last_sale.last_sale_date) 
    END AS days_without_sale,
    COALESCE(last_sale.total_units_sold, 0) AS total_ever_sold,
    p.base_price,
    CASE 
        WHEN last_sale.last_sale_date IS NULL THEN 'NEVER SOLD'
        WHEN DATEDIFF(CURDATE(), last_sale.last_sale_date) > 180 THEN 'DEAD STOCK (180d+)'
        WHEN DATEDIFF(CURDATE(), last_sale.last_sale_date) > 90 THEN 'SLOW MOVING (90d+)'
        ELSE 'ACTIVE'
    END AS stock_health
FROM inventory i
JOIN product_variants pv ON i.variant_id = pv.variant_id
JOIN products p ON pv.product_id = p.product_id
LEFT JOIN (
    SELECT 
        oi.variant_id,
        MAX(o.created_at) AS last_sale_date,
        SUM(oi.quantity) AS total_units_sold
    FROM order_items oi
    JOIN orders o ON oi.order_id = o.order_id
    WHERE o.payment_status = 'paid'
    GROUP BY oi.variant_id
) last_sale ON pv.variant_id = last_sale.variant_id
WHERE i.quantity > 0
  AND p.status = 'active'
ORDER BY days_without_sale DESC;
```

---

## SECTION 7: Discount Analytics

### Query 17: Discount Effectiveness Analysis

```sql
WITH orders_with_discount AS (
    SELECT 
        coupon_code,
        COUNT(*) AS orders_with_coupon,
        SUM(subtotal) AS gross_revenue,
        SUM(discount_amount) AS total_discount_given,
        SUM(total_amount) AS net_revenue,
        AVG(total_amount) AS avg_order_value,
        SUM(discount_amount) / SUM(subtotal) * 100 AS avg_discount_depth_pct
    FROM orders
    WHERE payment_status = 'paid'
      AND status NOT IN ('cancelled', 'refunded')
      AND coupon_code IS NOT NULL
    GROUP BY coupon_code
),
orders_without_discount AS (
    SELECT 
        AVG(total_amount) AS baseline_aov,
        COUNT(*) AS total_orders
    FROM orders
    WHERE payment_status = 'paid'
      AND status NOT IN ('cancelled', 'refunded')
      AND coupon_code IS NULL
)
SELECT 
    cp.code,
    dr.name AS discount_rule,
    dr.discount_type,
    dr.discount_value,
    COALESCE(owd.orders_with_coupon, 0) AS orders_used,
    ROUND(COALESCE(owd.gross_revenue, 0), 2) AS gross_revenue,
    ROUND(COALESCE(owd.total_discount_given, 0), 2) AS total_discount,
    ROUND(COALESCE(owd.net_revenue, 0), 2) AS net_revenue,
    ROUND(COALESCE(owd.avg_order_value, 0), 2) AS avg_order_with_coupon,
    ROUND(nowd.baseline_aov, 2) AS baseline_aov,
    ROUND(COALESCE(owd.avg_order_value, 0) - nowd.baseline_aov, 2) AS aov_lift,
    ROUND(COALESCE(owd.avg_discount_depth_pct, 0), 1) AS avg_discount_pct,
    -- ROI of discount: Net Revenue / Discount Given
    ROUND(COALESCE(owd.net_revenue, 0) / NULLIF(owd.total_discount_given, 0), 2) AS discount_roi
FROM coupons cp
JOIN discount_rules dr ON cp.rule_id = dr.rule_id
LEFT JOIN orders_with_discount owd ON cp.code = owd.coupon_code
CROSS JOIN orders_without_discount nowd
ORDER BY owd.net_revenue DESC NULLS LAST;
```

---

### Query 18: Discount Impact on Profit Margin

```sql
SELECT 
    DATE_FORMAT(o.created_at, '%Y-%m') AS month,
    COUNT(DISTINCT o.order_id) AS total_orders,
    -- Revenue breakdown
    SUM(o.subtotal) AS gross_revenue,
    SUM(o.discount_amount) AS discounts,
    SUM(o.shipping_fee) AS shipping,
    SUM(o.tax_amount) AS tax,
    SUM(o.total_amount) AS net_revenue,
    -- Cost analysis
    SUM(oi.quantity * p.cost_price) AS cogs,
    -- Gross Profit
    SUM(o.subtotal) - SUM(oi.quantity * p.cost_price) AS gross_profit_before_discount,
    SUM(o.total_amount) - SUM(oi.quantity * p.cost_price) AS gross_profit_after_discount,
    -- Margin %
    ROUND(
        (SUM(o.total_amount) - SUM(oi.quantity * p.cost_price)) 
        * 100.0 / SUM(o.total_amount),
        1
    ) AS gross_margin_pct,
    -- Discount as % of revenue
    ROUND(SUM(o.discount_amount) * 100.0 / SUM(o.subtotal), 1) AS discount_rate_pct
FROM orders o
JOIN order_items oi ON o.order_id = oi.order_id
JOIN product_variants pv ON oi.variant_id = pv.variant_id
JOIN products p ON pv.product_id = p.product_id
WHERE o.payment_status = 'paid'
  AND o.status NOT IN ('cancelled', 'refunded')
GROUP BY DATE_FORMAT(o.created_at, '%Y-%m')
ORDER BY month DESC;
```

---

## SECTION 8: Return Rate Analytics

### Query 19: Return Rate Analysis (อัตราการคืนสินค้า)

```sql
WITH return_stats AS (
    SELECT 
        DATE_FORMAT(r.requested_at, '%Y-%m') AS month,
        COUNT(DISTINCT r.return_id) AS total_returns,
        COUNT(DISTINCT CASE WHEN r.status = 'completed' THEN r.return_id END) AS completed_returns,
        SUM(r.refund_amount) AS total_refunded,
        AVG(DATEDIFF(r.completed_at, r.requested_at)) AS avg_processing_days
    FROM returns r
    GROUP BY DATE_FORMAT(r.requested_at, '%Y-%m')
),
order_stats AS (
    SELECT 
        DATE_FORMAT(created_at, '%Y-%m') AS month,
        COUNT(*) AS total_orders,
        SUM(total_amount) AS total_revenue
    FROM orders
    WHERE payment_status = 'paid'
      AND status NOT IN ('cancelled')
    GROUP BY DATE_FORMAT(created_at, '%Y-%m')
)
SELECT 
    os.month,
    os.total_orders,
    COALESCE(rs.total_returns, 0) AS returns_initiated,
    COALESCE(rs.completed_returns, 0) AS returns_completed,
    ROUND(COALESCE(rs.total_returns, 0) * 100.0 / os.total_orders, 2) AS return_rate_pct,
    ROUND(COALESCE(rs.total_refunded, 0), 2) AS refund_amount,
    ROUND(os.total_revenue, 2) AS gross_revenue,
    ROUND(os.total_revenue - COALESCE(rs.total_refunded, 0), 2) AS net_revenue,
    ROUND(COALESCE(rs.avg_processing_days, 0), 1) AS avg_processing_days
FROM order_stats os
LEFT JOIN return_stats rs ON os.month = rs.month
ORDER BY os.month DESC;
```

---

### Query 20: Return Reasons Analysis

```sql
SELECT 
    r.reason,
    COUNT(*) AS return_count,
    COUNT(*) * 100.0 / SUM(COUNT(*)) OVER () AS pct_of_total,
    SUM(r.refund_amount) AS total_refunded,
    AVG(r.refund_amount) AS avg_refund_amount,
    -- Most returned product for each reason
    (
        SELECT p.name
        FROM return_items ri2
        JOIN returns r2 ON ri2.return_id = r2.return_id
        JOIN order_items oi ON ri2.order_item_id = oi.order_item_id
        JOIN product_variants pv ON oi.variant_id = pv.variant_id
        JOIN products p ON pv.product_id = p.product_id
        WHERE r2.reason = r.reason
        GROUP BY p.product_id
        ORDER BY COUNT(*) DESC
        LIMIT 1
    ) AS most_returned_product
FROM returns r
WHERE r.status = 'completed'
GROUP BY r.reason
ORDER BY return_count DESC;
```

---

## SECTION 9: Advanced Analytics

### Query 21: Revenue Contribution by Customer Segment (Pareto Analysis)

```sql
-- 80/20 Rule: ลูกค้าแค่ 20% สร้างรายได้ 80%
WITH customer_revenue AS (
    SELECT 
        c.customer_id,
        CONCAT(c.first_name, ' ', c.last_name) AS name,
        SUM(o.total_amount) AS total_revenue
    FROM customers c
    JOIN orders o ON c.customer_id = o.customer_id
    WHERE o.payment_status = 'paid'
      AND o.status NOT IN ('cancelled', 'refunded')
    GROUP BY c.customer_id, c.first_name, c.last_name
),
ranked AS (
    SELECT 
        *,
        RANK() OVER (ORDER BY total_revenue DESC) AS revenue_rank,
        COUNT(*) OVER () AS total_customers,
        SUM(total_revenue) OVER (ORDER BY total_revenue DESC 
                                  ROWS UNBOUNDED PRECEDING) AS cumulative_revenue,
        SUM(total_revenue) OVER () AS grand_total
    FROM customer_revenue
)
SELECT 
    revenue_rank,
    name,
    ROUND(total_revenue, 2) AS total_revenue,
    ROUND(total_revenue * 100.0 / grand_total, 2) AS revenue_pct,
    ROUND(cumulative_revenue, 2) AS cumulative_revenue,
    ROUND(cumulative_revenue * 100.0 / grand_total, 2) AS cumulative_pct,
    ROUND(revenue_rank * 100.0 / total_customers, 1) AS customer_percentile,
    CASE 
        WHEN cumulative_revenue * 100.0 / grand_total <= 80 THEN 'Top 80% Revenue'
        ELSE 'Bottom 20% Revenue'
    END AS segment
FROM ranked
ORDER BY revenue_rank;
```

---

### Query 22: Customer Churn Risk Scoring

```sql
WITH customer_metrics AS (
    SELECT 
        c.customer_id,
        CONCAT(c.first_name, ' ', c.last_name) AS name,
        c.email,
        COUNT(DISTINCT o.order_id) AS total_orders,
        DATEDIFF(CURDATE(), MAX(o.created_at)) AS days_since_last_order,
        DATEDIFF(CURDATE(), c.created_at) AS customer_age_days,
        SUM(o.total_amount) AS lifetime_value,
        AVG(o.total_amount) AS avg_order_value,
        -- Purchase interval
        CASE 
            WHEN COUNT(DISTINCT o.order_id) > 1 
            THEN DATEDIFF(MAX(o.created_at), MIN(o.created_at)) / (COUNT(DISTINCT o.order_id) - 1)
            ELSE NULL
        END AS avg_days_between_orders
    FROM customers c
    LEFT JOIN orders o ON c.customer_id = o.customer_id
        AND o.payment_status = 'paid'
        AND o.status NOT IN ('cancelled', 'refunded')
    GROUP BY c.customer_id, c.first_name, c.last_name, c.email, c.created_at
)
SELECT 
    customer_id,
    name,
    email,
    total_orders,
    ROUND(lifetime_value, 2) AS lifetime_value,
    days_since_last_order,
    ROUND(avg_days_between_orders, 0) AS avg_order_interval,
    -- Churn Score (0-100, higher = more likely to churn)
    LEAST(100,
        CASE 
            WHEN days_since_last_order > avg_days_between_orders * 3 THEN 80
            WHEN days_since_last_order > avg_days_between_orders * 2 THEN 60
            WHEN days_since_last_order > avg_days_between_orders * 1.5 THEN 40
            ELSE 20
        END +
        CASE WHEN total_orders = 1 THEN 20 ELSE 0 END
    ) AS churn_score,
    CASE 
        WHEN LEAST(100,
            CASE 
                WHEN days_since_last_order > COALESCE(avg_days_between_orders, 30) * 3 THEN 80
                WHEN days_since_last_order > COALESCE(avg_days_between_orders, 30) * 2 THEN 60
                ELSE 20
            END +
            CASE WHEN total_orders = 1 THEN 20 ELSE 0 END
        ) >= 70 THEN 'HIGH RISK'
        WHEN LEAST(100,
            CASE 
                WHEN days_since_last_order > COALESCE(avg_days_between_orders, 30) * 3 THEN 80
                ELSE 40
            END +
            CASE WHEN total_orders = 1 THEN 20 ELSE 0 END
        ) >= 50 THEN 'MEDIUM RISK'
        ELSE 'LOW RISK'
    END AS churn_risk_level
FROM customer_metrics
WHERE total_orders > 0
ORDER BY churn_score DESC;
```

---

### Query 23: Sales Funnel Analysis

```sql
-- Funnel: Views -> Cart Adds -> Orders -> Paid Orders
SELECT 
    'Product Views' AS funnel_stage,
    SUM(p.view_count) AS count,
    100.0 AS pct_of_previous,
    100.0 AS overall_pct
FROM products p
WHERE p.status = 'active'

UNION ALL

SELECT 
    'Cart Additions',
    COUNT(*) AS count,
    COUNT(*) * 100.0 / (SELECT SUM(view_count) FROM products WHERE status = 'active') AS pct_of_previous,
    COUNT(*) * 100.0 / (SELECT SUM(view_count) FROM products WHERE status = 'active') AS overall_pct
FROM cart_items

UNION ALL

SELECT 
    'Orders Placed',
    COUNT(*) AS count,
    COUNT(*) * 100.0 / (SELECT COUNT(*) FROM cart_items) AS pct_of_previous,
    COUNT(*) * 100.0 / (SELECT SUM(view_count) FROM products WHERE status = 'active') AS overall_pct
FROM orders

UNION ALL

SELECT 
    'Paid Orders',
    COUNT(*) AS count,
    COUNT(*) * 100.0 / (SELECT COUNT(*) FROM orders) AS pct_of_previous,
    COUNT(*) * 100.0 / (SELECT SUM(view_count) FROM products WHERE status = 'active') AS overall_pct
FROM orders
WHERE payment_status = 'paid';
```

---

### Query 24: Hour-of-Day Sales Pattern

```sql
SELECT 
    HOUR(created_at) AS hour_of_day,
    COUNT(DISTINCT order_id) AS orders,
    SUM(total_amount) AS revenue,
    AVG(total_amount) AS avg_order_value,
    -- แสดง Bar Chart ด้วย RPAD
    RPAD('|', ROUND(COUNT(DISTINCT order_id) * 30.0 / MAX(COUNT(DISTINCT order_id)) OVER ()), '=') AS volume_bar
FROM orders
WHERE payment_status = 'paid'
  AND status NOT IN ('cancelled', 'refunded')
GROUP BY HOUR(created_at)
ORDER BY hour_of_day;
```

---

### Query 25: Day-of-Week Sales Pattern

```sql
SELECT 
    DAYOFWEEK(created_at) AS day_num,
    DAYNAME(created_at) AS day_name,
    COUNT(DISTINCT order_id) AS orders,
    COUNT(DISTINCT customer_id) AS unique_customers,
    ROUND(SUM(total_amount), 2) AS revenue,
    ROUND(AVG(total_amount), 2) AS avg_order_value,
    ROUND(SUM(total_amount) * 100.0 / SUM(SUM(total_amount)) OVER (), 1) AS revenue_pct
FROM orders
WHERE payment_status = 'paid'
  AND status NOT IN ('cancelled', 'refunded')
GROUP BY DAYOFWEEK(created_at), DAYNAME(created_at)
ORDER BY day_num;
```

---

### Query 26: New vs Returning Customer Revenue Split

```sql
WITH customer_order_sequence AS (
    SELECT 
        customer_id,
        order_id,
        total_amount,
        ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY created_at) AS order_sequence
    FROM orders
    WHERE payment_status = 'paid'
      AND status NOT IN ('cancelled', 'refunded')
)
SELECT 
    DATE_FORMAT(o.created_at, '%Y-%m') AS month,
    SUM(CASE WHEN cos.order_sequence = 1 THEN o.total_amount ELSE 0 END) AS new_customer_revenue,
    COUNT(DISTINCT CASE WHEN cos.order_sequence = 1 THEN o.customer_id END) AS new_customers,
    SUM(CASE WHEN cos.order_sequence > 1 THEN o.total_amount ELSE 0 END) AS returning_customer_revenue,
    COUNT(DISTINCT CASE WHEN cos.order_sequence > 1 THEN o.customer_id END) AS returning_customers,
    ROUND(
        SUM(CASE WHEN cos.order_sequence > 1 THEN o.total_amount ELSE 0 END) * 100.0 / 
        NULLIF(SUM(o.total_amount), 0),
        1
    ) AS returning_revenue_pct
FROM orders o
JOIN customer_order_sequence cos ON o.order_id = cos.order_id
WHERE o.payment_status = 'paid'
  AND o.status NOT IN ('cancelled', 'refunded')
GROUP BY DATE_FORMAT(o.created_at, '%Y-%m')
ORDER BY month DESC;
```

---

### Query 27: Product Affinity Analysis (Market Basket)

```sql
-- สินค้าคู่ไหนที่มักถูกซื้อพร้อมกัน
WITH product_pairs AS (
    SELECT 
        oi1.order_id,
        p1.product_id AS product_a_id,
        p1.name AS product_a,
        p2.product_id AS product_b_id,
        p2.name AS product_b
    FROM order_items oi1
    JOIN order_items oi2 ON oi1.order_id = oi2.order_id
    JOIN product_variants pv1 ON oi1.variant_id = pv1.variant_id
    JOIN product_variants pv2 ON oi2.variant_id = pv2.variant_id
    JOIN products p1 ON pv1.product_id = p1.product_id
    JOIN products p2 ON pv2.product_id = p2.product_id
    WHERE oi1.variant_id < oi2.variant_id  -- avoid duplicates
    JOIN orders o ON oi1.order_id = o.order_id
    WHERE o.payment_status = 'paid'
)
SELECT 
    product_a,
    product_b,
    COUNT(*) AS co_purchase_count,
    -- Support: % of orders containing both
    ROUND(COUNT(*) * 100.0 / (SELECT COUNT(DISTINCT order_id) FROM orders WHERE payment_status = 'paid'), 2) AS support_pct,
    -- Confidence A->B: given A is bought, how often B is also bought
    ROUND(
        COUNT(*) * 100.0 / (
            SELECT COUNT(DISTINCT oi.order_id)
            FROM order_items oi
            JOIN product_variants pv ON oi.variant_id = pv.variant_id
            WHERE pv.product_id = product_a_id
        ),
        1
    ) AS confidence_a_to_b_pct
FROM product_pairs
GROUP BY product_a_id, product_a, product_b_id, product_b
HAVING co_purchase_count >= 1
ORDER BY co_purchase_count DESC
LIMIT 20;
```

---

### Query 28: Repeat Purchase Rate (อัตราการซื้อซ้ำ)

```sql
SELECT 
    DATE_FORMAT(first_order.first_order_month, '%Y-%m') AS acquisition_month,
    COUNT(DISTINCT first_order.customer_id) AS new_customers,
    COUNT(DISTINCT CASE WHEN second_order.order_id IS NOT NULL THEN first_order.customer_id END) AS made_second_purchase,
    ROUND(
        COUNT(DISTINCT CASE WHEN second_order.order_id IS NOT NULL THEN first_order.customer_id END) 
        * 100.0 / COUNT(DISTINCT first_order.customer_id),
        1
    ) AS second_purchase_rate_pct,
    AVG(DATEDIFF(second_order.second_order_date, first_order.first_order_date)) AS avg_days_to_second_purchase
FROM (
    SELECT 
        customer_id,
        MIN(created_at) AS first_order_date,
        DATE_FORMAT(MIN(created_at), '%Y-%m-01') AS first_order_month
    FROM orders
    WHERE payment_status = 'paid'
      AND status NOT IN ('cancelled', 'refunded')
    GROUP BY customer_id
) first_order
LEFT JOIN (
    SELECT 
        o.customer_id,
        o.order_id,
        o.created_at AS second_order_date,
        ROW_NUMBER() OVER (PARTITION BY o.customer_id ORDER BY o.created_at) AS order_rank
    FROM orders o
    WHERE o.payment_status = 'paid'
      AND o.status NOT IN ('cancelled', 'refunded')
) second_order ON first_order.customer_id = second_order.customer_id
    AND second_order.order_rank = 2
GROUP BY DATE_FORMAT(first_order.first_order_month, '%Y-%m')
ORDER BY acquisition_month DESC;
```

---

### Query 29: Average Order Value Trend with Moving Average

```sql
WITH daily_aov AS (
    SELECT 
        DATE(created_at) AS sale_date,
        COUNT(DISTINCT order_id) AS orders,
        SUM(total_amount) AS revenue,
        AVG(total_amount) AS daily_aov
    FROM orders
    WHERE payment_status = 'paid'
      AND status NOT IN ('cancelled', 'refunded')
    GROUP BY DATE(created_at)
)
SELECT 
    sale_date,
    orders,
    ROUND(revenue, 2) AS revenue,
    ROUND(daily_aov, 2) AS daily_aov,
    -- 7-day moving average
    ROUND(AVG(daily_aov) OVER (
        ORDER BY sale_date 
        ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
    ), 2) AS moving_avg_7d,
    -- 30-day moving average
    ROUND(AVG(daily_aov) OVER (
        ORDER BY sale_date 
        ROWS BETWEEN 29 PRECEDING AND CURRENT ROW
    ), 2) AS moving_avg_30d
FROM daily_aov
ORDER BY sale_date DESC;
```

---

### Query 30: Executive Dashboard Summary (สรุปภาพรวมสำหรับผู้บริหาร)

```sql
-- One-query executive summary
SELECT 
    'Total Revenue (All Time)' AS metric,
    CONCAT('฿', FORMAT(SUM(total_amount), 0)) AS value,
    NULL AS change_pct
FROM orders WHERE payment_status = 'paid' AND status NOT IN ('cancelled', 'refunded')

UNION ALL

SELECT 
    'Revenue This Month',
    CONCAT('฿', FORMAT(SUM(total_amount), 0)),
    CONCAT(
        ROUND(
            (SUM(total_amount) - LAG(SUM(total_amount)) OVER ()) * 100.0 / 
            NULLIF(LAG(SUM(total_amount)) OVER (), 0), 
            1
        ),
        '%'
    )
FROM orders 
WHERE payment_status = 'paid' 
  AND MONTH(created_at) = MONTH(CURDATE()) AND YEAR(created_at) = YEAR(CURDATE())

UNION ALL

SELECT 
    'Total Customers',
    FORMAT(COUNT(*), 0),
    NULL
FROM customers WHERE is_active = TRUE

UNION ALL

SELECT 
    'Orders This Month',
    FORMAT(COUNT(*), 0),
    NULL
FROM orders 
WHERE MONTH(created_at) = MONTH(CURDATE()) AND YEAR(created_at) = YEAR(CURDATE())
  AND payment_status = 'paid'

UNION ALL

SELECT 
    'Average Order Value',
    CONCAT('฿', FORMAT(AVG(total_amount), 0)),
    NULL
FROM orders WHERE payment_status = 'paid' AND status NOT IN ('cancelled', 'refunded')

UNION ALL

SELECT 
    'Pending Orders',
    FORMAT(COUNT(*), 0),
    NULL
FROM orders WHERE status = 'pending'

UNION ALL

SELECT 
    'Low Stock Products',
    FORMAT(COUNT(DISTINCT p.product_id), 0),
    NULL
FROM inventory i
JOIN product_variants pv ON i.variant_id = pv.variant_id
JOIN products p ON pv.product_id = p.product_id
WHERE i.quantity <= i.reorder_point AND p.status = 'active';
```

---

### Query 31: Brand Performance Comparison

```sql
SELECT 
    b.name AS brand_name,
    COUNT(DISTINCT p.product_id) AS products_count,
    COUNT(DISTINCT oi.order_id) AS orders,
    SUM(oi.quantity) AS units_sold,
    ROUND(SUM(oi.total_price), 2) AS revenue,
    ROUND(AVG(pr.rating), 2) AS avg_rating,
    COUNT(DISTINCT pr.review_id) AS total_reviews,
    ROUND(
        SUM(oi.total_price) * 100.0 / SUM(SUM(oi.total_price)) OVER (),
        1
    ) AS market_share_pct,
    -- Growth vs previous period
    ROUND(
        SUM(CASE WHEN o.created_at >= DATE_SUB(CURDATE(), INTERVAL 30 DAY) THEN oi.total_price ELSE 0 END) -
        SUM(CASE WHEN o.created_at >= DATE_SUB(CURDATE(), INTERVAL 60 DAY) 
                 AND o.created_at < DATE_SUB(CURDATE(), INTERVAL 30 DAY) THEN oi.total_price ELSE 0 END),
        2
    ) AS revenue_change_30d
FROM brands b
JOIN products p ON b.brand_id = p.brand_id
JOIN product_variants pv ON p.product_id = pv.product_id
JOIN order_items oi ON pv.variant_id = oi.variant_id
JOIN orders o ON oi.order_id = o.order_id
LEFT JOIN product_reviews pr ON p.product_id = pr.product_id AND pr.is_approved = TRUE
WHERE o.payment_status = 'paid'
  AND o.status NOT IN ('cancelled', 'refunded')
GROUP BY b.brand_id, b.name
ORDER BY revenue DESC;
```

---

## แบบฝึกหัด (Challenge Exercises)

1. **Seasonality Query**: เขียน Query วิเคราะห์ Seasonal Pattern โดยเปรียบเทียบยอดขายในแต่ละ Quarter ของปีต่างๆ และแสดง % เปลี่ยนแปลง

2. **Product Velocity**: เขียน Query คำนวณ "Product Velocity" (ความเร็วในการขาย) โดยดูจาก จำนวนวันที่ใช้ขายหมด 1 batch สต็อก

3. **Customer Segmentation**: สร้าง Query ที่แบ่ง Segment ลูกค้าตาม 5 กลุ่ม (High Value High Frequency, High Value Low Frequency, Low Value High Frequency, Low Value Low Frequency, Dormant) พร้อมนับจำนวนและยอดรวมของแต่ละกลุ่ม

4. **Revenue Attribution**: เขียน Query ว่า Revenue ที่ได้มาจาก "Direct" (ไม่มี coupon) vs "Coupon-driven" คิดเป็นกี่เปอร์เซ็นต์ของแต่ละเดือน

5. **Predictive Reorder**: เขียน Query ที่ทำนายวันที่สต็อกจะหมดโดยใช้ Average Daily Sales Rate ของ 30 วันที่ผ่านมา

6. **Anomaly Detection**: เขียน Query ที่ระบุวันที่ยอดขายสูงหรือต่ำกว่า 2 Standard Deviation จาก Mean (คล้ายการ Detect anomaly)

7. **Category Cannibalization**: หาสินค้าในหมวดเดียวกันที่มี Revenue เพิ่มขึ้นในขณะที่สินค้าอื่นในหมวดเดียวกันลดลง (พฤติกรรม Cannibalization)

8. **First-time vs Repeat Buyer Basket**: เปรียบเทียบ Average Basket Size และ Product Mix ระหว่างลูกค้าที่ซื้อครั้งแรก vs ครั้งที่ 2+ 

9. **Inventory Health Dashboard**: สร้าง Single Query ที่แสดง: Total SKUs, In Stock, Low Stock, Out of Stock, Total Inventory Value, Average Days of Supply

10. **Customer Journey**: เขียน Query ที่แสดง "Customer Journey" ว่าลูกค้าส่วนใหญ่ซื้อสินค้าอะไรเป็นอันดับ 1, 2, 3 ตามลำดับ (First, Second, Third purchase category)
