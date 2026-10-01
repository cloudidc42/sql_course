# Part 116: Inventory and Supply Chain Management

## บทนำ (Introduction)

ระบบบริหารคลังสินค้าและ Supply Chain ที่ดีช่วยลดต้นทุน เพิ่มประสิทธิภาพ และสร้างความพึงพอใจให้ลูกค้า บทนี้ครอบคลุม Multi-warehouse management, FIFO/LIFO costing, Reorder calculations และ Supply chain analytics

## 1. Complete Inventory Schema

```sql
-- =========================================
-- INVENTORY & SUPPLY CHAIN - COMPLETE DDL
-- =========================================

CREATE DATABASE inventory_db
    CHARACTER SET utf8mb4
    COLLATE utf8mb4_unicode_ci;

USE inventory_db;

-- =========================================
-- SECTION 1: WAREHOUSE MANAGEMENT
-- =========================================

-- ตาราง Warehouses (คลังสินค้า)
CREATE TABLE warehouses (
    warehouse_id    INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    code            VARCHAR(20) NOT NULL UNIQUE,
    name            VARCHAR(200) NOT NULL,
    type            ENUM('main','regional','retail','cross_dock','3pl') DEFAULT 'main',
    address         TEXT,
    city            VARCHAR(100),
    country_code    CHAR(2) DEFAULT 'TH',
    manager_name    VARCHAR(200),
    phone           VARCHAR(20),
    total_capacity  DECIMAL(12,2) COMMENT 'in cubic meters',
    used_capacity   DECIMAL(12,2) DEFAULT 0,
    is_active       BOOLEAN DEFAULT TRUE
) ENGINE=InnoDB;

-- ตาราง Warehouse Zones (โซนในคลัง)
CREATE TABLE warehouse_zones (
    zone_id         INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    warehouse_id    INT UNSIGNED NOT NULL,
    zone_code       VARCHAR(20) NOT NULL,
    zone_type       ENUM('storage','picking','packing','shipping','receiving','staging') DEFAULT 'storage',
    description     VARCHAR(200),
    FOREIGN KEY (warehouse_id) REFERENCES warehouses(warehouse_id),
    UNIQUE KEY uk_wh_zone (warehouse_id, zone_code)
) ENGINE=InnoDB;

-- ตาราง Stock Locations (ตำแหน่งจัดเก็บ)
CREATE TABLE stock_locations (
    location_id     INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    warehouse_id    INT UNSIGNED NOT NULL,
    zone_id         INT UNSIGNED,
    aisle           VARCHAR(10) NOT NULL,
    rack            VARCHAR(10) NOT NULL,
    level           VARCHAR(10) NOT NULL,
    bin             VARCHAR(10),
    location_code   VARCHAR(30) GENERATED ALWAYS AS (
        CONCAT(aisle, '-', rack, '-', level, IF(bin IS NOT NULL, CONCAT('-', bin), ''))
    ) STORED,
    capacity_units  INT DEFAULT 100,
    current_units   INT DEFAULT 0,
    is_active       BOOLEAN DEFAULT TRUE,
    FOREIGN KEY (warehouse_id) REFERENCES warehouses(warehouse_id),
    FOREIGN KEY (zone_id) REFERENCES warehouse_zones(zone_id) ON DELETE SET NULL,
    INDEX idx_warehouse (warehouse_id),
    INDEX idx_code (location_code)
) ENGINE=InnoDB;

-- =========================================
-- SECTION 2: PRODUCTS
-- =========================================

-- ตาราง Product Categories
CREATE TABLE categories (
    category_id     INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    parent_id       INT UNSIGNED,
    name            VARCHAR(200) NOT NULL,
    code            VARCHAR(20) NOT NULL UNIQUE,
    FOREIGN KEY (parent_id) REFERENCES categories(category_id) ON DELETE SET NULL
) ENGINE=InnoDB;

-- ตาราง Units of Measure
CREATE TABLE units_of_measure (
    uom_id          INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    code            VARCHAR(10) NOT NULL UNIQUE,
    name            VARCHAR(50) NOT NULL,
    base_quantity   DECIMAL(10,4) DEFAULT 1 COMMENT 'conversion to base UOM'
) ENGINE=InnoDB;

-- ตาราง Products (สินค้า/วัตถุดิบ)
CREATE TABLE products (
    product_id      INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    sku             VARCHAR(50) NOT NULL UNIQUE,
    barcode         VARCHAR(50),
    name            VARCHAR(300) NOT NULL,
    category_id     INT UNSIGNED,
    uom_id          INT UNSIGNED NOT NULL,
    weight_kg       DECIMAL(8,3),
    length_cm       DECIMAL(7,2),
    width_cm        DECIMAL(7,2),
    height_cm       DECIMAL(7,2),
    volume_cm3      DECIMAL(12,2) GENERATED ALWAYS AS (length_cm * width_cm * height_cm) STORED,
    -- Cost
    standard_cost   DECIMAL(12,4),
    last_purchase_cost DECIMAL(12,4),
    average_cost    DECIMAL(12,4),
    -- Config
    is_stockable    BOOLEAN DEFAULT TRUE,
    is_purchasable  BOOLEAN DEFAULT TRUE,
    is_sellable     BOOLEAN DEFAULT TRUE,
    costing_method  ENUM('fifo','lifo','average','standard') DEFAULT 'fifo',
    -- Reorder
    reorder_point   DECIMAL(10,2) DEFAULT 0,
    reorder_quantity DECIMAL(10,2) DEFAULT 0,
    min_stock_level DECIMAL(10,2) DEFAULT 0,
    max_stock_level DECIMAL(10,2) DEFAULT 0,
    lead_time_days  INT DEFAULT 7,
    is_active       BOOLEAN DEFAULT TRUE,
    FOREIGN KEY (category_id) REFERENCES categories(category_id) ON DELETE SET NULL,
    FOREIGN KEY (uom_id) REFERENCES units_of_measure(uom_id)
) ENGINE=InnoDB;

-- =========================================
-- SECTION 3: MULTI-WAREHOUSE INVENTORY
-- =========================================

-- ตาราง Stock On Hand (ยอดสต็อกปัจจุบัน)
CREATE TABLE stock_on_hand (
    soh_id          INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    product_id      INT UNSIGNED NOT NULL,
    warehouse_id    INT UNSIGNED NOT NULL,
    location_id     INT UNSIGNED,
    quantity        DECIMAL(12,4) NOT NULL DEFAULT 0,
    reserved_qty    DECIMAL(12,4) NOT NULL DEFAULT 0,
    available_qty   DECIMAL(12,4) GENERATED ALWAYS AS (quantity - reserved_qty) STORED,
    lot_number      VARCHAR(50),
    serial_number   VARCHAR(100),
    expiry_date     DATE,
    last_updated    TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    FOREIGN KEY (product_id) REFERENCES products(product_id),
    FOREIGN KEY (warehouse_id) REFERENCES warehouses(warehouse_id),
    FOREIGN KEY (location_id) REFERENCES stock_locations(location_id) ON DELETE SET NULL,
    INDEX idx_product_wh (product_id, warehouse_id),
    INDEX idx_expiry (expiry_date)
) ENGINE=InnoDB;

-- ตาราง Stock Moves (การเคลื่อนไหวสต็อก)
CREATE TABLE stock_moves (
    move_id         BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    move_type       ENUM('receipt','delivery','transfer','adjustment','return','scrap','production') NOT NULL,
    product_id      INT UNSIGNED NOT NULL,
    from_warehouse_id INT UNSIGNED,
    to_warehouse_id INT UNSIGNED,
    from_location_id INT UNSIGNED,
    to_location_id  INT UNSIGNED,
    quantity        DECIMAL(12,4) NOT NULL,
    uom_id          INT UNSIGNED NOT NULL,
    lot_number      VARCHAR(50),
    unit_cost       DECIMAL(12,4),
    total_cost      DECIMAL(16,4),
    reference_type  VARCHAR(50) COMMENT 'PO, SO, etc.',
    reference_id    INT UNSIGNED,
    reference_number VARCHAR(50),
    reason          TEXT,
    performed_by    INT UNSIGNED,
    performed_at    TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (product_id) REFERENCES products(product_id),
    FOREIGN KEY (from_warehouse_id) REFERENCES warehouses(warehouse_id) ON DELETE SET NULL,
    FOREIGN KEY (to_warehouse_id) REFERENCES warehouses(warehouse_id) ON DELETE SET NULL,
    INDEX idx_product (product_id),
    INDEX idx_type (move_type),
    INDEX idx_performed (performed_at),
    INDEX idx_reference (reference_type, reference_id)
) ENGINE=InnoDB;

-- =========================================
-- SECTION 4: SUPPLIERS
-- =========================================

-- ตาราง Suppliers (ผู้จัดจำหน่าย)
CREATE TABLE suppliers (
    supplier_id     INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    code            VARCHAR(20) NOT NULL UNIQUE,
    name            VARCHAR(300) NOT NULL,
    contact_name    VARCHAR(200),
    email           VARCHAR(255),
    phone           VARCHAR(30),
    address         TEXT,
    city            VARCHAR(100),
    country_code    CHAR(2) DEFAULT 'TH',
    payment_terms_days INT DEFAULT 30,
    currency        CHAR(3) DEFAULT 'THB',
    rating          DECIMAL(3,1),
    is_approved     BOOLEAN DEFAULT FALSE,
    is_active       BOOLEAN DEFAULT TRUE,
    created_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP
) ENGINE=InnoDB;

-- ตาราง Supplier Products (สินค้าของ Supplier แต่ละราย)
CREATE TABLE supplier_products (
    sp_id           INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    supplier_id     INT UNSIGNED NOT NULL,
    product_id      INT UNSIGNED NOT NULL,
    supplier_sku    VARCHAR(100),
    unit_price      DECIMAL(12,4) NOT NULL,
    min_order_qty   DECIMAL(10,2) DEFAULT 1,
    lead_time_days  INT DEFAULT 7,
    is_preferred    BOOLEAN DEFAULT FALSE,
    last_updated    TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    FOREIGN KEY (supplier_id) REFERENCES suppliers(supplier_id),
    FOREIGN KEY (product_id) REFERENCES products(product_id),
    UNIQUE KEY uk_supplier_product (supplier_id, product_id),
    INDEX idx_product (product_id)
) ENGINE=InnoDB;

-- =========================================
-- SECTION 5: PURCHASE ORDERS
-- =========================================

-- ตาราง Purchase Orders (ใบสั่งซื้อ)
CREATE TABLE purchase_orders (
    po_id           INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    po_number       VARCHAR(30) NOT NULL UNIQUE,
    supplier_id     INT UNSIGNED NOT NULL,
    warehouse_id    INT UNSIGNED NOT NULL,
    status          ENUM('draft','sent','confirmed','partial_received','received','cancelled','closed') DEFAULT 'draft',
    order_date      DATE NOT NULL,
    expected_date   DATE,
    actual_received_date DATE,
    subtotal        DECIMAL(15,2) DEFAULT 0,
    discount_amount DECIMAL(12,2) DEFAULT 0,
    tax_amount      DECIMAL(12,2) DEFAULT 0,
    shipping_cost   DECIMAL(10,2) DEFAULT 0,
    total_amount    DECIMAL(15,2) DEFAULT 0,
    currency        CHAR(3) DEFAULT 'THB',
    notes           TEXT,
    created_by      INT UNSIGNED,
    approved_by     INT UNSIGNED,
    created_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (supplier_id) REFERENCES suppliers(supplier_id),
    FOREIGN KEY (warehouse_id) REFERENCES warehouses(warehouse_id),
    INDEX idx_supplier (supplier_id),
    INDEX idx_status (status),
    INDEX idx_order_date (order_date)
) ENGINE=InnoDB;

-- ตาราง Purchase Order Lines
CREATE TABLE po_lines (
    line_id         INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    po_id           INT UNSIGNED NOT NULL,
    product_id      INT UNSIGNED NOT NULL,
    ordered_qty     DECIMAL(10,2) NOT NULL,
    received_qty    DECIMAL(10,2) DEFAULT 0,
    unit_price      DECIMAL(12,4) NOT NULL,
    discount_pct    DECIMAL(5,2) DEFAULT 0,
    line_total      DECIMAL(15,2) NOT NULL,
    expected_date   DATE,
    notes           TEXT,
    FOREIGN KEY (po_id) REFERENCES purchase_orders(po_id) ON DELETE CASCADE,
    FOREIGN KEY (product_id) REFERENCES products(product_id),
    INDEX idx_po (po_id),
    INDEX idx_product (product_id)
) ENGINE=InnoDB;

-- =========================================
-- SECTION 6: GOODS RECEIPT
-- =========================================

-- ตาราง Goods Receipt (ใบรับสินค้า)
CREATE TABLE goods_receipts (
    gr_id           INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    gr_number       VARCHAR(30) NOT NULL UNIQUE,
    po_id           INT UNSIGNED,
    supplier_id     INT UNSIGNED NOT NULL,
    warehouse_id    INT UNSIGNED NOT NULL,
    received_date   DATETIME NOT NULL,
    received_by     INT UNSIGNED,
    status          ENUM('draft','quality_check','accepted','rejected','partial_accepted') DEFAULT 'draft',
    notes           TEXT,
    FOREIGN KEY (po_id) REFERENCES purchase_orders(po_id) ON DELETE SET NULL,
    FOREIGN KEY (supplier_id) REFERENCES suppliers(supplier_id),
    FOREIGN KEY (warehouse_id) REFERENCES warehouses(warehouse_id)
) ENGINE=InnoDB;

-- ตาราง Goods Receipt Items
CREATE TABLE gr_items (
    item_id         INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    gr_id           INT UNSIGNED NOT NULL,
    po_line_id      INT UNSIGNED,
    product_id      INT UNSIGNED NOT NULL,
    received_qty    DECIMAL(10,2) NOT NULL,
    accepted_qty    DECIMAL(10,2) DEFAULT 0,
    rejected_qty    DECIMAL(10,2) DEFAULT 0,
    location_id     INT UNSIGNED,
    lot_number      VARCHAR(50),
    expiry_date     DATE,
    unit_cost       DECIMAL(12,4),
    qc_result       ENUM('pass','fail','conditional') DEFAULT 'pass',
    qc_notes        TEXT,
    FOREIGN KEY (gr_id) REFERENCES goods_receipts(gr_id) ON DELETE CASCADE,
    FOREIGN KEY (po_line_id) REFERENCES po_lines(line_id) ON DELETE SET NULL,
    FOREIGN KEY (product_id) REFERENCES products(product_id),
    FOREIGN KEY (location_id) REFERENCES stock_locations(location_id) ON DELETE SET NULL
) ENGINE=InnoDB;

-- =========================================
-- SECTION 7: FIFO COST LAYERS
-- =========================================

-- ตาราง FIFO Cost Layers (ชั้นต้นทุนแบบ FIFO)
CREATE TABLE fifo_cost_layers (
    layer_id        BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    product_id      INT UNSIGNED NOT NULL,
    warehouse_id    INT UNSIGNED NOT NULL,
    lot_number      VARCHAR(50),
    receipt_date    DATE NOT NULL,
    receipt_qty     DECIMAL(12,4) NOT NULL,
    remaining_qty   DECIMAL(12,4) NOT NULL,
    unit_cost       DECIMAL(12,4) NOT NULL,
    total_cost      DECIMAL(16,4) GENERATED ALWAYS AS (remaining_qty * unit_cost) STORED,
    gr_id           INT UNSIGNED,
    is_exhausted    BOOLEAN DEFAULT FALSE,
    FOREIGN KEY (product_id) REFERENCES products(product_id),
    FOREIGN KEY (warehouse_id) REFERENCES warehouses(warehouse_id),
    FOREIGN KEY (gr_id) REFERENCES goods_receipts(gr_id) ON DELETE SET NULL,
    INDEX idx_product_wh (product_id, warehouse_id, is_exhausted, receipt_date)
) ENGINE=InnoDB;
```

## 2. Sample Data

```sql
-- Warehouses
INSERT INTO warehouses (warehouse_id, code, name, type, city, total_capacity) VALUES
(1, 'WH-BKK-01', 'Bangkok Main Warehouse', 'main', 'Bangkok', 50000),
(2, 'WH-BKK-02', 'Bangkok 2', 'regional', 'Bangkok', 20000),
(3, 'WH-CNX', 'Chiang Mai Warehouse', 'regional', 'Chiang Mai', 15000),
(4, 'WH-KKN', 'Khon Kaen Warehouse', 'regional', 'Khon Kaen', 10000);

-- Units of Measure
INSERT INTO units_of_measure VALUES
(1, 'PCS', 'Pieces', 1),
(2, 'BOX', 'Box (12 pcs)', 12),
(3, 'KG', 'Kilogram', 1),
(4, 'L', 'Liter', 1),
(5, 'M', 'Meter', 1),
(6, 'PALLET', 'Pallet', 1);

-- Categories
INSERT INTO categories (category_id, parent_id, name, code) VALUES
(1, NULL, 'Electronics', 'ELEC'),
(2, NULL, 'Clothing', 'CLTH'),
(3, NULL, 'Food & Beverages', 'FOOD'),
(4, 1, 'Smartphones', 'PHONE'),
(5, 1, 'Accessories', 'ACC'),
(6, 3, 'Snacks', 'SNACK');

-- Products
INSERT INTO products (product_id, sku, name, category_id, uom_id, weight_kg, standard_cost, last_purchase_cost, average_cost, costing_method, reorder_point, reorder_quantity, min_stock_level, max_stock_level, lead_time_days) VALUES
(1, 'PHONE-001', 'Smartphone Model A', 4, 1, 0.2, 15000, 15200, 15100, 'fifo', 50, 200, 20, 500, 14),
(2, 'ACC-001', 'Phone Case Universal', 5, 1, 0.05, 80, 85, 82, 'fifo', 100, 500, 50, 2000, 7),
(3, 'SNACK-001', 'Potato Chips 30g', 6, 1, 0.03, 10, 11, 10.5, 'fifo', 200, 1000, 100, 5000, 3),
(4, 'ELEC-001', 'USB Cable Type-C 1m', 5, 1, 0.08, 50, 52, 51, 'fifo', 150, 600, 50, 3000, 5),
(5, 'CLTH-001', 'T-Shirt White M', 2, 1, 0.2, 120, 125, 122, 'fifo', 50, 200, 20, 1000, 7);

-- Suppliers
INSERT INTO suppliers (supplier_id, code, name, contact_name, email, phone, payment_terms_days, rating, is_approved) VALUES
(1, 'SUP-001', 'Tech Distributors Co., Ltd.', 'คุณสมศักดิ์', 'somsak@techdist.com', '021234567', 30, 4.5, TRUE),
(2, 'SUP-002', 'Fashion Wholesale Ltd.', 'คุณมาลี', 'malee@fashionwh.com', '022345678', 45, 4.2, TRUE),
(3, 'SUP-003', 'Food Import Corp.', 'คุณวิชัย', 'vichai@foodimp.com', '023456789', 15, 4.0, TRUE),
(4, 'SUP-004', 'Accessories Hub', 'คุณนิสา', 'nisa@acchub.com', '024567890', 30, 4.3, TRUE);

-- Supplier Products
INSERT INTO supplier_products (supplier_id, product_id, supplier_sku, unit_price, min_order_qty, lead_time_days, is_preferred) VALUES
(1, 1, 'TD-SMR-001', 15200, 10, 14, TRUE),
(4, 2, 'AH-CASE-001', 85, 100, 7, TRUE),
(3, 3, 'FI-CHIPS-001', 11, 500, 3, TRUE),
(4, 4, 'AH-USB-001', 52, 200, 5, TRUE),
(2, 5, 'FW-TS-WHM', 125, 50, 7, TRUE),
-- Alternative suppliers
(1, 4, 'TD-USB-001', 55, 100, 7, FALSE);

-- Stock On Hand (initial)
INSERT INTO stock_on_hand (product_id, warehouse_id, quantity, reserved_qty) VALUES
(1, 1, 150, 10),
(1, 2, 80, 5),
(2, 1, 500, 20),
(3, 1, 1200, 50),
(3, 3, 300, 0),
(4, 1, 400, 30),
(5, 1, 200, 15);

-- FIFO Cost Layers
INSERT INTO fifo_cost_layers (product_id, warehouse_id, receipt_date, receipt_qty, remaining_qty, unit_cost) VALUES
-- Phone: 3 batches at different costs
(1, 1, '2024-01-05', 100, 80, 14800),
(1, 1, '2024-02-01', 100, 70, 15200),
(1, 2, '2024-01-10', 100, 80, 14900),
-- USB Cables
(4, 1, '2024-01-15', 200, 150, 50),
(4, 1, '2024-02-20', 250, 250, 52);

-- Purchase Orders
INSERT INTO purchase_orders (po_id, po_number, supplier_id, warehouse_id, status, order_date, expected_date, subtotal, total_amount) VALUES
(1, 'PO-2024-000001', 1, 1, 'received', '2024-01-01', '2024-01-15', 152000, 162640),
(2, 'PO-2024-000002', 4, 1, 'received', '2024-01-10', '2024-01-17', 42500, 45475),
(3, 'PO-2024-000003', 3, 1, 'confirmed', '2024-02-20', '2024-02-23', 55000, 58850),
(4, 'PO-2024-000004', 1, 1, 'sent', '2024-03-01', '2024-03-15', 304000, 325280);

-- PO Lines
INSERT INTO po_lines (po_id, product_id, ordered_qty, received_qty, unit_price, line_total) VALUES
(1, 1, 10, 10, 15200, 152000),
(2, 4, 500, 500, 52, 26000),
(2, 2, 200, 200, 85, 17000),
(3, 3, 5000, 0, 11, 55000),
(4, 1, 20, 0, 15200, 304000);
```

## 3. Supply Chain Queries

### Query 1: Multi-Warehouse Stock Summary

```sql
SELECT 
    p.sku,
    p.name AS product_name,
    c.name AS category,
    -- Per warehouse
    SUM(CASE WHEN soh.warehouse_id = 1 THEN soh.quantity ELSE 0 END) AS wh_bangkok_main,
    SUM(CASE WHEN soh.warehouse_id = 2 THEN soh.quantity ELSE 0 END) AS wh_bangkok_2,
    SUM(CASE WHEN soh.warehouse_id = 3 THEN soh.quantity ELSE 0 END) AS wh_chiang_mai,
    SUM(CASE WHEN soh.warehouse_id = 4 THEN soh.quantity ELSE 0 END) AS wh_khon_kaen,
    -- Totals
    SUM(soh.quantity) AS total_quantity,
    SUM(soh.reserved_qty) AS total_reserved,
    SUM(soh.available_qty) AS total_available,
    -- Inventory Value
    ROUND(SUM(soh.quantity) * p.average_cost, 2) AS inventory_value,
    -- Stock Health
    p.reorder_point,
    p.min_stock_level,
    CASE 
        WHEN SUM(soh.quantity) = 0 THEN 'OUT OF STOCK'
        WHEN SUM(soh.quantity) < p.min_stock_level THEN 'CRITICAL'
        WHEN SUM(soh.quantity) < p.reorder_point THEN 'REORDER NEEDED'
        WHEN SUM(soh.quantity) > p.max_stock_level THEN 'OVERSTOCK'
        ELSE 'OK'
    END AS stock_status
FROM products p
LEFT JOIN stock_on_hand soh ON p.product_id = soh.product_id
LEFT JOIN categories c ON p.category_id = c.category_id
WHERE p.is_active = TRUE AND p.is_stockable = TRUE
GROUP BY p.product_id, p.sku, p.name, c.name, p.reorder_point, 
         p.min_stock_level, p.max_stock_level, p.average_cost
ORDER BY inventory_value DESC;
```

---

### Query 2: FIFO Cost of Goods Sold Calculation

```sql
-- คำนวณ COGS แบบ FIFO
-- สมมุติว่าขาย Product 1 จำนวน 120 units
WITH RECURSIVE fifo_consumption AS (
    -- เรียงตาม receipt_date (เก่าสุดก่อน = FIFO)
    SELECT 
        layer_id,
        product_id,
        warehouse_id,
        receipt_date,
        remaining_qty,
        unit_cost,
        0.0 AS consumed_so_far,
        120.0 AS qty_to_consume  -- จำนวนที่ต้องการขาย
    FROM fifo_cost_layers
    WHERE product_id = 1
      AND warehouse_id = 1
      AND is_exhausted = FALSE
    ORDER BY receipt_date ASC
    LIMIT 1  -- Start with oldest layer
),
fifo_layers_ordered AS (
    SELECT 
        layer_id,
        product_id,
        warehouse_id,
        receipt_date,
        remaining_qty,
        unit_cost,
        ROW_NUMBER() OVER (ORDER BY receipt_date ASC, layer_id ASC) AS rn
    FROM fifo_cost_layers
    WHERE product_id = 1
      AND warehouse_id = 1
      AND is_exhausted = FALSE
)
-- Manual FIFO calculation using set-based approach
SELECT 
    f.receipt_date,
    f.unit_cost,
    f.remaining_qty AS layer_qty,
    LEAST(
        f.remaining_qty,
        GREATEST(
            0,
            120 - COALESCE(
                (SELECT SUM(f2.remaining_qty) 
                 FROM fifo_layers_ordered f2 
                 WHERE f2.rn < f.rn),
                0
            )
        )
    ) AS consumed_from_layer,
    f.unit_cost * LEAST(
        f.remaining_qty,
        GREATEST(
            0,
            120 - COALESCE(
                (SELECT SUM(f2.remaining_qty) 
                 FROM fifo_layers_ordered f2 
                 WHERE f2.rn < f.rn),
                0
            )
        )
    ) AS layer_cost
FROM fifo_layers_ordered f
WHERE GREATEST(
    0,
    120 - COALESCE(
        (SELECT SUM(f2.remaining_qty) 
         FROM fifo_layers_ordered f2 
         WHERE f2.rn < f.rn),
        0
    )
) > 0;
```

---

### Query 3: Inventory Valuation (Average Cost)

```sql
SELECT 
    p.sku,
    p.name,
    p.costing_method,
    -- Current stock
    SUM(soh.quantity) AS on_hand_qty,
    -- Cost methods comparison
    ROUND(SUM(soh.quantity) * p.standard_cost, 2) AS standard_cost_valuation,
    ROUND(SUM(soh.quantity) * p.average_cost, 2) AS average_cost_valuation,
    -- FIFO Valuation (sum of remaining cost layers)
    COALESCE((
        SELECT ROUND(SUM(fcl.remaining_qty * fcl.unit_cost), 2)
        FROM fifo_cost_layers fcl
        WHERE fcl.product_id = p.product_id
          AND NOT fcl.is_exhausted
    ), 0) AS fifo_valuation,
    -- Difference between methods
    ROUND(SUM(soh.quantity) * p.average_cost - 
        COALESCE((
            SELECT SUM(fcl.remaining_qty * fcl.unit_cost)
            FROM fifo_cost_layers fcl
            WHERE fcl.product_id = p.product_id
              AND NOT fcl.is_exhausted
        ), 0), 
    2) AS avg_vs_fifo_diff
FROM products p
JOIN stock_on_hand soh ON p.product_id = soh.product_id
WHERE p.is_stockable = TRUE
GROUP BY p.product_id, p.sku, p.name, p.costing_method, 
         p.standard_cost, p.average_cost;
```

---

### Query 4: Reorder Point Analysis

```sql
-- คำนวณ Reorder Point แบบ Dynamic
WITH sales_velocity AS (
    SELECT 
        product_id,
        -- Average daily usage from stock moves (outbound)
        SUM(quantity) / NULLIF(DATEDIFF(MAX(performed_at), MIN(performed_at)), 0) AS daily_usage_rate,
        STDDEV(quantity) AS usage_stddev
    FROM stock_moves
    WHERE move_type = 'delivery'
      AND performed_at >= DATE_SUB(CURDATE(), INTERVAL 90 DAY)
    GROUP BY product_id
)
SELECT 
    p.sku,
    p.name,
    p.lead_time_days,
    -- Calculated optimal reorder point
    ROUND(sv.daily_usage_rate * p.lead_time_days, 0) AS lead_time_demand,
    -- Safety stock = Z * STDDEV * SQRT(lead_time)
    -- Z=1.65 for 95% service level
    ROUND(1.65 * sv.usage_stddev * SQRT(p.lead_time_days), 0) AS safety_stock_95pct,
    -- Optimal Reorder Point = Lead Time Demand + Safety Stock
    ROUND(
        sv.daily_usage_rate * p.lead_time_days + 
        1.65 * sv.usage_stddev * SQRT(p.lead_time_days),
        0
    ) AS optimal_reorder_point,
    p.reorder_point AS current_reorder_point,
    -- EOQ (Economic Order Quantity)
    -- EOQ = SQRT(2 * Annual Demand * Ordering Cost / Holding Cost)
    ROUND(SQRT(
        2 * sv.daily_usage_rate * 365 * 500 /  -- Ordering cost = 500 THB
        (p.standard_cost * 0.25)  -- Holding cost = 25% of unit cost
    ), 0) AS eoq,
    -- Current vs Optimal comparison
    p.reorder_point - ROUND(
        sv.daily_usage_rate * p.lead_time_days + 
        1.65 * sv.usage_stddev * SQRT(p.lead_time_days),
        0
    ) AS reorder_variance
FROM products p
LEFT JOIN sales_velocity sv ON p.product_id = sv.product_id
WHERE p.is_active = TRUE
ORDER BY ABS(p.reorder_point - ROUND(
    COALESCE(sv.daily_usage_rate, 0) * p.lead_time_days + 
    1.65 * COALESCE(sv.usage_stddev, 0) * SQRT(p.lead_time_days),
    0
)) DESC;
```

---

### Query 5: Dead Stock Analysis

```sql
SELECT 
    p.sku,
    p.name,
    w.name AS warehouse,
    soh.quantity AS on_hand,
    soh.reserved_qty,
    p.average_cost,
    ROUND(soh.quantity * p.average_cost, 2) AS inventory_value,
    -- Last movement
    last_move.last_move_date,
    DATEDIFF(CURDATE(), last_move.last_move_date) AS days_no_movement,
    -- Classify
    CASE 
        WHEN last_move.last_move_date IS NULL THEN 'NEVER MOVED'
        WHEN DATEDIFF(CURDATE(), last_move.last_move_date) > 365 THEN 'DEAD (1yr+)'
        WHEN DATEDIFF(CURDATE(), last_move.last_move_date) > 180 THEN 'DEAD (6mo+)'
        WHEN DATEDIFF(CURDATE(), last_move.last_move_date) > 90 THEN 'SLOW MOVING (90d+)'
        ELSE 'ACTIVE'
    END AS movement_status,
    -- Recommended Action
    CASE 
        WHEN DATEDIFF(CURDATE(), last_move.last_move_date) > 365 THEN 'Write-off or Liquidate'
        WHEN DATEDIFF(CURDATE(), last_move.last_move_date) > 180 THEN 'Deep Discount or Transfer'
        WHEN DATEDIFF(CURDATE(), last_move.last_move_date) > 90 THEN 'Promotion or Markdown'
        ELSE 'Monitor'
    END AS recommended_action
FROM stock_on_hand soh
JOIN products p ON soh.product_id = p.product_id
JOIN warehouses w ON soh.warehouse_id = w.warehouse_id
LEFT JOIN (
    SELECT 
        product_id,
        from_warehouse_id AS warehouse_id,
        MAX(performed_at) AS last_move_date
    FROM stock_moves
    WHERE move_type IN ('delivery', 'transfer')
    GROUP BY product_id, from_warehouse_id
) last_move ON soh.product_id = last_move.product_id 
            AND soh.warehouse_id = last_move.warehouse_id
WHERE soh.quantity > 0
ORDER BY days_no_movement DESC NULLS FIRST;
```

---

### Query 6: Purchase Order Status and Tracking

```sql
SELECT 
    po.po_number,
    s.name AS supplier,
    w.name AS delivery_warehouse,
    po.order_date,
    po.expected_date,
    DATEDIFF(CURDATE(), po.expected_date) AS days_overdue,
    po.status,
    COUNT(pol.line_id) AS total_lines,
    SUM(pol.ordered_qty) AS total_ordered,
    SUM(pol.received_qty) AS total_received,
    ROUND(SUM(pol.received_qty) * 100.0 / NULLIF(SUM(pol.ordered_qty), 0), 1) AS receipt_pct,
    po.total_amount,
    -- Items pending
    GROUP_CONCAT(
        CASE WHEN pol.received_qty < pol.ordered_qty 
             THEN CONCAT(p.name, ' (', pol.ordered_qty - pol.received_qty, ' pending)')
        END
        SEPARATOR '; '
    ) AS pending_items
FROM purchase_orders po
JOIN suppliers s ON po.supplier_id = s.supplier_id
JOIN warehouses w ON po.warehouse_id = w.warehouse_id
JOIN po_lines pol ON po.po_id = pol.po_id
JOIN products p ON pol.product_id = p.product_id
WHERE po.status NOT IN ('cancelled', 'closed')
GROUP BY po.po_id, po.po_number, s.name, w.name, po.order_date, 
         po.expected_date, po.status, po.total_amount
ORDER BY po.expected_date ASC;
```

---

### Query 7: Supplier Performance Scorecard

```sql
SELECT 
    s.supplier_id,
    s.name AS supplier_name,
    s.rating AS recorded_rating,
    COUNT(DISTINCT po.po_id) AS total_pos,
    SUM(po.total_amount) AS total_purchase_value,
    -- Delivery performance
    SUM(CASE WHEN po.actual_received_date <= po.expected_date THEN 1 ELSE 0 END) AS on_time_count,
    ROUND(
        SUM(CASE WHEN po.actual_received_date <= po.expected_date THEN 1 ELSE 0 END) 
        * 100.0 / COUNT(DISTINCT CASE WHEN po.actual_received_date IS NOT NULL THEN po.po_id END),
        1
    ) AS on_time_delivery_pct,
    AVG(DATEDIFF(po.actual_received_date, po.order_date)) AS avg_actual_lead_days,
    -- Quality (from GR items)
    ROUND(
        SUM(gri.accepted_qty) * 100.0 / NULLIF(SUM(gri.received_qty), 0),
        1
    ) AS acceptance_rate_pct,
    SUM(gri.rejected_qty) AS total_rejected_qty,
    -- Price competitiveness (vs average)
    ROUND(AVG(sp.unit_price), 2) AS avg_unit_price,
    -- Calculated score
    ROUND(
        (COALESCE(
            SUM(CASE WHEN po.actual_received_date <= po.expected_date THEN 1 ELSE 0 END) 
            * 100.0 / NULLIF(COUNT(DISTINCT CASE WHEN po.actual_received_date IS NOT NULL THEN po.po_id END), 0),
            0
        ) * 0.4) +
        (COALESCE(
            SUM(gri.accepted_qty) * 100.0 / NULLIF(SUM(gri.received_qty), 0),
            0
        ) * 0.4) +
        (s.rating * 20),  -- 20% weight on rating
        1
    ) AS performance_score
FROM suppliers s
LEFT JOIN purchase_orders po ON s.supplier_id = po.supplier_id
    AND po.status != 'cancelled'
LEFT JOIN goods_receipts gr ON po.po_id = gr.po_id
LEFT JOIN gr_items gri ON gr.gr_id = gri.gr_id
LEFT JOIN supplier_products sp ON s.supplier_id = sp.supplier_id
WHERE s.is_active = TRUE
GROUP BY s.supplier_id, s.name, s.rating
ORDER BY performance_score DESC;
```

---

### Query 8: ABC Analysis (Inventory Classification)

```sql
-- ABC Analysis: A=Top 20% items = 80% of value, B=Next 30%, C=Bottom 50%
WITH inventory_value AS (
    SELECT 
        p.product_id,
        p.sku,
        p.name,
        SUM(soh.quantity) AS on_hand_qty,
        p.average_cost,
        SUM(soh.quantity) * p.average_cost AS total_value,
        -- Also consider annual movement value
        COALESCE(annual_sales.annual_value, 0) AS annual_sales_value
    FROM products p
    LEFT JOIN stock_on_hand soh ON p.product_id = soh.product_id
    LEFT JOIN (
        SELECT 
            product_id,
            SUM(quantity * unit_cost) AS annual_value
        FROM stock_moves
        WHERE move_type = 'delivery'
          AND performed_at >= DATE_SUB(CURDATE(), INTERVAL 365 DAY)
        GROUP BY product_id
    ) annual_sales ON p.product_id = annual_sales.product_id
    WHERE p.is_active = TRUE
    GROUP BY p.product_id, p.sku, p.name, p.average_cost
),
ranked AS (
    SELECT 
        *,
        SUM(total_value) OVER () AS grand_total,
        SUM(total_value) OVER (ORDER BY total_value DESC ROWS UNBOUNDED PRECEDING) AS cumulative_value,
        ROUND(RANK() OVER (ORDER BY total_value DESC) * 100.0 / COUNT(*) OVER (), 1) AS item_percentile
    FROM inventory_value
)
SELECT 
    product_id,
    sku,
    name,
    on_hand_qty,
    average_cost,
    ROUND(total_value, 2) AS inventory_value,
    ROUND(total_value * 100.0 / grand_total, 2) AS value_pct,
    ROUND(cumulative_value * 100.0 / grand_total, 2) AS cumulative_pct,
    item_percentile,
    CASE 
        WHEN cumulative_value * 100.0 / grand_total <= 80 THEN 'A'
        WHEN cumulative_value * 100.0 / grand_total <= 95 THEN 'B'
        ELSE 'C'
    END AS abc_class,
    -- Management strategy
    CASE 
        WHEN cumulative_value * 100.0 / grand_total <= 80 THEN 'Tight control, frequent counts, precise reorder'
        WHEN cumulative_value * 100.0 / grand_total <= 95 THEN 'Moderate control, monthly counts'
        ELSE 'Simple control, periodic counts'
    END AS management_strategy
FROM ranked
ORDER BY total_value DESC;
```

---

### Query 9: Stock Aging Analysis

```sql
-- วิเคราะห์อายุสต็อก (สำหรับ FIFO / Products with expiry)
SELECT 
    p.sku,
    p.name,
    w.name AS warehouse,
    fcl.lot_number,
    fcl.receipt_date,
    DATEDIFF(CURDATE(), fcl.receipt_date) AS age_days,
    fcl.remaining_qty,
    fcl.unit_cost,
    ROUND(fcl.remaining_qty * fcl.unit_cost, 2) AS layer_value,
    -- Age buckets
    CASE 
        WHEN DATEDIFF(CURDATE(), fcl.receipt_date) <= 30 THEN '0-30 days'
        WHEN DATEDIFF(CURDATE(), fcl.receipt_date) <= 60 THEN '31-60 days'
        WHEN DATEDIFF(CURDATE(), fcl.receipt_date) <= 90 THEN '61-90 days'
        WHEN DATEDIFF(CURDATE(), fcl.receipt_date) <= 180 THEN '91-180 days'
        ELSE '180+ days (CONCERN)'
    END AS age_bucket
FROM fifo_cost_layers fcl
JOIN products p ON fcl.product_id = p.product_id
JOIN warehouses w ON fcl.warehouse_id = w.warehouse_id
WHERE fcl.is_exhausted = FALSE
  AND fcl.remaining_qty > 0
ORDER BY fcl.receipt_date ASC;
```

---

### Query 10: Transfer Order Recommendation

```sql
-- แนะนำการโอนสินค้าระหว่างคลัง
WITH wh_stock AS (
    SELECT 
        product_id,
        warehouse_id,
        SUM(quantity) AS qty,
        SUM(available_qty) AS avail_qty
    FROM stock_on_hand
    GROUP BY product_id, warehouse_id
),
-- ดูว่าคลังไหน Overstock และ Understock
wh_analysis AS (
    SELECT 
        ws.product_id,
        ws.warehouse_id,
        w.name AS warehouse_name,
        ws.qty,
        ws.avail_qty,
        p.reorder_point,
        p.max_stock_level,
        p.reorder_quantity,
        ws.qty - p.reorder_point AS surplus_above_reorder,
        p.max_stock_level - ws.qty AS space_to_max,
        CASE 
            WHEN ws.qty < p.reorder_point THEN 'UNDERSTOCKED'
            WHEN ws.qty > p.max_stock_level THEN 'OVERSTOCKED'
            ELSE 'OK'
        END AS stock_status
    FROM wh_stock ws
    JOIN products p ON ws.product_id = p.product_id
    JOIN warehouses w ON ws.warehouse_id = w.warehouse_id
)
SELECT 
    over_stk.product_id,
    p.name AS product_name,
    over_stk.warehouse_name AS source_warehouse,
    under_stk.warehouse_name AS destination_warehouse,
    over_stk.qty AS source_qty,
    under_stk.qty AS dest_qty,
    under_stk.reorder_point AS dest_reorder_point,
    -- Recommended transfer quantity
    LEAST(
        over_stk.surplus_above_reorder,
        under_stk.reorder_quantity
    ) AS recommended_transfer_qty,
    'Transfer Recommended' AS action
FROM wh_analysis over_stk
JOIN wh_analysis under_stk ON over_stk.product_id = under_stk.product_id
    AND over_stk.warehouse_id != under_stk.warehouse_id
JOIN products p ON over_stk.product_id = p.product_id
WHERE over_stk.stock_status = 'OVERSTOCKED'
  AND under_stk.stock_status = 'UNDERSTOCKED'
ORDER BY recommended_transfer_qty DESC;
```

---

## แบบฝึกหัด (Challenge Exercises)

1. **Cycle Count Planning**: เขียน Query สร้าง Cycle Count Schedule โดยจัดลำดับ Priority จาก ABC Class และ Days Since Last Count

2. **LIFO Calculation**: เขียน Query คำนวณ COGS แบบ LIFO (Last In First Out) สำหรับ 100 units ของ Product 4

3. **Demand Forecasting**: เขียน Query ทำนายยอดขาย 30 วันข้างหน้าโดยใช้ Simple Moving Average ของ 3 เดือนที่ผ่านมา

4. **Cross-Docking Analysis**: เขียน Query หา Inbound PO ที่ Match กับ Pending Orders เพื่อแนะนำ Cross-Docking โดยไม่ต้อง Putaway

5. **Landed Cost Calculation**: เขียน Query คำนวณต้นทุนรวมของสินค้า (Landed Cost) โดยรวม Unit Price + Shipping + Import Duty + Handling Fee

6. **Pick Wave Optimization**: สร้าง Query จัดกลุ่ม Orders สำหรับ Pick Wave โดย Cluster Orders ที่มีสินค้าในบริเวณเดียวกัน

7. **Supplier Consolidation**: วิเคราะห์ว่า Supplier ไหนที่สามารถ Consolidate ได้ โดยดูจาก Product Overlap และ Volume

8. **Stock Reconciliation**: เขียน Query เปรียบเทียบ Book Quantity (จาก Stock Moves) กับ Physical Count และ Flag ความแตกต่าง

9. **Purchase Price Variance**: คำนวณ PPV (Purchase Price Variance = Actual Cost - Standard Cost) สำหรับ Goods Received ในเดือนนี้

10. **Inventory Optimization Model**: สร้าง Query คำนวณ Total Inventory Cost = Ordering Cost + Holding Cost + Stockout Cost สำหรับ Different Order Quantities
