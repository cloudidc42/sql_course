# ตอนที่ 109: Database Security Deep Dive

## บทนำ

ความปลอดภัยของฐานข้อมูลเป็นเรื่องที่ต้องให้ความสำคัญสูงสุด การรั่วไหลของข้อมูลสามารถสร้างความเสียหายอย่างมหาศาลทั้งด้านการเงินและชื่อเสียง บทนี้ครอบคลุมตั้งแต่ authentication, authorization, Row Level Security ไปจนถึง SQL injection prevention, encryption, และ GDPR compliance

---

## 1. Authentication และ Authorization

### 1.1 PostgreSQL Users และ Roles

```sql
-- สร้าง roles สำหรับ RBAC
CREATE ROLE app_readonly;
CREATE ROLE app_readwrite;
CREATE ROLE app_admin;
CREATE ROLE data_analyst;
CREATE ROLE etl_user;

-- Grant privileges ให้ roles
-- readonly role
GRANT CONNECT ON DATABASE myapp TO app_readonly;
GRANT USAGE ON SCHEMA public TO app_readonly;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO app_readonly;
ALTER DEFAULT PRIVILEGES IN SCHEMA public 
    GRANT SELECT ON TABLES TO app_readonly;

-- readwrite role
GRANT app_readonly TO app_readwrite;  -- inherit readonly
GRANT INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO app_readwrite;
GRANT USAGE ON ALL SEQUENCES IN SCHEMA public TO app_readwrite;
ALTER DEFAULT PRIVILEGES IN SCHEMA public 
    GRANT INSERT, UPDATE, DELETE ON TABLES TO app_readwrite;
ALTER DEFAULT PRIVILEGES IN SCHEMA public 
    GRANT USAGE ON SEQUENCES TO app_readwrite;

-- admin role
GRANT app_readwrite TO app_admin;
GRANT CREATE ON SCHEMA public TO app_admin;
GRANT ALL PRIVILEGES ON ALL TABLES IN SCHEMA public TO app_admin;

-- data_analyst role - อ่านได้เฉพาะตาราง analytics
GRANT CONNECT ON DATABASE myapp TO data_analyst;
GRANT USAGE ON SCHEMA analytics TO data_analyst;
GRANT SELECT ON ALL TABLES IN SCHEMA analytics TO data_analyst;

-- สร้าง users และ assign roles
CREATE USER web_app_user WITH PASSWORD 'strong_password_here' 
    CONNECTION LIMIT 50;
GRANT app_readwrite TO web_app_user;

CREATE USER readonly_user WITH PASSWORD 'another_strong_password'
    CONNECTION LIMIT 10;
GRANT app_readonly TO readonly_user;

CREATE USER analyst WITH PASSWORD 'analyst_password'
    CONNECTION LIMIT 5;
GRANT data_analyst TO analyst;

-- ETL user ที่มี limited access
CREATE USER etl_service WITH PASSWORD 'etl_password';
GRANT CONNECT ON DATABASE myapp TO etl_service;
GRANT USAGE ON SCHEMA staging TO etl_service;
GRANT ALL ON ALL TABLES IN SCHEMA staging TO etl_service;
GRANT SELECT ON orders, customers, products TO etl_service;
```

### 1.2 pg_hba.conf Configuration

```
# /etc/postgresql/16/main/pg_hba.conf
# TYPE  DATABASE        USER            ADDRESS                 METHOD

# Local connections: peer auth
local   all             postgres                                peer
local   all             all                                     peer

# IPv4 local connections: md5
host    all             all             127.0.0.1/32            md5

# Application server: scram-sha-256 (more secure than md5)
host    myapp           web_app_user    10.0.1.0/24            scram-sha-256
host    myapp           readonly_user   10.0.2.0/24            scram-sha-256

# Analytics server: SSL required
hostssl myapp           analyst         10.0.3.0/24            scram-sha-256

# Reject everything else
host    all             all             0.0.0.0/0              reject
```

---

## 2. Row Level Security (RLS)

### 2.1 Multi-tenant RLS

```sql
-- Multi-tenant Application Security

-- เพิ่ม tenant_id ใน tables
ALTER TABLE orders ADD COLUMN tenant_id INTEGER NOT NULL;
ALTER TABLE customers ADD COLUMN tenant_id INTEGER NOT NULL;
ALTER TABLE products ADD COLUMN tenant_id INTEGER NOT NULL;

-- Enable RLS
ALTER TABLE orders ENABLE ROW LEVEL SECURITY;
ALTER TABLE customers ENABLE ROW LEVEL SECURITY;
ALTER TABLE products ENABLE ROW LEVEL SECURITY;

-- สร้าง policy
-- ลูกค้าเห็นเฉพาะข้อมูลของ tenant ตัวเอง
CREATE POLICY tenant_isolation ON orders
    USING (tenant_id = current_setting('app.current_tenant_id')::INTEGER);

CREATE POLICY tenant_isolation ON customers
    USING (tenant_id = current_setting('app.current_tenant_id')::INTEGER);

CREATE POLICY tenant_isolation ON products
    USING (tenant_id = current_setting('app.current_tenant_id')::INTEGER);

-- Super admin ข้ามได้ทุก tenant
CREATE POLICY superadmin_access ON orders
    TO superadmin
    USING (TRUE);

-- Set tenant context ใน application
-- ก่อน execute queries, set session variable
SET LOCAL app.current_tenant_id = '42';

-- ตรวจสอบว่า RLS ทำงาน
SELECT id, customer_id, total_amount FROM orders;
-- จะเห็นเฉพาะ orders ของ tenant 42!

-- Function สำหรับ set tenant context
CREATE OR REPLACE FUNCTION set_tenant(tenant_id INTEGER)
RETURNS VOID AS $$
BEGIN
    PERFORM set_config('app.current_tenant_id', tenant_id::TEXT, TRUE);
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;
```

### 2.2 User-based RLS

```sql
-- User-based access control

-- Employees เห็นเฉพาะ records ของตัวเอง
ALTER TABLE employee_records ENABLE ROW LEVEL SECURITY;

CREATE POLICY employee_self_access ON employee_records
    USING (employee_id = (
        SELECT id FROM employees 
        WHERE username = current_user
    ));

-- Manager เห็นเฉพาะทีมของตัวเอง
CREATE POLICY manager_team_access ON employee_records
    TO managers
    USING (
        employee_id IN (
            SELECT e.id 
            FROM employees e
            JOIN team_assignments ta ON e.id = ta.employee_id
            WHERE ta.manager_username = current_user
        )
    );

-- HR เห็นทั้งหมด
CREATE POLICY hr_full_access ON employee_records
    TO hr_role
    USING (TRUE);

-- Salary visibility (เพิ่ม column-level check)
CREATE VIEW employee_public_info AS
SELECT 
    id,
    first_name,
    last_name,
    department,
    title,
    CASE 
        WHEN current_user = username OR pg_has_role('hr_role', 'usage')
        THEN salary::TEXT
        ELSE '***'
    END AS salary
FROM employee_records;
```

---

## 3. Column-Level Security

```sql
-- Column Security ด้วย views
CREATE VIEW customers_masked AS
SELECT 
    id,
    first_name,
    last_name,
    -- Mask email
    REGEXP_REPLACE(email, '(.{2}).*(@.*)', '\1***\2') AS email,
    -- Mask phone: แสดงเฉพาะ 4 หลักสุดท้าย
    CONCAT('***-***-', RIGHT(phone, 4)) AS phone,
    -- Mask card: แสดงเฉพาะ 4 หลักสุดท้าย
    CASE 
        WHEN pg_has_role('payment_team', 'usage') 
        THEN credit_card_last4
        ELSE '****'
    END AS credit_card_last4,
    city,
    country,
    created_at
FROM customers;

-- Grant access to masked view only
REVOKE ALL ON customers FROM app_readonly;
GRANT SELECT ON customers_masked TO app_readonly;

-- Column-level GRANT
GRANT SELECT (id, first_name, last_name, city, country) ON customers TO public_facing_role;

-- Prevent SELECT * on sensitive tables
REVOKE SELECT ON customers FROM app_readwrite;
GRANT SELECT (id, first_name, last_name, email, city) ON customers TO app_readwrite;
```

---

## 4. Data Masking

### 4.1 Dynamic Data Masking

```sql
-- Dynamic Data Masking Functions

CREATE OR REPLACE FUNCTION mask_email(email TEXT, reveal BOOLEAN DEFAULT FALSE)
RETURNS TEXT AS $$
BEGIN
    IF reveal THEN
        RETURN email;
    END IF;
    RETURN REGEXP_REPLACE(email, '(.{2}).*(@.*)', '\1***\2');
END;
$$ LANGUAGE plpgsql IMMUTABLE;

CREATE OR REPLACE FUNCTION mask_phone(phone TEXT, reveal BOOLEAN DEFAULT FALSE)
RETURNS TEXT AS $$
BEGIN
    IF reveal THEN
        RETURN phone;
    END IF;
    RETURN CONCAT('0**-***-', RIGHT(REGEXP_REPLACE(phone, '[^0-9]', '', 'g'), 4));
END;
$$ LANGUAGE plpgsql IMMUTABLE;

CREATE OR REPLACE FUNCTION mask_national_id(nid TEXT, reveal BOOLEAN DEFAULT FALSE)
RETURNS TEXT AS $$
BEGIN
    IF reveal THEN
        RETURN nid;
    END IF;
    -- Thai ID: x-xxxx-xxxxx-xx-x → show only last 4
    RETURN CONCAT('*-****-*****-**-', RIGHT(REGEXP_REPLACE(nid, '[^0-9]', '', 'g'), 1));
END;
$$ LANGUAGE plpgsql IMMUTABLE;

-- Masking view with role-based reveal
CREATE OR REPLACE VIEW customers_secure AS
SELECT 
    id,
    first_name,
    last_name,
    mask_email(email, pg_has_role('pii_readers', 'usage')) AS email,
    mask_phone(phone, pg_has_role('pii_readers', 'usage')) AS phone,
    mask_national_id(national_id, pg_has_role('full_pii_readers', 'usage')) AS national_id,
    city,
    province,
    country,
    created_at
FROM customers;

-- Audit function สำหรับ unmasked access
CREATE OR REPLACE FUNCTION log_pii_access(
    p_accessor TEXT,
    p_table TEXT,
    p_record_id INTEGER,
    p_reason TEXT
)
RETURNS VOID AS $$
BEGIN
    INSERT INTO pii_access_log (accessor, table_name, record_id, reason, accessed_at)
    VALUES (p_accessor, p_table, p_record_id, p_reason, NOW());
END;
$$ LANGUAGE plpgsql;
```

---

## 5. SQL Injection Prevention

### 5.1 Parameterized Queries

```python
# Python - psycopg2

# ❌ NEVER DO THIS - SQL Injection vulnerable
def get_user_BAD(username):
    query = f"SELECT * FROM users WHERE username = '{username}'"
    cursor.execute(query)
    
# ✅ Parameterized query
def get_user_SAFE(username):
    query = "SELECT id, username, email FROM users WHERE username = %s"
    cursor.execute(query, (username,))
    return cursor.fetchone()

# ❌ Dynamic table name injection (อันตราย)
def get_data_BAD(table_name):
    cursor.execute(f"SELECT * FROM {table_name}")

# ✅ Whitelist approach สำหรับ dynamic table names
ALLOWED_TABLES = {'orders', 'products', 'customers'}

def get_data_SAFE(table_name):
    if table_name not in ALLOWED_TABLES:
        raise ValueError(f"Invalid table: {table_name}")
    cursor.execute(f"SELECT * FROM {table_name}")  # Safe: whitelist validated

# ✅ Multi-value parameterized
def get_users_by_ids(user_ids):
    query = "SELECT * FROM users WHERE id = ANY(%s)"
    cursor.execute(query, (list(user_ids),))
    return cursor.fetchall()

# ✅ Dynamic ORDER BY with whitelist
ALLOWED_SORT_COLUMNS = {
    'name': 'name',
    'price': 'price', 
    'created_at': 'created_at',
    'total': 'total_amount'
}

def get_products_sorted(sort_by, sort_dir='ASC'):
    if sort_by not in ALLOWED_SORT_COLUMNS:
        sort_by = 'name'
    if sort_dir not in ('ASC', 'DESC'):
        sort_dir = 'ASC'
    
    col = ALLOWED_SORT_COLUMNS[sort_by]
    query = f"SELECT * FROM products ORDER BY {col} {sort_dir}"
    cursor.execute(query)
    return cursor.fetchall()
```

### 5.2 Input Validation

```python
import re
from typing import Optional
from pydantic import BaseModel, validator, constr, condecimal

class ProductSearchRequest(BaseModel):
    keyword: Optional[constr(max_length=100, strip_whitespace=True)] = None
    min_price: Optional[condecimal(ge=0, max_digits=10, decimal_places=2)] = None
    max_price: Optional[condecimal(ge=0, max_digits=10, decimal_places=2)] = None
    category_id: Optional[int] = None
    page: int = 1
    per_page: int = 20
    sort_by: str = 'name'
    sort_dir: str = 'ASC'
    
    @validator('keyword')
    def validate_keyword(cls, v):
        if v:
            # Remove potentially dangerous characters
            v = re.sub(r'[<>&\'"\\;]', '', v)
            if len(v.strip()) == 0:
                return None
        return v
    
    @validator('page', 'per_page')
    def validate_positive(cls, v):
        if v < 1:
            raise ValueError('Must be positive')
        return v
    
    @validator('per_page')
    def validate_per_page_max(cls, v):
        return min(v, 100)  # Max 100 items per page
    
    @validator('sort_by')
    def validate_sort_by(cls, v):
        allowed = {'name', 'price', 'created_at', 'stock_quantity'}
        if v not in allowed:
            return 'name'
        return v
    
    @validator('sort_dir')
    def validate_sort_dir(cls, v):
        return 'DESC' if v.upper() == 'DESC' else 'ASC'

# SQLAlchemy ป้องกัน injection โดย default
from sqlalchemy import text, select

# ✅ Safe with SQLAlchemy
def search_products(session, keyword):
    stmt = select(Product).where(
        Product.name.ilike(f'%{keyword}%')
    )
    return session.scalars(stmt).all()

# ✅ Safe raw query ด้วย bindparams
def search_raw(session, keyword):
    result = session.execute(
        text("SELECT * FROM products WHERE name ILIKE :keyword"),
        {'keyword': f'%{keyword}%'}
    )
    return result.fetchall()
```

### 5.3 PostgreSQL Stored Procedure Security

```sql
-- Security Definer functions
CREATE OR REPLACE FUNCTION get_user_data(p_username TEXT)
RETURNS TABLE (
    id INTEGER,
    username VARCHAR,
    email VARCHAR,
    role_name VARCHAR
)
SECURITY DEFINER  -- Run as function owner (higher privilege)
SET search_path = public  -- Prevent search_path injection
LANGUAGE plpgsql AS $$
BEGIN
    -- Input validation
    IF p_username IS NULL OR LENGTH(TRIM(p_username)) = 0 THEN
        RAISE EXCEPTION 'Invalid username';
    END IF;
    
    -- Safe parameterized query inside function
    RETURN QUERY
    SELECT u.id, u.username, u.email, r.name
    FROM users u
    JOIN user_roles ur ON u.id = ur.user_id
    JOIN roles r ON ur.role_id = r.id
    WHERE u.username = p_username;
END;
$$;

-- Revoke direct table access, grant only function
REVOKE SELECT ON users FROM app_readonly;
GRANT EXECUTE ON FUNCTION get_user_data TO app_readonly;
```

---

## 6. Audit Logging

```sql
-- Comprehensive Audit System

CREATE TABLE audit_log (
    id              BIGSERIAL PRIMARY KEY,
    table_name      TEXT NOT NULL,
    operation       TEXT NOT NULL,  -- INSERT, UPDATE, DELETE
    record_id       BIGINT,
    old_data        JSONB,
    new_data        JSONB,
    changed_fields  TEXT[],
    performed_by    TEXT DEFAULT current_user,
    app_user        TEXT,  -- Application-level user
    ip_address      INET,
    user_agent      TEXT,
    performed_at    TIMESTAMPTZ DEFAULT NOW()
);

-- Partition audit log by month for performance
CREATE TABLE audit_log_2024_01 PARTITION OF audit_log_partitioned
FOR VALUES FROM ('2024-01-01') TO ('2024-02-01');

-- Generic audit trigger
CREATE OR REPLACE FUNCTION audit_trigger_function()
RETURNS TRIGGER AS $$
DECLARE
    v_old_data JSONB;
    v_new_data JSONB;
    v_changed_fields TEXT[];
BEGIN
    IF (TG_OP = 'DELETE') THEN
        v_old_data = to_jsonb(OLD);
        INSERT INTO audit_log (table_name, operation, record_id, old_data, performed_by)
        VALUES (TG_TABLE_NAME, TG_OP, OLD.id, v_old_data, 
                COALESCE(current_setting('app.current_user', TRUE), current_user));
        RETURN OLD;
        
    ELSIF (TG_OP = 'UPDATE') THEN
        v_old_data = to_jsonb(OLD);
        v_new_data = to_jsonb(NEW);
        
        -- Calculate changed fields
        SELECT ARRAY_AGG(key) INTO v_changed_fields
        FROM jsonb_each(v_old_data) old_vals
        WHERE old_vals.value IS DISTINCT FROM v_new_data->old_vals.key;
        
        INSERT INTO audit_log (table_name, operation, record_id, old_data, new_data, changed_fields)
        VALUES (TG_TABLE_NAME, TG_OP, NEW.id, v_old_data, v_new_data, v_changed_fields);
        RETURN NEW;
        
    ELSIF (TG_OP = 'INSERT') THEN
        v_new_data = to_jsonb(NEW);
        INSERT INTO audit_log (table_name, operation, record_id, new_data)
        VALUES (TG_TABLE_NAME, TG_OP, NEW.id, v_new_data);
        RETURN NEW;
    END IF;
    
    RETURN NULL;
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;

-- Attach trigger to sensitive tables
CREATE TRIGGER audit_customers
    AFTER INSERT OR UPDATE OR DELETE ON customers
    FOR EACH ROW EXECUTE FUNCTION audit_trigger_function();

CREATE TRIGGER audit_orders
    AFTER INSERT OR UPDATE OR DELETE ON orders
    FOR EACH ROW EXECUTE FUNCTION audit_trigger_function();

-- Query audit log
SELECT 
    al.performed_at,
    al.operation,
    al.table_name,
    al.record_id,
    al.performed_by,
    al.changed_fields,
    al.old_data->>'email' AS old_email,
    al.new_data->>'email' AS new_email
FROM audit_log al
WHERE al.table_name = 'customers'
AND al.performed_at >= NOW() - INTERVAL '24 hours'
ORDER BY al.performed_at DESC;
```

---

## 7. Encryption

### 7.1 Column-Level Encryption (PostgreSQL pgcrypto)

```sql
-- Enable pgcrypto extension
CREATE EXTENSION IF NOT EXISTS pgcrypto;

-- Table with encrypted columns
CREATE TABLE sensitive_data (
    id              SERIAL PRIMARY KEY,
    customer_id     INTEGER REFERENCES customers(id),
    national_id_enc BYTEA,   -- Encrypted national ID
    bank_account_enc BYTEA,  -- Encrypted bank account
    key_id          INTEGER, -- Which encryption key was used
    created_at      TIMESTAMPTZ DEFAULT NOW()
);

-- Function สำหรับ encrypt/decrypt
-- Key management: store key separately (e.g., in HashiCorp Vault)
CREATE OR REPLACE FUNCTION encrypt_data(p_data TEXT, p_key TEXT)
RETURNS BYTEA AS $$
BEGIN
    RETURN pgp_sym_encrypt(p_data, p_key, 'cipher-algo=aes256');
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;

CREATE OR REPLACE FUNCTION decrypt_data(p_encrypted BYTEA, p_key TEXT)
RETURNS TEXT AS $$
BEGIN
    RETURN pgp_sym_decrypt(p_encrypted, p_key);
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;

-- Insert encrypted data
INSERT INTO sensitive_data (customer_id, national_id_enc, key_id)
VALUES (
    42,
    encrypt_data('1234567890123', current_setting('app.encryption_key')),
    1
);

-- Read and decrypt
SELECT 
    s.id,
    s.customer_id,
    decrypt_data(s.national_id_enc, current_setting('app.encryption_key')) AS national_id
FROM sensitive_data s
WHERE s.customer_id = 42;

-- Hash for searching (one-way)
CREATE OR REPLACE FUNCTION hash_for_search(p_data TEXT)
RETURNS TEXT AS $$
BEGIN
    -- Use HMAC with secret key for security
    RETURN encode(
        hmac(
            LOWER(TRIM(p_data)),
            current_setting('app.hash_secret'),
            'sha256'
        ),
        'hex'
    );
END;
$$ LANGUAGE plpgsql IMMUTABLE SECURITY DEFINER;

-- Store hash for lookup
ALTER TABLE customers ADD COLUMN email_hash TEXT;
UPDATE customers SET email_hash = hash_for_search(email);
CREATE INDEX idx_customers_email_hash ON customers(email_hash);

-- Search by hash (ไม่ต้อง decrypt)
SELECT id, first_name, last_name
FROM customers
WHERE email_hash = hash_for_search('user@example.com');
```

### 7.2 Transparent Data Encryption

```sql
-- TDE: encrypt at tablespace level (SQL Server equivalent)
-- PostgreSQL: ใช้ full-disk encryption + pg_crypto

-- Key rotation procedure
CREATE OR REPLACE PROCEDURE rotate_encryption_key(
    p_old_key TEXT,
    p_new_key TEXT
)
LANGUAGE plpgsql AS $$
DECLARE
    v_record RECORD;
    v_decrypted TEXT;
    v_rows_updated INTEGER := 0;
BEGIN
    FOR v_record IN 
        SELECT id, national_id_enc, bank_account_enc
        FROM sensitive_data
    LOOP
        -- Decrypt with old key
        v_decrypted := decrypt_data(v_record.national_id_enc, p_old_key);
        
        -- Re-encrypt with new key
        UPDATE sensitive_data
        SET national_id_enc = encrypt_data(v_decrypted, p_new_key),
            key_id = key_id + 1
        WHERE id = v_record.id;
        
        v_rows_updated := v_rows_updated + 1;
    END LOOP;
    
    RAISE NOTICE 'Key rotation complete: % records updated', v_rows_updated;
END;
$$;
```

---

## 8. GDPR Compliance

### 8.1 Right to Erasure (Right to be Forgotten)

```sql
-- GDPR: Data Subject Rights

-- Anonymization function
CREATE OR REPLACE FUNCTION anonymize_customer(p_customer_id INTEGER)
RETURNS VOID AS $$
DECLARE
    v_anon_suffix TEXT;
BEGIN
    v_anon_suffix := gen_random_uuid()::TEXT;
    
    -- Anonymize customer data
    UPDATE customers SET
        first_name = 'ANONYMIZED',
        last_name = 'USER',
        email = 'deleted_' || v_anon_suffix || '@anonymized.invalid',
        phone = NULL,
        national_id = NULL,
        address = NULL,
        date_of_birth = NULL,
        ip_address = NULL,
        is_anonymized = TRUE,
        anonymized_at = NOW()
    WHERE id = p_customer_id;
    
    -- Remove from marketing lists
    DELETE FROM email_subscriptions WHERE customer_id = p_customer_id;
    DELETE FROM push_subscriptions WHERE customer_id = p_customer_id;
    DELETE FROM marketing_segments WHERE customer_id = p_customer_id;
    
    -- Remove session data
    DELETE FROM user_sessions WHERE customer_id = p_customer_id;
    DELETE FROM user_events WHERE user_id = p_customer_id;
    
    -- Log the erasure
    INSERT INTO gdpr_requests (
        customer_id, request_type, performed_at, performed_by
    ) VALUES (
        p_customer_id, 'ERASURE', NOW(), current_user
    );
    
    RAISE NOTICE 'Customer % anonymized successfully', p_customer_id;
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;

-- Data export สำหรับ Data Portability
CREATE OR REPLACE FUNCTION export_customer_data(p_customer_id INTEGER)
RETURNS JSONB AS $$
DECLARE
    v_customer JSONB;
    v_orders JSONB;
    v_events JSONB;
BEGIN
    SELECT to_jsonb(c.*) INTO v_customer
    FROM customers c WHERE c.id = p_customer_id;
    
    SELECT COALESCE(jsonb_agg(to_jsonb(o.*)), '[]') INTO v_orders
    FROM orders o WHERE o.customer_id = p_customer_id;
    
    SELECT COALESCE(jsonb_agg(to_jsonb(e.*)), '[]') INTO v_events
    FROM user_events e WHERE e.user_id = p_customer_id
    AND e.event_date >= NOW() - INTERVAL '2 years';
    
    RETURN jsonb_build_object(
        'personal_data', v_customer,
        'purchase_history', v_orders,
        'activity_log', v_events,
        'exported_at', NOW()
    );
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;

-- Data retention policy
CREATE OR REPLACE PROCEDURE enforce_data_retention()
LANGUAGE plpgsql AS $$
BEGIN
    -- ลบ user events เก่ากว่า 2 ปี
    DELETE FROM user_events
    WHERE event_date < NOW() - INTERVAL '2 years';
    
    -- Archive orders เก่ากว่า 7 ปี
    INSERT INTO orders_archive
    SELECT * FROM orders
    WHERE created_at < NOW() - INTERVAL '7 years';
    
    DELETE FROM orders
    WHERE created_at < NOW() - INTERVAL '7 years';
    
    -- ลบ session logs เก่ากว่า 90 วัน
    DELETE FROM user_sessions
    WHERE created_at < NOW() - INTERVAL '90 days';
    
    RAISE NOTICE 'Data retention enforcement completed at %', NOW();
END;
$$;
```

---

## 9. Database Firewall

```sql
-- Connection limiting
ALTER USER web_app_user CONNECTION LIMIT 50;
ALTER USER analyst CONNECTION LIMIT 5;

-- Time-based access restrictions
CREATE OR REPLACE FUNCTION check_business_hours()
RETURNS BOOLEAN AS $$
BEGIN
    -- Allow access only during business hours (7am-10pm)
    IF EXTRACT(HOUR FROM CURRENT_TIMESTAMP AT TIME ZONE 'Asia/Bangkok') 
       NOT BETWEEN 7 AND 22 THEN
        RAISE EXCEPTION 'Access denied outside business hours';
    END IF;
    RETURN TRUE;
END;
$$ LANGUAGE plpgsql;

-- Suspicious query detection
CREATE TABLE query_stats (
    id              BIGSERIAL PRIMARY KEY,
    username        TEXT,
    query_text      TEXT,
    rows_returned   BIGINT,
    execution_time  FLOAT,
    recorded_at     TIMESTAMPTZ DEFAULT NOW()
);

-- Alert on bulk data access (potential data exfiltration)
CREATE OR REPLACE FUNCTION check_bulk_access()
RETURNS TRIGGER AS $$
BEGIN
    IF NEW.rows_returned > 10000 THEN
        INSERT INTO security_alerts (
            alert_type, username, details, created_at
        ) VALUES (
            'BULK_DATA_ACCESS',
            NEW.username,
            format('Query returned %s rows', NEW.rows_returned),
            NOW()
        );
    END IF;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;
```

---

## 10. Least Privilege Principle

```sql
-- Minimal permission setup สำหรับ microservices

-- Order Service: เข้าถึงเฉพาะ orders tables
CREATE ROLE order_service;
GRANT CONNECT ON DATABASE myapp TO order_service;
GRANT USAGE ON SCHEMA public TO order_service;
GRANT SELECT, INSERT, UPDATE ON orders TO order_service;
GRANT SELECT, INSERT, UPDATE, DELETE ON order_items TO order_service;
GRANT SELECT ON products TO order_service;  -- Read-only products
GRANT SELECT ON customers TO order_service;  -- Read-only customers
GRANT USAGE ON SEQUENCE orders_id_seq TO order_service;
GRANT USAGE ON SEQUENCE order_items_id_seq TO order_service;

-- Notification Service: read-only access
CREATE ROLE notification_service;
GRANT CONNECT ON DATABASE myapp TO notification_service;
GRANT USAGE ON SCHEMA public TO notification_service;
GRANT SELECT ON orders, customers, email_templates TO notification_service;

-- Analytics Service: read-only on analytics schema
CREATE ROLE analytics_service;
GRANT CONNECT ON DATABASE myapp TO analytics_service;
GRANT USAGE ON SCHEMA analytics TO analytics_service;
GRANT SELECT ON ALL TABLES IN SCHEMA analytics TO analytics_service;
-- NO access to main schema!

-- Payment Service: highly restricted
CREATE ROLE payment_service;
GRANT CONNECT ON DATABASE myapp TO payment_service;
GRANT USAGE ON SCHEMA payments TO payment_service;
GRANT SELECT, INSERT, UPDATE ON payments.transactions TO payment_service;
GRANT SELECT ON public.orders TO payment_service;
-- Cannot access customer PII

-- Verify permissions
SELECT 
    grantee,
    table_schema,
    table_name,
    privilege_type
FROM information_schema.role_table_grants
WHERE grantee IN ('order_service', 'notification_service', 'analytics_service')
ORDER BY grantee, table_name;
```

---

## 11. MySQL Security

```sql
-- MySQL 8.0 RBAC

-- สร้าง roles
CREATE ROLE 'app_read', 'app_write', 'app_admin';

-- Grant ให้ roles
GRANT SELECT ON myapp.* TO 'app_read';
GRANT SELECT, INSERT, UPDATE, DELETE ON myapp.* TO 'app_write';
GRANT ALL PRIVILEGES ON myapp.* TO 'app_admin';

-- สร้าง users
CREATE USER 'webapp'@'10.0.1.%' 
    IDENTIFIED BY 'strong_password'
    PASSWORD EXPIRE INTERVAL 90 DAY
    FAILED_LOGIN_ATTEMPTS 5 
    PASSWORD_LOCK_TIME 2;

GRANT 'app_write' TO 'webapp'@'10.0.1.%';
SET DEFAULT ROLE 'app_write' TO 'webapp'@'10.0.1.%';

-- SSL enforcement
ALTER USER 'webapp'@'10.0.1.%' REQUIRE SSL;

-- Column privileges
GRANT SELECT (id, username, email, city) ON myapp.users TO 'limited_user'@'%';

-- Stored procedure security
DELIMITER //
CREATE PROCEDURE GetOrderDetails(IN p_order_id INT)
    SQL SECURITY DEFINER
BEGIN
    -- Validate input
    IF p_order_id IS NULL OR p_order_id <= 0 THEN
        SIGNAL SQLSTATE '45000' SET MESSAGE_TEXT = 'Invalid order ID';
    END IF;
    
    SELECT o.id, o.status, o.total_amount,
           c.first_name, c.last_name
    FROM orders o
    JOIN customers c ON o.customer_id = c.id
    WHERE o.id = p_order_id;
END //
DELIMITER ;

-- Grant only procedure execution, not table access
GRANT EXECUTE ON PROCEDURE GetOrderDetails TO 'limited_user'@'%';
```

---

## แบบฝึกหัด

### ข้อที่ 1: RLS Multi-tenant

**เฉลย:**
```sql
-- Multi-tenant RLS setup
ALTER TABLE products ENABLE ROW LEVEL SECURITY;

CREATE POLICY tenant_products ON products
    USING (tenant_id = current_setting('app.tenant_id', TRUE)::INTEGER);

-- Admin bypass
CREATE POLICY admin_bypass ON products
    TO superadmin USING (TRUE);

-- Application code
-- session.execute("SET LOCAL app.tenant_id = :tid", {'tid': tenant_id})
```

### ข้อที่ 2: Audit Trigger

**เฉลย:**
```sql
CREATE OR REPLACE FUNCTION price_change_audit()
RETURNS TRIGGER AS $$
BEGIN
    IF OLD.price != NEW.price THEN
        INSERT INTO price_audit_log (
            product_id, old_price, new_price,
            change_pct, changed_by, changed_at
        ) VALUES (
            NEW.id, OLD.price, NEW.price,
            ROUND(100.0 * (NEW.price - OLD.price) / OLD.price, 2),
            current_setting('app.current_user', TRUE),
            NOW()
        );
    END IF;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER audit_price_changes
AFTER UPDATE ON products
FOR EACH ROW WHEN (OLD.price IS DISTINCT FROM NEW.price)
EXECUTE FUNCTION price_change_audit();
```

### ข้อที่ 3-10 (เฉลย ย่อ)

```sql
-- ข้อที่ 3: Parameterized search (Python)
-- ✅ Safe
def search(keyword, min_price, category_id):
    return db.execute(
        "SELECT * FROM products WHERE name ILIKE %s AND price >= %s AND category_id = %s",
        (f'%{keyword}%', min_price, category_id)
    ).fetchall()

-- ข้อที่ 4: Column masking view
CREATE VIEW orders_masked AS
SELECT 
    id,
    customer_id,
    status,
    CASE WHEN pg_has_role('finance_team', 'usage') THEN total_amount::TEXT ELSE '***' END AS total_amount,
    DATE(created_at) AS order_date  -- ไม่มี timestamp สำหรับ privacy
FROM orders;

-- ข้อที่ 5: GDPR anonymization
CREATE OR REPLACE FUNCTION gdpr_erase(p_id INTEGER)
RETURNS TEXT AS $$
BEGIN
    UPDATE customers SET
        first_name = 'DELETED',
        last_name = 'USER',
        email = 'deleted_' || p_id || '@void.invalid',
        phone = NULL,
        address = NULL,
        is_anonymized = TRUE,
        anonymized_at = NOW()
    WHERE id = p_id;
    RETURN 'Customer ' || p_id || ' anonymized';
END;
$$ LANGUAGE plpgsql;

-- ข้อที่ 6: Least privilege roles
CREATE ROLE reporting_service;
GRANT CONNECT ON DATABASE myapp TO reporting_service;
GRANT USAGE ON SCHEMA reports TO reporting_service;
GRANT SELECT ON ALL TABLES IN SCHEMA reports TO reporting_service;
-- No DML, no other schemas!

-- ข้อที่ 7: pgcrypto encryption
-- Encrypt
INSERT INTO secure_data (customer_id, ssn_encrypted)
VALUES (42, pgp_sym_encrypt('123-45-6789', 'encryption_key'));

-- Decrypt
SELECT pgp_sym_decrypt(ssn_encrypted, 'encryption_key') AS ssn
FROM secure_data WHERE customer_id = 42;

-- ข้อที่ 8: Connection audit
CREATE TABLE connection_log (
    id BIGSERIAL PRIMARY KEY,
    username TEXT,
    ip_address INET,
    connected_at TIMESTAMPTZ DEFAULT NOW(),
    disconnected_at TIMESTAMPTZ,
    queries_executed INTEGER DEFAULT 0
);

-- ข้อที่ 9: SQL injection test
-- These should all fail safely:
-- keyword = "'; DROP TABLE users; --"  → parameterized handles it
-- keyword = "' OR '1'='1"              → parameterized handles it
-- All safe with %s parameterized queries!

-- ข้อที่ 10: Security audit query
SELECT 
    u.username,
    STRING_AGG(DISTINCT r.rolname, ', ') AS roles,
    u.valuntil AS password_expires,
    u.connlimit AS connection_limit
FROM pg_user u
LEFT JOIN pg_auth_members am ON u.usesysid = am.member
LEFT JOIN pg_roles r ON am.roleid = r.oid
WHERE u.usename NOT LIKE 'pg_%'
GROUP BY u.username, u.valuntil, u.connlimit
ORDER BY u.username;
```

---

## สรุป

บทนี้ครอบคลุม Database Security อย่างครบถ้วน:

1. **Authentication/Authorization** - Users, Roles, RBAC
2. **Row Level Security** - Multi-tenant isolation, user-based access
3. **Column-Level Security** - Masking, views, column grants
4. **Data Masking** - Dynamic masking, role-based reveal
5. **SQL Injection Prevention** - Parameterized queries, input validation, whitelists
6. **Audit Logging** - Triggers, change tracking
7. **Encryption** - pgcrypto, column encryption, key rotation
8. **GDPR** - Right to erasure, data portability, retention policies
9. **Database Firewall** - Connection limits, time restrictions, anomaly detection
10. **Least Privilege** - Microservice isolation

Security ที่ดีต้องใช้ defense in depth - ไม่พึ่งพาแค่ layer เดียว แต่ต้องมีทุก layer ทำงานร่วมกัน
