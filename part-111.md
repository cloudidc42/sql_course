# Part 111: Complete E-commerce Database - Design to Implementation

## บทนำ (Introduction)

ในบทนี้เราจะสร้างระบบฐานข้อมูล E-commerce ที่สมบูรณ์แบบตั้งแต่การออกแบบ Schema ไปจนถึงการ Implement จริง ครอบคลุมทุกส่วนของระบบ ตั้งแต่สินค้า หมวดหมู่ ตะกร้าสินค้า การสั่งซื้อ ไปจนถึงระบบรีวิวและคูปอง

## 1. การออกแบบ Schema (Schema Design)

### 1.1 ภาพรวมของระบบ

ระบบ E-commerce ของเราประกอบด้วยส่วนหลักดังนี้:
- **ลูกค้า (Customers)**: ข้อมูลลูกค้าและที่อยู่
- **สินค้า (Products)**: สินค้าพร้อม Variants (ขนาด/สี)
- **หมวดหมู่ (Categories)**: หมวดหมู่แบบ Nested (Tree Structure)
- **คลังสินค้า (Inventory)**: การจัดการสต็อก
- **ตะกร้าสินค้า (Cart)**: Shopping Cart
- **คำสั่งซื้อ (Orders)**: Order Management
- **การคืนสินค้า (Returns)**: Returns and Refunds
- **รีวิว (Reviews)**: Product Reviews and Ratings
- **Wishlist**: Customer Wishlists
- **ส่วนลด (Discounts)**: Discount and Coupon System
- **ภาษี (Tax)**: Tax Calculations

### 1.2 DDL - สร้างตารางทั้งหมด

```sql
-- =========================================
-- E-COMMERCE DATABASE - COMPLETE DDL
-- =========================================

-- สร้าง Database
CREATE DATABASE ecommerce_db
    CHARACTER SET utf8mb4
    COLLATE utf8mb4_unicode_ci;

USE ecommerce_db;

-- =========================================
-- SECTION 1: CUSTOMER MANAGEMENT
-- =========================================

-- ตาราง Customers (ลูกค้า)
CREATE TABLE customers (
    customer_id     INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    email           VARCHAR(255) NOT NULL UNIQUE,
    password_hash   VARCHAR(255) NOT NULL,
    first_name      VARCHAR(100) NOT NULL,
    last_name       VARCHAR(100) NOT NULL,
    phone           VARCHAR(20),
    date_of_birth   DATE,
    gender          ENUM('male','female','other','prefer_not_to_say'),
    avatar_url      VARCHAR(500),
    is_verified     BOOLEAN DEFAULT FALSE,
    is_active       BOOLEAN DEFAULT TRUE,
    loyalty_points  INT DEFAULT 0,
    created_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    last_login_at   TIMESTAMP NULL,
    INDEX idx_email (email),
    INDEX idx_is_active (is_active)
) ENGINE=InnoDB;

-- ตาราง Customer Addresses (ที่อยู่ลูกค้า)
CREATE TABLE customer_addresses (
    address_id      INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    customer_id     INT UNSIGNED NOT NULL,
    address_label   VARCHAR(50) DEFAULT 'Home',
    recipient_name  VARCHAR(200) NOT NULL,
    phone           VARCHAR(20),
    address_line1   VARCHAR(255) NOT NULL,
    address_line2   VARCHAR(255),
    district        VARCHAR(100),
    city            VARCHAR(100) NOT NULL,
    state_province  VARCHAR(100),
    postal_code     VARCHAR(20) NOT NULL,
    country_code    CHAR(2) NOT NULL DEFAULT 'TH',
    is_default      BOOLEAN DEFAULT FALSE,
    created_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (customer_id) REFERENCES customers(customer_id) ON DELETE CASCADE,
    INDEX idx_customer (customer_id)
) ENGINE=InnoDB;

-- =========================================
-- SECTION 2: PRODUCT CATALOG
-- =========================================

-- ตาราง Categories (หมวดหมู่แบบ Nested)
CREATE TABLE categories (
    category_id     INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    parent_id       INT UNSIGNED NULL,
    name            VARCHAR(200) NOT NULL,
    slug            VARCHAR(200) NOT NULL UNIQUE,
    description     TEXT,
    image_url       VARCHAR(500),
    sort_order      INT DEFAULT 0,
    is_active       BOOLEAN DEFAULT TRUE,
    meta_title      VARCHAR(255),
    meta_description VARCHAR(500),
    created_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    FOREIGN KEY (parent_id) REFERENCES categories(category_id) ON DELETE SET NULL,
    INDEX idx_parent (parent_id),
    INDEX idx_slug (slug),
    INDEX idx_active (is_active)
) ENGINE=InnoDB;

-- ตาราง Brands (แบรนด์)
CREATE TABLE brands (
    brand_id        INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    name            VARCHAR(200) NOT NULL,
    slug            VARCHAR(200) NOT NULL UNIQUE,
    description     TEXT,
    logo_url        VARCHAR(500),
    website_url     VARCHAR(500),
    is_active       BOOLEAN DEFAULT TRUE,
    created_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP
) ENGINE=InnoDB;

-- ตาราง Products (สินค้า)
CREATE TABLE products (
    product_id      INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    category_id     INT UNSIGNED NOT NULL,
    brand_id        INT UNSIGNED,
    sku             VARCHAR(100) NOT NULL UNIQUE,
    name            VARCHAR(500) NOT NULL,
    slug            VARCHAR(500) NOT NULL UNIQUE,
    short_description TEXT,
    description     LONGTEXT,
    base_price      DECIMAL(12,2) NOT NULL,
    sale_price      DECIMAL(12,2),
    cost_price      DECIMAL(12,2),
    weight_grams    INT,
    status          ENUM('draft','active','inactive','discontinued') DEFAULT 'draft',
    is_featured     BOOLEAN DEFAULT FALSE,
    meta_title      VARCHAR(255),
    meta_description VARCHAR(500),
    view_count      INT DEFAULT 0,
    created_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    FOREIGN KEY (category_id) REFERENCES categories(category_id),
    FOREIGN KEY (brand_id) REFERENCES brands(brand_id) ON DELETE SET NULL,
    INDEX idx_category (category_id),
    INDEX idx_brand (brand_id),
    INDEX idx_status (status),
    INDEX idx_featured (is_featured),
    FULLTEXT idx_search (name, short_description)
) ENGINE=InnoDB;

-- ตาราง Product Images (รูปภาพสินค้า)
CREATE TABLE product_images (
    image_id        INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    product_id      INT UNSIGNED NOT NULL,
    image_url       VARCHAR(500) NOT NULL,
    alt_text        VARCHAR(255),
    sort_order      INT DEFAULT 0,
    is_primary      BOOLEAN DEFAULT FALSE,
    created_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (product_id) REFERENCES products(product_id) ON DELETE CASCADE,
    INDEX idx_product (product_id)
) ENGINE=InnoDB;

-- ตาราง Attribute Types (ประเภท Attribute เช่น Size, Color)
CREATE TABLE attribute_types (
    attribute_type_id INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    name            VARCHAR(100) NOT NULL,
    display_name    VARCHAR(100) NOT NULL,
    input_type      ENUM('select','color','button','radio') DEFAULT 'select'
) ENGINE=InnoDB;

-- ตาราง Attribute Values (ค่า Attribute เช่น S, M, L, Red, Blue)
CREATE TABLE attribute_values (
    attribute_value_id INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    attribute_type_id INT UNSIGNED NOT NULL,
    value           VARCHAR(100) NOT NULL,
    display_name    VARCHAR(100) NOT NULL,
    color_hex       CHAR(7),
    sort_order      INT DEFAULT 0,
    FOREIGN KEY (attribute_type_id) REFERENCES attribute_types(attribute_type_id) ON DELETE CASCADE,
    INDEX idx_type (attribute_type_id)
) ENGINE=InnoDB;

-- ตาราง Product Variants (Variants ของสินค้า)
CREATE TABLE product_variants (
    variant_id      INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    product_id      INT UNSIGNED NOT NULL,
    sku             VARCHAR(100) NOT NULL UNIQUE,
    name            VARCHAR(255),
    price_adjustment DECIMAL(10,2) DEFAULT 0.00,
    weight_grams    INT,
    is_active       BOOLEAN DEFAULT TRUE,
    created_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (product_id) REFERENCES products(product_id) ON DELETE CASCADE,
    INDEX idx_product (product_id)
) ENGINE=InnoDB;

-- ตาราง Variant Attribute Values (ความสัมพันธ์ Variant กับ Attribute)
CREATE TABLE variant_attribute_values (
    variant_id          INT UNSIGNED NOT NULL,
    attribute_value_id  INT UNSIGNED NOT NULL,
    PRIMARY KEY (variant_id, attribute_value_id),
    FOREIGN KEY (variant_id) REFERENCES product_variants(variant_id) ON DELETE CASCADE,
    FOREIGN KEY (attribute_value_id) REFERENCES attribute_values(attribute_value_id) ON DELETE CASCADE
) ENGINE=InnoDB;

-- =========================================
-- SECTION 3: INVENTORY MANAGEMENT
-- =========================================

-- ตาราง Inventory (สต็อกสินค้า)
CREATE TABLE inventory (
    inventory_id    INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    variant_id      INT UNSIGNED NOT NULL UNIQUE,
    quantity        INT NOT NULL DEFAULT 0,
    reserved_qty    INT NOT NULL DEFAULT 0,
    reorder_point   INT DEFAULT 10,
    reorder_qty     INT DEFAULT 50,
    updated_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    FOREIGN KEY (variant_id) REFERENCES product_variants(variant_id) ON DELETE CASCADE
) ENGINE=InnoDB;

-- ตาราง Inventory Transactions (ประวัติการเคลื่อนไหวสต็อก)
CREATE TABLE inventory_transactions (
    transaction_id  INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    variant_id      INT UNSIGNED NOT NULL,
    transaction_type ENUM('purchase','sale','return','adjustment','reserved','released') NOT NULL,
    quantity_change INT NOT NULL,
    quantity_before INT NOT NULL,
    quantity_after  INT NOT NULL,
    reference_type  VARCHAR(50),
    reference_id    INT UNSIGNED,
    notes           TEXT,
    created_by      INT UNSIGNED,
    created_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (variant_id) REFERENCES product_variants(variant_id),
    INDEX idx_variant (variant_id),
    INDEX idx_reference (reference_type, reference_id),
    INDEX idx_created_at (created_at)
) ENGINE=InnoDB;

-- =========================================
-- SECTION 4: SHOPPING CART
-- =========================================

-- ตาราง Cart (ตะกร้าสินค้า)
CREATE TABLE carts (
    cart_id         INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    customer_id     INT UNSIGNED,
    session_id      VARCHAR(255),
    expires_at      TIMESTAMP,
    created_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    FOREIGN KEY (customer_id) REFERENCES customers(customer_id) ON DELETE CASCADE,
    INDEX idx_customer (customer_id),
    INDEX idx_session (session_id)
) ENGINE=InnoDB;

-- ตาราง Cart Items (รายการใน Cart)
CREATE TABLE cart_items (
    cart_item_id    INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    cart_id         INT UNSIGNED NOT NULL,
    variant_id      INT UNSIGNED NOT NULL,
    quantity        INT NOT NULL DEFAULT 1,
    unit_price      DECIMAL(12,2) NOT NULL,
    added_at        TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    FOREIGN KEY (cart_id) REFERENCES carts(cart_id) ON DELETE CASCADE,
    FOREIGN KEY (variant_id) REFERENCES product_variants(variant_id),
    UNIQUE KEY uk_cart_variant (cart_id, variant_id),
    INDEX idx_cart (cart_id)
) ENGINE=InnoDB;

-- =========================================
-- SECTION 5: ORDERS
-- =========================================

-- ตาราง Orders (คำสั่งซื้อ)
CREATE TABLE orders (
    order_id        INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    order_number    VARCHAR(50) NOT NULL UNIQUE,
    customer_id     INT UNSIGNED NOT NULL,
    status          ENUM('pending','confirmed','processing','shipped','delivered','cancelled','refunded') DEFAULT 'pending',
    payment_status  ENUM('pending','paid','failed','refunded','partial_refund') DEFAULT 'pending',
    
    -- ที่อยู่จัดส่ง (snapshot ณ เวลาสั่งซื้อ)
    shipping_name   VARCHAR(200) NOT NULL,
    shipping_phone  VARCHAR(20),
    shipping_address1 VARCHAR(255) NOT NULL,
    shipping_address2 VARCHAR(255),
    shipping_district VARCHAR(100),
    shipping_city   VARCHAR(100) NOT NULL,
    shipping_postal VARCHAR(20) NOT NULL,
    shipping_country CHAR(2) DEFAULT 'TH',
    
    -- ราคา
    subtotal        DECIMAL(12,2) NOT NULL DEFAULT 0.00,
    discount_amount DECIMAL(12,2) NOT NULL DEFAULT 0.00,
    shipping_fee    DECIMAL(10,2) NOT NULL DEFAULT 0.00,
    tax_amount      DECIMAL(10,2) NOT NULL DEFAULT 0.00,
    total_amount    DECIMAL(12,2) NOT NULL DEFAULT 0.00,
    
    -- ข้อมูลเพิ่มเติม
    coupon_code     VARCHAR(50),
    notes           TEXT,
    tracking_number VARCHAR(100),
    shipped_at      TIMESTAMP NULL,
    delivered_at    TIMESTAMP NULL,
    cancelled_at    TIMESTAMP NULL,
    cancel_reason   TEXT,
    created_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    
    FOREIGN KEY (customer_id) REFERENCES customers(customer_id),
    INDEX idx_customer (customer_id),
    INDEX idx_status (status),
    INDEX idx_payment_status (payment_status),
    INDEX idx_created_at (created_at),
    INDEX idx_order_number (order_number)
) ENGINE=InnoDB;

-- ตาราง Order Items (รายการสินค้าในคำสั่งซื้อ)
CREATE TABLE order_items (
    order_item_id   INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    order_id        INT UNSIGNED NOT NULL,
    variant_id      INT UNSIGNED NOT NULL,
    product_name    VARCHAR(500) NOT NULL,
    variant_name    VARCHAR(255),
    sku             VARCHAR(100) NOT NULL,
    quantity        INT NOT NULL,
    unit_price      DECIMAL(12,2) NOT NULL,
    discount_per_item DECIMAL(10,2) DEFAULT 0.00,
    total_price     DECIMAL(12,2) NOT NULL,
    FOREIGN KEY (order_id) REFERENCES orders(order_id) ON DELETE CASCADE,
    FOREIGN KEY (variant_id) REFERENCES product_variants(variant_id),
    INDEX idx_order (order_id),
    INDEX idx_variant (variant_id)
) ENGINE=InnoDB;

-- ตาราง Order Status History (ประวัติสถานะ)
CREATE TABLE order_status_history (
    history_id      INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    order_id        INT UNSIGNED NOT NULL,
    status          VARCHAR(50) NOT NULL,
    notes           TEXT,
    created_by      INT UNSIGNED,
    created_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (order_id) REFERENCES orders(order_id) ON DELETE CASCADE,
    INDEX idx_order (order_id)
) ENGINE=InnoDB;

-- =========================================
-- SECTION 6: PAYMENTS
-- =========================================

-- ตาราง Payments (การชำระเงิน)
CREATE TABLE payments (
    payment_id      INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    order_id        INT UNSIGNED NOT NULL,
    payment_method  ENUM('credit_card','debit_card','bank_transfer','promptpay','cod','wallet') NOT NULL,
    payment_gateway VARCHAR(50),
    gateway_transaction_id VARCHAR(255),
    amount          DECIMAL(12,2) NOT NULL,
    currency        CHAR(3) DEFAULT 'THB',
    status          ENUM('pending','success','failed','cancelled','refunded') DEFAULT 'pending',
    paid_at         TIMESTAMP NULL,
    refunded_at     TIMESTAMP NULL,
    metadata        JSON,
    created_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (order_id) REFERENCES orders(order_id),
    INDEX idx_order (order_id),
    INDEX idx_gateway_txn (gateway_transaction_id)
) ENGINE=InnoDB;

-- =========================================
-- SECTION 7: RETURNS AND REFUNDS
-- =========================================

-- ตาราง Returns (การคืนสินค้า)
CREATE TABLE returns (
    return_id       INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    order_id        INT UNSIGNED NOT NULL,
    customer_id     INT UNSIGNED NOT NULL,
    return_number   VARCHAR(50) NOT NULL UNIQUE,
    status          ENUM('requested','approved','received','inspecting','completed','rejected') DEFAULT 'requested',
    reason          ENUM('defective','wrong_item','not_as_described','changed_mind','other') NOT NULL,
    reason_details  TEXT,
    refund_method   ENUM('original_payment','store_credit','bank_transfer') DEFAULT 'original_payment',
    refund_amount   DECIMAL(12,2),
    requested_at    TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    processed_at    TIMESTAMP NULL,
    completed_at    TIMESTAMP NULL,
    FOREIGN KEY (order_id) REFERENCES orders(order_id),
    FOREIGN KEY (customer_id) REFERENCES customers(customer_id),
    INDEX idx_order (order_id),
    INDEX idx_customer (customer_id)
) ENGINE=InnoDB;

-- ตาราง Return Items (รายการสินค้าที่คืน)
CREATE TABLE return_items (
    return_item_id  INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    return_id       INT UNSIGNED NOT NULL,
    order_item_id   INT UNSIGNED NOT NULL,
    quantity        INT NOT NULL,
    condition       ENUM('new','like_new','used','damaged') DEFAULT 'new',
    FOREIGN KEY (return_id) REFERENCES returns(return_id) ON DELETE CASCADE,
    FOREIGN KEY (order_item_id) REFERENCES order_items(order_item_id),
    INDEX idx_return (return_id)
) ENGINE=InnoDB;

-- =========================================
-- SECTION 8: REVIEWS AND RATINGS
-- =========================================

-- ตาราง Product Reviews (รีวิวสินค้า)
CREATE TABLE product_reviews (
    review_id       INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    product_id      INT UNSIGNED NOT NULL,
    customer_id     INT UNSIGNED NOT NULL,
    order_item_id   INT UNSIGNED,
    rating          TINYINT NOT NULL CHECK (rating BETWEEN 1 AND 5),
    title           VARCHAR(255),
    body            TEXT,
    pros            TEXT,
    cons            TEXT,
    is_verified_purchase BOOLEAN DEFAULT FALSE,
    is_approved     BOOLEAN DEFAULT FALSE,
    helpful_count   INT DEFAULT 0,
    not_helpful_count INT DEFAULT 0,
    created_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    FOREIGN KEY (product_id) REFERENCES products(product_id) ON DELETE CASCADE,
    FOREIGN KEY (customer_id) REFERENCES customers(customer_id),
    FOREIGN KEY (order_item_id) REFERENCES order_items(order_item_id) ON DELETE SET NULL,
    UNIQUE KEY uk_customer_product (customer_id, product_id),
    INDEX idx_product (product_id),
    INDEX idx_rating (rating),
    INDEX idx_approved (is_approved)
) ENGINE=InnoDB;

-- ตาราง Review Images (รูปภาพประกอบรีวิว)
CREATE TABLE review_images (
    image_id        INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    review_id       INT UNSIGNED NOT NULL,
    image_url       VARCHAR(500) NOT NULL,
    sort_order      INT DEFAULT 0,
    FOREIGN KEY (review_id) REFERENCES product_reviews(review_id) ON DELETE CASCADE
) ENGINE=InnoDB;

-- =========================================
-- SECTION 9: WISHLISTS
-- =========================================

-- ตาราง Wishlists (รายการสินค้าที่ต้องการ)
CREATE TABLE wishlists (
    wishlist_id     INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    customer_id     INT UNSIGNED NOT NULL UNIQUE,
    created_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (customer_id) REFERENCES customers(customer_id) ON DELETE CASCADE
) ENGINE=InnoDB;

-- ตาราง Wishlist Items (รายการใน Wishlist)
CREATE TABLE wishlist_items (
    wishlist_item_id INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    wishlist_id     INT UNSIGNED NOT NULL,
    product_id      INT UNSIGNED NOT NULL,
    added_at        TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (wishlist_id) REFERENCES wishlists(wishlist_id) ON DELETE CASCADE,
    FOREIGN KEY (product_id) REFERENCES products(product_id) ON DELETE CASCADE,
    UNIQUE KEY uk_wishlist_product (wishlist_id, product_id),
    INDEX idx_wishlist (wishlist_id)
) ENGINE=InnoDB;

-- =========================================
-- SECTION 10: DISCOUNTS AND COUPONS
-- =========================================

-- ตาราง Discount Rules (กฎส่วนลด)
CREATE TABLE discount_rules (
    rule_id         INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    name            VARCHAR(200) NOT NULL,
    description     TEXT,
    discount_type   ENUM('percentage','fixed_amount','free_shipping','buy_x_get_y') NOT NULL,
    discount_value  DECIMAL(10,2) NOT NULL,
    min_order_amount DECIMAL(12,2),
    max_discount_amount DECIMAL(10,2),
    applies_to      ENUM('all','category','product','brand') DEFAULT 'all',
    applies_to_id   INT UNSIGNED,
    is_active       BOOLEAN DEFAULT TRUE,
    starts_at       TIMESTAMP NULL,
    ends_at         TIMESTAMP NULL,
    usage_limit     INT,
    usage_count     INT DEFAULT 0,
    created_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_active (is_active),
    INDEX idx_dates (starts_at, ends_at)
) ENGINE=InnoDB;

-- ตาราง Coupons (คูปองส่วนลด)
CREATE TABLE coupons (
    coupon_id       INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    rule_id         INT UNSIGNED NOT NULL,
    code            VARCHAR(50) NOT NULL UNIQUE,
    is_single_use   BOOLEAN DEFAULT FALSE,
    customer_id     INT UNSIGNED,
    usage_limit     INT DEFAULT 1,
    usage_count     INT DEFAULT 0,
    is_active       BOOLEAN DEFAULT TRUE,
    expires_at      TIMESTAMP NULL,
    created_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (rule_id) REFERENCES discount_rules(rule_id),
    FOREIGN KEY (customer_id) REFERENCES customers(customer_id) ON DELETE SET NULL,
    INDEX idx_code (code),
    INDEX idx_active (is_active)
) ENGINE=InnoDB;

-- ตาราง Coupon Usage (ประวัติการใช้คูปอง)
CREATE TABLE coupon_usage (
    usage_id        INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    coupon_id       INT UNSIGNED NOT NULL,
    order_id        INT UNSIGNED NOT NULL,
    customer_id     INT UNSIGNED NOT NULL,
    discount_applied DECIMAL(10,2) NOT NULL,
    used_at         TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (coupon_id) REFERENCES coupons(coupon_id),
    FOREIGN KEY (order_id) REFERENCES orders(order_id),
    FOREIGN KEY (customer_id) REFERENCES customers(customer_id),
    INDEX idx_coupon (coupon_id),
    INDEX idx_customer (customer_id)
) ENGINE=InnoDB;

-- =========================================
-- SECTION 11: TAX MANAGEMENT
-- =========================================

-- ตาราง Tax Rates (อัตราภาษี)
CREATE TABLE tax_rates (
    tax_rate_id     INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    name            VARCHAR(100) NOT NULL,
    rate_percentage DECIMAL(5,2) NOT NULL,
    country_code    CHAR(2) NOT NULL,
    applies_to      ENUM('all','category','product') DEFAULT 'all',
    applies_to_id   INT UNSIGNED,
    is_active       BOOLEAN DEFAULT TRUE,
    created_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_country (country_code),
    INDEX idx_active (is_active)
) ENGINE=InnoDB;

-- ตาราง Order Tax Details (รายละเอียดภาษีในคำสั่งซื้อ)
CREATE TABLE order_tax_details (
    tax_detail_id   INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    order_id        INT UNSIGNED NOT NULL,
    tax_rate_id     INT UNSIGNED NOT NULL,
    taxable_amount  DECIMAL(12,2) NOT NULL,
    tax_amount      DECIMAL(10,2) NOT NULL,
    FOREIGN KEY (order_id) REFERENCES orders(order_id) ON DELETE CASCADE,
    FOREIGN KEY (tax_rate_id) REFERENCES tax_rates(tax_rate_id),
    INDEX idx_order (order_id)
) ENGINE=InnoDB;

-- =========================================
-- SECTION 12: SHIPPING
-- =========================================

-- ตาราง Shipping Methods (วิธีการจัดส่ง)
CREATE TABLE shipping_methods (
    method_id       INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    name            VARCHAR(100) NOT NULL,
    carrier         VARCHAR(100),
    base_fee        DECIMAL(8,2) NOT NULL,
    per_kg_fee      DECIMAL(6,2) DEFAULT 0,
    min_days        INT,
    max_days        INT,
    is_active       BOOLEAN DEFAULT TRUE
) ENGINE=InnoDB;
```

## 2. ข้อมูลตัวอย่าง (Sample Data - 100+ INSERT Statements)

```sql
-- =========================================
-- SAMPLE DATA INSERTS
-- =========================================

-- Insert Categories (หมวดหมู่)
INSERT INTO categories (category_id, parent_id, name, slug, description, sort_order) VALUES
(1, NULL, 'เสื้อผ้า', 'clothing', 'เสื้อผ้าทุกประเภท', 1),
(2, NULL, 'อิเล็กทรอนิกส์', 'electronics', 'สินค้าอิเล็กทรอนิกส์', 2),
(3, NULL, 'บ้านและสวน', 'home-garden', 'สินค้าสำหรับบ้านและสวน', 3),
(4, NULL, 'กีฬาและกลางแจ้ง', 'sports-outdoor', 'อุปกรณ์กีฬา', 4),
(5, NULL, 'ความงามและสุขภาพ', 'beauty-health', 'สินค้าความงาม', 5),
(6, 1, 'เสื้อยืด', 'tshirts', 'เสื้อยืดทุกสไตล์', 1),
(7, 1, 'กางเกง', 'pants', 'กางเกงทุกประเภท', 2),
(8, 1, 'เดรส', 'dresses', 'เดรสผู้หญิง', 3),
(9, 1, 'แจ็คเก็ต', 'jackets', 'แจ็คเก็ตและเสื้อกันหนาว', 4),
(10, 2, 'สมาร์ทโฟน', 'smartphones', 'โทรศัพท์มือถือ', 1),
(11, 2, 'แล็ปท็อป', 'laptops', 'คอมพิวเตอร์พกพา', 2),
(12, 2, 'หูฟัง', 'headphones', 'หูฟังทุกประเภท', 3),
(13, 3, 'เฟอร์นิเจอร์', 'furniture', 'เฟอร์นิเจอร์บ้าน', 1),
(14, 3, 'ของตกแต่ง', 'decorations', 'ของตกแต่งบ้าน', 2),
(15, 4, 'รองเท้ากีฬา', 'sports-shoes', 'รองเท้าสำหรับกีฬา', 1);

-- Insert Brands (แบรนด์)
INSERT INTO brands (brand_id, name, slug, description) VALUES
(1, 'Nike', 'nike', 'Just Do It'),
(2, 'Adidas', 'adidas', 'Impossible is Nothing'),
(3, 'Apple', 'apple', 'Think Different'),
(4, 'Samsung', 'samsung', 'Do What You Can\'t'),
(5, 'Sony', 'sony', 'Make.Believe'),
(6, 'IKEA', 'ikea', 'The Wonderful Everyday'),
(7, 'Uniqlo', 'uniqlo', 'LifeWear'),
(8, 'Zara', 'zara', 'Love Your Curves');

-- Insert Attribute Types
INSERT INTO attribute_types (attribute_type_id, name, display_name, input_type) VALUES
(1, 'size', 'ขนาด', 'button'),
(2, 'color', 'สี', 'color'),
(3, 'storage', 'ความจุ', 'select'),
(4, 'material', 'วัสดุ', 'select');

-- Insert Attribute Values - Sizes
INSERT INTO attribute_values (attribute_value_id, attribute_type_id, value, display_name, sort_order) VALUES
(1, 1, 'XS', 'XS', 1),
(2, 1, 'S', 'S', 2),
(3, 1, 'M', 'M', 3),
(4, 1, 'L', 'L', 4),
(5, 1, 'XL', 'XL', 5),
(6, 1, 'XXL', 'XXL', 6),
-- Colors
(7, 2, 'white', 'ขาว', 1),
(8, 2, 'black', 'ดำ', 2),
(9, 2, 'red', 'แดง', 3),
(10, 2, 'blue', 'น้ำเงิน', 4),
(11, 2, 'navy', 'กรมท่า', 5),
(12, 2, 'gray', 'เทา', 6),
-- Storage
(13, 3, '128gb', '128 GB', 1),
(14, 3, '256gb', '256 GB', 2),
(15, 3, '512gb', '512 GB', 3),
(16, 3, '1tb', '1 TB', 4);

-- Update color hex values
UPDATE attribute_values SET color_hex = '#FFFFFF' WHERE attribute_value_id = 7;
UPDATE attribute_values SET color_hex = '#000000' WHERE attribute_value_id = 8;
UPDATE attribute_values SET color_hex = '#FF0000' WHERE attribute_value_id = 9;
UPDATE attribute_values SET color_hex = '#0000FF' WHERE attribute_value_id = 10;
UPDATE attribute_values SET color_hex = '#001F5B' WHERE attribute_value_id = 11;
UPDATE attribute_values SET color_hex = '#808080' WHERE attribute_value_id = 12;

-- Insert Products (สินค้า)
INSERT INTO products (product_id, category_id, brand_id, sku, name, slug, base_price, sale_price, cost_price, status, is_featured) VALUES
(1, 6, 7, 'UNIQ-TSHIRT-001', 'Uniqlo เสื้อยืด Supima Cotton', 'uniqlo-supima-cotton-tshirt', 590, 490, 200, 'active', TRUE),
(2, 6, 1, 'NIKE-TSHIRT-DRI001', 'Nike Dri-FIT เสื้อยืดออกกำลังกาย', 'nike-drifit-training-tshirt', 990, 790, 350, 'active', TRUE),
(3, 7, 8, 'ZARA-PANT-001', 'Zara กางเกงขายาว Slim Fit', 'zara-slim-fit-pants', 1290, NULL, 450, 'active', FALSE),
(4, 10, 3, 'APPLE-IP15-PRO', 'iPhone 15 Pro', 'iphone-15-pro', 42900, NULL, 30000, 'active', TRUE),
(5, 10, 4, 'SAMSUNG-S24U', 'Samsung Galaxy S24 Ultra', 'samsung-galaxy-s24-ultra', 44900, 41900, 31000, 'active', TRUE),
(6, 11, 3, 'APPLE-MBP-M3', 'MacBook Pro M3', 'macbook-pro-m3', 59900, NULL, 40000, 'active', TRUE),
(7, 12, 5, 'SONY-WH1000XM5', 'Sony WH-1000XM5 Wireless Headphones', 'sony-wh1000xm5', 12900, 10900, 7000, 'active', TRUE),
(8, 15, 1, 'NIKE-AIR-MAX270', 'Nike Air Max 270', 'nike-air-max-270', 4500, 3900, 1800, 'active', TRUE),
(9, 15, 2, 'ADIDAS-ULTRA4D', 'Adidas Ultraboost 4D', 'adidas-ultraboost-4d', 6500, NULL, 2500, 'active', FALSE),
(10, 8, 8, 'ZARA-DRESS-001', 'Zara เดรสลายดอก Maxi', 'zara-floral-maxi-dress', 1890, 1490, 600, 'active', TRUE),
(11, 9, 2, 'ADIDAS-TRACK-001', 'Adidas Track Jacket', 'adidas-track-jacket', 2200, 1800, 800, 'active', FALSE),
(12, 6, 7, 'UNIQ-HENLEY-001', 'Uniqlo Henley Neck Long Sleeve', 'uniqlo-henley-longsleeve', 790, NULL, 280, 'active', FALSE);

-- Insert Product Images
INSERT INTO product_images (product_id, image_url, alt_text, sort_order, is_primary) VALUES
(1, '/images/products/uniqlo-supima-white.jpg', 'Uniqlo Supima Cotton สีขาว', 1, TRUE),
(1, '/images/products/uniqlo-supima-black.jpg', 'Uniqlo Supima Cotton สีดำ', 2, FALSE),
(2, '/images/products/nike-drifit-front.jpg', 'Nike Dri-FIT ด้านหน้า', 1, TRUE),
(4, '/images/products/iphone15pro-natural.jpg', 'iPhone 15 Pro Natural Titanium', 1, TRUE),
(4, '/images/products/iphone15pro-black.jpg', 'iPhone 15 Pro Black Titanium', 2, FALSE),
(5, '/images/products/samsung-s24u-titanium.jpg', 'Samsung S24 Ultra Titanium', 1, TRUE),
(6, '/images/products/macbook-pro-m3-silver.jpg', 'MacBook Pro M3 Silver', 1, TRUE),
(7, '/images/products/sony-wh1000xm5-black.jpg', 'Sony WH-1000XM5 Black', 1, TRUE),
(8, '/images/products/nike-airmax270-white.jpg', 'Nike Air Max 270 White', 1, TRUE);

-- Insert Product Variants (Variants สำหรับเสื้อผ้า)
INSERT INTO product_variants (variant_id, product_id, sku, name, price_adjustment) VALUES
-- Uniqlo Supima Cotton (size x color)
(1, 1, 'UNIQ-SUPIMA-S-WHITE', 'S / ขาว', 0),
(2, 1, 'UNIQ-SUPIMA-M-WHITE', 'M / ขาว', 0),
(3, 1, 'UNIQ-SUPIMA-L-WHITE', 'L / ขาว', 0),
(4, 1, 'UNIQ-SUPIMA-XL-WHITE', 'XL / ขาว', 0),
(5, 1, 'UNIQ-SUPIMA-S-BLACK', 'S / ดำ', 0),
(6, 1, 'UNIQ-SUPIMA-M-BLACK', 'M / ดำ', 0),
(7, 1, 'UNIQ-SUPIMA-L-BLACK', 'L / ดำ', 0),
(8, 1, 'UNIQ-SUPIMA-XL-BLACK', 'XL / ดำ', 0),
-- Nike Dri-FIT
(9, 2, 'NIKE-DRIFIT-S-BLACK', 'S / ดำ', 0),
(10, 2, 'NIKE-DRIFIT-M-BLACK', 'M / ดำ', 0),
(11, 2, 'NIKE-DRIFIT-L-BLACK', 'L / ดำ', 0),
(12, 2, 'NIKE-DRIFIT-S-NAVY', 'S / กรมท่า', 0),
(13, 2, 'NIKE-DRIFIT-M-NAVY', 'M / กรมท่า', 0),
-- iPhone 15 Pro (storage only)
(14, 4, 'APPLE-IP15PRO-128', '128 GB', 0),
(15, 4, 'APPLE-IP15PRO-256', '256 GB', 3000),
(16, 4, 'APPLE-IP15PRO-512', '512 GB', 7000),
(17, 4, 'APPLE-IP15PRO-1TB', '1 TB', 13000),
-- Samsung S24 Ultra
(18, 5, 'SAM-S24U-256', '256 GB', 0),
(19, 5, 'SAM-S24U-512', '512 GB', 4000),
-- Sony Headphones (color only)
(20, 7, 'SONY-WH5-BLACK', 'สีดำ', 0),
(21, 7, 'SONY-WH5-SILVER', 'สีเงิน', 0),
-- Nike Air Max 270 (size only)
(22, 8, 'NIKE-AM270-40', 'EU 40', 0),
(23, 8, 'NIKE-AM270-41', 'EU 41', 0),
(24, 8, 'NIKE-AM270-42', 'EU 42', 0),
(25, 8, 'NIKE-AM270-43', 'EU 43', 0),
(26, 8, 'NIKE-AM270-44', 'EU 44', 0);

-- Insert Variant Attribute Values
INSERT INTO variant_attribute_values (variant_id, attribute_value_id) VALUES
(1, 2), (1, 7),   -- S, White
(2, 3), (2, 7),   -- M, White
(3, 4), (3, 7),   -- L, White
(4, 5), (4, 7),   -- XL, White
(5, 2), (5, 8),   -- S, Black
(6, 3), (6, 8),   -- M, Black
(7, 4), (7, 8),   -- L, Black
(8, 5), (8, 8),   -- XL, Black
(9, 2), (9, 8),   -- S, Black (Nike)
(10, 3), (10, 8), -- M, Black
(11, 4), (11, 8), -- L, Black
(12, 2), (12, 11),-- S, Navy
(13, 3), (13, 11),-- M, Navy
(14, 13),  -- 128GB
(15, 14),  -- 256GB
(16, 15),  -- 512GB
(17, 16),  -- 1TB
(18, 14),  -- 256GB Samsung
(19, 15),  -- 512GB Samsung
(20, 8),   -- Black Sony
(21, 12);  -- Silver Sony (gray)

-- Insert Inventory
INSERT INTO inventory (variant_id, quantity, reserved_qty, reorder_point) VALUES
(1, 50, 2, 10), (2, 80, 5, 10), (3, 60, 3, 10), (4, 40, 1, 10),
(5, 55, 4, 10), (6, 75, 6, 10), (7, 65, 2, 10), (8, 35, 1, 10),
(9, 30, 2, 5), (10, 45, 3, 5), (11, 40, 1, 5), (12, 25, 2, 5), (13, 35, 1, 5),
(14, 20, 3, 5), (15, 15, 2, 5), (16, 10, 1, 5), (17, 5, 0, 3),
(18, 18, 4, 5), (19, 8, 1, 3),
(20, 25, 3, 5), (21, 20, 1, 5),
(22, 15, 1, 5), (23, 20, 2, 5), (24, 25, 3, 5), (25, 18, 1, 5), (26, 12, 0, 5);

-- Insert Customers (ลูกค้า)
INSERT INTO customers (customer_id, email, password_hash, first_name, last_name, phone, date_of_birth, gender, is_verified, loyalty_points, created_at) VALUES
(1, 'somchai@email.com', '$2b$12$hash1', 'สมชาย', 'ใจดี', '0891234567', '1990-05-15', 'male', TRUE, 1250, '2023-01-15 10:00:00'),
(2, 'malee@email.com', '$2b$12$hash2', 'มาลี', 'สวยงาม', '0862345678', '1992-08-20', 'female', TRUE, 3500, '2023-02-20 14:30:00'),
(3, 'wichai@email.com', '$2b$12$hash3', 'วิชัย', 'เก่งมาก', '0813456789', '1985-12-01', 'male', TRUE, 800, '2023-03-10 09:15:00'),
(4, 'nisa@email.com', '$2b$12$hash4', 'นิสา', 'รักดี', '0904567890', '1995-03-25', 'female', TRUE, 5200, '2023-01-05 16:45:00'),
(5, 'prasit@email.com', '$2b$12$hash5', 'ประสิทธิ์', 'มีสุข', '0775678901', '1988-07-10', 'male', FALSE, 0, '2024-01-20 11:00:00'),
(6, 'orapin@email.com', '$2b$12$hash6', 'อรพิน', 'งามดี', '0826789012', '1993-11-18', 'female', TRUE, 2100, '2023-06-15 13:20:00'),
(7, 'krit@email.com', '$2b$12$hash7', 'กฤต', 'สุขใส', '0857890123', '1991-04-30', 'male', TRUE, 900, '2023-08-22 08:30:00'),
(8, 'patchara@email.com', '$2b$12$hash8', 'พัชรา', 'ดีใจ', '0838901234', '1996-09-14', 'female', TRUE, 4100, '2022-12-01 15:00:00'),
(9, 'tawee@email.com', '$2b$12$hash9', 'ทวี', 'แข็งแรง', '0799012345', '1983-02-28', 'male', TRUE, 650, '2024-02-10 10:45:00'),
(10, 'sunee@email.com', '$2b$12$hash10', 'สุนี', 'มีความสุข', '0810123456', '1994-06-05', 'female', TRUE, 1800, '2023-04-18 12:00:00');

-- Insert Customer Addresses
INSERT INTO customer_addresses (customer_id, address_label, recipient_name, phone, address_line1, city, postal_code, country_code, is_default) VALUES
(1, 'บ้าน', 'สมชาย ใจดี', '0891234567', '123 ถนนสุขุมวิท แขวงคลองตัน', 'กรุงเทพมหานคร', '10110', 'TH', TRUE),
(2, 'บ้าน', 'มาลี สวยงาม', '0862345678', '456 ถนนพระราม 9 แขวงห้วยขวาง', 'กรุงเทพมหานคร', '10310', 'TH', TRUE),
(3, 'ที่ทำงาน', 'วิชัย เก่งมาก', '0813456789', '789 ถนนสีลม แขวงบางรัก', 'กรุงเทพมหานคร', '10500', 'TH', TRUE),
(4, 'บ้าน', 'นิสา รักดี', '0904567890', '321 ถนนนิมมานเหมินท์', 'เชียงใหม่', '50200', 'TH', TRUE),
(5, 'บ้าน', 'ประสิทธิ์ มีสุข', '0775678901', '654 ถนนเยาวราช', 'กรุงเทพมหานคร', '10100', 'TH', TRUE),
(6, 'บ้าน', 'อรพิน งามดี', '0826789012', '987 ถนนราชดำเนิน', 'กรุงเทพมหานคร', '10200', 'TH', TRUE),
(8, 'บ้าน', 'พัชรา ดีใจ', '0838901234', '147 ถนนพระราม 4 แขวงสาทร', 'กรุงเทพมหานคร', '10120', 'TH', TRUE),
(10, 'บ้าน', 'สุนี มีความสุข', '0810123456', '258 ถนนรัชดาภิเษก แขวงลาดยาว', 'กรุงเทพมหานคร', '10900', 'TH', TRUE);

-- Insert Discount Rules
INSERT INTO discount_rules (rule_id, name, discount_type, discount_value, min_order_amount, is_active, starts_at, ends_at) VALUES
(1, 'ส่วนลด 10% สำหรับสมาชิกใหม่', 'percentage', 10.00, 500, TRUE, '2024-01-01', '2024-12-31'),
(2, 'ลด 100 บาท เมื่อซื้อครบ 1000', 'fixed_amount', 100.00, 1000, TRUE, '2024-01-01', '2024-12-31'),
(3, 'ส่งฟรี เมื่อซื้อครบ 1500 บาท', 'free_shipping', 0.00, 1500, TRUE, '2024-01-01', NULL),
(4, 'แฟลชเซล 20% ลดทั้งร้าน', 'percentage', 20.00, NULL, FALSE, '2024-11-11', '2024-11-11');

-- Insert Coupons
INSERT INTO coupons (coupon_id, rule_id, code, is_single_use, usage_limit, usage_count, is_active, expires_at) VALUES
(1, 1, 'NEWMEMBER10', FALSE, 1000, 245, TRUE, '2024-12-31 23:59:59'),
(2, 2, 'SAVE100', FALSE, 500, 89, TRUE, '2024-12-31 23:59:59'),
(3, 3, 'FREESHIP', FALSE, 9999, 1523, TRUE, NULL),
(4, 4, '11-11-FLASH', FALSE, 9999, 0, FALSE, '2024-11-11 23:59:59');

-- Insert Tax Rates
INSERT INTO tax_rates (name, rate_percentage, country_code, is_active) VALUES
('VAT 7%', 7.00, 'TH', TRUE),
('No Tax', 0.00, 'TH', FALSE);

-- Insert Shipping Methods
INSERT INTO shipping_methods (name, carrier, base_fee, per_kg_fee, min_days, max_days, is_active) VALUES
('ส่งธรรมดา', 'Thailand Post', 50, 10, 3, 7, TRUE),
('ส่งด่วน', 'Thailand Post EMS', 100, 20, 1, 3, TRUE),
('Kerry Express', 'Kerry', 60, 15, 1, 3, TRUE),
('Flash Express', 'Flash', 55, 12, 1, 2, TRUE),
('J&T Express', 'J&T', 50, 10, 1, 3, TRUE);

-- Insert Orders (คำสั่งซื้อ)
INSERT INTO orders (order_id, order_number, customer_id, status, payment_status,
    shipping_name, shipping_phone, shipping_address1, shipping_city, shipping_postal, shipping_country,
    subtotal, discount_amount, shipping_fee, tax_amount, total_amount, coupon_code, created_at) VALUES
(1, 'ORD-2024-000001', 1, 'delivered', 'paid',
    'สมชาย ใจดี', '0891234567', '123 ถนนสุขุมวิท', 'กรุงเทพมหานคร', '10110', 'TH',
    1480, 0, 60, 103.60, 1643.60, NULL, '2024-01-20 10:30:00'),
(2, 'ORD-2024-000002', 2, 'delivered', 'paid',
    'มาลี สวยงาม', '0862345678', '456 ถนนพระราม 9', 'กรุงเทพมหานคร', '10310', 'TH',
    42900, 0, 0, 3003, 45903, NULL, '2024-01-25 14:00:00'),
(3, 'ORD-2024-000003', 4, 'delivered', 'paid',
    'นิสา รักดี', '0904567890', '321 ถนนนิมมานเหมินท์', 'เชียงใหม่', '50200', 'TH',
    10900, 0, 100, 763, 11763, NULL, '2024-02-05 09:00:00'),
(4, 'ORD-2024-000004', 1, 'delivered', 'paid',
    'สมชาย ใจดี', '0891234567', '123 ถนนสุขุมวิท', 'กรุงเทพมหานคร', '10110', 'TH',
    980, 98, 60, 66.64, 1008.64, 'NEWMEMBER10', '2024-02-15 16:00:00'),
(5, 'ORD-2024-000005', 8, 'shipped', 'paid',
    'พัชรา ดีใจ', '0838901234', '147 ถนนพระราม 4', 'กรุงเทพมหานคร', '10120', 'TH',
    45900, 0, 0, 3213, 49113, NULL, '2024-03-01 11:30:00'),
(6, 'ORD-2024-000006', 3, 'processing', 'paid',
    'วิชัย เก่งมาก', '0813456789', '789 ถนนสีลม', 'กรุงเทพมหานคร', '10500', 'TH',
    1580, 100, 0, 103.6, 1583.6, 'SAVE100', '2024-03-10 08:45:00'),
(7, 'ORD-2024-000007', 6, 'confirmed', 'paid',
    'อรพิน งามดี', '0826789012', '987 ถนนราชดำเนิน', 'กรุงเทพมหานคร', '10200', 'TH',
    3900, 0, 60, 275.1, 4235.1, NULL, '2024-03-15 15:20:00'),
(8, 'ORD-2024-000008', 10, 'pending', 'pending',
    'สุนี มีความสุข', '0810123456', '258 ถนนรัชดาภิเษก', 'กรุงเทพมหานคร', '10900', 'TH',
    790, 0, 50, 58.8, 898.8, NULL, '2024-03-20 20:10:00');

-- Insert Order Items
INSERT INTO order_items (order_id, variant_id, product_name, variant_name, sku, quantity, unit_price, total_price) VALUES
-- Order 1: เสื้อยืด Uniqlo + Nike Dri-FIT
(1, 2, 'Uniqlo เสื้อยืด Supima Cotton', 'M / ขาว', 'UNIQ-SUPIMA-M-WHITE', 2, 490, 980),
(1, 10, 'Nike Dri-FIT เสื้อยืดออกกำลังกาย', 'M / ดำ', 'NIKE-DRIFIT-M-BLACK', 1, 790, 790),
-- Order 2: iPhone 15 Pro
(2, 15, 'iPhone 15 Pro', '256 GB', 'APPLE-IP15PRO-256', 1, 45900, 45900),
-- Order 3: Sony WH-1000XM5
(3, 20, 'Sony WH-1000XM5 Wireless Headphones', 'สีดำ', 'SONY-WH5-BLACK', 1, 10900, 10900),
-- Order 4: เสื้อยืด Nike
(4, 11, 'Nike Dri-FIT เสื้อยืดออกกำลังกาย', 'L / ดำ', 'NIKE-DRIFIT-L-BLACK', 1, 790, 790),
(4, 3, 'Uniqlo เสื้อยืด Supima Cotton', 'L / ขาว', 'UNIQ-SUPIMA-L-WHITE', 1, 490, 490),
-- Order 5: Samsung Galaxy S24 Ultra
(5, 19, 'Samsung Galaxy S24 Ultra', '512 GB', 'SAM-S24U-512', 1, 45900, 45900),
-- Order 6: Zara Pants + Uniqlo Supima
(6, 3, 'Uniqlo เสื้อยืด Supima Cotton', 'L / ขาว', 'UNIQ-SUPIMA-L-WHITE', 2, 490, 980),
(6, 6, 'Uniqlo เสื้อยืด Supima Cotton', 'M / ดำ', 'UNIQ-SUPIMA-M-BLACK', 2, 490, 980),
-- Order 7: Nike Air Max 270
(7, 24, 'Nike Air Max 270', 'EU 42', 'NIKE-AM270-42', 1, 3900, 3900),
-- Order 8: Uniqlo Henley
(8, 2, 'Uniqlo เสื้อยืด Supima Cotton', 'M / ขาว', 'UNIQ-SUPIMA-M-WHITE', 1, 490, 490),
(8, 6, 'Uniqlo เสื้อยืด Supima Cotton', 'M / ดำ', 'UNIQ-SUPIMA-M-BLACK', 1, 490, 490);

-- Insert Order Status History
INSERT INTO order_status_history (order_id, status, notes, created_at) VALUES
(1, 'pending', 'Order received', '2024-01-20 10:30:00'),
(1, 'confirmed', 'Payment confirmed', '2024-01-20 11:00:00'),
(1, 'processing', 'Preparing order', '2024-01-21 09:00:00'),
(1, 'shipped', 'Shipped via Kerry Express TRK001', '2024-01-22 14:00:00'),
(1, 'delivered', 'Delivered successfully', '2024-01-24 16:30:00'),
(2, 'pending', 'Order received', '2024-01-25 14:00:00'),
(2, 'confirmed', 'Payment confirmed', '2024-01-25 14:30:00'),
(2, 'processing', 'Preparing order', '2024-01-26 10:00:00'),
(2, 'shipped', 'Shipped via Flash Express TRK002', '2024-01-27 13:00:00'),
(2, 'delivered', 'Delivered', '2024-01-29 11:00:00');

-- Insert Payments
INSERT INTO payments (order_id, payment_method, payment_gateway, gateway_transaction_id, amount, status, paid_at) VALUES
(1, 'credit_card', 'Omise', 'chrg_test_001', 1643.60, 'success', '2024-01-20 10:35:00'),
(2, 'promptpay', 'SCB', 'scb_qr_002', 45903, 'success', '2024-01-25 14:05:00'),
(3, 'credit_card', 'Omise', 'chrg_test_003', 11763, 'success', '2024-02-05 09:10:00'),
(4, 'bank_transfer', 'BBL', 'bbl_xfer_004', 1008.64, 'success', '2024-02-15 17:00:00'),
(5, 'credit_card', 'Omise', 'chrg_test_005', 49113, 'success', '2024-03-01 11:35:00'),
(6, 'promptpay', 'KBANK', 'kbank_qr_006', 1583.6, 'success', '2024-03-10 08:50:00'),
(7, 'credit_card', 'Omise', 'chrg_test_007', 4235.1, 'success', '2024-03-15 15:25:00'),
(8, 'promptpay', 'SCB', 'pending_008', 898.8, 'pending', NULL);

-- Insert Product Reviews
INSERT INTO product_reviews (product_id, customer_id, order_item_id, rating, title, body, is_verified_purchase, is_approved, helpful_count) VALUES
(1, 1, 1, 5, 'เนื้อผ้าดีมาก', 'สวมใส่สบาย เนื้อผ้า Supima Cotton นุ่มมาก ซักแล้วไม่หด แนะนำมากครับ', TRUE, TRUE, 23),
(2, 1, 2, 4, 'ดีแต่ขนาดเล็กนิดนึง', 'คุณภาพดีครับ ระบายเหงื่อได้ดี แต่ Size อาจจะ Fit นิดนึง แนะนำให้ซื้อ Size ใหญ่กว่าปกติสักไซส์', TRUE, TRUE, 15),
(4, 2, 3, 5, 'iPhone 15 Pro สุดยอด', 'กล้องดีมาก Pro Max จะดีกว่า แต่ขนาดนี้ถืองั้นกว่า Battery ดีขึ้นเยอะมากจาก 14 Pro', TRUE, TRUE, 45),
(7, 4, 4, 5, 'หูฟังดีที่สุดที่เคยใช้', 'ตัด Noise ได้ดีมาก เสียงใส ใส่สบาย ชาร์จ 1 ครั้งฟังได้ทั้งวัน คุ้มมากกับราคา', TRUE, TRUE, 67),
(5, 8, 5, 4, 'S Pen ดีมาก', 'ตัว S Pen ที่รวมมาดีมาก กล้องดีมาก แบตเยี่ยม แต่ตัวเครื่องใหญ่เกินไปนิดนึงสำหรับมือเล็ก', TRUE, TRUE, 32),
(8, 7, 7, 4, 'Nike Air Max 270 สวมใส่สบาย', 'น้ำหนักเบา รองรับแรงกระแทกดี เหมาะสำหรับเดินทั้งวัน ราคาโอเคสำหรับคุณภาพ', TRUE, TRUE, 18);

-- Insert Wishlists
INSERT INTO wishlists (wishlist_id, customer_id) VALUES
(1, 1), (2, 2), (3, 4), (4, 6), (5, 8), (6, 10);

-- Insert Wishlist Items
INSERT INTO wishlist_items (wishlist_id, product_id, added_at) VALUES
(1, 4, '2024-01-10 10:00:00'),
(1, 7, '2024-01-15 14:00:00'),
(2, 6, '2024-02-01 09:00:00'),
(2, 5, '2024-02-05 11:00:00'),
(3, 2, '2024-02-10 15:00:00'),
(3, 8, '2024-02-12 16:00:00'),
(4, 1, '2024-03-01 10:00:00'),
(5, 9, '2024-03-05 13:00:00'),
(6, 3, '2024-03-08 09:00:00'),
(6, 11, '2024-03-08 09:05:00');

-- Insert Coupon Usage
INSERT INTO coupon_usage (coupon_id, order_id, customer_id, discount_applied, used_at) VALUES
(1, 4, 1, 98.00, '2024-02-15 16:00:00'),
(2, 6, 3, 100.00, '2024-03-10 08:45:00');

-- Insert Returns
INSERT INTO returns (return_id, order_id, customer_id, return_number, status, reason, reason_details, refund_method, refund_amount, requested_at) VALUES
(1, 1, 1, 'RET-2024-000001', 'completed', 'wrong_item', 'ได้รับสี White แต่ต้องการ Black', 'original_payment', 490, '2024-01-26 10:00:00');

INSERT INTO return_items (return_id, order_item_id, quantity, condition) VALUES
(1, 1, 1, 'new');
```

## 3. Business Queries (20+ Essential Queries)

### Query 1: แสดงสินค้าพร้อมหมวดหมู่แบบ Full Path (Category Breadcrumb)

```sql
-- แสดง Category Path แบบ Hierarchical ด้วย Recursive CTE
WITH RECURSIVE category_path AS (
    -- Base case: หมวดหมู่รากที่ไม่มี parent
    SELECT 
        category_id,
        parent_id,
        name,
        CAST(name AS CHAR(500)) AS full_path,
        0 AS depth
    FROM categories
    WHERE parent_id IS NULL
    
    UNION ALL
    
    -- Recursive case: หมวดหมู่ที่มี parent
    SELECT 
        c.category_id,
        c.parent_id,
        c.name,
        CONCAT(cp.full_path, ' > ', c.name) AS full_path,
        cp.depth + 1 AS depth
    FROM categories c
    INNER JOIN category_path cp ON c.parent_id = cp.category_id
)
SELECT 
    p.product_id,
    p.name AS product_name,
    cp.full_path AS category_path,
    cp.depth AS category_depth,
    p.base_price,
    p.sale_price,
    COALESCE(p.sale_price, p.base_price) AS effective_price
FROM products p
JOIN category_path cp ON p.category_id = cp.category_id
WHERE p.status = 'active'
ORDER BY cp.full_path, p.name;
```

### Query 2: สินค้าพร้อม Stock และ Variant Count

```sql
SELECT 
    p.product_id,
    p.name,
    p.sku,
    b.name AS brand_name,
    COALESCE(p.sale_price, p.base_price) AS effective_price,
    COUNT(DISTINCT pv.variant_id) AS total_variants,
    SUM(i.quantity) AS total_stock,
    SUM(i.reserved_qty) AS total_reserved,
    SUM(i.quantity - i.reserved_qty) AS available_stock,
    ROUND(AVG(pr.rating), 1) AS avg_rating,
    COUNT(DISTINCT pr.review_id) AS review_count,
    p.status
FROM products p
LEFT JOIN brands b ON p.brand_id = b.brand_id
LEFT JOIN product_variants pv ON p.product_id = pv.product_id AND pv.is_active = TRUE
LEFT JOIN inventory i ON pv.variant_id = i.variant_id
LEFT JOIN product_reviews pr ON p.product_id = pr.product_id AND pr.is_approved = TRUE
WHERE p.status = 'active'
GROUP BY p.product_id, p.name, p.sku, b.name, p.sale_price, p.base_price, p.status
ORDER BY total_stock DESC;
```

### Query 3: ดูรายละเอียดคำสั่งซื้อพร้อมรายการสินค้า

```sql
SELECT 
    o.order_number,
    o.created_at AS order_date,
    CONCAT(c.first_name, ' ', c.last_name) AS customer_name,
    c.email,
    o.status AS order_status,
    o.payment_status,
    
    -- รายการสินค้า (แสดงเป็น JSON Array)
    JSON_ARRAYAGG(
        JSON_OBJECT(
            'product', oi.product_name,
            'variant', oi.variant_name,
            'qty', oi.quantity,
            'price', oi.unit_price,
            'total', oi.total_price
        )
    ) AS items,
    
    o.subtotal,
    o.discount_amount,
    o.shipping_fee,
    o.tax_amount,
    o.total_amount,
    o.coupon_code
FROM orders o
JOIN customers c ON o.customer_id = c.customer_id
JOIN order_items oi ON o.order_id = oi.order_id
WHERE o.order_id = 1
GROUP BY 
    o.order_id, o.order_number, o.created_at, c.first_name, c.last_name, 
    c.email, o.status, o.payment_status, o.subtotal, o.discount_amount, 
    o.shipping_fee, o.tax_amount, o.total_amount, o.coupon_code;
```

### Query 4: คำนวณราคาสินค้าพร้อมภาษี VAT

```sql
-- คำนวณราคาสินค้า + VAT
SELECT 
    p.name AS product_name,
    pv.name AS variant_name,
    COALESCE(p.sale_price, p.base_price) + pv.price_adjustment AS base_price_excl_tax,
    tr.rate_percentage AS vat_rate,
    ROUND(
        (COALESCE(p.sale_price, p.base_price) + pv.price_adjustment) * tr.rate_percentage / 100,
        2
    ) AS vat_amount,
    ROUND(
        (COALESCE(p.sale_price, p.base_price) + pv.price_adjustment) * (1 + tr.rate_percentage / 100),
        2
    ) AS price_incl_tax
FROM products p
JOIN product_variants pv ON p.product_id = pv.product_id
CROSS JOIN tax_rates tr
WHERE p.status = 'active'
  AND pv.is_active = TRUE
  AND tr.is_active = TRUE
  AND tr.country_code = 'TH'
ORDER BY p.name, pv.price_adjustment;
```

### Query 5: ตรวจสอบ Inventory ที่ต้องสั่งซื้อเพิ่ม (Below Reorder Point)

```sql
SELECT 
    p.name AS product_name,
    pv.sku,
    pv.name AS variant_name,
    i.quantity AS current_stock,
    i.reserved_qty,
    i.quantity - i.reserved_qty AS available_stock,
    i.reorder_point,
    i.reorder_qty,
    CASE 
        WHEN i.quantity = 0 THEN 'OUT OF STOCK'
        WHEN i.quantity <= i.reorder_point THEN 'REORDER NOW'
        WHEN i.quantity <= i.reorder_point * 1.5 THEN 'LOW STOCK'
        ELSE 'OK'
    END AS stock_status
FROM inventory i
JOIN product_variants pv ON i.variant_id = pv.variant_id
JOIN products p ON pv.product_id = p.product_id
WHERE p.status = 'active'
  AND i.quantity <= i.reorder_point
ORDER BY i.quantity ASC, p.name;
```

### Query 6: สรุปยอดขายตามหมวดหมู่

```sql
SELECT 
    c.name AS category_name,
    COUNT(DISTINCT o.order_id) AS total_orders,
    SUM(oi.quantity) AS units_sold,
    SUM(oi.total_price) AS revenue,
    ROUND(AVG(oi.unit_price), 2) AS avg_item_price,
    COUNT(DISTINCT oi.variant_id) AS unique_products_sold
FROM order_items oi
JOIN product_variants pv ON oi.variant_id = pv.variant_id
JOIN products p ON pv.product_id = p.product_id
JOIN categories c ON p.category_id = c.category_id
JOIN orders o ON oi.order_id = o.order_id
WHERE o.status NOT IN ('cancelled', 'refunded')
  AND o.payment_status = 'paid'
GROUP BY c.category_id, c.name
ORDER BY revenue DESC;
```

### Query 7: ลูกค้า VIP (ยอดซื้อสูงสุด + Loyalty Points)

```sql
SELECT 
    c.customer_id,
    CONCAT(c.first_name, ' ', c.last_name) AS customer_name,
    c.email,
    c.loyalty_points,
    COUNT(DISTINCT o.order_id) AS total_orders,
    SUM(o.total_amount) AS total_spent,
    ROUND(AVG(o.total_amount), 2) AS avg_order_value,
    MAX(o.created_at) AS last_order_date,
    DATEDIFF(CURDATE(), MAX(o.created_at)) AS days_since_last_order,
    CASE 
        WHEN SUM(o.total_amount) >= 100000 THEN 'PLATINUM'
        WHEN SUM(o.total_amount) >= 50000 THEN 'GOLD'
        WHEN SUM(o.total_amount) >= 10000 THEN 'SILVER'
        ELSE 'BRONZE'
    END AS tier
FROM customers c
LEFT JOIN orders o ON c.customer_id = o.customer_id 
    AND o.status NOT IN ('cancelled')
    AND o.payment_status = 'paid'
GROUP BY c.customer_id, c.first_name, c.last_name, c.email, c.loyalty_points
ORDER BY total_spent DESC NULLS LAST;
```

### Query 8: สินค้าที่อยู่ใน Wishlist มากที่สุด (Popularity)

```sql
SELECT 
    p.product_id,
    p.name AS product_name,
    b.name AS brand_name,
    COALESCE(p.sale_price, p.base_price) AS price,
    COUNT(wi.wishlist_item_id) AS wishlist_count,
    COALESCE(AVG(pr.rating), 0) AS avg_rating,
    SUM(i.quantity) AS stock_available,
    CASE WHEN SUM(i.quantity) > 0 THEN 'In Stock' ELSE 'Out of Stock' END AS availability
FROM products p
LEFT JOIN brands b ON p.brand_id = b.brand_id
LEFT JOIN wishlist_items wi ON p.product_id = wi.product_id
LEFT JOIN product_variants pv ON p.product_id = pv.product_id
LEFT JOIN inventory i ON pv.variant_id = i.variant_id
LEFT JOIN product_reviews pr ON p.product_id = pr.product_id AND pr.is_approved = TRUE
WHERE p.status = 'active'
GROUP BY p.product_id, p.name, b.name, p.sale_price, p.base_price
HAVING wishlist_count > 0
ORDER BY wishlist_count DESC, avg_rating DESC;
```

### Query 9: วิเคราะห์อัตราการคืนสินค้า

```sql
SELECT 
    p.name AS product_name,
    COUNT(DISTINCT oi.order_item_id) AS total_sold_instances,
    SUM(oi.quantity) AS total_units_sold,
    COUNT(DISTINCT ri.return_item_id) AS return_instances,
    SUM(ri.quantity) AS total_units_returned,
    ROUND(
        COUNT(DISTINCT ri.return_item_id) * 100.0 / COUNT(DISTINCT oi.order_item_id),
        2
    ) AS return_rate_percent,
    GROUP_CONCAT(DISTINCT r.reason) AS return_reasons
FROM products p
JOIN product_variants pv ON p.product_id = pv.product_id
JOIN order_items oi ON pv.variant_id = oi.variant_id
JOIN orders o ON oi.order_id = o.order_id
LEFT JOIN return_items ri ON oi.order_item_id = ri.order_item_id
LEFT JOIN returns r ON ri.return_id = r.return_id
WHERE o.status NOT IN ('cancelled')
GROUP BY p.product_id, p.name
HAVING total_sold_instances > 0
ORDER BY return_rate_percent DESC;
```

### Query 10: ประสิทธิผลของคูปอง

```sql
SELECT 
    cp.code AS coupon_code,
    dr.name AS discount_name,
    dr.discount_type,
    dr.discount_value,
    cp.usage_limit,
    cp.usage_count,
    ROUND(cp.usage_count * 100.0 / cp.usage_limit, 1) AS usage_rate_percent,
    COUNT(cu.usage_id) AS confirmed_usage,
    SUM(cu.discount_applied) AS total_discount_given,
    SUM(o.total_amount) AS total_revenue_from_coupon_orders,
    ROUND(AVG(o.total_amount), 2) AS avg_order_value_with_coupon
FROM coupons cp
JOIN discount_rules dr ON cp.rule_id = dr.rule_id
LEFT JOIN coupon_usage cu ON cp.coupon_id = cu.coupon_id
LEFT JOIN orders o ON cu.order_id = o.order_id AND o.payment_status = 'paid'
GROUP BY cp.coupon_id, cp.code, dr.name, dr.discount_type, dr.discount_value, cp.usage_limit, cp.usage_count
ORDER BY confirmed_usage DESC;
```

### Query 11: ตรวจสอบ Cart Abandonment

```sql
-- Cart ที่ยังไม่ได้ Checkout (Abandoned Carts)
SELECT 
    cart.cart_id,
    CONCAT(c.first_name, ' ', c.last_name) AS customer_name,
    c.email,
    COUNT(ci.cart_item_id) AS items_in_cart,
    SUM(ci.quantity * ci.unit_price) AS cart_value,
    cart.created_at AS cart_created,
    cart.updated_at AS last_updated,
    TIMESTAMPDIFF(HOUR, cart.updated_at, NOW()) AS hours_abandoned
FROM carts cart
LEFT JOIN customers c ON cart.customer_id = c.customer_id
JOIN cart_items ci ON cart.cart_id = ci.cart_id
WHERE cart.updated_at < DATE_SUB(NOW(), INTERVAL 24 HOUR)
  AND NOT EXISTS (
      SELECT 1 FROM orders o 
      WHERE o.customer_id = cart.customer_id 
      AND o.created_at > cart.updated_at
  )
GROUP BY cart.cart_id, c.first_name, c.last_name, c.email, cart.created_at, cart.updated_at
HAVING cart_value > 0
ORDER BY cart_value DESC;
```

### Query 12: สรุปคะแนน Rating และ Review

```sql
SELECT 
    p.name AS product_name,
    COUNT(pr.review_id) AS total_reviews,
    ROUND(AVG(pr.rating), 2) AS avg_rating,
    SUM(CASE WHEN pr.rating = 5 THEN 1 ELSE 0 END) AS five_star,
    SUM(CASE WHEN pr.rating = 4 THEN 1 ELSE 0 END) AS four_star,
    SUM(CASE WHEN pr.rating = 3 THEN 1 ELSE 0 END) AS three_star,
    SUM(CASE WHEN pr.rating = 2 THEN 1 ELSE 0 END) AS two_star,
    SUM(CASE WHEN pr.rating = 1 THEN 1 ELSE 0 END) AS one_star,
    SUM(CASE WHEN pr.is_verified_purchase = TRUE THEN 1 ELSE 0 END) AS verified_reviews,
    SUM(pr.helpful_count) AS total_helpful_votes
FROM products p
LEFT JOIN product_reviews pr ON p.product_id = pr.product_id AND pr.is_approved = TRUE
WHERE p.status = 'active'
GROUP BY p.product_id, p.name
ORDER BY avg_rating DESC, total_reviews DESC;
```

### Query 13: สินค้าแนะนำตาม Purchase History (Collaborative Filtering แบบง่าย)

```sql
-- ลูกค้าที่ซื้อสินค้า A มักซื้อสินค้า B ด้วย
SELECT 
    p1.name AS product_a,
    p2.name AS product_b,
    COUNT(*) AS co_purchase_count
FROM order_items oi1
JOIN order_items oi2 ON oi1.order_id = oi2.order_id 
    AND oi1.variant_id < oi2.variant_id  -- ป้องกัน duplicate
JOIN product_variants pv1 ON oi1.variant_id = pv1.variant_id
JOIN product_variants pv2 ON oi2.variant_id = pv2.variant_id
JOIN products p1 ON pv1.product_id = p1.product_id
JOIN products p2 ON pv2.product_id = p2.product_id
JOIN orders o ON oi1.order_id = o.order_id
WHERE o.status NOT IN ('cancelled')
  AND p1.product_id != p2.product_id
GROUP BY p1.product_id, p1.name, p2.product_id, p2.name
HAVING co_purchase_count >= 1
ORDER BY co_purchase_count DESC
LIMIT 20;
```

### Query 14: Revenue Summary ประจำเดือน

```sql
SELECT 
    DATE_FORMAT(o.created_at, '%Y-%m') AS year_month,
    COUNT(DISTINCT o.order_id) AS total_orders,
    COUNT(DISTINCT o.customer_id) AS unique_customers,
    SUM(oi.quantity) AS units_sold,
    SUM(o.subtotal) AS gross_revenue,
    SUM(o.discount_amount) AS total_discounts,
    SUM(o.shipping_fee) AS shipping_revenue,
    SUM(o.tax_amount) AS tax_collected,
    SUM(o.total_amount) AS net_revenue,
    ROUND(SUM(o.total_amount) / COUNT(DISTINCT o.order_id), 2) AS avg_order_value
FROM orders o
JOIN order_items oi ON o.order_id = oi.order_id
WHERE o.status NOT IN ('cancelled', 'refunded')
  AND o.payment_status = 'paid'
GROUP BY DATE_FORMAT(o.created_at, '%Y-%m')
ORDER BY year_month DESC;
```

### Query 15: ประวัติการสั่งซื้อของลูกค้า (Customer Order History)

```sql
SELECT 
    o.order_number,
    o.created_at,
    o.status,
    o.payment_status,
    COUNT(oi.order_item_id) AS item_count,
    SUM(oi.quantity) AS total_units,
    o.subtotal,
    o.discount_amount,
    o.total_amount,
    o.coupon_code,
    GROUP_CONCAT(
        CONCAT(oi.product_name, ' (', oi.quantity, 'x)')
        ORDER BY oi.order_item_id
        SEPARATOR ', '
    ) AS items_summary
FROM orders o
JOIN order_items oi ON o.order_id = oi.order_id
WHERE o.customer_id = 1
GROUP BY o.order_id, o.order_number, o.created_at, o.status, o.payment_status,
         o.subtotal, o.discount_amount, o.total_amount, o.coupon_code
ORDER BY o.created_at DESC;
```

### Query 16: Top Products (Best Sellers)

```sql
SELECT 
    RANK() OVER (ORDER BY SUM(oi.quantity) DESC) AS rank_position,
    p.product_id,
    p.name AS product_name,
    b.name AS brand_name,
    c.name AS category_name,
    SUM(oi.quantity) AS units_sold,
    SUM(oi.total_price) AS total_revenue,
    ROUND(AVG(pr.rating), 1) AS avg_rating,
    COUNT(DISTINCT pr.review_id) AS review_count
FROM order_items oi
JOIN orders o ON oi.order_id = o.order_id
JOIN product_variants pv ON oi.variant_id = pv.variant_id
JOIN products p ON pv.product_id = p.product_id
LEFT JOIN brands b ON p.brand_id = b.brand_id
JOIN categories c ON p.category_id = c.category_id
LEFT JOIN product_reviews pr ON p.product_id = pr.product_id AND pr.is_approved = TRUE
WHERE o.status NOT IN ('cancelled', 'refunded')
  AND o.payment_status = 'paid'
GROUP BY p.product_id, p.name, b.name, c.name
ORDER BY units_sold DESC
LIMIT 10;
```

### Query 17: สินค้าที่ใกล้หมด Stock แต่ยังขายดี

```sql
SELECT 
    p.name AS product_name,
    pv.sku,
    pv.name AS variant_name,
    i.quantity AS current_stock,
    i.reorder_point,
    COALESCE(recent_sales.units_sold_30d, 0) AS units_sold_last_30_days,
    CASE 
        WHEN COALESCE(recent_sales.units_sold_30d, 0) > 0 
        THEN ROUND(i.quantity / (recent_sales.units_sold_30d / 30.0), 0)
        ELSE NULL
    END AS days_of_stock_remaining
FROM inventory i
JOIN product_variants pv ON i.variant_id = pv.variant_id
JOIN products p ON pv.product_id = p.product_id
LEFT JOIN (
    SELECT 
        oi.variant_id,
        SUM(oi.quantity) AS units_sold_30d
    FROM order_items oi
    JOIN orders o ON oi.order_id = o.order_id
    WHERE o.created_at >= DATE_SUB(CURDATE(), INTERVAL 30 DAY)
      AND o.status NOT IN ('cancelled')
    GROUP BY oi.variant_id
) recent_sales ON pv.variant_id = recent_sales.variant_id
WHERE p.status = 'active'
  AND i.quantity <= i.reorder_point * 2
ORDER BY days_of_stock_remaining ASC NULLS LAST;
```

### Query 18: สรุปภาษีที่ต้องส่ง (Tax Report)

```sql
SELECT 
    DATE_FORMAT(o.created_at, '%Y-%m') AS period,
    tr.name AS tax_name,
    tr.rate_percentage,
    COUNT(DISTINCT o.order_id) AS orders_with_tax,
    SUM(o.subtotal - o.discount_amount) AS taxable_amount,
    SUM(o.tax_amount) AS tax_collected,
    SUM(o.total_amount) AS gross_revenue
FROM orders o
JOIN order_tax_details otd ON o.order_id = otd.order_id
JOIN tax_rates tr ON otd.tax_rate_id = tr.tax_rate_id
WHERE o.payment_status = 'paid'
  AND o.status NOT IN ('cancelled', 'refunded')
GROUP BY DATE_FORMAT(o.created_at, '%Y-%m'), tr.tax_rate_id, tr.name, tr.rate_percentage
ORDER BY period DESC, tr.rate_percentage DESC;
```

### Query 19: Variant ที่ขายดีของแต่ละสินค้า

```sql
SELECT 
    p.name AS product_name,
    pv.name AS variant_name,
    pv.sku,
    SUM(oi.quantity) AS units_sold,
    SUM(oi.total_price) AS revenue,
    i.quantity AS current_stock,
    RANK() OVER (
        PARTITION BY p.product_id 
        ORDER BY SUM(oi.quantity) DESC
    ) AS rank_within_product
FROM products p
JOIN product_variants pv ON p.product_id = pv.product_id
LEFT JOIN order_items oi ON pv.variant_id = oi.variant_id
LEFT JOIN orders o ON oi.order_id = o.order_id AND o.status NOT IN ('cancelled')
LEFT JOIN inventory i ON pv.variant_id = i.variant_id
GROUP BY p.product_id, p.name, pv.variant_id, pv.name, pv.sku, i.quantity
ORDER BY p.name, rank_within_product;
```

### Query 20: Lifetime Value ของลูกค้า + Churn Risk

```sql
SELECT 
    c.customer_id,
    CONCAT(c.first_name, ' ', c.last_name) AS customer_name,
    c.created_at AS joined_date,
    DATEDIFF(CURDATE(), c.created_at) AS days_as_customer,
    COUNT(DISTINCT o.order_id) AS total_orders,
    SUM(o.total_amount) AS total_lifetime_value,
    ROUND(AVG(o.total_amount), 2) AS avg_order_value,
    MAX(o.created_at) AS last_purchase_date,
    DATEDIFF(CURDATE(), MAX(o.created_at)) AS days_since_last_purchase,
    ROUND(
        COUNT(DISTINCT o.order_id) / 
        (DATEDIFF(CURDATE(), c.created_at) / 30.0), 
        2
    ) AS orders_per_month,
    CASE 
        WHEN DATEDIFF(CURDATE(), MAX(o.created_at)) > 180 THEN 'HIGH CHURN RISK'
        WHEN DATEDIFF(CURDATE(), MAX(o.created_at)) > 90 THEN 'MEDIUM CHURN RISK'
        WHEN DATEDIFF(CURDATE(), MAX(o.created_at)) > 30 THEN 'LOW CHURN RISK'
        ELSE 'ACTIVE'
    END AS churn_risk
FROM customers c
LEFT JOIN orders o ON c.customer_id = o.customer_id 
    AND o.payment_status = 'paid'
    AND o.status NOT IN ('cancelled')
WHERE c.is_active = TRUE
GROUP BY c.customer_id, c.first_name, c.last_name, c.created_at
ORDER BY total_lifetime_value DESC NULLS LAST;
```

## 4. Stored Procedures สำคัญ

### Procedure สำหรับ Checkout

```sql
DELIMITER //

CREATE PROCEDURE process_checkout(
    IN p_customer_id INT UNSIGNED,
    IN p_cart_id INT UNSIGNED,
    IN p_address_id INT UNSIGNED,
    IN p_coupon_code VARCHAR(50),
    IN p_payment_method VARCHAR(50),
    OUT p_order_id INT UNSIGNED,
    OUT p_order_number VARCHAR(50),
    OUT p_total_amount DECIMAL(12,2)
)
BEGIN
    DECLARE v_subtotal DECIMAL(12,2) DEFAULT 0;
    DECLARE v_discount DECIMAL(12,2) DEFAULT 0;
    DECLARE v_shipping_fee DECIMAL(10,2) DEFAULT 60;
    DECLARE v_tax_rate DECIMAL(5,2) DEFAULT 7;
    DECLARE v_tax_amount DECIMAL(10,2);
    DECLARE v_total DECIMAL(12,2);
    DECLARE v_coupon_id INT UNSIGNED;
    DECLARE v_discount_type VARCHAR(50);
    DECLARE v_discount_value DECIMAL(10,2);
    DECLARE v_min_order DECIMAL(12,2);
    
    DECLARE EXIT HANDLER FOR SQLEXCEPTION
    BEGIN
        ROLLBACK;
        RESIGNAL;
    END;
    
    START TRANSACTION;
    
    -- คำนวณ subtotal จาก cart
    SELECT SUM(ci.quantity * ci.unit_price) INTO v_subtotal
    FROM cart_items ci
    WHERE ci.cart_id = p_cart_id;
    
    -- ตรวจสอบและคำนวณ coupon
    IF p_coupon_code IS NOT NULL AND p_coupon_code != '' THEN
        SELECT c.coupon_id, dr.discount_type, dr.discount_value, dr.min_order_amount
        INTO v_coupon_id, v_discount_type, v_discount_value, v_min_order
        FROM coupons c
        JOIN discount_rules dr ON c.rule_id = dr.rule_id
        WHERE c.code = p_coupon_code
          AND c.is_active = TRUE
          AND (c.expires_at IS NULL OR c.expires_at > NOW())
          AND c.usage_count < c.usage_limit;
        
        IF v_coupon_id IS NOT NULL THEN
            IF v_min_order IS NULL OR v_subtotal >= v_min_order THEN
                IF v_discount_type = 'percentage' THEN
                    SET v_discount = v_subtotal * v_discount_value / 100;
                ELSEIF v_discount_type = 'fixed_amount' THEN
                    SET v_discount = LEAST(v_discount_value, v_subtotal);
                ELSEIF v_discount_type = 'free_shipping' THEN
                    SET v_shipping_fee = 0;
                END IF;
            END IF;
        END IF;
    END IF;
    
    -- ส่งฟรีถ้ายอดครบ 1500
    IF v_subtotal >= 1500 THEN
        SET v_shipping_fee = 0;
    END IF;
    
    -- คำนวณ VAT
    SET v_tax_amount = ROUND((v_subtotal - v_discount) * v_tax_rate / 100, 2);
    SET v_total = v_subtotal - v_discount + v_shipping_fee + v_tax_amount;
    
    -- สร้าง order number
    SET p_order_number = CONCAT('ORD-', YEAR(NOW()), '-', LPAD(
        (SELECT COALESCE(MAX(order_id), 0) + 1 FROM orders),
        6, '0'
    ));
    
    -- สร้าง Order
    INSERT INTO orders (
        order_number, customer_id, status, payment_status,
        shipping_name, shipping_phone, shipping_address1, shipping_city, 
        shipping_postal, shipping_country,
        subtotal, discount_amount, shipping_fee, tax_amount, total_amount,
        coupon_code
    )
    SELECT 
        p_order_number, p_customer_id, 'pending', 'pending',
        ca.recipient_name, ca.phone, ca.address_line1, ca.city,
        ca.postal_code, ca.country_code,
        v_subtotal, v_discount, v_shipping_fee, v_tax_amount, v_total,
        p_coupon_code
    FROM customer_addresses ca
    WHERE ca.address_id = p_address_id;
    
    SET p_order_id = LAST_INSERT_ID();
    SET p_total_amount = v_total;
    
    -- คัดลอก Cart Items -> Order Items
    INSERT INTO order_items (order_id, variant_id, product_name, variant_name, sku, quantity, unit_price, total_price)
    SELECT 
        p_order_id,
        ci.variant_id,
        p.name,
        pv.name,
        pv.sku,
        ci.quantity,
        ci.unit_price,
        ci.quantity * ci.unit_price
    FROM cart_items ci
    JOIN product_variants pv ON ci.variant_id = pv.variant_id
    JOIN products p ON pv.product_id = p.product_id
    WHERE ci.cart_id = p_cart_id;
    
    -- อัพเดท Inventory (Reserve)
    UPDATE inventory i
    JOIN cart_items ci ON i.variant_id = ci.variant_id
    SET i.reserved_qty = i.reserved_qty + ci.quantity
    WHERE ci.cart_id = p_cart_id;
    
    -- อัพเดท Coupon usage
    IF v_coupon_id IS NOT NULL THEN
        UPDATE coupons SET usage_count = usage_count + 1 WHERE coupon_id = v_coupon_id;
        INSERT INTO coupon_usage (coupon_id, order_id, customer_id, discount_applied)
        VALUES (v_coupon_id, p_order_id, p_customer_id, v_discount);
    END IF;
    
    -- ลบ Cart
    DELETE FROM cart_items WHERE cart_id = p_cart_id;
    DELETE FROM carts WHERE cart_id = p_cart_id;
    
    COMMIT;
END //

DELIMITER ;
```

## 5. Views สำคัญ

```sql
-- View: สินค้าพร้อมข้อมูลรวม
CREATE VIEW v_product_summary AS
SELECT 
    p.product_id,
    p.name,
    p.sku,
    p.slug,
    b.name AS brand_name,
    c.name AS category_name,
    COALESCE(p.sale_price, p.base_price) AS selling_price,
    p.base_price,
    p.sale_price,
    CASE WHEN p.sale_price IS NOT NULL 
         THEN ROUND((p.base_price - p.sale_price) * 100 / p.base_price, 1)
         ELSE 0 
    END AS discount_percent,
    p.status,
    p.is_featured,
    SUM(i.quantity) AS total_stock,
    COUNT(DISTINCT pv.variant_id) AS variant_count,
    ROUND(AVG(pr.rating), 1) AS avg_rating,
    COUNT(DISTINCT pr.review_id) AS review_count,
    p.created_at
FROM products p
LEFT JOIN brands b ON p.brand_id = b.brand_id
LEFT JOIN categories c ON p.category_id = c.category_id
LEFT JOIN product_variants pv ON p.product_id = pv.product_id AND pv.is_active = TRUE
LEFT JOIN inventory i ON pv.variant_id = i.variant_id
LEFT JOIN product_reviews pr ON p.product_id = pr.product_id AND pr.is_approved = TRUE
GROUP BY p.product_id, p.name, p.sku, p.slug, b.name, c.name,
         p.sale_price, p.base_price, p.status, p.is_featured, p.created_at;

-- View: Order Summary
CREATE VIEW v_order_summary AS
SELECT 
    o.order_id,
    o.order_number,
    CONCAT(c.first_name, ' ', c.last_name) AS customer_name,
    c.email,
    o.status,
    o.payment_status,
    COUNT(oi.order_item_id) AS item_types,
    SUM(oi.quantity) AS total_units,
    o.subtotal,
    o.discount_amount,
    o.shipping_fee,
    o.tax_amount,
    o.total_amount,
    o.coupon_code,
    o.created_at
FROM orders o
JOIN customers c ON o.customer_id = c.customer_id
JOIN order_items oi ON o.order_id = oi.order_id
GROUP BY o.order_id, o.order_number, c.first_name, c.last_name, c.email,
         o.status, o.payment_status, o.subtotal, o.discount_amount,
         o.shipping_fee, o.tax_amount, o.total_amount, o.coupon_code, o.created_at;
```

## 6. Indexes สำหรับ Performance

```sql
-- Composite indexes สำหรับ queries ที่ใช้บ่อย
CREATE INDEX idx_orders_customer_status ON orders(customer_id, status, created_at);
CREATE INDEX idx_order_items_variant ON order_items(variant_id, order_id);
CREATE INDEX idx_inventory_stock ON inventory(quantity, reorder_point);
CREATE INDEX idx_reviews_product_approved ON product_reviews(product_id, is_approved, rating);
CREATE INDEX idx_products_category_status ON products(category_id, status, is_featured);
CREATE INDEX idx_coupons_code_active ON coupons(code, is_active, expires_at);
```

## แบบฝึกหัด (Challenge Exercises)

1. **Trigger**: เขียน Trigger ที่อัพเดท `inventory` เมื่อมีการเปลี่ยนแปลง order status เป็น 'delivered'

2. **Query**: เขียน Query หาสินค้าที่ถูกเพิ่มลง Wishlist แต่ยังไม่เคยถูกสั่งซื้อ (ข้อมูลเชิงลึกสำหรับ remarketing)

3. **Report**: สร้าง Monthly Cohort Report แสดงว่าลูกค้าที่สมัครในเดือนเดียวกัน กลับมาซื้อซ้ำในเดือนต่อๆ ไปกี่เปอร์เซ็นต์

4. **Stored Procedure**: เขียน SP สำหรับ Process Return Request ที่ตรวจสอบ policy, อัพเดท inventory และสร้าง refund record

5. **Query**: หา "Dead Stock" คือสินค้าที่มี stock > 0 แต่ไม่มี order ใน 90 วันที่ผ่านมา

6. **Performance**: เพิ่ม Materialized View (หรือ Summary Table) สำหรับ pre-calculate ยอดขายรายวัน เพื่อให้ Dashboard load เร็วขึ้น

7. **Data Integrity**: เขียน Query ตรวจสอบ data integrity เช่น order total ที่ไม่ตรงกับ sum ของ order items

8. **Analytics**: เขียน Query สำหรับ RFM Analysis (Recency, Frequency, Monetary) เพื่อแบ่ง segment ลูกค้า

9. **Hierarchy**: เขียน Query แสดง Category Tree ทุกระดับพร้อมนับจำนวนสินค้าในแต่ละ node (รวม sub-categories ด้วย)

10. **Optimization**: วิเคราะห์ด้วย EXPLAIN ว่า Query ไหนใช้ Full Table Scan และแก้ไขโดยเพิ่ม Index ที่เหมาะสม
