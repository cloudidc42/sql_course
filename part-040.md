# Part 040: Real-world Aggregation Projects

## บทนำ (Introduction)

ในบทสุดท้ายของ Section นี้ เราจะนำ **ทุกสิ่งที่เรียนมาตั้งแต่ Part 031-039** มาใช้งานจริงในรูปแบบ **Business Intelligence Queries** ที่สมบูรณ์

แต่ละ Project จะมีการอธิบายอย่างละเอียด:
1. **Background** - ทำไมถึงต้องการ Query นี้
2. **SQL Query** - โค้ดพร้อมอธิบาย
3. **Business Interpretation** - วิธีอ่านและใช้ผลลัพธ์

---

## การตั้งค่าฐานข้อมูล

```sql
USE ecommerce_db;

-- ตรวจสอบข้อมูลทั้งหมด
SELECT 
    'orders' AS tbl, COUNT(*) AS rows FROM orders
UNION ALL SELECT 'order_items', COUNT(*) FROM order_items
UNION ALL SELECT 'products', COUNT(*) FROM products
UNION ALL SELECT 'customers', COUNT(*) FROM customers
UNION ALL SELECT 'employees', COUNT(*) FROM employees
UNION ALL SELECT 'departments', COUNT(*) FROM departments;
```

---

## Project 1: Complete Sales Dashboard

### Background
CEO ต้องการ Dashboard สรุปยอดขายสำหรับการประชุมรายสัปดาห์ ต้องแสดง KPIs หลัก

```sql
-- ===== COMPLETE SALES DASHBOARD =====
-- Query 1: Executive KPI Summary

SELECT 
    -- Revenue KPIs
    SUM(CASE WHEN status NOT IN ('cancelled') THEN total_amount ELSE 0 END) AS total_revenue,
    SUM(CASE WHEN status NOT IN ('cancelled') THEN total_amount - discount_amt ELSE 0 END) AS net_revenue,
    SUM(discount_amt) AS total_discounts_given,
    ROUND(SUM(discount_amt) * 100.0 / NULLIF(SUM(total_amount), 0), 2) AS discount_rate_pct,
    
    -- Order KPIs
    COUNT(*) AS total_orders,
    COUNT(CASE WHEN status = 'delivered' THEN 1 END) AS delivered_orders,
    COUNT(CASE WHEN status = 'cancelled' THEN 1 END) AS cancelled_orders,
    ROUND(COUNT(CASE WHEN status = 'cancelled' THEN 1 END) * 100.0 / COUNT(*), 2) AS cancellation_rate,
    
    -- Customer KPIs
    COUNT(DISTINCT customer_id) AS unique_customers,
    ROUND(SUM(CASE WHEN status NOT IN ('cancelled') THEN total_amount ELSE 0 END) / 
          NULLIF(COUNT(DISTINCT customer_id), 0), 0) AS revenue_per_customer,
    
    -- Average KPIs
    ROUND(AVG(CASE WHEN status NOT IN ('cancelled') THEN total_amount END), 0) AS avg_order_value,
    ROUND(AVG(CASE WHEN status NOT IN ('cancelled') THEN shipping_fee END), 0) AS avg_shipping_fee
FROM orders;

-- ผลลัพธ์ควรแสดง:
-- total_revenue: ยอดขายรวมทั้งหมด
-- net_revenue: ยอดขายสุทธิ (หลังส่วนลด)
-- cancellation_rate: อัตราการยกเลิก (ควรต่ำกว่า 5%)
-- avg_order_value: ยอดเฉลี่ยต่อออเดอร์ (KPI สำคัญมาก)
```

---

## Project 2: Monthly Sales Report

### Background
Finance Team ต้องการ Monthly P&L Report เพื่อ track performance รายเดือน

```sql
-- ===== MONTHLY SALES REPORT =====
SELECT 
    DATE_FORMAT(order_date, '%Y-%m') AS month,
    -- Volume
    COUNT(*) AS total_orders,
    COUNT(CASE WHEN status = 'delivered' THEN 1 END) AS completed_orders,
    COUNT(CASE WHEN status = 'cancelled' THEN 1 END) AS cancelled_orders,
    COUNT(DISTINCT customer_id) AS unique_customers,
    
    -- Revenue
    SUM(CASE WHEN status NOT IN ('cancelled') THEN total_amount ELSE 0 END) AS gross_revenue,
    SUM(CASE WHEN status NOT IN ('cancelled') THEN discount_amt ELSE 0 END) AS discounts,
    SUM(CASE WHEN status NOT IN ('cancelled') THEN total_amount - discount_amt ELSE 0 END) AS net_revenue,
    SUM(CASE WHEN status NOT IN ('cancelled') THEN shipping_fee ELSE 0 END) AS shipping_income,
    
    -- Averages
    ROUND(AVG(CASE WHEN status NOT IN ('cancelled') THEN total_amount END), 0) AS avg_order_value,
    
    -- Running Total (Month-over-Month)
    SUM(SUM(CASE WHEN status NOT IN ('cancelled') THEN total_amount ELSE 0 END)) 
        OVER (ORDER BY DATE_FORMAT(order_date, '%Y-%m')) AS ytd_revenue,
    
    -- Growth vs Previous Month
    SUM(CASE WHEN status NOT IN ('cancelled') THEN total_amount ELSE 0 END) -
    LAG(SUM(CASE WHEN status NOT IN ('cancelled') THEN total_amount ELSE 0 END))
        OVER (ORDER BY DATE_FORMAT(order_date, '%Y-%m')) AS mom_revenue_change,
    
    ROUND(
        (SUM(CASE WHEN status NOT IN ('cancelled') THEN total_amount ELSE 0 END) -
         LAG(SUM(CASE WHEN status NOT IN ('cancelled') THEN total_amount ELSE 0 END))
             OVER (ORDER BY DATE_FORMAT(order_date, '%Y-%m'))) * 100.0 /
        NULLIF(LAG(SUM(CASE WHEN status NOT IN ('cancelled') THEN total_amount ELSE 0 END))
            OVER (ORDER BY DATE_FORMAT(order_date, '%Y-%m')), 0),
        2
    ) AS mom_growth_pct
FROM orders
GROUP BY DATE_FORMAT(order_date, '%Y-%m')
ORDER BY month;

-- วิธีอ่านผล:
-- mom_growth_pct > 0: การเติบโต (สัญญาณดี)
-- mom_growth_pct < 0: การลดลง (ต้องตรวจสอบ)
-- ytd_revenue: ติดตามว่าถึงเป้าหมายรายปีหรือยัง
```

---

## Project 3: Quarterly and Yearly Comparison

### Background
Board of Directors ต้องการรายงานเปรียบเทียบ YoY (Year-over-Year) performance

```sql
-- ===== QUARTERLY COMPARISON REPORT =====
SELECT 
    YEAR(order_date) AS year,
    QUARTER(order_date) AS quarter,
    COUNT(*) AS orders,
    COUNT(DISTINCT customer_id) AS customers,
    SUM(CASE WHEN status NOT IN ('cancelled') THEN total_amount ELSE 0 END) AS revenue,
    
    -- Year-over-Year Comparison
    LAG(SUM(CASE WHEN status NOT IN ('cancelled') THEN total_amount ELSE 0 END))
        OVER (PARTITION BY QUARTER(order_date) ORDER BY YEAR(order_date)) AS same_q_prev_year,
    
    ROUND(
        (SUM(CASE WHEN status NOT IN ('cancelled') THEN total_amount ELSE 0 END) -
         LAG(SUM(CASE WHEN status NOT IN ('cancelled') THEN total_amount ELSE 0 END))
             OVER (PARTITION BY QUARTER(order_date) ORDER BY YEAR(order_date))) * 100.0 /
        NULLIF(LAG(SUM(CASE WHEN status NOT IN ('cancelled') THEN total_amount ELSE 0 END))
            OVER (PARTITION BY QUARTER(order_date) ORDER BY YEAR(order_date)), 0),
        2
    ) AS yoy_growth_pct
FROM orders
GROUP BY YEAR(order_date), QUARTER(order_date)
ORDER BY year, quarter;
```

---

## Project 4: Top N Analysis

### Background
Product Team ต้องการรู้ว่าสินค้าไหนขายดีที่สุด และลูกค้าไหนมีคุณค่าสูงสุด

```sql
-- ===== TOP 10 PRODUCTS BY REVENUE =====
SELECT 
    rnk,
    product_name,
    category,
    brand,
    times_ordered,
    total_units_sold,
    FORMAT(total_revenue, 0) AS revenue_formatted,
    FORMAT(total_cost, 0) AS cost_formatted,
    FORMAT(gross_profit, 0) AS profit_formatted,
    CONCAT(gross_margin_pct, '%') AS margin,
    current_stock
FROM (
    SELECT 
        RANK() OVER (ORDER BY SUM(oi.quantity * oi.unit_price) DESC) AS rnk,
        p.product_name,
        p.category,
        p.brand,
        COUNT(DISTINCT oi.order_id) AS times_ordered,
        SUM(oi.quantity) AS total_units_sold,
        SUM(oi.quantity * oi.unit_price) AS total_revenue,
        SUM(oi.quantity * p.cost) AS total_cost,
        SUM(oi.quantity * (oi.unit_price - p.cost)) AS gross_profit,
        ROUND(
            SUM(oi.quantity * (oi.unit_price - p.cost)) * 100.0 /
            NULLIF(SUM(oi.quantity * oi.unit_price), 0), 1
        ) AS gross_margin_pct,
        p.stock_qty AS current_stock
    FROM products p
    JOIN order_items oi ON p.product_id = oi.product_id
    JOIN orders o ON oi.order_id = o.order_id
    WHERE o.status NOT IN ('cancelled')
    GROUP BY p.product_id, p.product_name, p.category, p.brand, p.stock_qty
) AS ranked_products
WHERE rnk <= 10
ORDER BY rnk;

-- วิธีอ่านผล:
-- สินค้าที่มี revenue สูงแต่ margin ต่ำ = ควรพิจารณาปรับราคา
-- สินค้าที่ stock น้อยแต่ขายดี = ต้องเติม stock ด่วน
```

---

## Project 5: Market Basket Analysis

### Background
Marketing Team ต้องการรู้ว่าสินค้าไหนมักถูกซื้อพร้อมกัน เพื่อทำ Bundle Promotion

```sql
-- ===== MARKET BASKET ANALYSIS =====
-- หาคู่สินค้าที่ถูกซื้อพร้อมกัน
SELECT 
    pa.product_name AS product_a,
    pb.product_name AS product_b,
    COUNT(*) AS co_purchase_count,
    -- Support: สัดส่วนของออเดอร์ที่มีทั้งคู่
    ROUND(COUNT(*) * 100.0 / (SELECT COUNT(DISTINCT order_id) FROM order_items), 2) AS support_pct,
    -- ราคารวมถ้า bundle
    pa.price + pb.price AS combined_price
FROM order_items a
JOIN order_items b ON a.order_id = b.order_id 
    AND a.product_id < b.product_id   -- ป้องกัน duplicate pairs
JOIN products pa ON a.product_id = pa.product_id
JOIN products pb ON b.product_id = pb.product_id
GROUP BY a.product_id, b.product_id, pa.product_name, pb.product_name, pa.price, pb.price
HAVING COUNT(*) >= 2
ORDER BY co_purchase_count DESC, support_pct DESC;

-- วิธีใช้ผล:
-- คู่สินค้าที่มี co_purchase_count สูง = ควรทำ Bundle Deal
-- เช่น iPhone + AirPods → "Apple Bundle"
```

---

## Project 6: Customer Segmentation

### Background
CRM Team ต้องการแบ่ง Segment ลูกค้าเพื่อทำ Targeted Marketing

```sql
-- ===== CUSTOMER SEGMENTATION (RFM Analysis) =====
WITH customer_rfm AS (
    SELECT 
        c.customer_id,
        CONCAT(c.first_name, ' ', c.last_name) AS customer_name,
        c.city,
        c.province,
        
        -- Recency: วันที่ซื้อล่าสุด (น้อย = ดี)
        DATEDIFF(CURDATE(), MAX(o.order_date)) AS recency_days,
        
        -- Frequency: จำนวนครั้งที่ซื้อ (มาก = ดี)
        COUNT(o.order_id) AS frequency,
        
        -- Monetary: ยอดเงินรวม (มาก = ดี)
        SUM(o.total_amount) AS monetary_value,
        
        -- Average Order Value
        ROUND(AVG(o.total_amount), 0) AS avg_order_value,
        
        -- Additional metrics
        MIN(o.order_date) AS first_purchase_date,
        MAX(o.order_date) AS last_purchase_date,
        c.loyalty_points
    FROM customers c
    JOIN orders o ON c.customer_id = o.customer_id
    WHERE o.status != 'cancelled'
    GROUP BY c.customer_id, c.first_name, c.last_name, c.city, c.province, c.loyalty_points
),
rfm_scored AS (
    SELECT *,
        -- RFM Scoring (5 = Best, 1 = Worst)
        NTILE(5) OVER (ORDER BY recency_days ASC) AS r_score,   -- น้อย = ดี
        NTILE(5) OVER (ORDER BY frequency DESC) AS f_score,       -- มาก = ดี
        NTILE(5) OVER (ORDER BY monetary_value DESC) AS m_score   -- มาก = ดี
    FROM customer_rfm
)
SELECT 
    customer_id,
    customer_name,
    city,
    recency_days,
    frequency,
    FORMAT(monetary_value, 0) AS lifetime_value,
    r_score,
    f_score,
    m_score,
    r_score + f_score + m_score AS rfm_total,
    -- Customer Segment
    CASE 
        WHEN r_score >= 4 AND f_score >= 4 AND m_score >= 4 THEN '🏆 Champions'
        WHEN r_score >= 3 AND f_score >= 3 AND m_score >= 3 THEN '⭐ Loyal Customers'
        WHEN r_score >= 4 THEN '🆕 Recent Customers'
        WHEN f_score >= 4 THEN '🔄 Frequent Buyers'
        WHEN m_score >= 4 THEN '💰 Big Spenders'
        WHEN r_score <= 2 AND f_score <= 2 THEN '⚠️ At Risk / Lost'
        WHEN r_score <= 2 THEN '😴 Hibernating'
        ELSE '🌱 Potential'
    END AS segment
FROM rfm_scored
ORDER BY rfm_total DESC, monetary_value DESC;

-- วิธีใช้ผล:
-- Champions: ส่งโปรโมชั่น Exclusive, ขอ Review
-- At Risk: ส่ง Re-engagement Email, ส่วนลด Win-back
-- Recent: แนะนำสินค้าอื่น, ทำให้กลายเป็น Loyal
```

---

## Project 7: KPI Calculations

### Background
Operations Team ต้องการ Dashboard KPIs สำหรับ Operational Excellence

```sql
-- ===== OPERATIONAL KPI DASHBOARD =====
SELECT 
    -- Delivery Performance
    'Delivery Rate' AS kpi,
    CONCAT(ROUND(
        COUNT(CASE WHEN status = 'delivered' THEN 1 END) * 100.0 / 
        NULLIF(COUNT(CASE WHEN status NOT IN ('cancelled') THEN 1 END), 0),
        1
    ), '%') AS value,
    '> 90%' AS target,
    CASE WHEN COUNT(CASE WHEN status = 'delivered' THEN 1 END) * 100.0 / 
              NULLIF(COUNT(CASE WHEN status NOT IN ('cancelled') THEN 1 END), 0) >= 90 
         THEN '✅ On Target' ELSE '❌ Below Target' END AS status_flag
FROM orders

UNION ALL

SELECT 
    'Cancellation Rate',
    CONCAT(ROUND(
        COUNT(CASE WHEN status = 'cancelled' THEN 1 END) * 100.0 / COUNT(*), 1
    ), '%'),
    '< 5%',
    CASE WHEN COUNT(CASE WHEN status = 'cancelled' THEN 1 END) * 100.0 / COUNT(*) < 5
         THEN '✅ On Target' ELSE '❌ Below Target' END
FROM orders

UNION ALL

SELECT 
    'Avg Order Value',
    CONCAT('฿', FORMAT(AVG(CASE WHEN status NOT IN ('cancelled') THEN total_amount END), 0)),
    '฿30,000+',
    CASE WHEN AVG(CASE WHEN status NOT IN ('cancelled') THEN total_amount END) >= 30000
         THEN '✅ On Target' ELSE '❌ Below Target' END
FROM orders

UNION ALL

SELECT 
    'Products In Stock',
    CONCAT(
        COUNT(CASE WHEN stock_qty >= min_stock THEN 1 END), '/',
        COUNT(*), ' items'
    ),
    '100%',
    CASE WHEN COUNT(CASE WHEN stock_qty < min_stock THEN 1 END) = 0
         THEN '✅ On Target' ELSE '⚠️ Low Stock Items' END
FROM products
WHERE is_active = TRUE;
```

---

## Project 8: Inventory Analytics

### Background
Supply Chain Team ต้องการรู้สถานะ Inventory และวางแผนการสั่งซื้อ

```sql
-- ===== INVENTORY ANALYTICS REPORT =====
SELECT 
    p.product_id,
    p.product_name,
    p.category,
    p.brand,
    
    -- Current Stock
    p.stock_qty AS current_stock,
    p.min_stock AS reorder_point,
    
    -- Sales Velocity (ชิ้นต่อเดือน)
    ROUND(
        COALESCE(SUM(oi.quantity), 0) / 
        NULLIF(COUNT(DISTINCT DATE_FORMAT(o.order_date, '%Y-%m')), 0),
        1
    ) AS monthly_velocity,
    
    -- Days of Inventory (วันที่สต็อกจะหมด)
    ROUND(
        p.stock_qty / 
        NULLIF(COALESCE(SUM(oi.quantity), 0) / 
               NULLIF(COUNT(DISTINCT DATE_FORMAT(o.order_date, '%Y-%m')), 0) / 30, 0),
        0
    ) AS days_of_inventory,
    
    -- Inventory Value
    FORMAT(p.stock_qty * p.cost, 0) AS inventory_cost,
    FORMAT(p.stock_qty * p.price, 0) AS inventory_retail,
    
    -- Stock Status
    CASE 
        WHEN p.stock_qty = 0                           THEN '🔴 OUT OF STOCK'
        WHEN p.stock_qty <= p.min_stock                THEN '🟠 REORDER NOW'
        WHEN p.stock_qty <= p.min_stock * 2            THEN '🟡 REORDER SOON'
        ELSE '🟢 ADEQUATE'
    END AS stock_status,
    
    -- Revenue since product was created
    COALESCE(SUM(oi.quantity * oi.unit_price), 0) AS total_revenue_ever
FROM products p
LEFT JOIN order_items oi ON p.product_id = oi.product_id
LEFT JOIN orders o ON oi.order_id = o.order_id 
    AND o.status NOT IN ('cancelled')
WHERE p.is_active = TRUE
GROUP BY p.product_id, p.product_name, p.category, p.brand, 
         p.stock_qty, p.min_stock, p.cost, p.price
ORDER BY 
    CASE 
        WHEN p.stock_qty = 0 THEN 1
        WHEN p.stock_qty <= p.min_stock THEN 2
        WHEN p.stock_qty <= p.min_stock * 2 THEN 3
        ELSE 4
    END,
    monthly_velocity DESC;
```

---

## Project 9: Executive Summary

### Background
สร้าง Executive Summary Report ที่ครอบคลุมทุก Department สำหรับ Board Meeting

```sql
-- ===== EXECUTIVE SUMMARY REPORT =====

-- SECTION 1: Business Overview
SELECT '=== BUSINESS OVERVIEW ===' AS section, '' AS metric, '' AS value

UNION ALL

SELECT '', 'Total Revenue (All Time)',
    CONCAT('฿', FORMAT(SUM(CASE WHEN status != 'cancelled' THEN total_amount ELSE 0 END), 0))
FROM orders

UNION ALL

SELECT '', 'Total Orders',
    FORMAT(COUNT(*), 0)
FROM orders

UNION ALL

SELECT '', 'Total Customers',
    FORMAT(COUNT(*), 0)
FROM customers

UNION ALL

SELECT '', 'Active Products',
    FORMAT(COUNT(*), 0)
FROM products WHERE is_active = TRUE

UNION ALL

SELECT '', 'Total Employees',
    FORMAT(COUNT(*), 0)
FROM employees

UNION ALL

SELECT '=== 2024 PERFORMANCE ===' AS section, '' AS metric, '' AS value

UNION ALL

SELECT '', '2024 Revenue',
    CONCAT('฿', FORMAT(
        (SELECT SUM(total_amount) FROM orders 
         WHERE YEAR(order_date) = 2024 AND status != 'cancelled'), 0
    ))

UNION ALL

SELECT '', '2024 Order Count',
    FORMAT((SELECT COUNT(*) FROM orders WHERE YEAR(order_date) = 2024), 0)

UNION ALL

SELECT '', 'Average Order Value 2024',
    CONCAT('฿', FORMAT(
        (SELECT AVG(total_amount) FROM orders 
         WHERE YEAR(order_date) = 2024 AND status != 'cancelled'), 0
    ))

UNION ALL

SELECT '', 'Cancellation Rate 2024',
    CONCAT(ROUND(
        (SELECT COUNT(*) FROM orders WHERE YEAR(order_date) = 2024 AND status = 'cancelled') * 100.0 /
        NULLIF((SELECT COUNT(*) FROM orders WHERE YEAR(order_date) = 2024), 0),
        1
    ), '%')

UNION ALL

SELECT '=== TOP METRICS ===' AS section, '' AS metric, '' AS value

UNION ALL

SELECT '', 'Best Selling Category',
    (SELECT p.category FROM products p
     JOIN order_items oi ON p.product_id = oi.product_id
     JOIN orders o ON oi.order_id = o.order_id
     WHERE o.status != 'cancelled'
     GROUP BY p.category
     ORDER BY SUM(oi.quantity * oi.unit_price) DESC LIMIT 1)

UNION ALL

SELECT '', 'Most Popular Payment',
    (SELECT payment_method FROM orders
     WHERE status != 'cancelled'
     GROUP BY payment_method
     ORDER BY COUNT(*) DESC LIMIT 1)

UNION ALL

SELECT '', 'Top Customer City',
    (SELECT c.city FROM customers c
     JOIN orders o ON c.customer_id = o.customer_id
     WHERE o.status != 'cancelled'
     GROUP BY c.city
     ORDER BY SUM(o.total_amount) DESC LIMIT 1);
```

---

## Project 10: Sales Trend Analysis

### Background
หา Trends ระยะยาวเพื่อวางแผนธุรกิจ

```sql
-- ===== SALES TREND ANALYSIS =====
WITH monthly_data AS (
    SELECT 
        DATE_FORMAT(order_date, '%Y-%m') AS month,
        YEAR(order_date) AS year,
        MONTH(order_date) AS month_num,
        COUNT(*) AS orders,
        COUNT(DISTINCT customer_id) AS customers,
        SUM(CASE WHEN status != 'cancelled' THEN total_amount ELSE 0 END) AS revenue
    FROM orders
    GROUP BY DATE_FORMAT(order_date, '%Y-%m'), YEAR(order_date), MONTH(order_date)
)
SELECT 
    month,
    orders,
    customers,
    FORMAT(revenue, 0) AS revenue,
    
    -- Month-over-Month
    revenue - LAG(revenue) OVER (ORDER BY month) AS mom_change,
    ROUND(
        (revenue - LAG(revenue) OVER (ORDER BY month)) * 100.0 /
        NULLIF(LAG(revenue) OVER (ORDER BY month), 0), 1
    ) AS mom_growth_pct,
    
    -- 3-Month Moving Average
    FORMAT(ROUND(AVG(revenue) OVER (
        ORDER BY month ROWS BETWEEN 2 PRECEDING AND CURRENT ROW
    ), 0), 0) AS ma3_revenue,
    
    -- YTD Running Total
    FORMAT(SUM(revenue) OVER (ORDER BY month), 0) AS ytd_revenue,
    
    -- Trend Direction
    CASE 
        WHEN revenue > LAG(revenue) OVER (ORDER BY month) THEN '📈 Growing'
        WHEN revenue < LAG(revenue) OVER (ORDER BY month) THEN '📉 Declining'
        ELSE '➡️ Flat'
    END AS trend
FROM monthly_data
ORDER BY month;
```

---

## Project 11: Product Profitability Analysis

### Background
Finance ต้องการรู้ว่าสินค้าไหนทำกำไรจริงๆ (ไม่ใช่แค่ยอดขาย)

```sql
-- ===== PRODUCT PROFITABILITY ANALYSIS =====
SELECT 
    p.product_id,
    p.product_name,
    p.category,
    p.brand,
    
    -- Volume
    COALESCE(SUM(oi.quantity), 0) AS units_sold,
    
    -- Revenue
    COALESCE(SUM(oi.quantity * oi.unit_price), 0) AS gross_revenue,
    
    -- Cost
    COALESCE(SUM(oi.quantity * p.cost), 0) AS total_cogs,
    
    -- Gross Profit
    COALESCE(SUM(oi.quantity * (oi.unit_price - p.cost)), 0) AS gross_profit,
    
    -- Margin
    ROUND(
        COALESCE(SUM(oi.quantity * (oi.unit_price - p.cost)), 0) * 100.0 /
        NULLIF(COALESCE(SUM(oi.quantity * oi.unit_price), 0), 0),
        2
    ) AS gross_margin_pct,
    
    -- Profitability Rank
    RANK() OVER (ORDER BY COALESCE(SUM(oi.quantity * (oi.unit_price - p.cost)), 0) DESC) AS profit_rank,
    
    -- Product Classification
    CASE 
        WHEN COALESCE(SUM(oi.quantity * (oi.unit_price - p.cost)), 0) * 100.0 /
             NULLIF(COALESCE(SUM(oi.quantity * oi.unit_price), 0), 0) >= 30 
             AND COALESCE(SUM(oi.quantity * oi.unit_price), 0) >= 
                 (SELECT AVG(rev) FROM (
                     SELECT SUM(oi2.quantity * oi2.unit_price) AS rev
                     FROM order_items oi2 GROUP BY oi2.product_id
                 ) AS avg_rev)
             THEN '🌟 Star Product'
        WHEN COALESCE(SUM(oi.quantity * (oi.unit_price - p.cost)), 0) * 100.0 /
             NULLIF(COALESCE(SUM(oi.quantity * oi.unit_price), 0), 0) >= 30 
             THEN '💎 High Margin'
        WHEN COALESCE(SUM(oi.quantity * oi.unit_price), 0) >= 
             (SELECT AVG(rev) FROM (
                 SELECT SUM(oi3.quantity * oi3.unit_price) AS rev
                 FROM order_items oi3 GROUP BY oi3.product_id
             ) AS avg_rev2)
             THEN '📦 Volume Driver'
        WHEN COALESCE(SUM(oi.quantity), 0) = 0 THEN '💤 No Sales'
        ELSE '⚠️ Review Needed'
    END AS product_classification
FROM products p
LEFT JOIN order_items oi ON p.product_id = oi.product_id
LEFT JOIN orders o ON oi.order_id = o.order_id 
    AND o.status NOT IN ('cancelled')
WHERE p.is_active = TRUE
GROUP BY p.product_id, p.product_name, p.category, p.brand, p.cost
ORDER BY gross_profit DESC;
```

---

## Project 12: Customer Retention Analysis

### Background
Marketing ต้องการวัด Customer Retention Rate รายเดือน

```sql
-- ===== CUSTOMER RETENTION ANALYSIS =====

-- หาลูกค้าที่ Active แต่ละเดือน
WITH monthly_customers AS (
    SELECT 
        customer_id,
        DATE_FORMAT(order_date, '%Y-%m') AS active_month
    FROM orders
    WHERE status != 'cancelled'
    GROUP BY customer_id, DATE_FORMAT(order_date, '%Y-%m')
),
retention AS (
    SELECT 
        curr.active_month AS current_month,
        COUNT(DISTINCT curr.customer_id) AS current_customers,
        -- ลูกค้าที่ยังคงอยู่จากเดือนก่อน (Retained)
        COUNT(DISTINCT CASE 
            WHEN prev.customer_id IS NOT NULL THEN curr.customer_id 
        END) AS retained_customers,
        -- ลูกค้าใหม่
        COUNT(DISTINCT CASE 
            WHEN prev.customer_id IS NULL THEN curr.customer_id 
        END) AS new_customers
    FROM monthly_customers curr
    LEFT JOIN monthly_customers prev 
        ON curr.customer_id = prev.customer_id
        AND prev.active_month = DATE_FORMAT(
            DATE_SUB(STR_TO_DATE(CONCAT(curr.active_month, '-01'), '%Y-%m-%d'), INTERVAL 1 MONTH),
            '%Y-%m'
        )
    GROUP BY curr.active_month
)
SELECT 
    current_month,
    current_customers,
    retained_customers,
    new_customers,
    current_customers - retained_customers - new_customers AS recovered_customers,
    ROUND(retained_customers * 100.0 / NULLIF(
        LAG(current_customers) OVER (ORDER BY current_month), 0
    ), 1) AS retention_rate_pct
FROM retention
ORDER BY current_month;
```

---

## Project 13: Revenue Attribution

### Background
ต้องการรู้ว่า Revenue มาจากช่องทางไหน (Province, Payment Method, Product Category)

```sql
-- ===== REVENUE ATTRIBUTION ANALYSIS =====
-- Multi-dimensional revenue breakdown

SELECT 
    'Province' AS dimension,
    c.province AS dimension_value,
    COUNT(DISTINCT o.order_id) AS orders,
    FORMAT(SUM(o.total_amount), 0) AS revenue,
    ROUND(SUM(o.total_amount) * 100.0 / 
        (SELECT SUM(total_amount) FROM orders WHERE status != 'cancelled'), 2) AS revenue_share_pct
FROM orders o
JOIN customers c ON o.customer_id = c.customer_id
WHERE o.status != 'cancelled'
GROUP BY c.province

UNION ALL

SELECT 
    'Payment Method',
    payment_method,
    COUNT(DISTINCT order_id),
    FORMAT(SUM(total_amount), 0),
    ROUND(SUM(total_amount) * 100.0 / 
        (SELECT SUM(total_amount) FROM orders WHERE status != 'cancelled'), 2)
FROM orders
WHERE status != 'cancelled'
GROUP BY payment_method

UNION ALL

SELECT 
    'Product Category',
    p.category,
    COUNT(DISTINCT o.order_id),
    FORMAT(SUM(oi.quantity * oi.unit_price), 0),
    ROUND(SUM(oi.quantity * oi.unit_price) * 100.0 /
        (SELECT SUM(oi2.quantity * oi2.unit_price) 
         FROM order_items oi2 
         JOIN orders o2 ON oi2.order_id = o2.order_id 
         WHERE o2.status != 'cancelled'), 2)
FROM orders o
JOIN order_items oi ON o.order_id = oi.order_id
JOIN products p ON oi.product_id = p.product_id
WHERE o.status != 'cancelled'
GROUP BY p.category

ORDER BY dimension, revenue_share_pct DESC;
```

---

## Project 14: Cohort Analysis

### Background
Product Team ต้องการดู Cohort Retention ว่าลูกค้าแต่ละ Cohort กลับมาซื้อซ้ำแค่ไหน

```sql
-- ===== COHORT RETENTION ANALYSIS =====
WITH first_orders AS (
    -- หาว่าลูกค้าแต่ละคนซื้อครั้งแรกเดือนไหน
    SELECT 
        customer_id,
        DATE_FORMAT(MIN(order_date), '%Y-%m') AS cohort_month
    FROM orders
    WHERE status != 'cancelled'
    GROUP BY customer_id
),
cohort_purchases AS (
    SELECT 
        fo.cohort_month,
        fo.customer_id,
        DATE_FORMAT(o.order_date, '%Y-%m') AS purchase_month
    FROM first_orders fo
    JOIN orders o ON fo.customer_id = o.customer_id
    WHERE o.status != 'cancelled'
),
cohort_summary AS (
    SELECT 
        cohort_month,
        purchase_month,
        COUNT(DISTINCT customer_id) AS customers
    FROM cohort_purchases
    GROUP BY cohort_month, purchase_month
)
SELECT 
    c.cohort_month,
    COUNT(DISTINCT fo.customer_id) AS cohort_size,
    -- Month 0: การซื้อครั้งแรก (ควรเป็น 100%)
    SUM(CASE WHEN c.purchase_month = c.cohort_month 
             THEN c.customers ELSE 0 END) AS m0,
    -- Month 1: กลับมาซื้อในเดือนถัดไป
    SUM(CASE WHEN PERIOD_DIFF(
                 EXTRACT(YEAR_MONTH FROM STR_TO_DATE(CONCAT(c.purchase_month, '-01'), '%Y-%m-%d')),
                 EXTRACT(YEAR_MONTH FROM STR_TO_DATE(CONCAT(c.cohort_month, '-01'), '%Y-%m-%d'))
             ) = 1 THEN c.customers ELSE 0 END) AS m1,
    -- Month 2:
    SUM(CASE WHEN PERIOD_DIFF(
                 EXTRACT(YEAR_MONTH FROM STR_TO_DATE(CONCAT(c.purchase_month, '-01'), '%Y-%m-%d')),
                 EXTRACT(YEAR_MONTH FROM STR_TO_DATE(CONCAT(c.cohort_month, '-01'), '%Y-%m-%d'))
             ) = 2 THEN c.customers ELSE 0 END) AS m2,
    -- Month 3:
    SUM(CASE WHEN PERIOD_DIFF(
                 EXTRACT(YEAR_MONTH FROM STR_TO_DATE(CONCAT(c.purchase_month, '-01'), '%Y-%m-%d')),
                 EXTRACT(YEAR_MONTH FROM STR_TO_DATE(CONCAT(c.cohort_month, '-01'), '%Y-%m-%d'))
             ) = 3 THEN c.customers ELSE 0 END) AS m3
FROM cohort_summary c
JOIN first_orders fo ON c.cohort_month = fo.cohort_month
GROUP BY c.cohort_month
ORDER BY cohort_month;
```

---

## Project 15: Complete Business Intelligence Query Suite

### Background
สร้าง Comprehensive BI Report ที่รวมทุกมิติของธุรกิจ

```sql
-- ===== COMPLETE BI REPORT =====
-- รันครั้งเดียว ได้ข้อมูลครบทุกด้าน

-- === PART 1: Revenue Summary ===
SELECT '1. REVENUE SUMMARY' AS section, NULL AS category, NULL AS year,
    SUM(total_amount) AS revenue, COUNT(*) AS orders,
    COUNT(DISTINCT customer_id) AS customers
FROM orders WHERE status != 'cancelled'

UNION ALL

-- === PART 2: Revenue by Category ===
SELECT '2. CATEGORY BREAKDOWN', p.category, NULL,
    SUM(oi.quantity * oi.unit_price),
    COUNT(DISTINCT o.order_id),
    COUNT(DISTINCT o.customer_id)
FROM orders o
JOIN order_items oi ON o.order_id = oi.order_id
JOIN products p ON oi.product_id = p.product_id
WHERE o.status != 'cancelled'
GROUP BY p.category

UNION ALL

-- === PART 3: Revenue by Year ===
SELECT '3. YEARLY BREAKDOWN', NULL, CAST(YEAR(order_date) AS CHAR),
    SUM(total_amount), COUNT(*), COUNT(DISTINCT customer_id)
FROM orders WHERE status != 'cancelled'
GROUP BY YEAR(order_date)

ORDER BY section, revenue DESC;
```

---

## สรุปบทที่ 40 และ Section ทั้งหมด

### สิ่งที่เรียนมาตั้งแต่ Part 031-040

| Part | หัวข้อ | Skills |
|------|--------|--------|
| 031 | Aggregate Functions | COUNT, SUM, AVG, MIN, MAX |
| 032 | GROUP BY | Grouping Data |
| 033 | HAVING | Filter Groups |
| 034 | Advanced Aggregation | Conditional, Pivot, Percentage |
| 035 | ROLLUP | Subtotals |
| 036 | CUBE | All Combinations |
| 037 | GROUPING SETS | Custom Groupings |
| 038 | Window Functions | Running Totals, Ranking |
| 039 | Statistical Functions | STDDEV, Percentiles |
| 040 | BI Projects | Real-world Applications |

---

## แบบฝึกหัดท้ายบท

### แบบฝึกหัดที่ 1
สร้าง Complete Sales KPI Report ที่แสดง Revenue, Orders, Customers, Avg Order Value

```sql
-- เฉลย:
SELECT 
    COUNT(*) AS total_orders,
    COUNT(CASE WHEN status != 'cancelled' THEN 1 END) AS completed_orders,
    SUM(CASE WHEN status != 'cancelled' THEN total_amount ELSE 0 END) AS total_revenue,
    COUNT(DISTINCT customer_id) AS unique_customers,
    ROUND(AVG(CASE WHEN status != 'cancelled' THEN total_amount END), 0) AS avg_order_value,
    ROUND(COUNT(CASE WHEN status = 'cancelled' THEN 1 END) * 100.0 / COUNT(*), 2) AS cancel_rate
FROM orders;
```

### แบบฝึกหัดที่ 2
สร้าง Monthly Revenue Report พร้อม Running Total และ MoM Growth

```sql
-- เฉลย:
SELECT 
    DATE_FORMAT(order_date, '%Y-%m') AS month,
    SUM(total_amount) AS revenue,
    SUM(SUM(total_amount)) OVER (ORDER BY DATE_FORMAT(order_date, '%Y-%m')) AS ytd,
    SUM(total_amount) - LAG(SUM(total_amount)) OVER (ORDER BY DATE_FORMAT(order_date, '%Y-%m')) AS mom_change,
    ROUND((SUM(total_amount) - LAG(SUM(total_amount)) OVER (ORDER BY DATE_FORMAT(order_date, '%Y-%m'))) * 100.0 /
          NULLIF(LAG(SUM(total_amount)) OVER (ORDER BY DATE_FORMAT(order_date, '%Y-%m')), 0), 2) AS mom_pct
FROM orders WHERE status != 'cancelled'
GROUP BY DATE_FORMAT(order_date, '%Y-%m')
ORDER BY month;
```

### แบบฝึกหัดที่ 3
หา Top 5 Products ตาม Revenue

```sql
-- เฉลย:
SELECT 
    p.product_name,
    p.category,
    SUM(oi.quantity * oi.unit_price) AS revenue,
    SUM(oi.quantity) AS units_sold
FROM products p
JOIN order_items oi ON p.product_id = oi.product_id
JOIN orders o ON oi.order_id = o.order_id
WHERE o.status != 'cancelled'
GROUP BY p.product_id, p.product_name, p.category
ORDER BY revenue DESC
LIMIT 5;
```

### แบบฝึกหัดที่ 4
Segment ลูกค้าเป็น Champions, Loyal, At Risk ตาม RFM

```sql
-- เฉลย:
SELECT 
    c.customer_id,
    CONCAT(c.first_name, ' ', c.last_name) AS name,
    COUNT(o.order_id) AS frequency,
    SUM(o.total_amount) AS monetary,
    DATEDIFF(CURDATE(), MAX(o.order_date)) AS recency,
    CASE 
        WHEN COUNT(o.order_id) >= 4 AND SUM(o.total_amount) >= 100000 THEN 'Champions'
        WHEN COUNT(o.order_id) >= 2 AND DATEDIFF(CURDATE(), MAX(o.order_date)) <= 90 THEN 'Loyal'
        WHEN DATEDIFF(CURDATE(), MAX(o.order_date)) > 180 THEN 'At Risk'
        ELSE 'Regular'
    END AS segment
FROM customers c
JOIN orders o ON c.customer_id = o.customer_id
WHERE o.status != 'cancelled'
GROUP BY c.customer_id, c.first_name, c.last_name
ORDER BY monetary DESC;
```

### แบบฝึกหัดที่ 5
สร้าง Product Profitability Report พร้อม Margin %

```sql
-- เฉลย:
SELECT 
    p.product_name,
    p.category,
    SUM(oi.quantity * oi.unit_price) AS revenue,
    SUM(oi.quantity * p.cost) AS cost,
    SUM(oi.quantity * (oi.unit_price - p.cost)) AS profit,
    ROUND(SUM(oi.quantity * (oi.unit_price - p.cost)) * 100.0 /
          NULLIF(SUM(oi.quantity * oi.unit_price), 0), 2) AS margin_pct
FROM products p
JOIN order_items oi ON p.product_id = oi.product_id
JOIN orders o ON oi.order_id = o.order_id
WHERE o.status != 'cancelled'
GROUP BY p.product_id, p.product_name, p.category, p.cost
ORDER BY profit DESC;
```

### แบบฝึกหัดที่ 6
สร้าง Inventory Status Report

```sql
-- เฉลย:
SELECT 
    product_name,
    category,
    stock_qty,
    min_stock,
    CASE 
        WHEN stock_qty = 0 THEN 'Out of Stock'
        WHEN stock_qty < min_stock THEN 'Reorder Now'
        WHEN stock_qty < min_stock * 2 THEN 'Reorder Soon'
        ELSE 'OK'
    END AS status,
    stock_qty * price AS inventory_value
FROM products
WHERE is_active = TRUE
ORDER BY 
    CASE WHEN stock_qty = 0 THEN 1
         WHEN stock_qty < min_stock THEN 2
         WHEN stock_qty < min_stock * 2 THEN 3
         ELSE 4 END,
    product_name;
```

### แบบฝึกหัดที่ 7
คำนวณ YoY Revenue Growth แต่ละ Category

```sql
-- เฉลย:
SELECT 
    p.category,
    SUM(CASE WHEN YEAR(o.order_date) = 2024 THEN oi.quantity * oi.unit_price ELSE 0 END) AS rev_2024,
    SUM(CASE WHEN YEAR(o.order_date) = 2023 THEN oi.quantity * oi.unit_price ELSE 0 END) AS rev_2023,
    ROUND(
        (SUM(CASE WHEN YEAR(o.order_date) = 2024 THEN oi.quantity * oi.unit_price ELSE 0 END) -
         SUM(CASE WHEN YEAR(o.order_date) = 2023 THEN oi.quantity * oi.unit_price ELSE 0 END)) * 100.0 /
        NULLIF(SUM(CASE WHEN YEAR(o.order_date) = 2023 THEN oi.quantity * oi.unit_price ELSE 0 END), 0),
        2
    ) AS yoy_growth_pct
FROM products p
JOIN order_items oi ON p.product_id = oi.product_id
JOIN orders o ON oi.order_id = o.order_id
WHERE o.status != 'cancelled'
GROUP BY p.category
ORDER BY yoy_growth_pct DESC;
```

### แบบฝึกหัดที่ 8
สร้าง HR Summary Report แต่ละแผนก

```sql
-- เฉลย:
SELECT 
    d.department_name,
    COUNT(e.employee_id) AS headcount,
    MIN(e.salary) AS min_salary,
    MAX(e.salary) AS max_salary,
    ROUND(AVG(e.salary), 0) AS avg_salary,
    SUM(e.salary) AS monthly_payroll,
    SUM(e.salary) * 12 AS annual_payroll,
    d.budget,
    ROUND(SUM(e.salary) * 12 * 100.0 / NULLIF(d.budget, 0), 1) AS payroll_budget_pct
FROM departments d
LEFT JOIN employees e ON d.department_id = e.department_id
WHERE e.salary IS NOT NULL
GROUP BY d.department_id, d.department_name, d.budget
ORDER BY monthly_payroll DESC;
```

### แบบฝึกหัดที่ 9
สร้าง Customer Geographic Analysis

```sql
-- เฉลย:
SELECT 
    c.province,
    COUNT(DISTINCT c.customer_id) AS customers,
    COUNT(o.order_id) AS orders,
    SUM(o.total_amount) AS revenue,
    ROUND(AVG(o.total_amount), 0) AS avg_order,
    ROUND(SUM(o.total_amount) * 100.0 / 
          (SELECT SUM(total_amount) FROM orders WHERE status != 'cancelled'), 2) AS revenue_share
FROM customers c
LEFT JOIN orders o ON c.customer_id = o.customer_id
    AND o.status != 'cancelled'
GROUP BY c.province
ORDER BY revenue DESC;
```

### แบบฝึกหัดที่ 10
สร้าง Complete Executive Dashboard ที่รวมทุก KPI สำคัญ

```sql
-- เฉลย:
SELECT 
    -- Revenue
    FORMAT(SUM(CASE WHEN status != 'cancelled' THEN total_amount ELSE 0 END), 0) AS total_revenue,
    FORMAT(SUM(CASE WHEN status != 'cancelled' AND YEAR(order_date) = 2024 
               THEN total_amount ELSE 0 END), 0) AS revenue_2024,
    
    -- Orders
    COUNT(*) AS total_orders,
    ROUND(COUNT(CASE WHEN status = 'cancelled' THEN 1 END) * 100.0 / COUNT(*), 2) AS cancel_rate,
    
    -- Customers
    COUNT(DISTINCT customer_id) AS unique_customers,
    FORMAT(ROUND(SUM(CASE WHEN status != 'cancelled' THEN total_amount ELSE 0 END) /
           NULLIF(COUNT(DISTINCT customer_id), 0), 0), 0) AS revenue_per_customer,
    
    -- Averages
    FORMAT(ROUND(AVG(CASE WHEN status != 'cancelled' THEN total_amount END), 0), 0) AS avg_order_value,
    FORMAT(ROUND(AVG(CASE WHEN status != 'cancelled' THEN shipping_fee END), 0), 0) AS avg_shipping
FROM orders;
```

---

## บทสรุปส่วนที่ 3 (Part 031-040)

ยินดีด้วย! คุณได้เรียนรู้ **Aggregation และ Analytical SQL** อย่างครบถ้วนแล้ว

### ทักษะที่ได้รับ

1. **Aggregate Functions** - นับ, รวม, เฉลี่ย, หาค่า min/max
2. **GROUP BY** - จัดกลุ่มข้อมูลสำหรับการวิเคราะห์
3. **HAVING** - กรองกลุ่มที่ต้องการ
4. **Advanced Patterns** - Conditional Aggregation, Pivot, Percentage
5. **ROLLUP** - Hierarchical Subtotals
6. **CUBE** - Multi-dimensional Analysis
7. **GROUPING SETS** - Custom Grouping Combinations
8. **Window Functions** - Running Totals, Rankings
9. **Statistical Functions** - STDDEV, Percentiles, Z-Score
10. **BI Projects** - Real-world Business Intelligence

### บทต่อไป

Section ถัดไปจะเรียน **Subqueries, CTEs, และ Advanced Query Techniques** ซึ่งจะทำให้ Query ของคุณทรงพลังและอ่านง่ายยิ่งขึ้น!

---

*จบบทที่ 040 - Real-world Aggregation Projects*

*จบ Part 031-040: Aggregation and Business Intelligence*
