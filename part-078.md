# Part 78: Optimistic vs Pessimistic Locking

## บทนำ

เมื่อหลาย users ต้องการแก้ไขข้อมูลเดียวกัน เราต้องตัดสินใจว่าจะจัดการ concurrency อย่างไร มีสองแนวทางหลัก:

- **Pessimistic Locking**: "ฉันสงสัยว่าคนอื่นจะมาขัดขวาง → ล็อกก่อน"
- **Optimistic Locking**: "น่าจะไม่มีปัญหา → ทำแล้วค่อยตรวจสอบ"

---

## 1. Optimistic Locking Concept

Optimistic Locking ตั้งอยู่บนสมมติฐานว่า conflicts ไม่ค่อยเกิดขึ้น ดังนั้นแทนที่จะ lock ข้อมูลตั้งแต่ต้น เราทำงานกับข้อมูลได้เลย แล้วค่อย **ตรวจสอบตอน commit** ว่ามีคนอื่นแก้ไขระหว่างนั้นหรือไม่

```
Optimistic Locking Flow:

1. READ:   อ่านข้อมูลพร้อม version/timestamp
2. MODIFY: แก้ไขข้อมูล (ไม่มี lock)
3. WRITE:  ตรวจสอบว่า version ยังตรงกัน แล้วค่อย UPDATE
4. หาก version เปลี่ยน → Conflict! → Retry หรือ Error

เปรียบเทียบ:
- Low contention environment → Optimistic ดีกว่า
- High contention environment → Pessimistic ดีกว่า
```

---

## 2. Version Column Pattern

วิธีที่ใช้บ่อยที่สุด: เพิ่ม `version` column ที่ increment ทุกครั้งที่ UPDATE

```sql
-- สร้างตารางที่รองรับ Optimistic Locking
CREATE TABLE documents (
    doc_id      SERIAL PRIMARY KEY,
    title       VARCHAR(200) NOT NULL,
    content     TEXT,
    version     INT NOT NULL DEFAULT 1,
    created_by  INT NOT NULL,
    updated_by  INT,
    created_at  TIMESTAMPTZ DEFAULT NOW(),
    updated_at  TIMESTAMPTZ DEFAULT NOW()
);

-- ใส่ข้อมูลทดสอบ
INSERT INTO documents (title, content, created_by) 
VALUES ('Project Plan', 'Initial draft content...', 1);

-- ตัวอย่างที่ 1: Basic Optimistic Lock Update
-- Step 1: อ่านข้อมูล
SELECT doc_id, title, content, version FROM documents WHERE doc_id = 1;
-- ได้: version = 1

-- Step 2: แก้ไขและ save (ใช้ WHERE version = old_version)
UPDATE documents 
SET 
    content = 'Updated content by User A',
    version = version + 1,
    updated_by = 101,
    updated_at = NOW()
WHERE doc_id = 1 
  AND version = 1;  -- ← Optimistic Lock Check!

-- ตรวจสอบ: ถ้า rowcount = 0 → version เปลี่ยนไปแล้ว = CONFLICT!
-- ถ้า rowcount = 1 → สำเร็จ

-- ตัวอย่างที่ 2: สาธิต conflict
-- User A และ User B อ่าน document เดียวกัน (version = 1)

-- User A update ก่อน:
UPDATE documents
SET content = 'User A changes', version = version + 1
WHERE doc_id = 1 AND version = 1;
-- UPDATED: 1 row, version เป็น 2 แล้ว

-- User B พยายาม update (ยังคิดว่า version = 1):
UPDATE documents
SET content = 'User B changes', version = version + 1  
WHERE doc_id = 1 AND version = 1;  -- version = 1 ไม่มีอีกแล้ว!
-- UPDATED: 0 rows → CONFLICT DETECTED!

-- Function สำหรับ Optimistic Update
CREATE OR REPLACE FUNCTION update_document(
    p_doc_id     INT,
    p_version    INT,
    p_content    TEXT,
    p_updated_by INT
) RETURNS JSONB AS $$
DECLARE
    v_rows_affected INT;
    v_new_version   INT;
BEGIN
    UPDATE documents
    SET 
        content    = p_content,
        version    = version + 1,
        updated_by = p_updated_by,
        updated_at = NOW()
    WHERE doc_id = p_doc_id 
      AND version = p_version;
    
    GET DIAGNOSTICS v_rows_affected = ROW_COUNT;
    
    IF v_rows_affected = 0 THEN
        -- ตรวจสอบว่า document มีอยู่จริง
        IF EXISTS (SELECT 1 FROM documents WHERE doc_id = p_doc_id) THEN
            RETURN jsonb_build_object(
                'success', false,
                'error', 'Conflict: Document was modified by another user',
                'error_code', 'OPTIMISTIC_LOCK_CONFLICT',
                'current_version', (SELECT version FROM documents WHERE doc_id = p_doc_id)
            );
        ELSE
            RETURN jsonb_build_object(
                'success', false,
                'error', 'Document not found',
                'error_code', 'NOT_FOUND'
            );
        END IF;
    END IF;
    
    SELECT version INTO v_new_version FROM documents WHERE doc_id = p_doc_id;
    
    RETURN jsonb_build_object(
        'success', true,
        'doc_id', p_doc_id,
        'new_version', v_new_version,
        'message', 'Document updated successfully'
    );
END;
$$ LANGUAGE plpgsql;

-- ทดสอบ
BEGIN;
SELECT update_document(1, 2, 'New content from User A', 101);
COMMIT;

-- Conflict test
BEGIN;
SELECT update_document(1, 2, 'Conflicting content from User B', 102);
-- จะ error เพราะ version = 2 ไม่มีแล้ว (เป็น 3 หลังจาก User A update)
COMMIT;
```

---

## 3. Timestamp-Based Optimistic Locking

แทนที่จะใช้ version number ใช้ timestamp ของการแก้ไขครั้งล่าสุด

```sql
-- Timestamp-based approach
CREATE TABLE products_ts (
    product_id   SERIAL PRIMARY KEY,
    name         VARCHAR(200) NOT NULL,
    price        DECIMAL(10,2) NOT NULL,
    stock        INT NOT NULL DEFAULT 0,
    last_modified TIMESTAMPTZ DEFAULT NOW()
);

INSERT INTO products_ts (name, price, stock) VALUES
    ('Widget', 9.99, 100),
    ('Gadget', 19.99, 50);

-- Update ด้วย timestamp check
CREATE OR REPLACE FUNCTION update_product_price(
    p_product_id   INT,
    p_last_modified TIMESTAMPTZ,
    p_new_price    DECIMAL
) RETURNS JSONB AS $$
DECLARE
    v_rows INT;
    v_actual_modified TIMESTAMPTZ;
BEGIN
    UPDATE products_ts
    SET price = p_new_price, last_modified = NOW()
    WHERE product_id = p_product_id
      AND last_modified = p_last_modified;  -- Timestamp lock check
    
    GET DIAGNOSTICS v_rows = ROW_COUNT;
    
    IF v_rows = 0 THEN
        SELECT last_modified INTO v_actual_modified 
        FROM products_ts WHERE product_id = p_product_id;
        
        RETURN jsonb_build_object(
            'success', false,
            'error', 'Stale data: product was modified since you loaded it',
            'your_timestamp', p_last_modified,
            'current_timestamp', v_actual_modified
        );
    END IF;
    
    RETURN jsonb_build_object('success', true, 'new_price', p_new_price);
END;
$$ LANGUAGE plpgsql;

-- ทดสอบ
-- อ่านข้อมูล
SELECT product_id, name, price, last_modified FROM products_ts WHERE product_id = 1;

BEGIN;
-- Update โดยใช้ timestamp ที่อ่านมา
SELECT update_product_price(1, NOW() - INTERVAL '1 hour', 12.99);
-- จะ fail เพราะ timestamp ไม่ตรง
COMMIT;

BEGIN;
SELECT update_product_price(
    1, 
    (SELECT last_modified FROM products_ts WHERE product_id = 1),
    12.99
);
-- จะสำเร็จเพราะ timestamp ตรง
COMMIT;

-- ข้อควรระวัง: Timestamp precision อาจทำให้เกิด false conflicts
-- ใน PostgreSQL: TIMESTAMPTZ มี microsecond precision → ดี
-- แต่ถ้า clock skew หรือ concurrent writes ในเวลา < 1 microsecond → ปัญหาได้
```

---

## 4. Hash/Checksum-Based Optimistic Locking

อีกวิธีหนึ่งคือใช้ hash ของ data เพื่อ detect changes

```sql
-- Hash-based approach
CREATE TABLE settings (
    setting_id  SERIAL PRIMARY KEY,
    category    VARCHAR(100),
    key         VARCHAR(100),
    value       TEXT,
    data_hash   TEXT GENERATED ALWAYS AS (
                    md5(category || key || COALESCE(value, ''))
                ) STORED,
    UNIQUE(category, key)
);

INSERT INTO settings (category, key, value) VALUES
    ('email', 'smtp_host', 'mail.example.com'),
    ('email', 'smtp_port', '587');

-- Update with hash check
CREATE OR REPLACE FUNCTION update_setting(
    p_setting_id INT,
    p_expected_hash TEXT,
    p_new_value TEXT
) RETURNS JSONB AS $$
DECLARE
    v_rows INT;
BEGIN
    UPDATE settings
    SET value = p_new_value
    WHERE setting_id = p_setting_id
      AND data_hash = p_expected_hash;
    
    GET DIAGNOSTICS v_rows = ROW_COUNT;
    
    IF v_rows = 0 THEN
        RETURN jsonb_build_object(
            'success', false,
            'error', 'Setting was changed by another user (hash mismatch)'
        );
    END IF;
    
    RETURN jsonb_build_object('success', true, 'updated_value', p_new_value);
END;
$$ LANGUAGE plpgsql;

-- อ่านพร้อม hash
SELECT setting_id, category, key, value, data_hash FROM settings WHERE setting_id = 1;

-- Update โดยใช้ hash
BEGIN;
SELECT update_setting(1, 
    (SELECT data_hash FROM settings WHERE setting_id = 1),
    'new-mail.example.com');
COMMIT;
```

---

## 5. Application Implementation Patterns

### Pattern 1: CAS (Compare-And-Swap)

```sql
-- CAS (Compare And Swap) pattern ใน SQL
CREATE OR REPLACE FUNCTION cas_update(
    p_table      TEXT,
    p_id_col     TEXT,
    p_id_val     INT,
    p_version_col TEXT,
    p_version_val INT,
    p_set_clause TEXT
) RETURNS BOOLEAN AS $$
DECLARE
    v_rows INT;
BEGIN
    EXECUTE format(
        'UPDATE %I SET %s, %I = %I + 1, updated_at = NOW() 
         WHERE %I = $1 AND %I = $2',
        p_table, p_set_clause, p_version_col, p_version_col,
        p_id_col, p_version_col
    ) USING p_id_val, p_version_val;
    
    GET DIAGNOSTICS v_rows = ROW_COUNT;
    RETURN v_rows = 1;
END;
$$ LANGUAGE plpgsql;

-- ใช้งาน
BEGIN;
SELECT cas_update('documents', 'doc_id', 1, 'version', 3, 'content = ''CAS updated content''');
COMMIT;
```

### Pattern 2: Retry Loop

```sql
-- Optimistic lock ด้วย retry
CREATE OR REPLACE FUNCTION optimistic_update_with_retry(
    p_doc_id     INT,
    p_new_content TEXT,
    p_user_id    INT,
    p_max_retries INT DEFAULT 5
) RETURNS JSONB AS $$
DECLARE
    v_version   INT;
    v_attempt   INT := 0;
    v_rows      INT;
    v_delay_ms  INT;
BEGIN
    LOOP
        v_attempt := v_attempt + 1;
        
        -- อ่าน current version
        SELECT version INTO v_version FROM documents WHERE doc_id = p_doc_id;
        
        IF v_version IS NULL THEN
            RETURN jsonb_build_object('success', false, 'error', 'Document not found');
        END IF;
        
        -- Attempt update
        UPDATE documents
        SET content = p_new_content, 
            version = version + 1,
            updated_by = p_user_id,
            updated_at = NOW()
        WHERE doc_id = p_doc_id AND version = v_version;
        
        GET DIAGNOSTICS v_rows = ROW_COUNT;
        
        IF v_rows = 1 THEN
            RETURN jsonb_build_object(
                'success', true,
                'attempts', v_attempt,
                'final_version', v_version + 1
            );
        END IF;
        
        -- Conflict: another update happened
        IF v_attempt >= p_max_retries THEN
            RETURN jsonb_build_object(
                'success', false,
                'error', format('Could not update after %s attempts', v_attempt),
                'error_code', 'MAX_RETRIES_EXCEEDED'
            );
        END IF;
        
        -- Backoff with jitter
        v_delay_ms := (random() * 100 + 10)::INT * v_attempt;
        RAISE NOTICE 'Conflict on attempt %, waiting %ms', v_attempt, v_delay_ms;
        PERFORM pg_sleep(v_delay_ms / 1000.0);
    END LOOP;
END;
$$ LANGUAGE plpgsql;

-- ทดสอบ
BEGIN;
SELECT optimistic_update_with_retry(1, 'Retry pattern content', 101, 3);
COMMIT;
```

### Pattern 3: Merge (Last Write Wins vs First Write Wins)

```sql
-- Last Write Wins: ข้อมูลล่าสุดชนะเสมอ (ง่ายแต่อาจสูญข้อมูล)
CREATE OR REPLACE FUNCTION last_write_wins_update(
    p_doc_id   INT,
    p_content  TEXT,
    p_user_id  INT
) RETURNS VOID AS $$
BEGIN
    UPDATE documents
    SET content = p_content, version = version + 1, updated_by = p_user_id, updated_at = NOW()
    WHERE doc_id = p_doc_id;
    -- ไม่ตรวจ version → last writer wins
END;
$$ LANGUAGE plpgsql;

-- First Write Wins: คนแรกสำเร็จ คนหลังต้อง retry
CREATE OR REPLACE FUNCTION first_write_wins_update(
    p_doc_id  INT,
    p_version INT,  -- version ที่อ่านมา
    p_content TEXT,
    p_user_id INT
) RETURNS BOOLEAN AS $$
DECLARE
    v_rows INT;
BEGIN
    UPDATE documents
    SET content = p_content, version = version + 1, updated_by = p_user_id, updated_at = NOW()
    WHERE doc_id = p_doc_id AND version = p_version;
    
    GET DIAGNOSTICS v_rows = ROW_COUNT;
    RETURN v_rows = 1;  -- TRUE = success, FALSE = conflict (someone else wrote first)
END;
$$ LANGUAGE plpgsql;

-- Merge Strategy: รวม changes
CREATE TABLE document_history (
    history_id  SERIAL PRIMARY KEY,
    doc_id      INT REFERENCES documents(doc_id),
    version     INT NOT NULL,
    content     TEXT,
    changed_by  INT,
    changed_at  TIMESTAMPTZ DEFAULT NOW()
);

CREATE OR REPLACE FUNCTION merge_update(
    p_doc_id      INT,
    p_base_version INT,  -- version ที่ user เริ่มต้นอ่าน
    p_new_content  TEXT,
    p_user_id      INT
) RETURNS JSONB AS $$
DECLARE
    v_current_version INT;
    v_rows INT;
BEGIN
    SELECT version INTO v_current_version FROM documents WHERE doc_id = p_doc_id;
    
    IF v_current_version = p_base_version THEN
        -- ไม่มี conflict → direct update
        UPDATE documents
        SET content = p_new_content, version = version + 1, 
            updated_by = p_user_id, updated_at = NOW()
        WHERE doc_id = p_doc_id;
        
        RETURN jsonb_build_object('success', true, 'strategy', 'direct_update');
    ELSE
        -- Conflict → บันทึก history และ resolve
        -- บันทึก user's changes ใน history
        INSERT INTO document_history (doc_id, version, content, changed_by)
        VALUES (p_doc_id, p_base_version + 1, p_new_content, p_user_id);
        
        RETURN jsonb_build_object(
            'success', false,
            'strategy', 'needs_merge',
            'your_changes_saved', true,
            'current_version', v_current_version,
            'message', 'Conflict detected. Your changes saved in history for manual merge.'
        );
    END IF;
END;
$$ LANGUAGE plpgsql;
```

---

## 6. Pessimistic Locking (SELECT FOR UPDATE)

Pessimistic Locking ตั้งอยู่บนสมมติฐานว่า conflicts จะเกิดขึ้น ดังนั้น Lock ก่อนทำงาน

```sql
-- Basic Pessimistic Lock
BEGIN;
    SELECT doc_id, title, content, version 
    FROM documents 
    WHERE doc_id = 1 
    FOR UPDATE;  -- Lock row ทันที!
    
    -- ไม่มีใคร update row นี้ได้จนกว่าจะ COMMIT
    UPDATE documents
    SET content = 'Pessimistic update', version = version + 1
    WHERE doc_id = 1;
COMMIT;

-- Pessimistic Lock ด้วย Function
CREATE OR REPLACE FUNCTION pessimistic_update_document(
    p_doc_id    INT,
    p_content   TEXT,
    p_user_id   INT
) RETURNS JSONB AS $$
DECLARE
    v_doc RECORD;
BEGIN
    -- Lock document ก่อน
    SELECT doc_id, title, content, version 
    INTO v_doc
    FROM documents 
    WHERE doc_id = p_doc_id
    FOR UPDATE;  -- Exclusive lock
    
    IF NOT FOUND THEN
        RETURN jsonb_build_object('success', false, 'error', 'Document not found');
    END IF;
    
    -- ตอนนี้ row ถูก lock → ทำงานได้อย่างปลอดภัย
    UPDATE documents
    SET content = p_content,
        version = version + 1,
        updated_by = p_user_id,
        updated_at = NOW()
    WHERE doc_id = p_doc_id;
    
    RETURN jsonb_build_object(
        'success', true,
        'old_version', v_doc.version,
        'new_version', v_doc.version + 1
    );
END;
$$ LANGUAGE plpgsql;

-- Pessimistic Lock สำหรับ inventory
CREATE OR REPLACE FUNCTION reserve_stock_pessimistic(
    p_product_id INT,
    p_quantity   INT,
    p_order_id   INT
) RETURNS JSONB AS $$
DECLARE
    v_available INT;
BEGIN
    -- Lock product row ก่อน
    SELECT stock - COALESCE(reserved, 0) INTO v_available
    FROM inventory
    WHERE product_id = p_product_id
    FOR UPDATE;  -- No one else can reserve simultaneously!
    
    IF v_available < p_quantity THEN
        RETURN jsonb_build_object(
            'success', false,
            'error', 'Insufficient stock',
            'available', v_available,
            'requested', p_quantity
        );
    END IF;
    
    UPDATE inventory 
    SET reserved = reserved + p_quantity
    WHERE product_id = p_product_id;
    
    INSERT INTO reservations (product_id, order_id, quantity, reserved_at)
    VALUES (p_product_id, p_order_id, p_quantity, NOW());
    
    RETURN jsonb_build_object(
        'success', true,
        'reserved', p_quantity,
        'remaining', v_available - p_quantity
    );
END;
$$ LANGUAGE plpgsql;
```

---

## 7. Choosing Between Optimistic and Pessimistic

```sql
-- Decision Matrix
/*
Use Optimistic When:
✓ Low contention (conflicts rare)
✓ Read-heavy workloads
✓ Long read operations before update
✓ Cannot afford lock hold time
✓ Distributed systems (microservices)
✓ Retry cost is low
Example: Blog posts, user profiles, product catalog

Use Pessimistic When:
✓ High contention (conflicts frequent)
✓ Write-heavy workloads
✓ Updates are expensive to redo
✓ Short lock hold time acceptable
✓ Single database, tight consistency needed
✓ Banking, inventory systems
Example: Bank transfers, ticket booking, stock updates
*/

-- Profiling contention สำหรับตัดสินใจ
CREATE OR REPLACE FUNCTION measure_contention(
    p_table_name TEXT,
    p_duration   INTERVAL DEFAULT '5 minutes'
) RETURNS TABLE(
    metric      TEXT,
    value       NUMERIC,
    suggestion  TEXT
) AS $$
DECLARE
    v_updates  BIGINT;
    v_conflicts BIGINT;
    v_hot_rate NUMERIC;
BEGIN
    -- อ่าน stats ปัจจุบัน
    SELECT n_tup_upd, n_tup_hot_upd 
    INTO v_updates, v_hot_rate
    FROM pg_stat_user_tables 
    WHERE tablename = p_table_name;
    
    metric     := 'Total Updates';
    value      := v_updates;
    suggestion := 'High = high contention possible';
    RETURN NEXT;
    
    metric     := 'HOT Update Rate (%)';
    value      := ROUND(v_hot_rate::NUMERIC / NULLIF(v_updates, 0) * 100, 2);
    suggestion := CASE 
        WHEN v_hot_rate::NUMERIC / NULLIF(v_updates, 0) > 0.7 THEN 'Use optimistic locking'
        WHEN v_hot_rate::NUMERIC / NULLIF(v_updates, 0) < 0.3 THEN 'Consider pessimistic locking'
        ELSE 'Either approach may work'
    END;
    RETURN NEXT;
    
    metric     := 'Lock Waits';
    value      := (SELECT COUNT(*) FROM pg_locks 
                   WHERE NOT granted AND relation = p_table_name::regclass);
    suggestion := CASE 
        WHEN (SELECT COUNT(*) FROM pg_locks WHERE NOT granted AND relation = p_table_name::regclass) > 10
        THEN 'High lock waits - consider optimistic locking or redesign'
        ELSE 'Lock waits normal'
    END;
    RETURN NEXT;
END;
$$ LANGUAGE plpgsql;

BEGIN;
SELECT * FROM measure_contention('documents');
COMMIT;
```

---

## 8. Stale Data Detection

```sql
-- ตรวจจับและจัดการ stale data อย่างมีประสิทธิภาพ

-- Pattern 1: Version-based Stale Check
CREATE OR REPLACE FUNCTION is_stale(
    p_doc_id    INT,
    p_my_version INT
) RETURNS BOOLEAN AS $$
DECLARE
    v_current_version INT;
BEGIN
    SELECT version INTO v_current_version FROM documents WHERE doc_id = p_doc_id;
    RETURN v_current_version != p_my_version;
END;
$$ LANGUAGE plpgsql;

-- Pattern 2: Timestamp-based Stale Check
CREATE OR REPLACE FUNCTION has_changed_since(
    p_doc_id      INT,
    p_my_timestamp TIMESTAMPTZ
) RETURNS BOOLEAN AS $$
DECLARE
    v_last_modified TIMESTAMPTZ;
BEGIN
    SELECT updated_at INTO v_last_modified FROM documents WHERE doc_id = p_doc_id;
    RETURN v_last_modified > p_my_timestamp;
END;
$$ LANGUAGE plpgsql;

-- Pattern 3: Comprehensive Stale Detection
CREATE OR REPLACE FUNCTION check_document_freshness(
    p_doc_id      INT,
    p_my_version  INT,
    p_my_ts       TIMESTAMPTZ
) RETURNS JSONB AS $$
DECLARE
    v_doc RECORD;
BEGIN
    SELECT doc_id, version, updated_at, updated_by
    INTO v_doc
    FROM documents
    WHERE doc_id = p_doc_id;
    
    IF NOT FOUND THEN
        RETURN jsonb_build_object('fresh', false, 'reason', 'Document deleted');
    END IF;
    
    IF v_doc.version != p_my_version THEN
        RETURN jsonb_build_object(
            'fresh', false,
            'reason', 'Version mismatch',
            'your_version', p_my_version,
            'current_version', v_doc.version,
            'last_modified', v_doc.updated_at,
            'modified_by', v_doc.updated_by,
            'time_since_your_read', NOW() - p_my_ts
        );
    END IF;
    
    RETURN jsonb_build_object(
        'fresh', true,
        'version', v_doc.version
    );
END;
$$ LANGUAGE plpgsql;

-- ใช้ใน application:
DO $$
DECLARE
    v_read_time TIMESTAMPTZ := NOW();
    v_version   INT;
    v_freshness JSONB;
BEGIN
    -- อ่านข้อมูล
    SELECT version INTO v_version FROM documents WHERE doc_id = 1;
    
    -- ทำงานบางอย่าง (อาจใช้เวลา)
    PERFORM pg_sleep(0.001);
    
    -- ตรวจสอบก่อน write
    v_freshness := check_document_freshness(1, v_version, v_read_time);
    
    IF (v_freshness->>'fresh')::BOOLEAN THEN
        -- อัปเดตได้
        UPDATE documents SET content = 'Safe update' WHERE doc_id = 1;
        RAISE NOTICE 'Update successful';
    ELSE
        RAISE NOTICE 'Stale data detected: %', v_freshness;
    END IF;
END;
$$;
```

---

## 9. Complete Optimistic Lock Implementation

```sql
-- สร้าง full implementation ที่ใช้ใน production

-- ตาราง base ที่รองรับ optimistic locking
CREATE TABLE ol_base_table (
    id          BIGSERIAL PRIMARY KEY,
    data        JSONB NOT NULL DEFAULT '{}',
    version     BIGINT NOT NULL DEFAULT 1,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    created_by  INT,
    updated_by  INT,
    is_deleted  BOOLEAN NOT NULL DEFAULT FALSE
);

-- ตาราง conflict log
CREATE TABLE ol_conflicts (
    conflict_id  SERIAL PRIMARY KEY,
    table_name   TEXT,
    record_id    BIGINT,
    user_id      INT,
    expected_version BIGINT,
    actual_version   BIGINT,
    occurred_at  TIMESTAMPTZ DEFAULT NOW()
);

-- Generic optimistic update function
CREATE OR REPLACE FUNCTION optimistic_update(
    p_table_name TEXT,
    p_id         BIGINT,
    p_version    BIGINT,
    p_data       JSONB,
    p_user_id    INT DEFAULT NULL
) RETURNS JSONB AS $$
DECLARE
    v_rows       INT;
    v_new_ver    BIGINT;
    v_cur_ver    BIGINT;
BEGIN
    EXECUTE format(
        'UPDATE %I SET data = $1, version = version + 1, updated_at = NOW(), updated_by = $2
         WHERE id = $3 AND version = $4 AND is_deleted = FALSE',
        p_table_name
    ) USING p_data, p_user_id, p_id, p_version;
    
    GET DIAGNOSTICS v_rows = ROW_COUNT;
    
    IF v_rows = 0 THEN
        -- Detect reason for failure
        EXECUTE format('SELECT version FROM %I WHERE id = $1', p_table_name)
        INTO v_cur_ver USING p_id;
        
        -- Log conflict
        INSERT INTO ol_conflicts (table_name, record_id, user_id, expected_version, actual_version)
        VALUES (p_table_name, p_id, p_user_id, p_version, v_cur_ver);
        
        IF v_cur_ver IS NULL THEN
            RETURN jsonb_build_object('success', false, 'error', 'Record not found or deleted');
        ELSE
            RETURN jsonb_build_object(
                'success', false,
                'error', 'Optimistic lock conflict',
                'expected_version', p_version,
                'current_version', v_cur_ver
            );
        END IF;
    END IF;
    
    -- Get new version
    EXECUTE format('SELECT version FROM %I WHERE id = $1', p_table_name)
    INTO v_new_ver USING p_id;
    
    RETURN jsonb_build_object(
        'success', true,
        'id', p_id,
        'new_version', v_new_ver
    );
END;
$$ LANGUAGE plpgsql;

-- ดู conflict statistics
CREATE OR REPLACE VIEW ol_conflict_summary AS
SELECT 
    table_name,
    COUNT(*) AS total_conflicts,
    COUNT(DISTINCT user_id) AS affected_users,
    MAX(occurred_at) AS last_conflict,
    AVG(actual_version - expected_version) AS avg_version_gap
FROM ol_conflicts
WHERE occurred_at > NOW() - INTERVAL '24 hours'
GROUP BY table_name
ORDER BY total_conflicts DESC;

SELECT * FROM ol_conflict_summary;
```

---

## 10. Optimistic Locking ใน Distributed Systems

```sql
-- ใน microservices: แต่ละ service มี database แยก
-- Optimistic locking ช่วยได้แม้ข้ามระบบ

-- Pattern: ETag-style locking (HTTP-inspired)
CREATE TABLE api_resources (
    resource_id  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    resource_type VARCHAR(100) NOT NULL,
    data         JSONB NOT NULL,
    etag         TEXT GENERATED ALWAYS AS (
                     md5(data::TEXT || resource_id::TEXT)
                 ) STORED,
    version      BIGINT NOT NULL DEFAULT 1,
    updated_at   TIMESTAMPTZ DEFAULT NOW()
);

-- Update with ETag check (HTTP If-Match header equivalent)
CREATE OR REPLACE FUNCTION update_resource(
    p_resource_id UUID,
    p_etag       TEXT,   -- ETag from previous GET request
    p_new_data   JSONB
) RETURNS JSONB AS $$
DECLARE
    v_rows INT;
    v_new_etag TEXT;
BEGIN
    UPDATE api_resources
    SET data = p_new_data, version = version + 1, updated_at = NOW()
    WHERE resource_id = p_resource_id AND etag = p_etag;
    
    GET DIAGNOSTICS v_rows = ROW_COUNT;
    
    IF v_rows = 0 THEN
        RETURN jsonb_build_object(
            'status', 412,  -- 412 Precondition Failed (HTTP)
            'error', 'ETag mismatch - resource was modified'
        );
    END IF;
    
    SELECT etag INTO v_new_etag FROM api_resources WHERE resource_id = p_resource_id;
    
    RETURN jsonb_build_object(
        'status', 200,
        'new_etag', v_new_etag,
        'resource_id', p_resource_id
    );
END;
$$ LANGUAGE plpgsql;
```

---

## 11. Version Column ใน ORM Frameworks

```sql
-- Hibernate/JPA (Java): @Version annotation
-- SQLAlchemy (Python): version_id_col
-- ActiveRecord (Ruby): lock_version column
-- Entity Framework (.NET): RowVersion/Timestamp

-- จำลอง ORM-style versioning ใน SQL
CREATE TABLE orm_entities (
    id            SERIAL PRIMARY KEY,
    name          TEXT NOT NULL,
    data          JSONB,
    lock_version  INT NOT NULL DEFAULT 0  -- Hibernate-style
);

-- Hibernate จะ generate SQL ประมาณนี้:
-- UPDATE orm_entities SET name=?, data=?, lock_version=lock_version+1
-- WHERE id=? AND lock_version=?

-- ถ้า rowcount = 0 → throws OptimisticLockException

-- Python SQLAlchemy equivalent:
/*
class Document(Base):
    __tablename__ = 'documents'
    id = Column(Integer, primary_key=True)
    content = Column(Text)
    version_id = Column(Integer, nullable=False)
    __mapper_args__ = {"version_id_col": version_id}
*/

-- สร้าง trigger ที่ auto-increment version (สำหรับ ORM ที่ไม่ manage version เอง)
CREATE OR REPLACE FUNCTION auto_increment_version()
RETURNS TRIGGER AS $$
BEGIN
    IF TG_OP = 'UPDATE' THEN
        NEW.lock_version := OLD.lock_version + 1;
        NEW.updated_at := NOW();
    END IF;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER tr_auto_version
    BEFORE UPDATE ON orm_entities
    FOR EACH ROW EXECUTE FUNCTION auto_increment_version();

-- ทดสอบ trigger
INSERT INTO orm_entities (name, data) VALUES ('Test Entity', '{"key": "value"}');

-- Update ครั้งแรก
BEGIN;
UPDATE orm_entities SET name = 'Updated Entity' WHERE id = 1 AND lock_version = 0;
-- lock_version จะถูก increment เป็น 1 โดย trigger
COMMIT;

SELECT id, name, lock_version FROM orm_entities;
-- lock_version = 1

-- Conflict test
BEGIN;
UPDATE orm_entities SET name = 'Another Update' WHERE id = 1 AND lock_version = 0;
-- 0 rows affected (version = 1 แล้ว)
ROLLBACK;
```

---

## 12. Performance Comparison

```sql
-- Benchmark: Optimistic vs Pessimistic ใน different scenarios

-- Setup
CREATE TABLE bench_items (
    id      SERIAL PRIMARY KEY,
    value   INT NOT NULL DEFAULT 0,
    version INT NOT NULL DEFAULT 1
);

INSERT INTO bench_items (value) SELECT generate_series(1, 1000);

-- Pessimistic benchmark
CREATE OR REPLACE FUNCTION bench_pessimistic(p_iterations INT DEFAULT 100)
RETURNS INTERVAL AS $$
DECLARE
    v_start TIMESTAMPTZ := clock_timestamp();
    i INT;
    v_id INT;
BEGIN
    FOR i IN 1..p_iterations LOOP
        v_id := (random() * 999 + 1)::INT;
        
        BEGIN
            SELECT * FROM bench_items WHERE id = v_id FOR UPDATE;
            UPDATE bench_items SET value = value + 1 WHERE id = v_id;
        EXCEPTION WHEN OTHERS THEN NULL;
        END;
    END LOOP;
    
    RETURN clock_timestamp() - v_start;
END;
$$ LANGUAGE plpgsql;

-- Optimistic benchmark
CREATE OR REPLACE FUNCTION bench_optimistic(p_iterations INT DEFAULT 100)
RETURNS TABLE(duration INTERVAL, conflicts INT) AS $$
DECLARE
    v_start TIMESTAMPTZ := clock_timestamp();
    i INT;
    v_id INT;
    v_version INT;
    v_rows INT;
    v_conflicts INT := 0;
BEGIN
    FOR i IN 1..p_iterations LOOP
        v_id := (random() * 999 + 1)::INT;
        
        -- Read
        SELECT version INTO v_version FROM bench_items WHERE id = v_id;
        
        -- Update with version check
        UPDATE bench_items 
        SET value = value + 1, version = version + 1
        WHERE id = v_id AND version = v_version;
        
        GET DIAGNOSTICS v_rows = ROW_COUNT;
        
        IF v_rows = 0 THEN
            v_conflicts := v_conflicts + 1;
        END IF;
    END LOOP;
    
    duration  := clock_timestamp() - v_start;
    conflicts := v_conflicts;
    RETURN NEXT;
END;
$$ LANGUAGE plpgsql;

BEGIN;
RAISE NOTICE 'Pessimistic duration: %', bench_pessimistic(1000);
COMMIT;

BEGIN;
SELECT * FROM bench_optimistic(1000);
COMMIT;
```

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: Basic Optimistic Lock

```sql
-- คำตอบ
CREATE TABLE ol_products (
    product_id  SERIAL PRIMARY KEY,
    name        VARCHAR(200),
    price       DECIMAL(10,2),
    version     INT NOT NULL DEFAULT 1,
    updated_at  TIMESTAMPTZ DEFAULT NOW()
);

INSERT INTO ol_products (name, price) VALUES ('Laptop', 35000), ('Phone', 15000);

-- Optimistic update function
CREATE OR REPLACE FUNCTION ol_update_price(
    p_product_id INT,
    p_version    INT,
    p_new_price  DECIMAL
) RETURNS TEXT AS $$
DECLARE
    v_rows INT;
BEGIN
    UPDATE ol_products
    SET price = p_new_price, version = version + 1, updated_at = NOW()
    WHERE product_id = p_product_id AND version = p_version;
    
    GET DIAGNOSTICS v_rows = ROW_COUNT;
    
    RETURN CASE v_rows 
        WHEN 1 THEN 'Success'
        ELSE 'Conflict: product was modified'
    END;
END;
$$ LANGUAGE plpgsql;

BEGIN;
SELECT ol_update_price(1, 1, 32000);  -- Should succeed
SELECT ol_update_price(1, 1, 30000);  -- Should conflict (version is now 2)
COMMIT;
```

### แบบฝึกหัดที่ 2: Version Conflict Resolution

```sql
-- คำตอบ
CREATE OR REPLACE FUNCTION resolve_conflict(
    p_doc_id      INT,
    p_my_version  INT,
    p_my_content  TEXT,
    p_strategy    TEXT DEFAULT 'fail'  -- 'fail', 'retry', 'overwrite'
) RETURNS JSONB AS $$
DECLARE
    v_current RECORD;
    v_rows    INT;
BEGIN
    SELECT version, content, updated_at INTO v_current
    FROM documents WHERE doc_id = p_doc_id;
    
    IF v_current.version = p_my_version THEN
        UPDATE documents SET content = p_my_content, version = version + 1
        WHERE doc_id = p_doc_id;
        RETURN jsonb_build_object('result', 'success', 'strategy', 'no_conflict');
    END IF;
    
    CASE p_strategy
        WHEN 'fail' THEN
            RETURN jsonb_build_object(
                'result', 'conflict',
                'current_version', v_current.version,
                'current_content', v_current.content
            );
        WHEN 'overwrite' THEN
            UPDATE documents SET content = p_my_content, version = version + 1
            WHERE doc_id = p_doc_id;
            RETURN jsonb_build_object('result', 'overwritten', 'version', v_current.version + 1);
        WHEN 'retry' THEN
            -- Return current data so client can re-read
            RETURN jsonb_build_object(
                'result', 'retry',
                'current_version', v_current.version,
                'current_content', v_current.content,
                'hint', 'Re-read and reapply your changes'
            );
    END CASE;
END;
$$ LANGUAGE plpgsql;
```

### แบบฝึกหัดที่ 3: Shopping Cart with Optimistic Lock

```sql
-- คำตอบ
CREATE TABLE shopping_carts (
    cart_id    SERIAL PRIMARY KEY,
    user_id    INT NOT NULL UNIQUE,
    items      JSONB NOT NULL DEFAULT '[]',
    version    INT NOT NULL DEFAULT 1,
    updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE OR REPLACE FUNCTION add_to_cart(
    p_user_id   INT,
    p_item_id   INT,
    p_quantity  INT
) RETURNS JSONB AS $$
DECLARE
    v_cart     RECORD;
    v_rows     INT;
    v_attempts INT := 0;
    v_new_items JSONB;
BEGIN
    LOOP
        v_attempts := v_attempts + 1;
        
        -- Read current cart
        SELECT cart_id, items, version INTO v_cart
        FROM shopping_carts WHERE user_id = p_user_id;
        
        IF NOT FOUND THEN
            -- Create cart if not exists
            INSERT INTO shopping_carts (user_id, items)
            VALUES (p_user_id, jsonb_build_array(
                jsonb_build_object('item_id', p_item_id, 'quantity', p_quantity)
            ));
            RETURN jsonb_build_object('success', true, 'created', true);
        END IF;
        
        -- Build new items list
        v_new_items := v_cart.items || jsonb_build_object('item_id', p_item_id, 'quantity', p_quantity);
        
        -- Optimistic update
        UPDATE shopping_carts
        SET items = v_new_items, version = version + 1, updated_at = NOW()
        WHERE cart_id = v_cart.cart_id AND version = v_cart.version;
        
        GET DIAGNOSTICS v_rows = ROW_COUNT;
        
        IF v_rows = 1 THEN
            RETURN jsonb_build_object('success', true, 'attempts', v_attempts);
        END IF;
        
        IF v_attempts >= 5 THEN
            RETURN jsonb_build_object('success', false, 'error', 'Too many conflicts');
        END IF;
        
        PERFORM pg_sleep(0.01 * v_attempts);  -- Backoff
    END LOOP;
END;
$$ LANGUAGE plpgsql;
```

### แบบฝึกหัดที่ 4: Compare Conflict Rates

```sql
-- คำตอบ
CREATE OR REPLACE FUNCTION simulate_concurrent_updates(
    p_num_users    INT DEFAULT 10,
    p_num_updates  INT DEFAULT 100,
    p_contention   FLOAT DEFAULT 0.3  -- 30% chance same item
) RETURNS TABLE(
    total_attempts INT,
    conflicts      INT,
    conflict_rate  NUMERIC
) AS $$
DECLARE
    i           INT;
    v_item_id   INT;
    v_version   INT;
    v_rows      INT;
    v_total     INT := 0;
    v_conflicts INT := 0;
BEGIN
    -- Create test data
    CREATE TEMP TABLE ol_sim (id INT PRIMARY KEY, value INT DEFAULT 0, version INT DEFAULT 1);
    INSERT INTO ol_sim SELECT generate_series(1, 10), 0, 1;
    
    FOR i IN 1..p_num_updates LOOP
        -- Simulate contention: 30% chance of picking same hot item
        IF random() < p_contention THEN
            v_item_id := 1;  -- Hot item
        ELSE
            v_item_id := (random() * 9 + 1)::INT;  -- Random item
        END IF;
        
        SELECT version INTO v_version FROM ol_sim WHERE id = v_item_id;
        
        PERFORM pg_sleep(0.0001);  -- Simulate processing time
        
        UPDATE ol_sim SET value = value + 1, version = version + 1
        WHERE id = v_item_id AND version = v_version;
        
        GET DIAGNOSTICS v_rows = ROW_COUNT;
        
        v_total := v_total + 1;
        IF v_rows = 0 THEN v_conflicts := v_conflicts + 1; END IF;
    END LOOP;
    
    DROP TABLE ol_sim;
    
    total_attempts := v_total;
    conflicts      := v_conflicts;
    conflict_rate  := ROUND(v_conflicts::NUMERIC / v_total * 100, 2);
    RETURN NEXT;
END;
$$ LANGUAGE plpgsql;

BEGIN;
SELECT * FROM simulate_concurrent_updates(10, 200, 0.1);  -- Low contention
SELECT * FROM simulate_concurrent_updates(10, 200, 0.5);  -- High contention
COMMIT;
```

### แบบฝึกหัดที่ 5: Pessimistic vs Optimistic Decision Helper

```sql
-- คำตอบ
CREATE OR REPLACE FUNCTION recommend_locking_strategy(
    p_table_name TEXT
) RETURNS TABLE(
    recommendation TEXT,
    reason        TEXT,
    metrics       JSONB
) AS $$
DECLARE
    v_updates    BIGINT;
    v_hot_pct    NUMERIC;
    v_seq_scans  BIGINT;
    v_idx_scans  BIGINT;
    v_lock_waits INT;
BEGIN
    SELECT n_tup_upd, 
           ROUND(n_tup_hot_upd::NUMERIC / NULLIF(n_tup_upd, 0) * 100, 2),
           seq_scan,
           idx_scan
    INTO v_updates, v_hot_pct, v_seq_scans, v_idx_scans
    FROM pg_stat_user_tables WHERE tablename = p_table_name;
    
    SELECT COUNT(*) INTO v_lock_waits
    FROM pg_locks WHERE NOT granted AND relation = p_table_name::regclass;
    
    IF v_lock_waits > 10 THEN
        recommendation := 'Optimistic Locking';
        reason := 'High lock contention detected';
    ELSIF v_hot_pct < 30 THEN
        recommendation := 'Review indexing and use Optimistic Locking';
        reason := 'Low HOT update rate suggests high contention or many indexed column updates';
    ELSIF v_updates > 100000 THEN
        recommendation := 'Pessimistic Locking with SKIP LOCKED';
        reason := 'High update volume - pessimistic is more reliable';
    ELSE
        recommendation := 'Either strategy works';
        reason := 'Normal contention levels';
    END IF;
    
    metrics := jsonb_build_object(
        'updates', v_updates,
        'hot_pct', v_hot_pct,
        'lock_waits', v_lock_waits
    );
    RETURN NEXT;
END;
$$ LANGUAGE plpgsql;

SELECT * FROM recommend_locking_strategy('documents');
```

### แบบฝึกหัดที่ 6: Multi-Entity Optimistic Lock

```sql
-- คำตอบ
CREATE OR REPLACE FUNCTION batch_optimistic_update(
    p_updates JSONB[]  -- [{"id": 1, "version": 1, "data": {...}}, ...]
) RETURNS JSONB AS $$
DECLARE
    v_update    JSONB;
    v_rows      INT;
    v_success   INT := 0;
    v_failed    INT := 0;
    v_errors    JSONB[] := '{}';
BEGIN
    FOREACH v_update IN ARRAY p_updates
    LOOP
        SAVEPOINT item_update;
        
        BEGIN
            UPDATE documents
            SET data = v_update->'data',
                version = version + 1,
                updated_at = NOW()
            WHERE doc_id = (v_update->>'id')::INT
              AND version = (v_update->>'version')::INT;
            
            GET DIAGNOSTICS v_rows = ROW_COUNT;
            
            IF v_rows = 1 THEN
                RELEASE SAVEPOINT item_update;
                v_success := v_success + 1;
            ELSE
                ROLLBACK TO SAVEPOINT item_update;
                RELEASE SAVEPOINT item_update;
                v_failed := v_failed + 1;
                v_errors := array_append(v_errors, jsonb_build_object(
                    'id', v_update->>'id',
                    'error', 'Version conflict'
                ));
            END IF;
        EXCEPTION WHEN OTHERS THEN
            ROLLBACK TO SAVEPOINT item_update;
            RELEASE SAVEPOINT item_update;
            v_failed := v_failed + 1;
            v_errors := array_append(v_errors, jsonb_build_object(
                'id', v_update->>'id',
                'error', SQLERRM
            ));
        END;
    END LOOP;
    
    RETURN jsonb_build_object(
        'success', v_success,
        'failed', v_failed,
        'errors', to_jsonb(v_errors)
    );
END;
$$ LANGUAGE plpgsql;
```

### แบบฝึกหัดที่ 7: Version History

```sql
-- คำตอบ
CREATE TABLE document_versions (
    version_id  SERIAL PRIMARY KEY,
    doc_id      INT NOT NULL,
    version     INT NOT NULL,
    content     TEXT,
    changed_by  INT,
    changed_at  TIMESTAMPTZ DEFAULT NOW(),
    UNIQUE(doc_id, version)
);

-- Trigger สำหรับ auto-save version history
CREATE OR REPLACE FUNCTION save_version_history()
RETURNS TRIGGER AS $$
BEGIN
    IF TG_OP = 'UPDATE' AND OLD.version != NEW.version THEN
        INSERT INTO document_versions (doc_id, version, content, changed_by)
        VALUES (OLD.doc_id, OLD.version, OLD.content, NEW.updated_by);
    END IF;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER tr_document_versions
AFTER UPDATE ON documents
FOR EACH ROW EXECUTE FUNCTION save_version_history();

-- ดู history
SELECT dv.version, dv.content, dv.changed_at
FROM document_versions dv
WHERE dv.doc_id = 1
ORDER BY dv.version DESC;

-- Restore to version
CREATE OR REPLACE FUNCTION restore_version(
    p_doc_id         INT,
    p_target_version INT,
    p_user_id        INT
) RETURNS JSONB AS $$
DECLARE
    v_old_content TEXT;
    v_cur_version INT;
BEGIN
    SELECT content INTO v_old_content FROM document_versions 
    WHERE doc_id = p_doc_id AND version = p_target_version;
    
    SELECT version INTO v_cur_version FROM documents WHERE doc_id = p_doc_id;
    
    IF v_old_content IS NULL THEN
        RETURN jsonb_build_object('success', false, 'error', 'Version not found');
    END IF;
    
    UPDATE documents 
    SET content = v_old_content, version = version + 1, 
        updated_by = p_user_id, updated_at = NOW()
    WHERE doc_id = p_doc_id;
    
    RETURN jsonb_build_object(
        'success', true,
        'restored_from', p_target_version,
        'new_version', v_cur_version + 1
    );
END;
$$ LANGUAGE plpgsql;
```

### แบบฝึกหัดที่ 8: Optimistic Lock for Distributed Use

```sql
-- คำตอบ
CREATE TABLE distributed_resources (
    resource_id   UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    resource_type VARCHAR(50),
    data          JSONB,
    version       BIGINT DEFAULT 1,
    etag          TEXT,
    last_updated  TIMESTAMPTZ DEFAULT NOW()
);

-- Auto-generate ETag
CREATE OR REPLACE FUNCTION generate_etag()
RETURNS TRIGGER AS $$
BEGIN
    NEW.etag := encode(sha256((NEW.data::TEXT || NEW.version::TEXT || 
                               NEW.resource_id::TEXT)::BYTEA), 'hex');
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER tr_etag
BEFORE INSERT OR UPDATE ON distributed_resources
FOR EACH ROW EXECUTE FUNCTION generate_etag();

-- RESTful-style update with ETag
CREATE OR REPLACE FUNCTION api_update_resource(
    p_resource_id UUID,
    p_if_match    TEXT,  -- ETag from GET response
    p_new_data    JSONB
) RETURNS TABLE(
    http_status INT,
    new_etag    TEXT,
    data        JSONB
) AS $$
DECLARE
    v_rows INT;
    v_new_etag TEXT;
BEGIN
    UPDATE distributed_resources
    SET data = p_new_data, version = version + 1, last_updated = NOW()
    WHERE resource_id = p_resource_id AND etag = p_if_match;
    
    GET DIAGNOSTICS v_rows = ROW_COUNT;
    
    IF v_rows = 0 THEN
        http_status := 412;  -- Precondition Failed
        new_etag    := NULL;
        data        := NULL;
        RETURN NEXT;
        RETURN;
    END IF;
    
    SELECT etag, distributed_resources.data INTO v_new_etag, data
    FROM distributed_resources WHERE resource_id = p_resource_id;
    
    http_status := 200;
    new_etag    := v_new_etag;
    RETURN NEXT;
END;
$$ LANGUAGE plpgsql;
```

### แบบฝึกหัดที่ 9: Conflict Analytics

```sql
-- คำตอบ
CREATE OR REPLACE VIEW conflict_analytics AS
SELECT 
    DATE_TRUNC('hour', occurred_at) AS hour,
    table_name,
    COUNT(*) AS total_conflicts,
    COUNT(DISTINCT user_id) AS affected_users,
    AVG(actual_version - expected_version) AS avg_version_gap,
    MAX(actual_version - expected_version) AS max_version_gap
FROM ol_conflicts
WHERE occurred_at > NOW() - INTERVAL '7 days'
GROUP BY DATE_TRUNC('hour', occurred_at), table_name
ORDER BY hour DESC, total_conflicts DESC;

-- Alert สำหรับ high conflict rate
CREATE OR REPLACE FUNCTION check_conflict_rate(
    p_window INTERVAL DEFAULT '15 minutes',
    p_threshold INT DEFAULT 10
) RETURNS TABLE(
    table_name    TEXT,
    conflict_count INT,
    alert_level   TEXT
) AS $$
BEGIN
    RETURN QUERY
    SELECT 
        oc.table_name,
        COUNT(*)::INT AS conflict_count,
        CASE 
            WHEN COUNT(*) > p_threshold * 3 THEN 'CRITICAL'
            WHEN COUNT(*) > p_threshold THEN 'WARNING'
            ELSE 'OK'
        END AS alert_level
    FROM ol_conflicts oc
    WHERE occurred_at > NOW() - p_window
    GROUP BY oc.table_name
    HAVING COUNT(*) > p_threshold / 2
    ORDER BY COUNT(*) DESC;
END;
$$ LANGUAGE plpgsql;
```

### แบบฝึกหัดที่ 10: Complete System Test

```sql
-- คำตอบ
CREATE OR REPLACE FUNCTION test_locking_strategies()
RETURNS TABLE(
    strategy    TEXT,
    test_name   TEXT,
    passed      BOOLEAN,
    details     TEXT
) AS $$
DECLARE
    v_result JSONB;
    v_doc_id INT;
BEGIN
    -- Setup
    INSERT INTO documents (title, content, created_by) VALUES ('Test Doc', 'Initial', 1)
    RETURNING doc_id INTO v_doc_id;
    
    -- Test 1: Optimistic success
    strategy  := 'Optimistic';
    test_name := 'Successful update';
    
    v_result := update_document(v_doc_id, 1, 'Updated content', 101);
    passed    := (v_result->>'success')::BOOLEAN;
    details   := v_result::TEXT;
    RETURN NEXT;
    
    -- Test 2: Optimistic conflict
    strategy  := 'Optimistic';
    test_name := 'Conflict detection';
    
    v_result := update_document(v_doc_id, 1, 'Conflicting content', 102);
    passed    := NOT (v_result->>'success')::BOOLEAN;  -- Should fail!
    details   := v_result::TEXT;
    RETURN NEXT;
    
    -- Test 3: Pessimistic success
    strategy  := 'Pessimistic';
    test_name := 'Successful lock and update';
    
    v_result := pessimistic_update_document(v_doc_id, 'Pessimistic update', 101);
    passed    := (v_result->>'success')::BOOLEAN;
    details   := v_result::TEXT;
    RETURN NEXT;
    
    -- Cleanup
    DELETE FROM documents WHERE doc_id = v_doc_id;
END;
$$ LANGUAGE plpgsql;

BEGIN;
SELECT * FROM test_locking_strategies();
COMMIT;
```

---

## สรุป

ใน Part นี้เราได้เรียนรู้:

1. **Optimistic Locking**:
   - ไม่ lock ข้อมูล → อ่าน แก้ไข แล้วค่อยตรวจ conflict
   - ใช้ `version` column หรือ timestamp
   - เหมาะกับ low contention environments
   - ต้องมี retry logic สำหรับ conflicts

2. **Pessimistic Locking**:
   - Lock ก่อนแก้ไข → `SELECT FOR UPDATE`
   - Guarantee ไม่มี conflict
   - เหมาะกับ high contention environments
   - ระวัง deadlocks

3. **Version Column Pattern**: เพิ่ม `version INT DEFAULT 1` ใน table

4. **Timestamp Pattern**: ใช้ `last_modified TIMESTAMPTZ`

5. **ETag Pattern**: สำหรับ REST APIs

6. **Retry Logic**: Exponential backoff สำหรับ conflict handling

7. **Conflict Analytics**: Monitor conflict rates เพื่อ optimize

ใน Part 79 เราจะเรียนรู้เรื่อง Connection Pooling ซึ่งเป็นสิ่งสำคัญสำหรับ production systems
