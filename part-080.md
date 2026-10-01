# Part 80: Distributed Transactions and Consistency

## บทนำ

เมื่อ application ขยายใหญ่ขึ้น เราต้องจัดการ transactions ที่ครอบคลุมหลาย databases หรือหลาย services การรักษา consistency ใน distributed system เป็นความท้าทายที่สำคัญที่สุดอย่างหนึ่งในการออกแบบระบบ

---

## 1. CAP Theorem

```
CAP Theorem (Brewer's Theorem, 2000):
ระบบ distributed ไม่สามารถ guarantee ทั้งสามอย่างพร้อมกัน:

C - Consistency:    ทุก node เห็นข้อมูลเดียวกัน ณ เวลาเดียวกัน
A - Availability:   ทุก request ได้รับ response (อาจไม่ใช่ข้อมูลล่าสุด)
P - Partition Tolerance: ระบบทำงานต่อได้แม้มี network partition

Network partition ไม่สามารถหลีกเลี่ยงได้ในระบบจริง ดังนั้น:
- CA systems: Traditional RDBMS (single node) - ไม่ tolerant ต่อ partition
- CP systems: MongoDB, HBase, ZooKeeper - เลือก consistency over availability
- AP systems: Cassandra, DynamoDB, CouchDB - เลือก availability over consistency
```

```sql
-- สาธิต: ความสำคัญของ Consistency ใน distributed scenario
CREATE TABLE distributed_accounts (
    account_id  INT PRIMARY KEY,
    balance     DECIMAL(15,2) NOT NULL,
    last_updated TIMESTAMPTZ DEFAULT NOW(),
    node_id     TEXT DEFAULT 'node-1'
);

-- Simulated Node 1
INSERT INTO distributed_accounts VALUES (1, 10000, NOW(), 'node-1');
-- Simulated Node 2 (replica - might be behind)
-- INSERT INTO distributed_accounts VALUES (1, 10000, NOW() - INTERVAL '2 seconds', 'node-2');

-- CP behavior: ถ้าไม่สามารถ sync กับ quorum → fail
-- AP behavior: ถ้าไม่สามารถ sync → return stale data แต่ available
```

---

## 2. Two-Phase Commit (2PC)

2PC เป็น protocol สำหรับทำ distributed transaction ให้เป็น atomic ประกอบด้วยสองขั้นตอน:
- **Phase 1 (Prepare)**: Coordinator ถาม participants ว่าพร้อมจะ commit หรือไม่
- **Phase 2 (Commit/Rollback)**: ถ้าทุกคนพร้อม → COMMIT, มีคนไม่พร้อม → ROLLBACK ทั้งหมด

```
2PC Flow:
                    Coordinator
                       |
          ┌────────────┼────────────┐
          ▼            ▼            ▼
      DB Node 1    DB Node 2    DB Node 3

Phase 1 - PREPARE:
Coordinator → "PREPARE TO COMMIT txn-123" → all nodes
Node 1 → "VOTE: YES (logged to disk)"
Node 2 → "VOTE: YES (logged to disk)"
Node 3 → "VOTE: YES (logged to disk)"

Phase 2 - COMMIT:
Coordinator → "COMMIT txn-123" → all nodes
All nodes commit and release locks
```

```sql
-- PostgreSQL PREPARE TRANSACTION (ส่วนหนึ่งของ 2PC)
-- สร้าง tables
CREATE TABLE orders (
    order_id    SERIAL PRIMARY KEY,
    customer_id INT NOT NULL,
    total       DECIMAL(10,2) NOT NULL,
    status      VARCHAR(20) DEFAULT 'pending',
    created_at  TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE inventory (
    product_id  INT PRIMARY KEY,
    stock       INT NOT NULL
);

CREATE TABLE payments (
    payment_id  SERIAL PRIMARY KEY,
    order_id    INT NOT NULL,
    amount      DECIMAL(10,2) NOT NULL,
    status      VARCHAR(20) DEFAULT 'pending',
    created_at  TIMESTAMPTZ DEFAULT NOW()
);

INSERT INTO inventory VALUES (101, 50), (102, 30);

-- 2PC Example: Order + Payment across two conceptual services
-- Service 1: Orders Database
BEGIN;
    INSERT INTO orders (customer_id, total) VALUES (1001, 299.99);
    UPDATE inventory SET stock = stock - 1 WHERE product_id = 101;
PREPARE TRANSACTION 'txn-order-1001-20240101';
-- Transaction is now in "prepared" state

-- Service 2: Payments Database (บน database อื่น)
-- BEGIN;
--     INSERT INTO payments (order_id, amount) VALUES (1, 299.99);
-- PREPARE TRANSACTION 'txn-payment-1001-20240101';

-- Coordinator checks all votes: YES, YES
-- Phase 2: Commit all
COMMIT PREPARED 'txn-order-1001-20240101';
-- COMMIT PREPARED 'txn-payment-1001-20240101';  -- บน database อื่น

-- ถ้ามี failure:
-- ROLLBACK PREPARED 'txn-order-1001-20240101';
-- ROLLBACK PREPARED 'txn-payment-1001-20240101';

-- ดู prepared transactions ที่รอ commit/rollback
SELECT * FROM pg_prepared_xacts;

-- Cleanup orphaned prepared transactions (อันตราย! ตรวจสอบก่อน)
-- DO $$
-- DECLARE v_txn RECORD;
-- BEGIN
--     FOR v_txn IN SELECT gid FROM pg_prepared_xacts 
--                  WHERE prepared < NOW() - INTERVAL '1 hour' LOOP
--         EXECUTE 'ROLLBACK PREPARED ' || quote_literal(v_txn.gid);
--     END LOOP;
-- END;
-- $$;
```

---

## 3. ปัญหาของ 2PC

```
2PC มีปัญหาหลัก:

1. Blocking Protocol:
   - ถ้า Coordinator crash ระหว่าง Phase 2
   - Participants จะ BLOCK จนกว่า Coordinator กลับมา
   - ทรัพยากรถูก lock นาน

2. Single Point of Failure:
   - Coordinator เป็น SPOF

3. Latency:
   - ต้องรอ 2 round trips ข้ามเครือข่าย
   - ไม่เหมาะกับ microservices ที่ต้องการ high throughput

4. Not suitable for long-running transactions:
   - Participants hold locks ตลอดช่วง Phase 1
```

```sql
-- Monitor ปัญหา prepared transactions
CREATE OR REPLACE FUNCTION check_stuck_prepared_transactions(
    p_timeout INTERVAL DEFAULT '30 minutes'
) RETURNS TABLE(
    gid         TEXT,
    owner       TEXT,
    database    TEXT,
    prepared    TIMESTAMPTZ,
    stuck_for   INTERVAL,
    action      TEXT
) AS $$
BEGIN
    RETURN QUERY
    SELECT 
        px.gid,
        px.owner,
        px.database,
        px.prepared,
        NOW() - px.prepared AS stuck_for,
        'ROLLBACK PREPARED ' || quote_literal(px.gid) AS action
    FROM pg_prepared_xacts px
    WHERE px.prepared < NOW() - p_timeout
    ORDER BY px.prepared;
END;
$$ LANGUAGE plpgsql;

SELECT * FROM check_stuck_prepared_transactions('1 hour');
```

---

## 4. Saga Pattern

Saga แก้ปัญหา 2PC โดยแบ่ง long-running transaction เป็น sequence ของ local transactions แต่ละอัน publish event ไปยัง service ถัดไป ถ้ามี failure → ใช้ **compensating transactions** เพื่อ undo

```
Saga Types:
1. Choreography: Services communicate via events (no central coordinator)
2. Orchestration: Central Saga Orchestrator controls the flow

E-commerce Order Saga (Choreography):
1. Order Service: createOrder → OrderCreated event
2. Inventory Service: reserveStock → StockReserved event  
3. Payment Service: processPayment → PaymentProcessed event
4. Shipping Service: scheduleShipment → ShipmentScheduled event

Failure Case:
3. Payment fails → PaymentFailed event
2. Inventory Service: releaseStock (compensating) → StockReleased event
1. Order Service: cancelOrder (compensating) → OrderCancelled event
```

```sql
-- Saga Implementation ใน PostgreSQL

-- Saga State Machine
CREATE TABLE saga_instances (
    saga_id     UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    saga_type   VARCHAR(50) NOT NULL,
    status      VARCHAR(20) DEFAULT 'started',
    current_step VARCHAR(50),
    context     JSONB NOT NULL DEFAULT '{}',
    started_at  TIMESTAMPTZ DEFAULT NOW(),
    updated_at  TIMESTAMPTZ DEFAULT NOW(),
    completed_at TIMESTAMPTZ
);

-- Saga Steps Log
CREATE TABLE saga_steps (
    step_id      SERIAL PRIMARY KEY,
    saga_id      UUID REFERENCES saga_instances(saga_id),
    step_name    VARCHAR(100) NOT NULL,
    status       VARCHAR(20) NOT NULL,  -- pending, executing, completed, compensating, compensated, failed
    payload      JSONB,
    result       JSONB,
    error        TEXT,
    executed_at  TIMESTAMPTZ DEFAULT NOW()
);

-- Order Saga Tables
CREATE TABLE saga_orders (
    order_id    UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    saga_id     UUID REFERENCES saga_instances(saga_id),
    customer_id INT NOT NULL,
    product_id  INT NOT NULL,
    quantity    INT NOT NULL,
    total       DECIMAL(10,2) NOT NULL,
    status      VARCHAR(20) DEFAULT 'pending'
);

CREATE TABLE saga_stock_reservations (
    reservation_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    saga_id        UUID REFERENCES saga_instances(saga_id),
    product_id     INT NOT NULL,
    quantity       INT NOT NULL,
    status         VARCHAR(20) DEFAULT 'reserved'
);

CREATE TABLE saga_payments (
    payment_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    saga_id    UUID REFERENCES saga_instances(saga_id),
    amount     DECIMAL(10,2) NOT NULL,
    status     VARCHAR(20) DEFAULT 'pending'
);

-- Step 1: Create Order
CREATE OR REPLACE FUNCTION saga_create_order(
    p_saga_id   UUID,
    p_customer  INT,
    p_product   INT,
    p_quantity  INT,
    p_total     DECIMAL
) RETURNS JSONB AS $$
DECLARE
    v_order_id UUID;
BEGIN
    INSERT INTO saga_orders (saga_id, customer_id, product_id, quantity, total)
    VALUES (p_saga_id, p_customer, p_product, p_quantity, p_total)
    RETURNING order_id INTO v_order_id;
    
    INSERT INTO saga_steps (saga_id, step_name, status, payload, result)
    VALUES (p_saga_id, 'create_order', 'completed', 
            jsonb_build_object('customer', p_customer, 'product', p_product),
            jsonb_build_object('order_id', v_order_id));
    
    RETURN jsonb_build_object('order_id', v_order_id, 'success', true);
END;
$$ LANGUAGE plpgsql;

-- Step 1 Compensation: Cancel Order
CREATE OR REPLACE FUNCTION saga_cancel_order(p_saga_id UUID) RETURNS VOID AS $$
BEGIN
    UPDATE saga_orders SET status = 'cancelled' WHERE saga_id = p_saga_id;
    INSERT INTO saga_steps (saga_id, step_name, status) 
    VALUES (p_saga_id, 'cancel_order', 'compensated');
END;
$$ LANGUAGE plpgsql;

-- Step 2: Reserve Stock
CREATE OR REPLACE FUNCTION saga_reserve_stock(p_saga_id UUID, p_product INT, p_qty INT)
RETURNS JSONB AS $$
DECLARE
    v_available INT;
    v_res_id    UUID;
BEGIN
    SELECT stock INTO v_available FROM inventory WHERE product_id = p_product FOR UPDATE;
    
    IF v_available < p_qty THEN
        INSERT INTO saga_steps (saga_id, step_name, status, error)
        VALUES (p_saga_id, 'reserve_stock', 'failed', 'Insufficient stock');
        RETURN jsonb_build_object('success', false, 'error', 'Insufficient stock');
    END IF;
    
    UPDATE inventory SET stock = stock - p_qty WHERE product_id = p_product;
    
    INSERT INTO saga_stock_reservations (saga_id, product_id, quantity)
    VALUES (p_saga_id, p_product, p_qty) RETURNING reservation_id INTO v_res_id;
    
    INSERT INTO saga_steps (saga_id, step_name, status, result)
    VALUES (p_saga_id, 'reserve_stock', 'completed', 
            jsonb_build_object('reservation_id', v_res_id));
    
    RETURN jsonb_build_object('reservation_id', v_res_id, 'success', true);
END;
$$ LANGUAGE plpgsql;

-- Step 2 Compensation: Release Stock
CREATE OR REPLACE FUNCTION saga_release_stock(p_saga_id UUID) RETURNS VOID AS $$
DECLARE
    v_res RECORD;
BEGIN
    SELECT * INTO v_res FROM saga_stock_reservations WHERE saga_id = p_saga_id;
    IF FOUND THEN
        UPDATE inventory SET stock = stock + v_res.quantity WHERE product_id = v_res.product_id;
        UPDATE saga_stock_reservations SET status = 'released' WHERE saga_id = p_saga_id;
        INSERT INTO saga_steps (saga_id, step_name, status)
        VALUES (p_saga_id, 'release_stock', 'compensated');
    END IF;
END;
$$ LANGUAGE plpgsql;

-- Step 3: Process Payment
CREATE OR REPLACE FUNCTION saga_process_payment(p_saga_id UUID, p_amount DECIMAL)
RETURNS JSONB AS $$
DECLARE
    v_pay_id UUID;
    v_success BOOLEAN := (random() > 0.2);  -- 80% success rate simulation
BEGIN
    IF NOT v_success THEN
        INSERT INTO saga_steps (saga_id, step_name, status, error)
        VALUES (p_saga_id, 'process_payment', 'failed', 'Payment declined');
        RETURN jsonb_build_object('success', false, 'error', 'Payment declined');
    END IF;
    
    INSERT INTO saga_payments (saga_id, amount, status)
    VALUES (p_saga_id, p_amount, 'completed') RETURNING payment_id INTO v_pay_id;
    
    INSERT INTO saga_steps (saga_id, step_name, status, result)
    VALUES (p_saga_id, 'process_payment', 'completed',
            jsonb_build_object('payment_id', v_pay_id));
    
    RETURN jsonb_build_object('payment_id', v_pay_id, 'success', true);
END;
$$ LANGUAGE plpgsql;

-- Saga Orchestrator
CREATE OR REPLACE FUNCTION execute_order_saga(
    p_customer  INT,
    p_product   INT,
    p_quantity  INT,
    p_total     DECIMAL
) RETURNS JSONB AS $$
DECLARE
    v_saga_id UUID;
    v_result  JSONB;
BEGIN
    -- Start saga
    INSERT INTO saga_instances (saga_type, status, current_step, context)
    VALUES ('order_saga', 'started', 'create_order',
            jsonb_build_object('customer', p_customer, 'product', p_product))
    RETURNING saga_id INTO v_saga_id;
    
    -- Step 1: Create Order
    v_result := saga_create_order(v_saga_id, p_customer, p_product, p_quantity, p_total);
    IF NOT (v_result->>'success')::BOOLEAN THEN GOTO compensate; END IF;
    
    UPDATE saga_instances SET current_step = 'reserve_stock' WHERE saga_id = v_saga_id;
    
    -- Step 2: Reserve Stock
    v_result := saga_reserve_stock(v_saga_id, p_product, p_quantity);
    IF NOT (v_result->>'success')::BOOLEAN THEN
        PERFORM saga_cancel_order(v_saga_id);  -- Compensate step 1
        GOTO compensate;
    END IF;
    
    UPDATE saga_instances SET current_step = 'process_payment' WHERE saga_id = v_saga_id;
    
    -- Step 3: Process Payment
    v_result := saga_process_payment(v_saga_id, p_total);
    IF NOT (v_result->>'success')::BOOLEAN THEN
        PERFORM saga_release_stock(v_saga_id);  -- Compensate step 2
        PERFORM saga_cancel_order(v_saga_id);   -- Compensate step 1
        GOTO compensate;
    END IF;
    
    -- Success
    UPDATE saga_instances 
    SET status = 'completed', current_step = 'done', completed_at = NOW(), updated_at = NOW()
    WHERE saga_id = v_saga_id;
    
    RETURN jsonb_build_object('success', true, 'saga_id', v_saga_id);
    
    <<compensate>>
    UPDATE saga_instances 
    SET status = 'compensated', updated_at = NOW()
    WHERE saga_id = v_saga_id;
    
    RETURN jsonb_build_object('success', false, 'saga_id', v_saga_id, 'error', v_result->>'error');
END;
$$ LANGUAGE plpgsql;

-- ทดสอบ
BEGIN;
INSERT INTO inventory VALUES (201, 10) ON CONFLICT (product_id) DO UPDATE SET stock = 10;
SELECT execute_order_saga(1001, 201, 2, 599.98);
COMMIT;

-- ดู saga status
SELECT s.saga_id, s.status, s.current_step, s.started_at, s.completed_at,
       array_agg(st.step_name ORDER BY st.step_id) AS steps_executed
FROM saga_instances s
LEFT JOIN saga_steps st ON st.saga_id = s.saga_id
GROUP BY s.saga_id, s.status, s.current_step, s.started_at, s.completed_at
ORDER BY s.started_at DESC;
```

---

## 5. Outbox Pattern

Outbox Pattern แก้ปัญหา "dual write" - การเขียนไปที่ database และ message queue พร้อมกัน

```
ปัญหาที่ Outbox แก้:
Order Service ต้องการทั้งสองอย่างนี้:
  1. บันทึก order ใน DB
  2. ส่ง event ไปยัง Kafka/RabbitMQ

ถ้า DB commit สำเร็จ แต่ message queue fail → inconsistency!
ถ้าส่ง message ก่อน แต่ DB fail → ghost messages!

Outbox Solution:
  1. บันทึก order + outbox message ใน SAME DB transaction
  2. Message Relay service อ่านจาก outbox table แล้วส่ง
  3. ทำ idempotent consumer ที่ปลายทาง
```

```sql
-- Outbox Pattern Implementation
CREATE TABLE outbox_events (
    event_id    UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    event_type  VARCHAR(100) NOT NULL,
    aggregate_type VARCHAR(50),
    aggregate_id   VARCHAR(100),
    payload     JSONB NOT NULL,
    status      VARCHAR(20) DEFAULT 'pending',
    attempts    INT DEFAULT 0,
    max_attempts INT DEFAULT 5,
    next_retry  TIMESTAMPTZ DEFAULT NOW(),
    created_at  TIMESTAMPTZ DEFAULT NOW(),
    processed_at TIMESTAMPTZ
);

CREATE INDEX ix_outbox_pending ON outbox_events(next_retry, created_at)
WHERE status = 'pending';

-- Create order + queue event atomically
CREATE OR REPLACE FUNCTION create_order_with_event(
    p_customer_id INT,
    p_items       JSONB
) RETURNS JSONB AS $$
DECLARE
    v_order_id INT;
    v_event_id UUID;
BEGIN
    -- 1. Create order
    INSERT INTO orders (customer_id, total)
    VALUES (p_customer_id, (p_items->>'total')::DECIMAL)
    RETURNING order_id INTO v_order_id;
    
    -- 2. Write outbox event (SAME transaction)
    INSERT INTO outbox_events (event_type, aggregate_type, aggregate_id, payload)
    VALUES (
        'order.created',
        'order',
        v_order_id::TEXT,
        jsonb_build_object(
            'order_id', v_order_id,
            'customer_id', p_customer_id,
            'items', p_items,
            'timestamp', NOW()
        )
    ) RETURNING event_id INTO v_event_id;
    
    -- Both succeed or both fail - atomically!
    RETURN jsonb_build_object(
        'order_id', v_order_id,
        'event_id', v_event_id,
        'success', true
    );
END;
$$ LANGUAGE plpgsql;

-- Message Relay: ส่ง events จาก outbox ไปยัง message queue
-- (เรียกจาก background process ทุก N seconds)
CREATE OR REPLACE FUNCTION relay_outbox_events(
    p_batch_size INT DEFAULT 100
) RETURNS TABLE(event_id UUID, event_type TEXT, delivered BOOLEAN) AS $$
DECLARE
    v_event RECORD;
    v_delivered BOOLEAN;
BEGIN
    FOR v_event IN
        SELECT * FROM outbox_events
        WHERE status = 'pending'
          AND next_retry <= NOW()
          AND attempts < max_attempts
        ORDER BY created_at
        LIMIT p_batch_size
        FOR UPDATE SKIP LOCKED
    LOOP
        -- Mark as processing
        UPDATE outbox_events 
        SET status = 'processing', attempts = attempts + 1
        WHERE outbox_events.event_id = v_event.event_id;
        
        -- Simulate: จริงๆ ต้องส่งไป Kafka/RabbitMQ ที่นี่
        -- v_delivered := send_to_kafka(v_event.event_type, v_event.payload);
        v_delivered := TRUE;  -- simulate success
        
        IF v_delivered THEN
            UPDATE outbox_events 
            SET status = 'delivered', processed_at = NOW()
            WHERE outbox_events.event_id = v_event.event_id;
        ELSE
            -- Exponential backoff retry
            UPDATE outbox_events 
            SET status = 'pending',
                next_retry = NOW() + (INTERVAL '1 minute' * POWER(2, v_event.attempts))
            WHERE outbox_events.event_id = v_event.event_id;
        END IF;
        
        event_id   := v_event.event_id;
        event_type := v_event.event_type;
        delivered  := v_delivered;
        RETURN NEXT;
    END LOOP;
END;
$$ LANGUAGE plpgsql;

-- Outbox monitoring
CREATE OR REPLACE VIEW outbox_health AS
SELECT 
    status,
    COUNT(*) AS count,
    MIN(created_at) AS oldest_event,
    MAX(attempts) AS max_attempts
FROM outbox_events
GROUP BY status;

SELECT * FROM outbox_health;

-- Dead letter events (failed max retries)
SELECT event_id, event_type, aggregate_id, attempts, created_at
FROM outbox_events
WHERE status = 'pending' AND attempts >= max_attempts
ORDER BY created_at;
```

---

## 6. Eventual Consistency

```
Eventual Consistency:
ระบบ guarantee ว่าถ้าไม่มี new updates → ในที่สุดทุก replica จะ converge ไปยัง value เดียวกัน
ไม่ guarantee timing ว่าจะ consistent เมื่อไหร่

Techniques:
1. Last Write Wins (LWW): ใช้ timestamp หรือ version
2. CRDTs (Conflict-free Replicated Data Types): merge โดยอัตโนมัติ
3. Vector Clocks: track causality ระหว่าง nodes
4. Read Repair: แก้ inconsistency ตอน read
```

```sql
-- Eventual Consistency via Replication
-- Primary: การเขียน
-- Replica: การอ่าน (อาจ lag ได้)

-- ตรวจสอบ replication lag
SELECT 
    application_name AS replica,
    state,
    sent_lsn,
    replay_lsn,
    pg_wal_lsn_diff(sent_lsn, replay_lsn) AS lag_bytes,
    replay_lag AS lag_time
FROM pg_stat_replication
ORDER BY lag_bytes DESC;

-- Monotonic reads: การอ่านหลังจาก write ต้องเห็น write นั้น
-- Solution: อ่านจาก primary เสมอหลังจาก write สำคัญ

CREATE OR REPLACE FUNCTION consistent_read_after_write(
    p_key TEXT,
    p_write_lsn pg_lsn DEFAULT NULL
) RETURNS BOOLEAN AS $$
BEGIN
    -- ถ้าอยู่บน replica ตรวจว่า LSN ผ่านมาแล้ว
    IF pg_is_in_recovery() AND p_write_lsn IS NOT NULL THEN
        -- รอจนกว่า replica จะ catch up
        RETURN pg_wal_replay_wait(p_write_lsn, timeout := 5.0);
    END IF;
    RETURN TRUE;
END;
$$ LANGUAGE plpgsql;

-- Pattern: Sticky sessions หรือ read your own writes
-- หลัง write ที่ primary → ส่ง LSN ไปกับ response
-- Read requests: ตรวจ LSN ก่อนอ่านจาก replica
```

---

## 7. CQRS (Command Query Responsibility Segregation)

```
CQRS แยก operation ออกเป็น:
- Command: Write operations (เปลี่ยน state)
- Query: Read operations (ไม่เปลี่ยน state)

Benefits:
- Scale reads and writes independently
- Optimize read models separately
- Support Event Sourcing
- Cleaner code separation
```

```sql
-- CQRS Implementation

-- Write Model (Command side)
CREATE TABLE products_write (
    product_id  SERIAL PRIMARY KEY,
    name        VARCHAR(200) NOT NULL,
    price       DECIMAL(10,2) NOT NULL,
    stock       INT NOT NULL DEFAULT 0,
    category_id INT,
    updated_at  TIMESTAMPTZ DEFAULT NOW()
);

-- Read Models (Query side) - denormalized views
CREATE TABLE products_read_cache (
    product_id        INT PRIMARY KEY,
    name              VARCHAR(200),
    price             DECIMAL(10,2),
    stock             INT,
    category_name     TEXT,
    total_sold        INT DEFAULT 0,
    avg_rating        DECIMAL(3,2),
    search_keywords   TEXT,   -- denormalized for full-text search
    last_synced       TIMESTAMPTZ DEFAULT NOW()
);

-- Command: Update product price
CREATE OR REPLACE FUNCTION cmd_update_price(
    p_product_id INT,
    p_new_price  DECIMAL,
    p_user_id    INT
) RETURNS VOID AS $$
BEGIN
    -- Write model update
    UPDATE products_write 
    SET price = p_new_price, updated_at = NOW()
    WHERE product_id = p_product_id;
    
    -- Publish event (via outbox)
    INSERT INTO outbox_events (event_type, aggregate_type, aggregate_id, payload)
    VALUES ('product.price_updated', 'product', p_product_id::TEXT,
            jsonb_build_object(
                'product_id', p_product_id,
                'new_price', p_new_price,
                'updated_by', p_user_id,
                'timestamp', NOW()
            ));
END;
$$ LANGUAGE plpgsql;

-- Query: Get product (from read model - optimized)
CREATE OR REPLACE FUNCTION qry_get_product(p_product_id INT)
RETURNS JSONB AS $$
DECLARE
    v_product RECORD;
BEGIN
    SELECT * INTO v_product FROM products_read_cache WHERE product_id = p_product_id;
    
    IF NOT FOUND THEN
        RETURN jsonb_build_object('error', 'Product not found');
    END IF;
    
    RETURN row_to_json(v_product)::JSONB;
END;
$$ LANGUAGE plpgsql;

-- Projection: สร้าง/อัปเดต read model จาก events
CREATE OR REPLACE FUNCTION project_price_update(p_event JSONB) RETURNS VOID AS $$
BEGIN
    UPDATE products_read_cache
    SET price = (p_event->>'new_price')::DECIMAL,
        last_synced = NOW()
    WHERE product_id = (p_event->>'product_id')::INT;
END;
$$ LANGUAGE plpgsql;

-- Search Query (optimized read model)
SELECT product_id, name, price, category_name, avg_rating
FROM products_read_cache
WHERE search_keywords ILIKE '%laptop%'
  AND stock > 0
  AND price BETWEEN 10000 AND 50000
ORDER BY avg_rating DESC
LIMIT 20;
```

---

## 8. Event Sourcing

```
Event Sourcing: แทนที่จะเก็บ current state → เก็บ sequence of events
State = apply(events)

Benefits:
- Complete audit trail
- Time travel (rewind to any point)
- Event replay for projections
- No lost updates

Trade-offs:
- Query complexity (ต้อง rebuild state)
- Event schema evolution
- Storage growth
```

```sql
-- Event Store
CREATE TABLE event_store (
    event_id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    stream_id       TEXT NOT NULL,   -- aggregate id (e.g., "order-1234")
    stream_type     TEXT NOT NULL,   -- aggregate type (e.g., "order")
    event_type      TEXT NOT NULL,   -- e.g., "OrderCreated", "ItemAdded"
    event_version   INT NOT NULL,    -- version within stream
    payload         JSONB NOT NULL,
    metadata        JSONB DEFAULT '{}',
    created_at      TIMESTAMPTZ DEFAULT NOW(),
    UNIQUE(stream_id, event_version)  -- prevent duplicate versions
);

CREATE INDEX ix_es_stream ON event_store(stream_id, event_version);
CREATE INDEX ix_es_type_time ON event_store(stream_type, created_at);

-- Append events (ONLY append, never update!)
CREATE OR REPLACE FUNCTION append_event(
    p_stream_id    TEXT,
    p_stream_type  TEXT,
    p_event_type   TEXT,
    p_payload      JSONB,
    p_expected_ver INT DEFAULT NULL  -- Optimistic lock
) RETURNS INT AS $$
DECLARE
    v_current_ver INT;
    v_new_ver     INT;
BEGIN
    -- Get current version
    SELECT COALESCE(MAX(event_version), 0) INTO v_current_ver
    FROM event_store WHERE stream_id = p_stream_id;
    
    -- Optimistic concurrency check
    IF p_expected_ver IS NOT NULL AND v_current_ver != p_expected_ver THEN
        RAISE EXCEPTION 'Concurrency conflict: expected version % but got %',
            p_expected_ver, v_current_ver;
    END IF;
    
    v_new_ver := v_current_ver + 1;
    
    INSERT INTO event_store (stream_id, stream_type, event_type, event_version, payload)
    VALUES (p_stream_id, p_stream_type, p_event_type, v_new_ver, p_payload);
    
    RETURN v_new_ver;
END;
$$ LANGUAGE plpgsql;

-- Read events for stream
CREATE OR REPLACE FUNCTION get_events(
    p_stream_id  TEXT,
    p_from_ver   INT DEFAULT 1,
    p_to_ver     INT DEFAULT NULL
) RETURNS TABLE(event_version INT, event_type TEXT, payload JSONB, created_at TIMESTAMPTZ) AS $$
BEGIN
    RETURN QUERY
    SELECT es.event_version, es.event_type, es.payload, es.created_at
    FROM event_store es
    WHERE es.stream_id = p_stream_id
      AND es.event_version >= p_from_ver
      AND (p_to_ver IS NULL OR es.event_version <= p_to_ver)
    ORDER BY es.event_version;
END;
$$ LANGUAGE plpgsql;

-- Rebuild Order aggregate from events
CREATE OR REPLACE FUNCTION rebuild_order_state(p_order_id INT)
RETURNS JSONB AS $$
DECLARE
    v_event   RECORD;
    v_state   JSONB := '{"items": [], "status": "draft", "total": 0}';
BEGIN
    FOR v_event IN
        SELECT event_type, payload FROM event_store
        WHERE stream_id = 'order-' || p_order_id
        ORDER BY event_version
    LOOP
        v_state := CASE v_event.event_type
            WHEN 'OrderCreated' THEN
                v_state || jsonb_build_object(
                    'customer_id', v_event.payload->>'customer_id',
                    'status', 'created'
                )
            WHEN 'ItemAdded' THEN
                jsonb_set(v_state, '{items}', 
                    (v_state->'items') || v_event.payload->'item') ||
                jsonb_build_object('total', 
                    (v_state->>'total')::DECIMAL + (v_event.payload->>'price')::DECIMAL)
            WHEN 'OrderConfirmed' THEN
                jsonb_set(v_state, '{status}', '"confirmed"')
            WHEN 'OrderCancelled' THEN
                jsonb_set(v_state, '{status}', '"cancelled"')
            ELSE v_state
        END;
    END LOOP;
    
    RETURN v_state;
END;
$$ LANGUAGE plpgsql;

-- ทดสอบ Event Sourcing
BEGIN;
SELECT append_event('order-1', 'order', 'OrderCreated', 
    '{"customer_id": 1001, "created_at": "2024-01-01"}', 0);
SELECT append_event('order-1', 'order', 'ItemAdded',
    '{"item": {"product": "Laptop", "qty": 1}, "price": 35000}', 1);
SELECT append_event('order-1', 'order', 'OrderConfirmed', 
    '{"confirmed_at": "2024-01-01"}', 2);
COMMIT;

SELECT * FROM get_events('order-1');
SELECT rebuild_order_state(1);
```

---

## 9. Distributed Transaction Patterns: XA Transactions

```sql
-- XA (eXtended Architecture) เป็น standard สำหรับ distributed transactions
-- PostgreSQL รองรับผ่าน max_prepared_transactions setting

-- ใน postgresql.conf:
-- max_prepared_transactions = 100  (default = 0, ต้อง enable)

-- ตรวจสอบ setting
SHOW max_prepared_transactions;

-- XA-style implementation สำหรับ transfer ระหว่าง 2 databases
-- (สมมติว่าทำผ่าน application layer)

CREATE OR REPLACE FUNCTION xa_prepare_transfer(
    p_from_account INT,
    p_to_account   INT,
    p_amount       DECIMAL,
    p_xa_id        TEXT
) RETURNS BOOLEAN AS $$
DECLARE
    v_balance DECIMAL;
BEGIN
    -- Phase 1: Prepare
    SELECT balance INTO v_balance FROM distributed_accounts 
    WHERE account_id = p_from_account FOR UPDATE;
    
    IF v_balance < p_amount THEN
        RAISE EXCEPTION 'Insufficient funds';
    END IF;
    
    UPDATE distributed_accounts 
    SET balance = balance - p_amount
    WHERE account_id = p_from_account;
    
    -- Log prepared state
    INSERT INTO outbox_events (event_type, aggregate_type, aggregate_id, payload)
    VALUES ('transfer.prepared', 'account', p_from_account::TEXT,
            jsonb_build_object('xa_id', p_xa_id, 'amount', p_amount, 'to', p_to_account));
    
    PREPARE TRANSACTION p_xa_id;
    RETURN TRUE;
END;
$$ LANGUAGE plpgsql;

-- Phase 2a: Commit (call when all participants voted YES)
CREATE OR REPLACE FUNCTION xa_commit(p_xa_id TEXT) RETURNS VOID AS $$
BEGIN
    COMMIT PREPARED p_xa_id;
END;
$$ LANGUAGE plpgsql;

-- Phase 2b: Rollback (call when any participant voted NO)
CREATE OR REPLACE FUNCTION xa_rollback(p_xa_id TEXT) RETURNS VOID AS $$
BEGIN
    ROLLBACK PREPARED p_xa_id;
END;
$$ LANGUAGE plpgsql;
```

---

## 10. Idempotency Patterns

```sql
-- Idempotency Key: ทำให้ operation ปลอดภัยต่อการ retry

CREATE TABLE idempotency_keys (
    key          TEXT PRIMARY KEY,
    operation    TEXT NOT NULL,
    payload      JSONB,
    result       JSONB,
    status       VARCHAR(20) DEFAULT 'processing',
    created_at   TIMESTAMPTZ DEFAULT NOW(),
    expires_at   TIMESTAMPTZ DEFAULT NOW() + INTERVAL '24 hours'
);

-- Idempotent operation pattern
CREATE OR REPLACE FUNCTION idempotent_operation(
    p_idempotency_key TEXT,
    p_operation       TEXT,
    p_payload         JSONB
) RETURNS JSONB AS $$
DECLARE
    v_existing RECORD;
    v_result   JSONB;
BEGIN
    -- Check if already processed
    SELECT * INTO v_existing FROM idempotency_keys WHERE key = p_idempotency_key;
    
    IF FOUND THEN
        IF v_existing.status = 'completed' THEN
            -- Return cached result
            RETURN v_existing.result || jsonb_build_object('idempotent', true);
        ELSIF v_existing.status = 'processing' THEN
            -- Another request is in progress
            RETURN jsonb_build_object('error', 'Request still processing', 'retry_after', 5);
        END IF;
    END IF;
    
    -- Register idempotency key
    INSERT INTO idempotency_keys (key, operation, payload)
    VALUES (p_idempotency_key, p_operation, p_payload)
    ON CONFLICT (key) DO NOTHING;
    
    -- Perform actual operation
    v_result := CASE p_operation
        WHEN 'transfer' THEN 
            jsonb_build_object('success', true, 'transferred', p_payload->>'amount')
        ELSE
            jsonb_build_object('error', 'Unknown operation')
    END;
    
    -- Mark as completed with result
    UPDATE idempotency_keys
    SET status = 'completed', result = v_result
    WHERE key = p_idempotency_key;
    
    RETURN v_result;
END;
$$ LANGUAGE plpgsql;

-- Cleanup expired idempotency keys
DELETE FROM idempotency_keys WHERE expires_at < NOW();

-- ทดสอบ idempotency
BEGIN;
SELECT idempotent_operation('pay-order-1001-v1', 'transfer', '{"amount": 100}');
-- จะทำงาน
SELECT idempotent_operation('pay-order-1001-v1', 'transfer', '{"amount": 100}');
-- จะ return cached result (idempotent: true)
COMMIT;
```

---

## 11. Change Data Capture (CDC)

```sql
-- CDC ดักจับ changes ใน database เพื่อส่งไปยัง downstream systems

-- Method 1: Trigger-based CDC
CREATE TABLE cdc_changes (
    change_id   BIGSERIAL PRIMARY KEY,
    operation   CHAR(1) NOT NULL,  -- I=Insert, U=Update, D=Delete
    table_name  TEXT NOT NULL,
    row_id      TEXT,
    old_data    JSONB,
    new_data    JSONB,
    changed_at  TIMESTAMPTZ DEFAULT NOW(),
    processed   BOOLEAN DEFAULT FALSE
);

CREATE OR REPLACE FUNCTION capture_changes() RETURNS TRIGGER AS $$
BEGIN
    INSERT INTO cdc_changes (operation, table_name, row_id, old_data, new_data)
    VALUES (
        LEFT(TG_OP, 1),
        TG_TABLE_NAME,
        CASE TG_OP WHEN 'DELETE' THEN OLD.order_id::TEXT ELSE NEW.order_id::TEXT END,
        CASE WHEN TG_OP != 'INSERT' THEN row_to_json(OLD)::JSONB END,
        CASE WHEN TG_OP != 'DELETE' THEN row_to_json(NEW)::JSONB END
    );
    RETURN COALESCE(NEW, OLD);
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER tr_cdc_orders
AFTER INSERT OR UPDATE OR DELETE ON orders
FOR EACH ROW EXECUTE FUNCTION capture_changes();

-- ดู recent changes
SELECT change_id, operation, table_name, row_id, 
       changed_at, processed
FROM cdc_changes
WHERE NOT processed
ORDER BY changed_at
LIMIT 50;

-- Method 2: Logical Replication (production-grade)
-- ใช้ wal2json หรือ pgoutput plugin
-- PostgreSQL WAL-based CDC tools: Debezium, pg_logical
/*
-- สร้าง logical replication slot
SELECT pg_create_logical_replication_slot('debezium_slot', 'pgoutput');

-- ดู replication slots
SELECT slot_name, plugin, slot_type, active, restart_lsn
FROM pg_replication_slots;

-- Release slot
SELECT pg_drop_replication_slot('debezium_slot');
*/
```

---

## 12. Distributed Consistency Monitoring

```sql
-- Monitor distributed system consistency

-- Replication lag alert
CREATE OR REPLACE FUNCTION check_replication_lag(
    p_max_lag_mb  NUMERIC DEFAULT 100,
    p_max_lag_sec INTERVAL DEFAULT '30 seconds'
) RETURNS TABLE(
    replica     TEXT,
    lag_mb      NUMERIC,
    lag_time    INTERVAL,
    status      TEXT
) AS $$
BEGIN
    RETURN QUERY
    SELECT 
        application_name,
        ROUND(pg_wal_lsn_diff(sent_lsn, replay_lsn) / 1024.0 / 1024.0, 2),
        replay_lag,
        CASE 
            WHEN replay_lag > p_max_lag_sec THEN 'CRITICAL: High lag!'
            WHEN pg_wal_lsn_diff(sent_lsn, replay_lsn) / 1024.0 / 1024.0 > p_max_lag_mb 
            THEN 'WARNING: Large WAL backlog'
            ELSE 'OK'
        END
    FROM pg_stat_replication
    ORDER BY replay_lag DESC NULLS LAST;
END;
$$ LANGUAGE plpgsql;

-- Saga health check
CREATE OR REPLACE VIEW saga_health AS
SELECT 
    saga_type,
    status,
    COUNT(*) AS count,
    AVG(EXTRACT(EPOCH FROM (COALESCE(completed_at, NOW()) - started_at))) AS avg_duration_sec,
    MAX(started_at) AS last_saga
FROM saga_instances
WHERE started_at > NOW() - INTERVAL '1 hour'
GROUP BY saga_type, status
ORDER BY saga_type, status;

SELECT * FROM saga_health;

-- Outbox lag
SELECT 
    COUNT(*) FILTER (WHERE status = 'pending') AS pending_events,
    COUNT(*) FILTER (WHERE status = 'delivered') AS delivered_last_hour,
    MIN(created_at) FILTER (WHERE status = 'pending') AS oldest_pending,
    COUNT(*) FILTER (WHERE attempts >= max_attempts) AS dead_letters
FROM outbox_events
WHERE created_at > NOW() - INTERVAL '1 hour';

-- Overall distributed system health
CREATE OR REPLACE VIEW distributed_system_health AS
SELECT 
    'Saga Success Rate' AS metric,
    ROUND(COUNT(*) FILTER (WHERE status = 'completed')::NUMERIC / 
          NULLIF(COUNT(*), 0) * 100, 1)::TEXT || '%' AS value,
    'Last 1 hour' AS window
FROM saga_instances WHERE started_at > NOW() - INTERVAL '1 hour'

UNION ALL

SELECT 
    'Outbox Pending',
    COUNT(*)::TEXT,
    'Current'
FROM outbox_events WHERE status = 'pending'

UNION ALL

SELECT 
    'Prepared Transactions',
    COUNT(*)::TEXT,
    'Stuck > 5 min'
FROM pg_prepared_xacts WHERE prepared < NOW() - INTERVAL '5 minutes';

SELECT * FROM distributed_system_health;
```

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: CAP Theorem Classification

```sql
-- คำตอบ
CREATE OR REPLACE FUNCTION classify_database_cap(p_db_type TEXT)
RETURNS TABLE(
    property    TEXT,
    supported   TEXT,
    trade_off   TEXT
) AS $$
BEGIN
    CASE UPPER(p_db_type)
        WHEN 'POSTGRESQL_SINGLE' THEN
            property := 'Consistency'; supported := 'YES'; trade_off := 'ACID guarantees'; RETURN NEXT;
            property := 'Availability'; supported := 'YES (single node)'; trade_off := 'Single point of failure'; RETURN NEXT;
            property := 'Partition Tolerance'; supported := 'NO'; trade_off := 'Stops if network partition'; RETURN NEXT;
        WHEN 'POSTGRESQL_CLUSTER' THEN
            property := 'Consistency'; supported := 'YES (sync replication)'; trade_off := 'Higher latency'; RETURN NEXT;
            property := 'Availability'; supported := 'PARTIAL'; trade_off := 'May refuse writes during partition'; RETURN NEXT;
            property := 'Partition Tolerance'; supported := 'YES'; trade_off := 'Chooses CP'; RETURN NEXT;
        WHEN 'CASSANDRA' THEN
            property := 'Consistency'; supported := 'Eventual'; trade_off := 'May read stale data'; RETURN NEXT;
            property := 'Availability'; supported := 'YES'; trade_off := 'Always responds'; RETURN NEXT;
            property := 'Partition Tolerance'; supported := 'YES'; trade_off := 'Chooses AP'; RETURN NEXT;
        ELSE
            property := 'Unknown'; supported := 'Unknown'; trade_off := p_db_type || ' not in database'; RETURN NEXT;
    END CASE;
END;
$$ LANGUAGE plpgsql;

SELECT * FROM classify_database_cap('POSTGRESQL_CLUSTER');
SELECT * FROM classify_database_cap('CASSANDRA');
```

### แบบฝึกหัดที่ 2: 2PC Coordinator

```sql
-- คำตอบ
CREATE TABLE xa_transactions (
    xa_id        TEXT PRIMARY KEY,
    status       VARCHAR(20) DEFAULT 'preparing',
    participants JSONB,
    created_at   TIMESTAMPTZ DEFAULT NOW(),
    completed_at TIMESTAMPTZ
);

CREATE OR REPLACE FUNCTION xa_coordinator_begin(
    p_xa_id       TEXT,
    p_participants JSONB  -- ["db1", "db2", "db3"]
) RETURNS VOID AS $$
BEGIN
    INSERT INTO xa_transactions (xa_id, participants)
    VALUES (p_xa_id, p_participants);
END;
$$ LANGUAGE plpgsql;

CREATE OR REPLACE FUNCTION xa_coordinator_vote(
    p_xa_id    TEXT,
    p_node     TEXT,
    p_vote     BOOLEAN  -- TRUE=YES, FALSE=NO
) RETURNS TEXT AS $$
DECLARE
    v_xa RECORD;
    v_votes JSONB;
    v_all_yes BOOLEAN;
BEGIN
    SELECT * INTO v_xa FROM xa_transactions WHERE xa_id = p_xa_id;
    
    -- Record vote
    v_votes := COALESCE(v_xa.participants->'votes', '{}') 
               || jsonb_build_object(p_node, p_vote);
    
    UPDATE xa_transactions 
    SET participants = participants || jsonb_build_object('votes', v_votes)
    WHERE xa_id = p_xa_id;
    
    -- Check if all voted
    v_all_yes := NOT EXISTS (
        SELECT 1 FROM jsonb_each_text(v_votes) WHERE value = 'false'
    );
    
    IF jsonb_object_keys(v_votes)::TEXT = (SELECT COUNT(*)::TEXT FROM jsonb_array_elements_text(v_xa.participants)) THEN
        IF v_all_yes THEN
            UPDATE xa_transactions SET status = 'committed', completed_at = NOW()
            WHERE xa_id = p_xa_id;
            RETURN 'COMMIT';
        ELSE
            UPDATE xa_transactions SET status = 'aborted', completed_at = NOW()
            WHERE xa_id = p_xa_id;
            RETURN 'ROLLBACK';
        END IF;
    END IF;
    
    RETURN 'WAITING';
END;
$$ LANGUAGE plpgsql;
```

### แบบฝึกหัดที่ 3: Simple Saga

```sql
-- คำตอบ
CREATE OR REPLACE FUNCTION bank_transfer_saga(
    p_from_account INT,
    p_to_account   INT,
    p_amount       DECIMAL
) RETURNS JSONB AS $$
DECLARE
    v_saga_id UUID;
    v_from_balance DECIMAL;
    v_step TEXT;
BEGIN
    v_saga_id := gen_random_uuid();
    v_step    := 'debit';
    
    -- Step 1: Debit source account
    SELECT balance INTO v_from_balance
    FROM distributed_accounts 
    WHERE account_id = p_from_account FOR UPDATE;
    
    IF v_from_balance < p_amount THEN
        RETURN jsonb_build_object('success', false, 'error', 'Insufficient funds');
    END IF;
    
    UPDATE distributed_accounts SET balance = balance - p_amount
    WHERE account_id = p_from_account;
    
    v_step := 'credit';
    
    -- Step 2: Credit target account
    UPDATE distributed_accounts SET balance = balance + p_amount
    WHERE account_id = p_to_account;
    
    IF NOT FOUND THEN
        -- Compensate: Rollback debit
        UPDATE distributed_accounts SET balance = balance + p_amount
        WHERE account_id = p_from_account;
        RETURN jsonb_build_object('success', false, 'error', 'Target account not found');
    END IF;
    
    RETURN jsonb_build_object('success', true, 'saga_id', v_saga_id, 'amount', p_amount);
    
EXCEPTION WHEN OTHERS THEN
    -- Compensate based on current step
    IF v_step = 'credit' THEN
        UPDATE distributed_accounts SET balance = balance + p_amount
        WHERE account_id = p_from_account;
    END IF;
    RETURN jsonb_build_object('success', false, 'error', SQLERRM);
END;
$$ LANGUAGE plpgsql;

BEGIN;
INSERT INTO distributed_accounts VALUES (10, 50000), (20, 10000) 
ON CONFLICT (account_id) DO UPDATE SET balance = EXCLUDED.balance;
SELECT bank_transfer_saga(10, 20, 5000);
SELECT account_id, balance FROM distributed_accounts WHERE account_id IN (10, 20);
COMMIT;
```

### แบบฝึกหัดที่ 4: Outbox Monitor

```sql
-- คำตอบ
CREATE OR REPLACE FUNCTION outbox_health_check()
RETURNS TABLE(
    check_name TEXT,
    status     TEXT,
    value      TEXT,
    action     TEXT
) AS $$
DECLARE
    v_pending     INT;
    v_dead_letter INT;
    v_oldest      INTERVAL;
BEGIN
    SELECT COUNT(*), COUNT(*) FILTER (WHERE attempts >= max_attempts),
           NOW() - MIN(created_at)
    INTO v_pending, v_dead_letter, v_oldest
    FROM outbox_events WHERE status = 'pending';
    
    check_name := 'Pending Events';
    value      := v_pending::TEXT;
    status     := CASE WHEN v_pending > 1000 THEN 'CRITICAL' WHEN v_pending > 100 THEN 'WARNING' ELSE 'OK' END;
    action     := CASE WHEN v_pending > 1000 THEN 'Check relay service' ELSE 'Monitor' END;
    RETURN NEXT;
    
    check_name := 'Dead Letter Events';
    value      := v_dead_letter::TEXT;
    status     := CASE WHEN v_dead_letter > 0 THEN 'WARNING' ELSE 'OK' END;
    action     := CASE WHEN v_dead_letter > 0 THEN 'Investigate failed events' ELSE 'None' END;
    RETURN NEXT;
    
    check_name := 'Oldest Pending Age';
    value      := COALESCE(v_oldest::TEXT, 'None');
    status     := CASE WHEN v_oldest > INTERVAL '1 hour' THEN 'CRITICAL' 
                       WHEN v_oldest > INTERVAL '10 minutes' THEN 'WARNING' ELSE 'OK' END;
    action     := CASE WHEN v_oldest > INTERVAL '1 hour' THEN 'Restart relay service immediately' ELSE 'Monitor' END;
    RETURN NEXT;
END;
$$ LANGUAGE plpgsql;

SELECT * FROM outbox_health_check();
```

### แบบฝึกหัดที่ 5: Idempotent Transfer

```sql
-- คำตอบ
CREATE OR REPLACE FUNCTION idempotent_transfer(
    p_idempotency_key TEXT,
    p_from_account    INT,
    p_to_account      INT,
    p_amount          DECIMAL
) RETURNS JSONB AS $$
DECLARE
    v_existing RECORD;
    v_result   JSONB;
BEGIN
    -- Check idempotency
    SELECT * INTO v_existing FROM idempotency_keys WHERE key = p_idempotency_key;
    
    IF FOUND AND v_existing.status = 'completed' THEN
        RETURN v_existing.result || '{"idempotent": true}';
    END IF;
    
    -- Register
    INSERT INTO idempotency_keys (key, operation, payload)
    VALUES (p_idempotency_key, 'transfer', 
            jsonb_build_object('from', p_from_account, 'to', p_to_account, 'amount', p_amount))
    ON CONFLICT (key) DO NOTHING;
    
    -- Perform transfer
    v_result := bank_transfer_saga(p_from_account, p_to_account, p_amount);
    
    -- Save result
    UPDATE idempotency_keys
    SET status = 'completed', result = v_result
    WHERE key = p_idempotency_key;
    
    RETURN v_result;
END;
$$ LANGUAGE plpgsql;

-- ทดสอบ: ส่ง 2 ครั้งด้วย key เดิม → ได้ผลลัพธ์เดิม
BEGIN;
SELECT idempotent_transfer('transfer-20240101-001', 10, 20, 1000);
SELECT idempotent_transfer('transfer-20240101-001', 10, 20, 1000);  -- idempotent = true
COMMIT;
```

### แบบฝึกหัดที่ 6: Saga Recovery

```sql
-- คำตอบ
CREATE OR REPLACE FUNCTION recover_stuck_sagas(
    p_stuck_after INTERVAL DEFAULT '30 minutes'
) RETURNS TABLE(
    saga_id   UUID,
    saga_type TEXT,
    action    TEXT,
    steps     TEXT
) AS $$
DECLARE
    v_saga RECORD;
BEGIN
    FOR v_saga IN
        SELECT s.saga_id, s.saga_type, s.current_step, s.started_at,
               array_agg(st.step_name ORDER BY st.step_id) AS completed_steps
        FROM saga_instances s
        LEFT JOIN saga_steps st ON st.saga_id = s.saga_id AND st.status IN ('completed', 'compensated')
        WHERE s.status IN ('started', 'processing')
          AND s.updated_at < NOW() - p_stuck_after
        GROUP BY s.saga_id, s.saga_type, s.current_step, s.started_at
    LOOP
        -- Trigger compensation
        UPDATE saga_instances 
        SET status = 'compensating', updated_at = NOW()
        WHERE saga_instances.saga_id = v_saga.saga_id;
        
        saga_id   := v_saga.saga_id;
        saga_type := v_saga.saga_type;
        action    := 'COMPENSATING: ' || v_saga.current_step;
        steps     := array_to_string(v_saga.completed_steps, ' → ');
        RETURN NEXT;
    END LOOP;
END;
$$ LANGUAGE plpgsql;

SELECT * FROM recover_stuck_sagas('5 minutes');
```

### แบบฝึกหัดที่ 7: Event Store Snapshot

```sql
-- คำตอบ
CREATE TABLE event_store_snapshots (
    snapshot_id  SERIAL PRIMARY KEY,
    stream_id    TEXT NOT NULL,
    version      INT NOT NULL,
    state        JSONB NOT NULL,
    created_at   TIMESTAMPTZ DEFAULT NOW(),
    UNIQUE(stream_id, version)
);

CREATE OR REPLACE FUNCTION take_snapshot(
    p_stream_id TEXT,
    p_version   INT
) RETURNS VOID AS $$
DECLARE
    v_state JSONB;
BEGIN
    -- Rebuild state up to version
    v_state := '{}';
    -- (ใน real app: apply events one by one)
    
    INSERT INTO event_store_snapshots (stream_id, version, state)
    VALUES (p_stream_id, p_version, v_state)
    ON CONFLICT (stream_id, version) DO NOTHING;
END;
$$ LANGUAGE plpgsql;

-- Load from snapshot + newer events
CREATE OR REPLACE FUNCTION load_aggregate(p_stream_id TEXT)
RETURNS JSONB AS $$
DECLARE
    v_snapshot RECORD;
    v_state    JSONB;
BEGIN
    -- Find latest snapshot
    SELECT * INTO v_snapshot
    FROM event_store_snapshots
    WHERE stream_id = p_stream_id
    ORDER BY version DESC
    LIMIT 1;
    
    IF NOT FOUND THEN
        v_state    := '{}';
    ELSE
        v_state    := v_snapshot.state;
    END IF;
    
    -- Apply events after snapshot
    SELECT COUNT(*) INTO v_state FROM event_store
    WHERE stream_id = p_stream_id
      AND event_version > COALESCE(v_snapshot.version, 0);
    
    RETURN jsonb_build_object(
        'state', v_state,
        'loaded_from_snapshot', v_snapshot IS NOT NULL,
        'snapshot_version', v_snapshot.version
    );
END;
$$ LANGUAGE plpgsql;
```

### แบบฝึกหัดที่ 8: Distributed Counter (CRDT-inspired)

```sql
-- คำตอบ
-- G-Counter (Grow-only): CRDT ที่ง่ายที่สุด
CREATE TABLE distributed_counter (
    node_id    TEXT NOT NULL,
    counter_id TEXT NOT NULL,
    value      BIGINT NOT NULL DEFAULT 0,
    updated_at TIMESTAMPTZ DEFAULT NOW(),
    PRIMARY KEY (node_id, counter_id)
);

-- Increment (only on local node)
CREATE OR REPLACE FUNCTION increment_counter(
    p_counter_id TEXT,
    p_node_id    TEXT DEFAULT 'node-1',
    p_amount     BIGINT DEFAULT 1
) RETURNS BIGINT AS $$
BEGIN
    INSERT INTO distributed_counter (node_id, counter_id, value)
    VALUES (p_node_id, p_counter_id, p_amount)
    ON CONFLICT (node_id, counter_id) DO UPDATE
    SET value = distributed_counter.value + p_amount,
        updated_at = NOW();
    
    -- Return total (sum across all nodes)
    RETURN (SELECT SUM(value) FROM distributed_counter WHERE counter_id = p_counter_id);
END;
$$ LANGUAGE plpgsql;

-- Merge (during sync between nodes)
CREATE OR REPLACE FUNCTION merge_counter(
    p_counter_id TEXT,
    p_node_id    TEXT,
    p_value      BIGINT
) RETURNS VOID AS $$
BEGIN
    INSERT INTO distributed_counter (node_id, counter_id, value)
    VALUES (p_node_id, p_counter_id, p_value)
    ON CONFLICT (node_id, counter_id) DO UPDATE
    SET value = GREATEST(distributed_counter.value, EXCLUDED.value);  -- Take max!
END;
$$ LANGUAGE plpgsql;

-- Read total
CREATE OR REPLACE FUNCTION get_counter(p_counter_id TEXT) RETURNS BIGINT AS $$
    SELECT COALESCE(SUM(value), 0) FROM distributed_counter WHERE counter_id = p_counter_id;
$$ LANGUAGE SQL;

-- ทดสอบ
BEGIN;
SELECT increment_counter('page_views', 'node-1', 5);
SELECT increment_counter('page_views', 'node-2', 3);
SELECT get_counter('page_views');  -- 8
COMMIT;
```

### แบบฝึกหัดที่ 9: Consistency Check Report

```sql
-- คำตอบ
CREATE OR REPLACE FUNCTION distributed_consistency_report()
RETURNS TABLE(
    system      TEXT,
    metric      TEXT,
    value       TEXT,
    health      TEXT
) AS $$
BEGIN
    -- Replication lag
    FOR system, metric, value, health IN
        SELECT 
            application_name,
            'Replication Lag',
            COALESCE(replay_lag::TEXT, 'N/A'),
            CASE WHEN replay_lag > INTERVAL '30 seconds' THEN 'CRITICAL'
                 WHEN replay_lag > INTERVAL '5 seconds' THEN 'WARNING'
                 ELSE 'OK' END
        FROM pg_stat_replication
    LOOP RETURN NEXT; END LOOP;
    
    -- Prepared transactions
    system := 'PostgreSQL';
    metric := 'Prepared Transactions';
    value  := (SELECT COUNT(*)::TEXT FROM pg_prepared_xacts WHERE prepared < NOW() - INTERVAL '1 hour');
    health := CASE WHEN (SELECT COUNT(*) FROM pg_prepared_xacts WHERE prepared < NOW() - INTERVAL '1 hour') > 0 
              THEN 'WARNING: Stuck prepared transactions' ELSE 'OK' END;
    RETURN NEXT;
    
    -- Outbox
    system := 'Outbox';
    metric := 'Pending Events';
    value  := (SELECT COUNT(*)::TEXT FROM outbox_events WHERE status = 'pending');
    health := CASE WHEN (SELECT COUNT(*) FROM outbox_events WHERE status = 'pending') > 100 
              THEN 'WARNING' ELSE 'OK' END;
    RETURN NEXT;
    
    -- Saga
    system := 'Saga';
    metric := 'Stuck Sagas';
    value  := (SELECT COUNT(*)::TEXT FROM saga_instances 
               WHERE status IN ('started', 'processing') 
               AND updated_at < NOW() - INTERVAL '30 minutes');
    health := CASE WHEN (SELECT COUNT(*) FROM saga_instances 
                         WHERE status IN ('started', 'processing') 
                         AND updated_at < NOW() - INTERVAL '30 minutes') > 0
              THEN 'WARNING' ELSE 'OK' END;
    RETURN NEXT;
END;
$$ LANGUAGE plpgsql;

SELECT * FROM distributed_consistency_report();
```

### แบบฝึกหัดที่ 10: Full Distributed Transaction Test

```sql
-- คำตอบ
CREATE OR REPLACE FUNCTION test_distributed_patterns()
RETURNS TABLE(
    pattern    TEXT,
    test       TEXT,
    passed     BOOLEAN,
    notes      TEXT
) AS $$
DECLARE
    v_result JSONB;
    v_count  INT;
BEGIN
    -- Test 1: Outbox atomicity
    pattern := 'Outbox Pattern';
    test    := 'Order and event created atomically';
    BEGIN
        v_result := create_order_with_event(9001, '{"total": 100}');
        passed := (v_result->>'success')::BOOLEAN;
        notes  := v_result::TEXT;
    EXCEPTION WHEN OTHERS THEN
        passed := FALSE;
        notes  := SQLERRM;
    END;
    RETURN NEXT;
    
    -- Test 2: Idempotency
    pattern := 'Idempotency';
    test    := 'Same key returns same result';
    BEGIN
        v_result := idempotent_operation('test-key-001', 'transfer', '{"amount": 50}');
        v_result := idempotent_operation('test-key-001', 'transfer', '{"amount": 50}');
        passed := (v_result->>'idempotent')::BOOLEAN = TRUE;
        notes  := 'Second call: idempotent = ' || (v_result->>'idempotent');
    EXCEPTION WHEN OTHERS THEN
        passed := FALSE;
        notes  := SQLERRM;
    END;
    RETURN NEXT;
    
    -- Test 3: Saga compensation
    pattern := 'Saga Pattern';
    test    := 'Failed saga triggers compensation';
    BEGIN
        -- Create scenario where payment will fail
        v_result := execute_order_saga(9999, 999, 100, 9999.99);
        -- Payment fail rate is ~20%, so success OR failure is valid
        passed := v_result ? 'saga_id';
        notes  := 'Saga result: ' || (v_result->>'success') || ', ID: ' || (v_result->>'saga_id');
    EXCEPTION WHEN OTHERS THEN
        passed := FALSE;
        notes  := SQLERRM;
    END;
    RETURN NEXT;
    
    -- Test 4: Event store append-only
    pattern := 'Event Sourcing';
    test    := 'Events cannot be deleted';
    BEGIN
        PERFORM append_event('test-stream-1', 'test', 'TestEvent', '{}', 0);
        SELECT COUNT(*) INTO v_count FROM event_store WHERE stream_id = 'test-stream-1';
        passed := v_count > 0;
        notes  := 'Event count: ' || v_count;
    EXCEPTION WHEN OTHERS THEN
        passed := FALSE;
        notes  := SQLERRM;
    END;
    RETURN NEXT;
END;
$$ LANGUAGE plpgsql;

BEGIN;
SELECT * FROM test_distributed_patterns();
COMMIT;
```

---

## สรุป

ใน Part นี้เราได้เรียนรู้:

1. **CAP Theorem**: เลือกได้สองจากสามเสมอ (C/A/P) - Network partition หลีกเลี่ยงไม่ได้

2. **Two-Phase Commit (2PC)**:
   - Phase 1: Prepare - ทุก participant ลง vote
   - Phase 2: Commit หรือ Rollback ตาม votes
   - ปัญหา: Blocking, SPOF, latency

3. **Saga Pattern**: แทน 2PC ด้วย local transactions + compensating transactions
   - Choreography: event-driven, no central coordinator
   - Orchestration: central saga orchestrator

4. **Outbox Pattern**: แก้ dual-write problem โดยเขียน event ใน same DB transaction

5. **Eventual Consistency**: ข้อมูล converge ในที่สุด แต่ไม่ guarantee timing

6. **CQRS**: แยก read และ write models เพื่อ optimize แต่ละอย่าง

7. **Event Sourcing**: เก็บ events แทน state - complete audit trail, time travel

8. **Idempotency**: ทำให้ retry ปลอดภัย ด้วย idempotency keys

9. **CDC (Change Data Capture)**: ดักจับ changes เพื่อส่ง downstream

10. **CRDT**: Data structures ที่ merge ได้โดยอัตโนมัติ ใน distributed systems

---

## บทสรุป Chapter: Transactions and Concurrency (Parts 71-80)

ตลอด 10 parts เราได้ครอบคลุม:

| Part | หัวข้อ | Key Concept |
|------|--------|-------------|
| 071 | ACID Properties | Atomicity, Consistency, Isolation, Durability |
| 072 | Transactions | BEGIN, COMMIT, ROLLBACK |
| 073 | Savepoints | Partial rollback, nested transactions |
| 074 | Isolation Levels | Dirty Read, Non-repeatable, Phantom |
| 075 | Locking | Row/Table locks, Advisory locks |
| 076 | Deadlocks | Detection, prevention, retry |
| 077 | MVCC | xmin/xmax, snapshot isolation, VACUUM |
| 078 | Optimistic vs Pessimistic | Version column, SELECT FOR UPDATE |
| 079 | Connection Pooling | PgBouncer, HikariCP, pool sizing |
| 080 | Distributed Transactions | 2PC, Saga, Outbox, CQRS |
