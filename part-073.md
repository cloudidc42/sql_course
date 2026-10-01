# Part 73: Savepoints

## บทนำ

**Savepoint** คือจุดตรวจสอบ (checkpoint) ภายใน transaction ที่ช่วยให้เราสามารถ rollback ได้เพียงบางส่วนของ transaction โดยไม่ต้อง rollback ทั้งหมด คิดว่ามันเหมือน "เซฟเกม" - ถ้าเกมแพ้หลังจากจุดบันทึก ก็โหลดจากจุดนั้น โดยไม่ต้องเริ่มใหม่ตั้งแต่ต้น

---

## 1. SAVEPOINT - Syntax พื้นฐาน

```sql
-- รูปแบบพื้นฐาน
BEGIN;
    -- ทำ operations บางอย่าง
    SAVEPOINT savepoint_name;
    -- ทำ operations เพิ่มเติม
    ROLLBACK TO SAVEPOINT savepoint_name;  -- กลับไปที่ savepoint
    -- หรือ
    RELEASE SAVEPOINT savepoint_name;       -- ลบ savepoint (ไม่ rollback)
COMMIT;

-- ตัวอย่างง่ายๆ
BEGIN;
    INSERT INTO accounts (name, balance) VALUES ('Alice', 1000);
    SAVEPOINT after_alice;
    
    INSERT INTO accounts (name, balance) VALUES ('Bob', 2000);
    SAVEPOINT after_bob;
    
    INSERT INTO accounts (name, balance) VALUES ('Charlie', 3000);
    
    -- เปลี่ยนใจ: ไม่เอา Charlie
    ROLLBACK TO SAVEPOINT after_bob;
    
    -- Alice และ Bob ยังอยู่, Charlie ถูกยกเลิก
COMMIT;

SELECT name, balance FROM accounts;
-- Alice: 1000
-- Bob: 2000
```

---

## 2. ROLLBACK TO SAVEPOINT

```sql
-- ตัวอย่างที่ 1: Simple rollback to savepoint
BEGIN;
    UPDATE products SET price = price * 1.1;  -- เพิ่มราคา 10%
    SAVEPOINT price_update;
    
    UPDATE products SET price = price * 1.2 WHERE category = 'electronics';  -- เพิ่มอีก 20%
    
    -- ตรวจสอบ
    SELECT category, AVG(price) FROM products GROUP BY category;
    
    -- Electronics แพงเกินไป → rollback แค่ส่วนนั้น
    ROLLBACK TO SAVEPOINT price_update;
    
    -- ตอนนี้ทุก category ได้ 10% แล้ว แต่ electronics ไม่ได้ 20%
    UPDATE products SET price = price * 1.05 WHERE category = 'electronics';  -- เพิ่มแค่ 5%
COMMIT;

-- ตัวอย่างที่ 2: Multiple savepoints
BEGIN;
    SAVEPOINT start;
    
    INSERT INTO orders (customer_id, status) VALUES (1, 'new') RETURNING order_id INTO v_order_id;
    SAVEPOINT after_order;
    
    INSERT INTO order_items (order_id, product_id, qty) VALUES (v_order_id, 101, 2);
    SAVEPOINT after_item1;
    
    INSERT INTO order_items (order_id, product_id, qty) VALUES (v_order_id, 102, 1);
    SAVEPOINT after_item2;
    
    INSERT INTO order_items (order_id, product_id, qty) VALUES (v_order_id, 999, 1);
    -- สมมติ product 999 ไม่มีสต็อก
    -- ต้องการยกเลิกแค่ item สุดท้าย
    ROLLBACK TO SAVEPOINT after_item2;
    
    -- order ยังมี item 101 และ 102
    UPDATE orders SET status = 'confirmed' WHERE order_id = v_order_id;
COMMIT;

-- ตัวอย่างที่ 3: Rollback ไปยัง savepoint ที่อยู่ก่อนหน้า
BEGIN;
    INSERT INTO log_entries VALUES (1, 'Step 1 started', NOW());
    SAVEPOINT step1;
    
    INSERT INTO log_entries VALUES (2, 'Step 2 started', NOW());
    SAVEPOINT step2;
    
    INSERT INTO log_entries VALUES (3, 'Step 3 started', NOW());
    SAVEPOINT step3;
    
    -- Step 3 ล้มเหลว ต้องการ rollback ไปที่ step1
    ROLLBACK TO SAVEPOINT step1;
    -- step2 และ step3 ถูกลบ, step1 savepoint ยังอยู่ใช้ได้อีก
    
    -- Savepoints ที่ถูก rollback past จะถูกลบด้วย (step2, step3)
    -- แต่ step1 ยังอยู่
    
    INSERT INTO log_entries VALUES (2, 'Step 2 retry', NOW());
    SAVEPOINT step2;  -- สร้าง savepoint ใหม่
    
COMMIT;
```

---

## 3. RELEASE SAVEPOINT

```sql
-- RELEASE ลบ savepoint โดยไม่ rollback
-- ใช้เพื่อ "commit" nested transaction ย่อย

BEGIN;
    INSERT INTO main_records VALUES (1, 'main');
    SAVEPOINT nested_start;
    
    INSERT INTO detail_records VALUES (1, 1, 'detail 1');
    INSERT INTO detail_records VALUES (2, 1, 'detail 2');
    
    -- ทุกอย่างดี → release savepoint (ยืนยัน nested section)
    RELEASE SAVEPOINT nested_start;
    
    -- ตอนนี้ไม่สามารถ ROLLBACK TO nested_start ได้แล้ว
    -- แต่ยังอยู่ใน outer transaction อยู่
    
    INSERT INTO summary VALUES (1, 2, NOW());
COMMIT;

-- Pattern: Release เมื่อสำเร็จ, Rollback เมื่อล้มเหลว
CREATE OR REPLACE FUNCTION process_nested_section()
RETURNS VOID AS $$
BEGIN
    SAVEPOINT nested;
    
    -- ทำงาน
    INSERT INTO table_a VALUES (...);
    INSERT INTO table_b VALUES (...);
    
    -- สำเร็จ → release
    RELEASE SAVEPOINT nested;
    
EXCEPTION
    WHEN OTHERS THEN
        -- ล้มเหลว → rollback
        ROLLBACK TO SAVEPOINT nested;
        RAISE NOTICE 'Nested section failed: %', SQLERRM;
END;
$$ LANGUAGE plpgsql;
```

---

## 4. Nested Transactions Simulation

PostgreSQL ไม่มี "true nested transactions" แต่เราสามารถจำลองได้ด้วย Savepoints:

```sql
-- Pattern: Simulating nested transactions with savepoints
CREATE OR REPLACE FUNCTION begin_nested() RETURNS VOID AS $$
BEGIN
    EXECUTE 'SAVEPOINT nested_' || txid_current();
END;
$$ LANGUAGE plpgsql;

CREATE OR REPLACE FUNCTION commit_nested() RETURNS VOID AS $$
BEGIN
    EXECUTE 'RELEASE SAVEPOINT nested_' || txid_current();
END;
$$ LANGUAGE plpgsql;

CREATE OR REPLACE FUNCTION rollback_nested() RETURNS VOID AS $$
BEGIN
    EXECUTE 'ROLLBACK TO SAVEPOINT nested_' || txid_current();
    EXECUTE 'RELEASE SAVEPOINT nested_' || txid_current();
END;
$$ LANGUAGE plpgsql;

-- ตัวอย่างการใช้งาน
BEGIN;  -- Outer transaction

    INSERT INTO parent_table VALUES (1, 'parent');
    
    -- "Begin" nested transaction
    SAVEPOINT nested_1;
    BEGIN
        INSERT INTO child_table VALUES (1, 1, 'child A');
        INSERT INTO child_table VALUES (2, 1, 'child B');
        -- สำเร็จ
    EXCEPTION
        WHEN OTHERS THEN
            ROLLBACK TO SAVEPOINT nested_1;
    END;
    RELEASE SAVEPOINT nested_1;  -- "Commit" nested
    
    -- "Begin" another nested transaction
    SAVEPOINT nested_2;
    BEGIN
        INSERT INTO child_table VALUES (3, 1, 'child C');
        RAISE EXCEPTION 'Something failed';  -- จำลอง error
    EXCEPTION
        WHEN OTHERS THEN
            ROLLBACK TO SAVEPOINT nested_2;
            RAISE NOTICE 'Nested 2 rolled back: %', SQLERRM;
    END;
    RELEASE SAVEPOINT nested_2;  -- Release แม้จะ rollback
    
    -- Outer transaction ยังคงใช้งานได้
    UPDATE parent_table SET status = 'processed' WHERE id = 1;
    
COMMIT;

-- Result: parent_table มี row 1 'processed'
-- child_table มี children A และ B (แต่ไม่มี C เพราะถูก rollback)
```

---

## 5. Partial Rollback Patterns

### Pattern 1: Try-Catch with Savepoints

```sql
-- Pattern ที่ใช้บ่อยมากใน production
CREATE OR REPLACE FUNCTION process_items_with_partial_rollback(
    p_items INT[]
) RETURNS TABLE(item_id INT, status TEXT, message TEXT) AS $$
DECLARE
    v_item_id INT;
    v_savepoint_name TEXT;
BEGIN
    FOREACH v_item_id IN ARRAY p_items
    LOOP
        v_savepoint_name := 'item_' || v_item_id;
        
        BEGIN
            EXECUTE 'SAVEPOINT ' || v_savepoint_name;
            
            -- Process this item
            INSERT INTO processed_items (item_id, processed_at)
            VALUES (v_item_id, NOW());
            
            UPDATE items SET status = 'processed' WHERE id = v_item_id;
            
            EXECUTE 'RELEASE SAVEPOINT ' || v_savepoint_name;
            
            item_id := v_item_id;
            status := 'success';
            message := 'Processed successfully';
            RETURN NEXT;
            
        EXCEPTION
            WHEN OTHERS THEN
                EXECUTE 'ROLLBACK TO SAVEPOINT ' || v_savepoint_name;
                EXECUTE 'RELEASE SAVEPOINT ' || v_savepoint_name;
                
                item_id := v_item_id;
                status := 'failed';
                message := SQLERRM;
                RETURN NEXT;
        END;
    END LOOP;
END;
$$ LANGUAGE plpgsql;

-- ใช้งาน
BEGIN;
SELECT * FROM process_items_with_partial_rollback(ARRAY[1, 2, 3, 4, 5]);
COMMIT;
```

### Pattern 2: Multi-Step Process with Savepoints

```sql
-- การ import ข้อมูลแบบ partial success
CREATE OR REPLACE FUNCTION import_products(
    p_products JSONB  -- [{"name": "Widget", "price": 9.99, "stock": 100}, ...]
) RETURNS JSONB AS $$
DECLARE
    v_product     JSONB;
    v_success_count INT := 0;
    v_fail_count    INT := 0;
    v_errors        JSONB[] := '{}';
    v_sp_name       TEXT;
BEGIN
    FOR v_product IN SELECT * FROM jsonb_array_elements(p_products)
    LOOP
        v_sp_name := 'import_' || md5(random()::text);
        
        BEGIN
            EXECUTE 'SAVEPOINT ' || v_sp_name;
            
            INSERT INTO products (name, price, stock)
            VALUES (
                v_product->>'name',
                (v_product->>'price')::DECIMAL,
                (v_product->>'stock')::INT
            );
            
            EXECUTE 'RELEASE SAVEPOINT ' || v_sp_name;
            v_success_count := v_success_count + 1;
            
        EXCEPTION
            WHEN unique_violation THEN
                EXECUTE 'ROLLBACK TO SAVEPOINT ' || v_sp_name;
                EXECUTE 'RELEASE SAVEPOINT ' || v_sp_name;
                v_fail_count := v_fail_count + 1;
                v_errors := array_append(v_errors, 
                    jsonb_build_object(
                        'product', v_product->>'name',
                        'error', 'Duplicate product name'
                    )
                );
            WHEN check_violation THEN
                EXECUTE 'ROLLBACK TO SAVEPOINT ' || v_sp_name;
                EXECUTE 'RELEASE SAVEPOINT ' || v_sp_name;
                v_fail_count := v_fail_count + 1;
                v_errors := array_append(v_errors, 
                    jsonb_build_object(
                        'product', v_product->>'name',
                        'error', 'Invalid data: ' || SQLERRM
                    )
                );
        END;
    END LOOP;
    
    RETURN jsonb_build_object(
        'success_count', v_success_count,
        'fail_count', v_fail_count,
        'errors', to_jsonb(v_errors)
    );
END;
$$ LANGUAGE plpgsql;

-- ทดสอบ
BEGIN;
SELECT import_products('[
    {"name": "Widget A", "price": 9.99, "stock": 100},
    {"name": "Widget B", "price": -5.00, "stock": 50},
    {"name": "Widget C", "price": 15.99, "stock": 75}
]');
COMMIT;
```

### Pattern 3: Validation with Rollback on Failure

```sql
-- Validate ข้อมูลทีละ batch แล้วรวม
CREATE OR REPLACE FUNCTION validate_and_insert_batch(
    p_data JSONB[]
) RETURNS TABLE(
    batch_index    INT,
    rows_inserted  INT,
    errors         TEXT
) AS $$
DECLARE
    v_idx   INT;
    v_item  JSONB;
    v_sp    TEXT;
    v_rows  INT;
BEGIN
    v_idx := 0;
    
    FOREACH v_item IN ARRAY p_data
    LOOP
        v_idx := v_idx + 1;
        v_sp := 'batch_' || v_idx;
        
        SAVEPOINT "batch_sp";
        
        BEGIN
            -- Process batch
            WITH batch_data AS (
                SELECT * FROM jsonb_to_recordset(v_item) AS t(id INT, name TEXT, value DECIMAL)
            )
            INSERT INTO target_table (id, name, value)
            SELECT id, name, value FROM batch_data;
            
            GET DIAGNOSTICS v_rows = ROW_COUNT;
            
            RELEASE SAVEPOINT "batch_sp";
            
            batch_index   := v_idx;
            rows_inserted := v_rows;
            errors        := NULL;
            RETURN NEXT;
            
        EXCEPTION
            WHEN OTHERS THEN
                ROLLBACK TO SAVEPOINT "batch_sp";
                RELEASE SAVEPOINT "batch_sp";
                
                batch_index   := v_idx;
                rows_inserted := 0;
                errors        := SQLERRM;
                RETURN NEXT;
        END;
    END LOOP;
END;
$$ LANGUAGE plpgsql;
```

---

## 6. Savepoints ใน Application Code

### Python (psycopg2)

```sql
-- SQL สำหรับทดสอบ pattern จาก Python
-- ใน Python จะเป็น:
-- conn = psycopg2.connect(...)
-- conn.autocommit = False
-- cur = conn.cursor()
-- cur.execute("BEGIN")
-- cur.execute("SAVEPOINT sp1")
-- try:
--     cur.execute("INSERT ...")
--     cur.execute("RELEASE SAVEPOINT sp1")
-- except Exception:
--     cur.execute("ROLLBACK TO SAVEPOINT sp1")
-- conn.commit()

-- จำลองใน SQL:
DO $$
DECLARE
    i INT;
    errors TEXT[] := '{}';
BEGIN
    FOR i IN 1..10
    LOOP
        BEGIN
            SAVEPOINT loop_item;
            
            IF i % 3 = 0 THEN
                RAISE EXCEPTION 'Simulated error at %', i;
            END IF;
            
            INSERT INTO batch_results (item_number, status)
            VALUES (i, 'success');
            
            RELEASE SAVEPOINT loop_item;
            
        EXCEPTION WHEN OTHERS THEN
            ROLLBACK TO SAVEPOINT loop_item;
            RELEASE SAVEPOINT loop_item;
            
            INSERT INTO batch_results (item_number, status, error_msg)
            VALUES (i, 'failed', SQLERRM);
            
            errors := array_append(errors, format('Item %s: %s', i, SQLERRM));
        END;
    END LOOP;
    
    IF array_length(errors, 1) > 0 THEN
        RAISE NOTICE 'Completed with % errors: %', 
                      array_length(errors, 1), 
                      array_to_string(errors, '; ');
    END IF;
END;
$$;
```

### JDBC Pattern (Java-style in SQL)

```sql
-- Pattern ที่ใช้ใน Java JDBC:
-- conn.setAutoCommit(false);
-- Savepoint sp = conn.setSavepoint("before_update");
-- try {
--     stmt.executeUpdate("UPDATE ...");
--     conn.releaseSavepoint(sp);
-- } catch (SQLException e) {
--     conn.rollback(sp);
--     conn.releaseSavepoint(sp);
-- }
-- conn.commit();

-- จำลองใน PL/pgSQL:
CREATE OR REPLACE FUNCTION jdbc_style_savepoint_demo()
RETURNS TEXT AS $$
DECLARE
    v_result TEXT := '';
BEGIN
    -- "setSavepoint"
    SAVEPOINT before_update;
    
    BEGIN
        UPDATE accounts SET balance = balance + 100 WHERE account_id = 1;
        -- "releaseSavepoint" - success
        RELEASE SAVEPOINT before_update;
        v_result := v_result || 'Update 1 succeeded; ';
    EXCEPTION WHEN OTHERS THEN
        -- "rollback(sp)"
        ROLLBACK TO SAVEPOINT before_update;
        RELEASE SAVEPOINT before_update;
        v_result := v_result || format('Update 1 failed (%s); ', SQLERRM);
    END;
    
    -- Another operation
    SAVEPOINT before_insert;
    
    BEGIN
        INSERT INTO audit_log (action, timestamp) VALUES ('balance_update', NOW());
        RELEASE SAVEPOINT before_insert;
        v_result := v_result || 'Insert succeeded';
    EXCEPTION WHEN OTHERS THEN
        ROLLBACK TO SAVEPOINT before_insert;
        RELEASE SAVEPOINT before_insert;
        v_result := v_result || format('Insert failed (%s)', SQLERRM);
    END;
    
    RETURN v_result;
END;
$$ LANGUAGE plpgsql;

BEGIN;
SELECT jdbc_style_savepoint_demo();
COMMIT;
```

---

## 7. Savepoints ใน SQL Server

```sql
-- SQL Server ใช้ SAVE TRANSACTION แทน SAVEPOINT
BEGIN TRANSACTION;

    INSERT INTO Customers (Name) VALUES ('Alice');
    SAVE TRANSACTION after_alice;  -- SQL Server syntax
    
    INSERT INTO Orders (CustomerID, Total) VALUES (1, 5000);
    SAVE TRANSACTION after_order;
    
    -- ลอง insert ที่ล้มเหลว
    BEGIN TRY
        INSERT INTO OrderItems (OrderID, ProductID, Qty) VALUES (1, 9999, 1);
        -- Product 9999 ไม่มี → Foreign key violation
    END TRY
    BEGIN CATCH
        IF XACT_STATE() = -1  -- Uncommittable transaction
        BEGIN
            ROLLBACK TRANSACTION;
            THROW;
        END ELSE IF XACT_STATE() = 1  -- Active, committable transaction
        BEGIN
            ROLLBACK TRANSACTION after_order;  -- Rollback ถึง savepoint
            -- ยังอยู่ใน transaction
        END
    END CATCH;
    
    -- ยังสามารถทำงานต่อได้
    INSERT INTO OrderItems (OrderID, ProductID, Qty) VALUES (1, 101, 1);
    
COMMIT TRANSACTION;
```

---

## 8. Savepoints ใน MySQL

```sql
-- MySQL รองรับ SAVEPOINT เหมือน PostgreSQL
START TRANSACTION;

    INSERT INTO users (name, email) VALUES ('Alice', 'alice@example.com');
    SAVEPOINT after_alice;
    
    INSERT INTO users (name, email) VALUES ('Bob', 'bob@example.com');
    SAVEPOINT after_bob;
    
    -- สมมติมีปัญหากับ Bob's email
    INSERT INTO email_verification (user_id, token) 
    VALUES (LAST_INSERT_ID(), 'invalid_token_format');
    -- หรืออาจมี constraint violation
    
    ROLLBACK TO SAVEPOINT after_alice;
    -- Bob และ email verification ถูกยกเลิก
    
    -- Alice ยังอยู่ ทำต่อ
    INSERT INTO users (name, email) VALUES ('Bob Fixed', 'bob.fixed@example.com');

COMMIT;

-- ดู Savepoints ปัจจุบัน (MySQL ไม่มี system table สำหรับนี้)
-- ต้องจัดการเองใน application code
```

---

## 9. Savepoint Use Cases ใน Real World

### Use Case 1: Import ข้อมูลแบบ Continue-on-Error

```sql
CREATE TABLE import_log (
    import_id   SERIAL PRIMARY KEY,
    row_number  INT,
    data        JSONB,
    status      VARCHAR(20),
    error_msg   TEXT,
    imported_at TIMESTAMP DEFAULT NOW()
);

CREATE OR REPLACE FUNCTION import_csv_data(
    p_import_id INT,
    p_rows      JSONB[]
) RETURNS JSONB AS $$
DECLARE
    v_row      JSONB;
    v_row_num  INT := 0;
    v_success  INT := 0;
    v_failed   INT := 0;
    v_sp       TEXT;
BEGIN
    FOREACH v_row IN ARRAY p_rows
    LOOP
        v_row_num := v_row_num + 1;
        v_sp := 'row_' || v_row_num;
        
        EXECUTE format('SAVEPOINT %I', v_sp);
        
        BEGIN
            -- Validate required fields
            IF v_row->>'customer_name' IS NULL THEN
                RAISE EXCEPTION 'customer_name is required';
            END IF;
            
            IF (v_row->>'amount')::DECIMAL <= 0 THEN
                RAISE EXCEPTION 'amount must be positive';
            END IF;
            
            -- Insert the data
            INSERT INTO transactions (
                customer_name,
                amount,
                transaction_date,
                import_id
            ) VALUES (
                v_row->>'customer_name',
                (v_row->>'amount')::DECIMAL,
                COALESCE((v_row->>'transaction_date')::DATE, CURRENT_DATE),
                p_import_id
            );
            
            -- Log success
            INSERT INTO import_log (import_id, row_number, data, status)
            VALUES (p_import_id, v_row_num, v_row, 'success');
            
            EXECUTE format('RELEASE SAVEPOINT %I', v_sp);
            v_success := v_success + 1;
            
        EXCEPTION
            WHEN OTHERS THEN
                EXECUTE format('ROLLBACK TO SAVEPOINT %I', v_sp);
                EXECUTE format('RELEASE SAVEPOINT %I', v_sp);
                
                -- Log failure
                INSERT INTO import_log (import_id, row_number, data, status, error_msg)
                VALUES (p_import_id, v_row_num, v_row, 'failed', SQLERRM);
                
                v_failed := v_failed + 1;
        END;
    END LOOP;
    
    RETURN jsonb_build_object(
        'total', v_row_num,
        'success', v_success,
        'failed', v_failed,
        'success_rate', ROUND(v_success::NUMERIC / v_row_num * 100, 2)
    );
END;
$$ LANGUAGE plpgsql;
```

### Use Case 2: Multi-Step Form Processing

```sql
-- สมมติมี wizard form ที่มีหลาย step
CREATE OR REPLACE FUNCTION process_registration_wizard(
    p_session_id UUID,
    p_step       INT,
    p_data       JSONB
) RETURNS JSONB AS $$
DECLARE
    v_user_id    INT;
    v_profile_id INT;
    v_sp_name    TEXT;
BEGIN
    v_sp_name := 'wizard_step_' || p_step;
    
    EXECUTE format('SAVEPOINT %I', v_sp_name);
    
    BEGIN
        CASE p_step
            WHEN 1 THEN
                -- Step 1: Create user account
                INSERT INTO users (email, password_hash)
                VALUES (p_data->>'email', p_data->>'password_hash')
                RETURNING user_id INTO v_user_id;
                
                INSERT INTO wizard_state (session_id, user_id, last_step)
                VALUES (p_session_id, v_user_id, 1)
                ON CONFLICT (session_id) DO UPDATE 
                SET user_id = EXCLUDED.user_id, last_step = 1;
                
            WHEN 2 THEN
                -- Step 2: Create profile
                SELECT user_id INTO v_user_id FROM wizard_state WHERE session_id = p_session_id;
                
                INSERT INTO user_profiles (user_id, first_name, last_name, phone)
                VALUES (
                    v_user_id,
                    p_data->>'first_name',
                    p_data->>'last_name',
                    p_data->>'phone'
                )
                RETURNING profile_id INTO v_profile_id;
                
                UPDATE wizard_state SET last_step = 2 WHERE session_id = p_session_id;
                
            WHEN 3 THEN
                -- Step 3: Complete registration
                SELECT user_id INTO v_user_id FROM wizard_state WHERE session_id = p_session_id;
                
                UPDATE users SET is_verified = TRUE, verified_at = NOW()
                WHERE user_id = v_user_id;
                
                DELETE FROM wizard_state WHERE session_id = p_session_id;
                
        END CASE;
        
        EXECUTE format('RELEASE SAVEPOINT %I', v_sp_name);
        
        RETURN jsonb_build_object('success', true, 'step', p_step);
        
    EXCEPTION
        WHEN OTHERS THEN
            EXECUTE format('ROLLBACK TO SAVEPOINT %I', v_sp_name);
            EXECUTE format('RELEASE SAVEPOINT %I', v_sp_name);
            
            RETURN jsonb_build_object('success', false, 'error', SQLERRM, 'step', p_step);
    END;
END;
$$ LANGUAGE plpgsql;
```

### Use Case 3: Financial Reconciliation

```sql
-- การกระทบยอดบัญชีที่ต้องทำทีละรายการ
CREATE OR REPLACE FUNCTION reconcile_transactions(
    p_statement_entries JSONB[]
) RETURNS TABLE(
    transaction_ref  TEXT,
    matched         BOOLEAN,
    action_taken    TEXT
) AS $$
DECLARE
    v_entry JSONB;
    v_sp    TEXT;
    v_ref   TEXT;
    v_amount DECIMAL;
    v_matched_id INT;
BEGIN
    FOREACH v_entry IN ARRAY p_statement_entries
    LOOP
        v_ref    := v_entry->>'reference';
        v_amount := (v_entry->>'amount')::DECIMAL;
        v_sp     := 'recon_' || md5(v_ref);
        
        EXECUTE format('SAVEPOINT %I', v_sp);
        
        BEGIN
            -- ค้นหา transaction ที่ match
            SELECT transaction_id INTO v_matched_id
            FROM pending_transactions
            WHERE reference = v_ref
              AND amount = v_amount
              AND status = 'pending'
            LIMIT 1
            FOR UPDATE;
            
            IF v_matched_id IS NOT NULL THEN
                -- Match found
                UPDATE pending_transactions 
                SET status = 'reconciled', reconciled_at = NOW()
                WHERE transaction_id = v_matched_id;
                
                INSERT INTO reconciliation_log (reference, amount, status, matched_id)
                VALUES (v_ref, v_amount, 'matched', v_matched_id);
                
                EXECUTE format('RELEASE SAVEPOINT %I', v_sp);
                
                transaction_ref := v_ref;
                matched         := TRUE;
                action_taken    := 'Matched transaction ' || v_matched_id;
            ELSE
                -- No match found
                INSERT INTO unmatched_entries (reference, amount, entry_date)
                VALUES (v_ref, v_amount, NOW());
                
                EXECUTE format('RELEASE SAVEPOINT %I', v_sp);
                
                transaction_ref := v_ref;
                matched         := FALSE;
                action_taken    := 'Saved as unmatched';
            END IF;
            
            RETURN NEXT;
            
        EXCEPTION
            WHEN OTHERS THEN
                EXECUTE format('ROLLBACK TO SAVEPOINT %I', v_sp);
                EXECUTE format('RELEASE SAVEPOINT %I', v_sp);
                
                transaction_ref := v_ref;
                matched         := FALSE;
                action_taken    := 'Error: ' || SQLERRM;
                RETURN NEXT;
        END;
    END LOOP;
END;
$$ LANGUAGE plpgsql;
```

---

## 10. Performance Considerations

```sql
-- Savepoints มี overhead เล็กน้อย แต่ส่วนใหญ่ไม่มีนัยสำคัญ

-- ตัวอย่าง: เปรียบเทียบ performance
DO $$
DECLARE
    v_start TIMESTAMPTZ;
    v_end TIMESTAMPTZ;
    i INT;
BEGIN
    -- Test 1: ไม่ใช้ savepoints
    v_start := clock_timestamp();
    FOR i IN 1..1000
    LOOP
        INSERT INTO perf_test (val) VALUES (i);
    END LOOP;
    v_end := clock_timestamp();
    RAISE NOTICE 'Without savepoints: %ms', EXTRACT(MILLISECONDS FROM v_end - v_start);
    
    -- Test 2: ใช้ savepoint ทุก 100 rows
    TRUNCATE perf_test;
    v_start := clock_timestamp();
    FOR i IN 1..1000
    LOOP
        IF i % 100 = 1 THEN
            EXECUTE 'SAVEPOINT batch_' || i;
        END IF;
        INSERT INTO perf_test (val) VALUES (i);
        IF i % 100 = 0 THEN
            EXECUTE 'RELEASE SAVEPOINT batch_' || (i - 99);
        END IF;
    END LOOP;
    v_end := clock_timestamp();
    RAISE NOTICE 'With savepoints every 100 rows: %ms', EXTRACT(MILLISECONDS FROM v_end - v_start);
    
    -- Test 3: ใช้ savepoint ทุก row (overhead สูงมาก)
    TRUNCATE perf_test;
    v_start := clock_timestamp();
    FOR i IN 1..1000
    LOOP
        EXECUTE 'SAVEPOINT row_' || i;
        INSERT INTO perf_test (val) VALUES (i);
        EXECUTE 'RELEASE SAVEPOINT row_' || i;
    END LOOP;
    v_end := clock_timestamp();
    RAISE NOTICE 'With savepoints every row: %ms', EXTRACT(MILLISECONDS FROM v_end - v_start);
END;
$$;

-- คำแนะนำ: ใช้ savepoints เมื่อจำเป็นจริงๆ เช่น
-- 1. Loop ที่บางรายการอาจล้มเหลว
-- 2. Multi-step process ที่ต้องการ partial rollback
-- 3. Nested operations ใน stored procedures
-- ไม่ควรใช้สำหรับทุก statement ที่ทำ
```

---

## 11. Debugging Savepoints

```sql
-- PostgreSQL ไม่มี system view สำหรับ active savepoints
-- แต่เราสามารถ track ได้เองใน application

-- Pattern: Savepoint Stack Tracking
CREATE TABLE savepoint_audit (
    session_id    TEXT DEFAULT pg_backend_pid()::TEXT,
    sp_name       TEXT,
    action        TEXT CHECK (action IN ('create', 'rollback', 'release')),
    created_at    TIMESTAMPTZ DEFAULT NOW()
);

CREATE OR REPLACE FUNCTION tracked_savepoint(
    p_name   TEXT,
    p_action TEXT DEFAULT 'create'
) RETURNS VOID AS $$
BEGIN
    INSERT INTO savepoint_audit (sp_name, action) VALUES (p_name, p_action);
    
    CASE p_action
        WHEN 'create'   THEN EXECUTE 'SAVEPOINT ' || quote_ident(p_name);
        WHEN 'rollback' THEN EXECUTE 'ROLLBACK TO SAVEPOINT ' || quote_ident(p_name);
        WHEN 'release'  THEN EXECUTE 'RELEASE SAVEPOINT ' || quote_ident(p_name);
    END CASE;
END;
$$ LANGUAGE plpgsql;

-- ใช้งาน
BEGIN;
SELECT tracked_savepoint('step1');
INSERT INTO test_table VALUES (1);

SELECT tracked_savepoint('step2');
INSERT INTO test_table VALUES (2);

SELECT tracked_savepoint('step2', 'rollback');

SELECT tracked_savepoint('step2', 'release');

COMMIT;

-- ดูประวัติ
SELECT * FROM savepoint_audit ORDER BY created_at;
```

---

## 12. Common Mistakes และวิธีแก้

```sql
-- Mistake 1: ใช้ชื่อ savepoint ซ้ำ
BEGIN;
    SAVEPOINT sp1;
    INSERT INTO t1 VALUES (1);
    
    SAVEPOINT sp1;  -- ชื่อซ้ำ! PostgreSQL สร้าง savepoint ใหม่แทน
    INSERT INTO t1 VALUES (2);
    
    ROLLBACK TO SAVEPOINT sp1;  -- จะ rollback ไปที่อันที่สองเท่านั้น
    -- ไม่ใช่อันแรก!
COMMIT;

-- วิธีแก้: ใช้ชื่อที่ unique
BEGIN;
    SAVEPOINT operation_1;
    INSERT INTO t1 VALUES (1);
    
    SAVEPOINT operation_2;  -- ชื่อต่างกัน
    INSERT INTO t1 VALUES (2);
    
    ROLLBACK TO SAVEPOINT operation_1;  -- แน่ใจว่า rollback ไปที่ถูก
COMMIT;

-- Mistake 2: Rollback past savepoint
BEGIN;
    SAVEPOINT sp1;
    INSERT INTO t1 VALUES (1);
    SAVEPOINT sp2;
    INSERT INTO t1 VALUES (2);
    SAVEPOINT sp3;
    INSERT INTO t1 VALUES (3);
    
    ROLLBACK TO SAVEPOINT sp1;  -- sp2 และ sp3 ถูกลบ!
    
    -- ไม่สามารถ ROLLBACK TO sp2 ได้แล้ว!
    -- ROLLBACK TO SAVEPOINT sp2;  -- ERROR: savepoint "sp2" does not exist
    
    -- แต่ sp1 ยังใช้ได้
    ROLLBACK TO SAVEPOINT sp1;  -- OK
COMMIT;

-- Mistake 3: ลืม Release savepoints
BEGIN;
    FOR i IN 1..10000
    LOOP
        EXECUTE 'SAVEPOINT sp_' || i;
        -- ลืม RELEASE!
    END LOOP;
    -- Memory usage สูงมาก! Savepoints ทั้งหมดถูกเก็บไว้
COMMIT;

-- วิธีแก้: Release เมื่อไม่ต้องการแล้ว
BEGIN;
    FOR i IN 1..10000
    LOOP
        EXECUTE 'SAVEPOINT sp_' || i;
        -- ทำงาน
        EXECUTE 'RELEASE SAVEPOINT sp_' || i;  -- ลบทันที
    END LOOP;
COMMIT;

-- Mistake 4: Savepoint หลัง transaction error
BEGIN;
    INSERT INTO t1 VALUES (1);
    
    BEGIN
        RAISE EXCEPTION 'Error!';
    EXCEPTION WHEN OTHERS THEN
        NULL;  -- Swallow error
    END;
    
    -- Transaction ยังอยู่ใน ABORTED state ใน PostgreSQL!
    INSERT INTO t1 VALUES (2);  -- ERROR: current transaction is aborted
    
    -- ต้องใช้ SAVEPOINT เพื่อหลีกเลี่ยงสิ่งนี้
ROLLBACK;

-- วิธีที่ถูกต้อง:
BEGIN;
    SAVEPOINT before_risky;
    
    BEGIN
        INSERT INTO t1 VALUES (1);
        RAISE EXCEPTION 'Error!';
    EXCEPTION WHEN OTHERS THEN
        ROLLBACK TO SAVEPOINT before_risky;
    END;
    
    -- Transaction ยังใช้งานได้
    INSERT INTO t1 VALUES (2);  -- OK!
COMMIT;
```

---

## 13. Savepoints กับ Triggers

```sql
-- Triggers ทำงานภายใน transaction context เดียวกัน
-- Savepoints ใน triggers อาจทำให้เกิดปัญหาได้

CREATE TABLE trigger_demo (
    id    SERIAL PRIMARY KEY,
    value TEXT NOT NULL
);

CREATE TABLE trigger_log (
    log_id    SERIAL PRIMARY KEY,
    action    TEXT,
    old_value TEXT,
    new_value TEXT,
    logged_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE OR REPLACE FUNCTION log_changes() RETURNS TRIGGER AS $$
BEGIN
    -- Trigger สร้าง savepoint ได้
    SAVEPOINT trigger_log_sp;
    
    BEGIN
        INSERT INTO trigger_log (action, old_value, new_value)
        VALUES (
            TG_OP,
            OLD.value,
            NEW.value
        );
        
        RELEASE SAVEPOINT trigger_log_sp;
    EXCEPTION WHEN OTHERS THEN
        ROLLBACK TO SAVEPOINT trigger_log_sp;
        RELEASE SAVEPOINT trigger_log_sp;
        -- Log the logging failure without failing the main operation
        RAISE WARNING 'Could not log change: %', SQLERRM;
    END;
    
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER after_update_log
AFTER UPDATE ON trigger_demo
FOR EACH ROW EXECUTE FUNCTION log_changes();

-- ทดสอบ
BEGIN;
UPDATE trigger_demo SET value = 'new_value' WHERE id = 1;
COMMIT;

-- Trigger's savepoint will be within the outer transaction
```

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: Basic Savepoint

สร้าง transaction ที่ใช้ savepoints เพื่อ insert หลาย rows โดยสามารถยกเลิกแค่บางรายการได้:

```sql
-- คำตอบ
BEGIN;
    INSERT INTO products (name, price, category) VALUES ('Product A', 100, 'Electronics');
    SAVEPOINT after_a;
    
    INSERT INTO products (name, price, category) VALUES ('Product B', -50, 'Electronics');  -- invalid price
    
    -- ตรวจสอบว่า price valid
    DO $$
    BEGIN
        IF (SELECT price FROM products WHERE name = 'Product B') < 0 THEN
            RAISE EXCEPTION 'Negative price';
        END IF;
    END;
    $$;
    
    RELEASE SAVEPOINT after_a;
    
COMMIT;
```

### แบบฝึกหัดที่ 2: Savepoint in Loop

```sql
-- คำตอบ
CREATE OR REPLACE FUNCTION bulk_insert_with_savepoints(
    p_count INT
) RETURNS TABLE(inserted INT, failed INT) AS $$
DECLARE
    i           INT;
    v_inserted  INT := 0;
    v_failed    INT := 0;
BEGIN
    FOR i IN 1..p_count
    LOOP
        SAVEPOINT item_sp;
        
        BEGIN
            -- สมมติ 10% fail
            IF random() < 0.1 THEN
                RAISE EXCEPTION 'Random failure at %', i;
            END IF;
            
            INSERT INTO bulk_test (num, created_at) VALUES (i, NOW());
            RELEASE SAVEPOINT item_sp;
            v_inserted := v_inserted + 1;
            
        EXCEPTION WHEN OTHERS THEN
            ROLLBACK TO SAVEPOINT item_sp;
            RELEASE SAVEPOINT item_sp;
            v_failed := v_failed + 1;
        END;
    END LOOP;
    
    inserted := v_inserted;
    failed   := v_failed;
    RETURN NEXT;
END;
$$ LANGUAGE plpgsql;

BEGIN;
SELECT * FROM bulk_insert_with_savepoints(100);
COMMIT;
```

### แบบฝึกหัดที่ 3: Multi-Level Savepoints

```sql
-- คำตอบ
BEGIN;
    -- Level 1
    INSERT INTO categories (name) VALUES ('Electronics');
    SAVEPOINT level1;
    
    -- Level 2
    INSERT INTO subcategories (category_id, name) VALUES (1, 'Phones');
    SAVEPOINT level2;
    
    -- Level 3
    INSERT INTO products (subcategory_id, name, price) VALUES (1, 'iPhone', 35000);
    SAVEPOINT level3;
    
    INSERT INTO products (subcategory_id, name, price) VALUES (1, 'Samsung', 30000);
    
    -- rollback Level 3 (ยกเลิก iPhone และ Samsung)
    ROLLBACK TO SAVEPOINT level3;
    
    -- ใส่ iPhone อีกครั้งแต่ราคาต่างออกไป
    INSERT INTO products (subcategory_id, name, price) VALUES (1, 'iPhone', 32000);
    
    -- rollback ไปที่ level1 (ยกเลิก subcategories และ products ทั้งหมด)
    ROLLBACK TO SAVEPOINT level1;
    
    -- เพิ่ม subcategory ใหม่
    INSERT INTO subcategories (category_id, name) VALUES (1, 'Laptops');

COMMIT;
-- ผล: category Electronics + subcategory Laptops เท่านั้น
```

### แบบฝึกหัดที่ 4: Error Recovery Pattern

```sql
-- คำตอบ
CREATE OR REPLACE FUNCTION resilient_data_migration()
RETURNS TABLE(step TEXT, status TEXT, rows_affected INT) AS $$
DECLARE
    v_rows INT;
BEGIN
    -- Step 1: Migrate users
    SAVEPOINT migrate_users;
    BEGIN
        INSERT INTO new_users SELECT id, name, email FROM old_users;
        GET DIAGNOSTICS v_rows = ROW_COUNT;
        RELEASE SAVEPOINT migrate_users;
        step := 'Migrate Users'; status := 'success'; rows_affected := v_rows;
        RETURN NEXT;
    EXCEPTION WHEN OTHERS THEN
        ROLLBACK TO SAVEPOINT migrate_users;
        RELEASE SAVEPOINT migrate_users;
        step := 'Migrate Users'; status := 'failed: ' || SQLERRM; rows_affected := 0;
        RETURN NEXT;
    END;
    
    -- Step 2: Migrate orders
    SAVEPOINT migrate_orders;
    BEGIN
        INSERT INTO new_orders SELECT id, user_id, total FROM old_orders;
        GET DIAGNOSTICS v_rows = ROW_COUNT;
        RELEASE SAVEPOINT migrate_orders;
        step := 'Migrate Orders'; status := 'success'; rows_affected := v_rows;
        RETURN NEXT;
    EXCEPTION WHEN OTHERS THEN
        ROLLBACK TO SAVEPOINT migrate_orders;
        RELEASE SAVEPOINT migrate_orders;
        step := 'Migrate Orders'; status := 'failed: ' || SQLERRM; rows_affected := 0;
        RETURN NEXT;
    END;
END;
$$ LANGUAGE plpgsql;
```

### แบบฝึกหัดที่ 5: Savepoint Stack Management

```sql
-- คำตอบ
CREATE OR REPLACE FUNCTION manage_savepoint_stack(
    p_operations TEXT[][]  -- [['create', 'sp1'], ['create', 'sp2'], ['rollback', 'sp1'], ...]
) RETURNS TEXT AS $$
DECLARE
    v_op    TEXT[];
    v_log   TEXT := '';
BEGIN
    FOREACH v_op SLICE 1 IN ARRAY p_operations
    LOOP
        CASE v_op[1]
            WHEN 'create' THEN
                EXECUTE format('SAVEPOINT %I', v_op[2]);
                v_log := v_log || format('Created: %s; ', v_op[2]);
            WHEN 'rollback' THEN
                EXECUTE format('ROLLBACK TO SAVEPOINT %I', v_op[2]);
                v_log := v_log || format('Rolled back to: %s; ', v_op[2]);
            WHEN 'release' THEN
                EXECUTE format('RELEASE SAVEPOINT %I', v_op[2]);
                v_log := v_log || format('Released: %s; ', v_op[2]);
        END CASE;
    END LOOP;
    
    RETURN v_log;
EXCEPTION WHEN OTHERS THEN
    RETURN v_log || 'ERROR: ' || SQLERRM;
END;
$$ LANGUAGE plpgsql;
```

### แบบฝึกหัดที่ 6: Order Processing with Savepoints

```sql
-- คำตอบ
CREATE OR REPLACE FUNCTION process_order_with_savepoints(
    p_customer_id INT,
    p_items JSONB[]
) RETURNS JSONB AS $$
DECLARE
    v_order_id INT;
    v_item     JSONB;
    v_result   JSONB[] := '{}';
    v_success  INT := 0;
    v_failed   INT := 0;
BEGIN
    -- Create order header
    SAVEPOINT create_order;
    BEGIN
        INSERT INTO orders (customer_id, status)
        VALUES (p_customer_id, 'pending')
        RETURNING order_id INTO v_order_id;
        
        RELEASE SAVEPOINT create_order;
    EXCEPTION WHEN OTHERS THEN
        ROLLBACK TO SAVEPOINT create_order;
        RAISE;
    END;
    
    -- Process each item
    FOREACH v_item IN ARRAY p_items
    LOOP
        SAVEPOINT process_item;
        BEGIN
            INSERT INTO order_items (
                order_id, product_id, quantity, unit_price
            )
            SELECT v_order_id, 
                   (v_item->>'product_id')::INT,
                   (v_item->>'quantity')::INT,
                   price
            FROM products 
            WHERE product_id = (v_item->>'product_id')::INT
              AND stock >= (v_item->>'quantity')::INT;
            
            IF NOT FOUND THEN
                RAISE EXCEPTION 'Product % unavailable', v_item->>'product_id';
            END IF;
            
            UPDATE products 
            SET stock = stock - (v_item->>'quantity')::INT
            WHERE product_id = (v_item->>'product_id')::INT;
            
            RELEASE SAVEPOINT process_item;
            v_success := v_success + 1;
            v_result := array_append(v_result, 
                jsonb_build_object('product_id', v_item->>'product_id', 'status', 'added'));
            
        EXCEPTION WHEN OTHERS THEN
            ROLLBACK TO SAVEPOINT process_item;
            RELEASE SAVEPOINT process_item;
            v_failed := v_failed + 1;
            v_result := array_append(v_result,
                jsonb_build_object('product_id', v_item->>'product_id', 
                                   'status', 'failed', 'error', SQLERRM));
        END;
    END LOOP;
    
    IF v_success = 0 THEN
        RAISE EXCEPTION 'No items could be added to order';
    END IF;
    
    UPDATE orders SET status = 'confirmed' WHERE order_id = v_order_id;
    
    RETURN jsonb_build_object(
        'order_id', v_order_id,
        'items_added', v_success,
        'items_failed', v_failed,
        'details', to_jsonb(v_result)
    );
END;
$$ LANGUAGE plpgsql;
```

### แบบฝึกหัดที่ 7: Savepoint for Data Validation

```sql
-- คำตอบ
CREATE OR REPLACE FUNCTION validate_and_save(
    p_data JSONB
) RETURNS JSONB AS $$
DECLARE
    v_errors TEXT[] := '{}';
    v_warnings TEXT[] := '{}';
BEGIN
    SAVEPOINT validation_start;
    
    -- Validate email
    IF p_data->>'email' !~ '^[^@]+@[^@]+\.[^@]+$' THEN
        v_errors := array_append(v_errors, 'Invalid email format');
    END IF;
    
    -- Validate age
    IF (p_data->>'age')::INT NOT BETWEEN 0 AND 150 THEN
        v_errors := array_append(v_errors, 'Invalid age');
    END IF;
    
    IF array_length(v_errors, 1) > 0 THEN
        ROLLBACK TO SAVEPOINT validation_start;
        RETURN jsonb_build_object('valid', false, 'errors', to_jsonb(v_errors));
    END IF;
    
    -- Save data
    INSERT INTO validated_users (email, age, data)
    VALUES (p_data->>'email', (p_data->>'age')::INT, p_data);
    
    RELEASE SAVEPOINT validation_start;
    
    RETURN jsonb_build_object('valid', true, 'warnings', to_jsonb(v_warnings));
    
EXCEPTION WHEN OTHERS THEN
    ROLLBACK TO SAVEPOINT validation_start;
    RETURN jsonb_build_object('valid', false, 'errors', ARRAY[SQLERRM]);
END;
$$ LANGUAGE plpgsql;
```

### แบบฝึกหัดที่ 8: Nested Business Logic

```sql
-- คำตอบ
CREATE OR REPLACE FUNCTION nested_business_logic()
RETURNS TEXT AS $$
DECLARE
    v_log TEXT := '';
BEGIN
    -- Outer operation
    INSERT INTO main_process (name, started_at) VALUES ('Process A', NOW());
    
    -- Sub-operation 1
    SAVEPOINT sub_op1;
    BEGIN
        INSERT INTO sub_tasks (process_name, task) VALUES ('Process A', 'Task 1');
        v_log := v_log || 'Task 1 done; ';
        
        -- Nested sub-operation
        SAVEPOINT sub_op1_nested;
        BEGIN
            INSERT INTO task_details (task, detail) VALUES ('Task 1', 'Detail 1a');
            INSERT INTO task_details (task, detail) VALUES ('Task 1', 'Detail 1b');
            v_log := v_log || 'Details added; ';
            RELEASE SAVEPOINT sub_op1_nested;
        EXCEPTION WHEN OTHERS THEN
            ROLLBACK TO SAVEPOINT sub_op1_nested;
            RELEASE SAVEPOINT sub_op1_nested;
            v_log := v_log || 'Details failed (continuing); ';
        END;
        
        RELEASE SAVEPOINT sub_op1;
    EXCEPTION WHEN OTHERS THEN
        ROLLBACK TO SAVEPOINT sub_op1;
        RELEASE SAVEPOINT sub_op1;
        v_log := v_log || 'Sub-op1 failed; ';
    END;
    
    -- Sub-operation 2
    SAVEPOINT sub_op2;
    BEGIN
        INSERT INTO sub_tasks (process_name, task) VALUES ('Process A', 'Task 2');
        RELEASE SAVEPOINT sub_op2;
        v_log := v_log || 'Task 2 done; ';
    EXCEPTION WHEN OTHERS THEN
        ROLLBACK TO SAVEPOINT sub_op2;
        RELEASE SAVEPOINT sub_op2;
        v_log := v_log || 'Task 2 failed; ';
    END;
    
    UPDATE main_process SET completed_at = NOW() WHERE name = 'Process A';
    
    RETURN v_log;
END;
$$ LANGUAGE plpgsql;

BEGIN;
SELECT nested_business_logic();
COMMIT;
```

### แบบฝึกหัดที่ 9: Savepoint Benchmark

```sql
-- คำตอบ
CREATE OR REPLACE FUNCTION benchmark_savepoints(
    p_row_count INT DEFAULT 1000,
    p_sp_frequency INT DEFAULT 100
) RETURNS TABLE(
    method TEXT,
    rows_inserted INT,
    duration_ms NUMERIC
) AS $$
DECLARE
    v_start TIMESTAMPTZ;
    v_end TIMESTAMPTZ;
    i INT;
BEGIN
    CREATE TEMP TABLE bench_data (id INT, val TEXT) ON COMMIT DROP;
    
    -- Method 1: No savepoints
    TRUNCATE bench_data;
    v_start := clock_timestamp();
    FOR i IN 1..p_row_count LOOP
        INSERT INTO bench_data VALUES (i, 'test_' || i);
    END LOOP;
    v_end := clock_timestamp();
    
    method := 'No savepoints';
    rows_inserted := p_row_count;
    duration_ms := EXTRACT(MILLISECONDS FROM v_end - v_start);
    RETURN NEXT;
    
    -- Method 2: Savepoints every N rows
    TRUNCATE bench_data;
    v_start := clock_timestamp();
    FOR i IN 1..p_row_count LOOP
        IF i % p_sp_frequency = 1 THEN
            EXECUTE 'SAVEPOINT batch_' || i;
        END IF;
        INSERT INTO bench_data VALUES (i, 'test_' || i);
        IF i % p_sp_frequency = 0 THEN
            EXECUTE 'RELEASE SAVEPOINT batch_' || (i - p_sp_frequency + 1);
        END IF;
    END LOOP;
    v_end := clock_timestamp();
    
    method := 'Savepoints every ' || p_sp_frequency || ' rows';
    rows_inserted := p_row_count;
    duration_ms := EXTRACT(MILLISECONDS FROM v_end - v_start);
    RETURN NEXT;
END;
$$ LANGUAGE plpgsql;

BEGIN;
SELECT * FROM benchmark_savepoints(1000, 100);
COMMIT;
```

### แบบฝึกหัดที่ 10: Complete Savepoint System

```sql
-- คำตอบ: Savepoint management system
CREATE TYPE sp_action AS ENUM ('create', 'rollback', 'release');

CREATE TABLE savepoint_stack (
    session_id TEXT DEFAULT pg_backend_pid()::TEXT,
    sp_name    TEXT NOT NULL,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    action     sp_action NOT NULL,
    PRIMARY KEY (session_id, sp_name, created_at)
);

CREATE OR REPLACE FUNCTION sp_create(p_name TEXT) RETURNS VOID AS $$
BEGIN
    EXECUTE format('SAVEPOINT %I', p_name);
    INSERT INTO savepoint_stack (sp_name, action) VALUES (p_name, 'create');
END;
$$ LANGUAGE plpgsql;

CREATE OR REPLACE FUNCTION sp_rollback(p_name TEXT) RETURNS VOID AS $$
BEGIN
    EXECUTE format('ROLLBACK TO SAVEPOINT %I', p_name);
    INSERT INTO savepoint_stack (sp_name, action) VALUES (p_name, 'rollback');
END;
$$ LANGUAGE plpgsql;

CREATE OR REPLACE FUNCTION sp_release(p_name TEXT) RETURNS VOID AS $$
BEGIN
    EXECUTE format('RELEASE SAVEPOINT %I', p_name);
    INSERT INTO savepoint_stack (sp_name, action) VALUES (p_name, 'release');
END;
$$ LANGUAGE plpgsql;

CREATE OR REPLACE VIEW active_savepoints AS
SELECT DISTINCT ON (session_id, sp_name) 
    session_id,
    sp_name,
    created_at,
    action
FROM savepoint_stack
ORDER BY session_id, sp_name, created_at DESC
WHERE action = 'create';  -- Only show savepoints not yet rolled back or released

-- การใช้งาน
BEGIN;
    SELECT sp_create('operation_1');
    INSERT INTO test_table VALUES (1);
    
    SELECT sp_create('operation_2');
    INSERT INTO test_table VALUES (2);
    
    SELECT sp_rollback('operation_1');  -- Rollback ทั้งสอง
    
    SELECT * FROM active_savepoints;  -- ดู savepoints ที่ยังใช้งานได้
COMMIT;
```

---

## สรุป

Savepoints เป็นเครื่องมือที่ทรงพลังสำหรับ:

1. **Partial Rollback** - ยกเลิกเพียงส่วนหนึ่งของ transaction
2. **Error Recovery** - จัดการ errors ในระดับ sub-transaction
3. **Batch Processing** - ประมวลผลทีละรายการ โดยรายการที่ล้มเหลวไม่กระทบรายการอื่น
4. **Nested Operations** - จำลอง nested transactions
5. **Import Operations** - นำเข้าข้อมูลโดยข้ามรายการที่มีปัญหา

**Key Points:**
- `SAVEPOINT name` - สร้าง checkpoint
- `ROLLBACK TO SAVEPOINT name` - กลับไปที่ checkpoint (savepoints ที่สร้างหลังจากนั้นจะถูกลบ)
- `RELEASE SAVEPOINT name` - ลบ checkpoint (ยืนยัน nested section)
- การ rollback ผ่าน savepoint จะลบ savepoints ที่สร้างหลังจากนั้น
- รองรับใน PostgreSQL, MySQL, SQLite, Oracle แต่ syntax แตกต่างกันเล็กน้อยใน SQL Server

ใน Part 74 เราจะเรียนรู้เรื่อง Isolation Levels ซึ่งเป็นหัวใจของการจัดการ concurrent transactions
