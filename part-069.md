# Part 069: Partitioning for Performance

## Table Partitioning เพื่อประสิทธิภาพ

---

## บทนำ

Table Partitioning คือการแบ่งตารางขนาดใหญ่ออกเป็นชิ้นเล็กๆ (partitions) ที่สามารถ query ได้แยกกัน ทำให้:
- **Partition Pruning**: Query เฉพาะ partitions ที่จำเป็น ข้ามที่ไม่จำเป็น
- **Parallel Operations**: Process หลาย partitions พร้อมกัน
- **Maintenance**: VACUUM, ANALYZE, Archive บาง partitions ได้อิสระ
- **Performance**: Index เล็กลง, cache efficient มากขึ้น

---

## 1. What is Table Partitioning?

```
ตาราง orders (500 ล้านแถว) ไม่มี partition:
┌──────────────────────────────────────────────────────────┐
│  orders (500M rows)                                      │
│  ┌────────────────────────────────────────────────────┐  │
│  │ 2020 data │ 2021 data │ 2022 data │ 2023 data │... │  │
│  └────────────────────────────────────────────────────┘  │
│  Query: WHERE order_date >= '2024-01-01'                 │
│  ผล: Scan 500M rows → ช้ามาก!                           │
└──────────────────────────────────────────────────────────┘

ตาราง orders พร้อม range partition ตามปี:
┌──────────────────────────────────────────────────────────┐
│  orders (parent)                                         │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐   │
│  │orders_20 │ │orders_21 │ │orders_22 │ │orders_23 │   │
│  │ (100M)   │ │ (110M)   │ │ (120M)   │ │ (130M)   │   │
│  └──────────┘ └──────────┘ └──────────┘ └──────────┘   │
│  Query: WHERE order_date >= '2024-01-01'                 │
│  ผล: Scan เฉพาะ orders_23 (130M rows) → เร็วขึ้น 4x!  │
└──────────────────────────────────────────────────────────┘
```

---

## 2. Range Partitioning

Range partitioning แบ่งข้อมูลตามช่วงค่า เหมาะสำหรับ date/time columns

### PostgreSQL Declarative Partitioning (PG10+)

```sql
-- สร้างตาราง parent (partitioned table)
CREATE TABLE orders (
    order_id    BIGINT,
    customer_id INT,
    order_date  DATE NOT NULL,
    total_amount DECIMAL(12,2),
    status      VARCHAR(20),
    PRIMARY KEY (order_id, order_date)  -- partition key ต้องอยู่ใน PK!
) PARTITION BY RANGE (order_date);

-- สร้าง partitions สำหรับแต่ละปี
CREATE TABLE orders_2022 PARTITION OF orders
    FOR VALUES FROM ('2022-01-01') TO ('2023-01-01');

CREATE TABLE orders_2023 PARTITION OF orders
    FOR VALUES FROM ('2023-01-01') TO ('2024-01-01');

CREATE TABLE orders_2024 PARTITION OF orders
    FOR VALUES FROM ('2024-01-01') TO ('2025-01-01');

-- Default partition (rows ที่ไม่ตรง partition ใดๆ)
CREATE TABLE orders_default PARTITION OF orders DEFAULT;
```

```sql
-- ตรวจสอบ partition structure
SELECT 
    parent.relname AS parent_table,
    child.relname AS partition_name,
    pg_get_expr(child.relpartbound, child.oid) AS partition_bounds
FROM pg_class parent
JOIN pg_inherits ON pg_inherits.inhparent = parent.oid
JOIN pg_class child ON child.oid = pg_inherits.inhrelid
WHERE parent.relname = 'orders';
```

### Range Partitioning แบบ Monthly

```sql
-- Partitioned by month - ดีสำหรับ high-volume systems
CREATE TABLE events (
    event_id    BIGSERIAL,
    user_id     INT,
    event_type  VARCHAR(50),
    event_time  TIMESTAMP NOT NULL,
    payload     JSONB,
    PRIMARY KEY (event_id, event_time)
) PARTITION BY RANGE (event_time);

-- สร้าง monthly partitions
CREATE TABLE events_2024_01 PARTITION OF events
    FOR VALUES FROM ('2024-01-01') TO ('2024-02-01');

CREATE TABLE events_2024_02 PARTITION OF events
    FOR VALUES FROM ('2024-02-01') TO ('2024-03-01');

-- Script เพื่อ generate partition statements อัตโนมัติ
DO $$
DECLARE
    v_start DATE := '2024-01-01';
    v_end   DATE;
    v_month INT;
BEGIN
    FOR v_month IN 1..12 LOOP
        v_end := v_start + INTERVAL '1 month';
        EXECUTE format(
            'CREATE TABLE IF NOT EXISTS events_%s_%s PARTITION OF events 
             FOR VALUES FROM (%L) TO (%L)',
            EXTRACT(YEAR FROM v_start),
            LPAD(EXTRACT(MONTH FROM v_start)::text, 2, '0'),
            v_start,
            v_end
        );
        RAISE NOTICE 'Created partition for %', v_start;
        v_start := v_end;
    END LOOP;
END $$;
```

---

## 3. List Partitioning

List partitioning แบ่งข้อมูลตามค่าที่ระบุ เหมาะสำหรับ categorical data

```sql
-- ตัวอย่าง: Partitioned by region
CREATE TABLE sales (
    sale_id     BIGINT,
    region      VARCHAR(20) NOT NULL,
    sale_date   DATE,
    amount      DECIMAL(12,2),
    PRIMARY KEY (sale_id, region)
) PARTITION BY LIST (region);

-- สร้าง partitions ตาม region
CREATE TABLE sales_north PARTITION OF sales
    FOR VALUES IN ('north', 'north-east', 'north-west');

CREATE TABLE sales_south PARTITION OF sales
    FOR VALUES IN ('south', 'south-east', 'south-west');

CREATE TABLE sales_central PARTITION OF sales
    FOR VALUES IN ('central', 'bangkok');

CREATE TABLE sales_other PARTITION OF sales DEFAULT;

-- Query ที่ใช้ partition pruning:
SELECT * FROM sales WHERE region = 'north';
-- Scan เฉพาะ sales_north!
```

---

## 4. Hash Partitioning

Hash partitioning กระจายข้อมูล evenly ตาม hash ของ column ไม่มี natural order

```sql
-- Hash partitioning สำหรับ even distribution
CREATE TABLE user_sessions (
    session_id  VARCHAR(50) NOT NULL,
    user_id     INT,
    created_at  TIMESTAMP DEFAULT NOW(),
    data        JSONB,
    PRIMARY KEY (session_id)
) PARTITION BY HASH (session_id);

-- สร้าง 4 partitions (0-3)
CREATE TABLE user_sessions_p0 PARTITION OF user_sessions
    FOR VALUES WITH (MODULUS 4, REMAINDER 0);

CREATE TABLE user_sessions_p1 PARTITION OF user_sessions
    FOR VALUES WITH (MODULUS 4, REMAINDER 1);

CREATE TABLE user_sessions_p2 PARTITION OF user_sessions
    FOR VALUES WITH (MODULUS 4, REMAINDER 2);

CREATE TABLE user_sessions_p3 PARTITION OF user_sessions
    FOR VALUES WITH (MODULUS 4, REMAINDER 3);

-- ข้อดี: ข้อมูลกระจาย evenly
-- ข้อเสีย: ไม่ได้รับประโยชน์จาก partition pruning แบบ range/list
-- เหมาะสำหรับ: write scalability, ตาราง session/cache
```

---

## 5. Partition Pruning

Partition pruning คือการที่ planner "ข้าม" partitions ที่ไม่ตรงเงื่อนไข

```sql
-- ตรวจสอบ partition pruning ด้วย EXPLAIN
EXPLAIN SELECT * FROM orders WHERE order_date >= '2024-01-01';

/*
Append  (cost=0.00..2500.00 rows=130000 width=150)
  ->  Seq Scan on orders_2024  (cost=0.00..2500.00 rows=130000 width=150)
        Filter: (order_date >= '2024-01-01'::date)
*/
-- ดู: เฉพาะ orders_2024 ถูก scan! ข้าม 2022, 2023 ทั้งหมด

-- เปรียบเทียบกับ query ที่ span หลาย partitions:
EXPLAIN SELECT * FROM orders WHERE order_date >= '2023-06-01';
/*
Append  (cost=0.00..15000.00 rows=630000 width=150)
  ->  Seq Scan on orders_2023  (cost=0.00..5500.00 rows=130000 width=150)
  ->  Seq Scan on orders_2024  (cost=0.00..9500.00 rows=500000 width=150)
*/
-- Scan 2 partitions (ครึ่งปี 2023 + ทั้งปี 2024)
```

```sql
-- ตรวจสอบว่า runtime pruning ทำงานหรือไม่
-- (สำหรับ parameterized queries)
SHOW enable_partition_pruning;  -- ควร 'on'

SET enable_partition_pruning = ON;

-- EXPLAIN สำหรับ parameterized query
PREPARE p AS SELECT * FROM orders WHERE order_date = $1;
EXPLAIN EXECUTE p('2024-06-15');
-- ควรเห็น "Partitions excluded:" ใน plan
```

---

## 6. Partition Indexes

```sql
-- สร้าง index บน partitioned table (สร้างอัตโนมัติบนทุก partitions)
CREATE INDEX idx_orders_customer ON orders(customer_id);
-- PostgreSQL สร้าง index บน orders_2022, orders_2023, orders_2024 ให้อัตโนมัติ!

CREATE INDEX idx_orders_status_date ON orders(status, order_date);

-- ดู partition indexes
SELECT 
    p.relname AS partition_name,
    i.relname AS index_name,
    ix.indisvalid
FROM pg_class p
JOIN pg_inherits ON pg_inherits.inhrelid = p.oid
JOIN pg_index ix ON ix.indrelid = p.oid
JOIN pg_class i ON i.oid = ix.indexrelid
WHERE pg_inherits.inhparent = 'orders'::regclass
ORDER BY p.relname, i.relname;

-- สร้าง index บน partition เดียวเท่านั้น:
CREATE INDEX idx_orders_2024_customer ON orders_2024(customer_id);
-- ใช้ตอนต้องการ index เฉพาะ partition ใหม่ก่อนที่ global index จะพร้อม
```

---

## 7. MySQL Partitioning

### MySQL Range Partitioning

```sql
-- MySQL Range Partition
CREATE TABLE orders (
    order_id    INT AUTO_INCREMENT,
    customer_id INT,
    order_date  DATE NOT NULL,
    amount      DECIMAL(10,2),
    PRIMARY KEY (order_id, order_date)  -- PK ต้องมี partition key
) PARTITION BY RANGE (YEAR(order_date)) (
    PARTITION p2020 VALUES LESS THAN (2021),
    PARTITION p2021 VALUES LESS THAN (2022),
    PARTITION p2022 VALUES LESS THAN (2023),
    PARTITION p2023 VALUES LESS THAN (2024),
    PARTITION p2024 VALUES LESS THAN (2025),
    PARTITION p_future VALUES LESS THAN MAXVALUE
);

-- Range by UNIX timestamp (เร็วกว่า YEAR())
CREATE TABLE events_mysql (
    id          INT AUTO_INCREMENT,
    event_time  TIMESTAMP NOT NULL,
    data        TEXT,
    PRIMARY KEY (id, event_time)
) PARTITION BY RANGE (UNIX_TIMESTAMP(event_time)) (
    PARTITION p_2024_q1 VALUES LESS THAN (UNIX_TIMESTAMP('2024-04-01')),
    PARTITION p_2024_q2 VALUES LESS THAN (UNIX_TIMESTAMP('2024-07-01')),
    PARTITION p_2024_q3 VALUES LESS THAN (UNIX_TIMESTAMP('2024-10-01')),
    PARTITION p_2024_q4 VALUES LESS THAN (UNIX_TIMESTAMP('2025-01-01')),
    PARTITION p_future  VALUES LESS THAN MAXVALUE
);
```

```sql
-- ดู partition info ใน MySQL
SELECT 
    PARTITION_NAME,
    PARTITION_EXPRESSION,
    PARTITION_DESCRIPTION,
    TABLE_ROWS,
    AVG_ROW_LENGTH,
    DATA_LENGTH,
    INDEX_LENGTH
FROM information_schema.PARTITIONS
WHERE TABLE_SCHEMA = DATABASE()
AND TABLE_NAME = 'orders';

-- EXPLAIN partition pruning ใน MySQL
EXPLAIN PARTITIONS
SELECT * FROM orders WHERE order_date >= '2024-01-01';
-- ดู column "partitions" - เห็นว่า partition ไหนถูก scan
```

---

## 8. Partition Maintenance (Adding/Dropping)

### Adding New Partitions

```sql
-- PostgreSQL: เพิ่ม partition ใหม่สำหรับปีหน้า
CREATE TABLE orders_2025 PARTITION OF orders
    FOR VALUES FROM ('2025-01-01') TO ('2026-01-01');

-- สร้าง indexes บน partition ใหม่ (ถ้า parent มี index อยู่แล้ว จะสร้างอัตโนมัติ)

-- ตรวจสอบ:
SELECT 
    child.relname AS partition_name,
    pg_size_pretty(pg_relation_size(child.oid)) AS size
FROM pg_class parent
JOIN pg_inherits ON pg_inherits.inhparent = parent.oid
JOIN pg_class child ON child.oid = pg_inherits.inhrelid
WHERE parent.relname = 'orders'
ORDER BY child.relname;
```

```sql
-- Automation: สร้าง partition ล่วงหน้า (scheduled job)
CREATE OR REPLACE FUNCTION create_monthly_partition(
    p_table TEXT,
    p_year INT,
    p_month INT
) RETURNS VOID AS $$
DECLARE
    v_partition_name TEXT;
    v_start_date DATE;
    v_end_date DATE;
BEGIN
    v_partition_name := p_table || '_' || p_year || '_' || LPAD(p_month::text, 2, '0');
    v_start_date := make_date(p_year, p_month, 1);
    v_end_date := v_start_date + INTERVAL '1 month';
    
    EXECUTE format(
        'CREATE TABLE IF NOT EXISTS %I PARTITION OF %I 
         FOR VALUES FROM (%L) TO (%L)',
        v_partition_name,
        p_table,
        v_start_date,
        v_end_date
    );
    
    RAISE NOTICE 'Created partition: % (% to %)', 
        v_partition_name, v_start_date, v_end_date;
END;
$$ LANGUAGE plpgsql;

-- ใช้งาน:
SELECT create_monthly_partition('events', 2025, 1);
SELECT create_monthly_partition('events', 2025, 2);
```

### Dropping Old Partitions (Data Archival)

```sql
-- ลบ partition เก่า (เร็วมากกว่า DELETE!)
-- PostgreSQL:
DROP TABLE orders_2019;  -- instant! ไม่ต้อง VACUUM

-- MySQL:
ALTER TABLE orders DROP PARTITION p2019;

-- เทียบกับ DELETE:
DELETE FROM orders WHERE order_date < '2020-01-01';
-- ช้ามาก! ต้องอ่านทุก row, อัพเดต index, generate WAL/binlog

-- ขั้นตอนที่ดีกว่า: Detach แล้ว Archive ก่อน Drop
-- PostgreSQL:
ALTER TABLE orders DETACH PARTITION orders_2019;
-- orders_2019 กลายเป็นตารางอิสระ
-- Archive ด้วย pg_dump:
-- pg_dump -t orders_2019 mydb > orders_2019_archive.sql
-- แล้วค่อย DROP:
DROP TABLE orders_2019;
```

```sql
-- Attach existing table เป็น partition
-- ดีสำหรับ: bulk load โดยไม่กระทบ production
CREATE TABLE orders_2025_new (LIKE orders INCLUDING ALL);
-- Load ข้อมูลเข้า orders_2025_new
-- ...
-- Attach เป็น partition:
ALTER TABLE orders ATTACH PARTITION orders_2025_new
    FOR VALUES FROM ('2025-01-01') TO ('2026-01-01');
-- Fast! ไม่ต้อง copy ข้อมูล
```

---

## 9. When to Use Partitioning

### Partitioning ช่วยได้เมื่อ:

```
✓ ตารางใหญ่มาก (> 10-50GB หรือ > 100M rows)
✓ Query มักกรองตาม partition key (เช่น date range)
✓ Data lifecycle ชัดเจน (archive เก่า, drop เก่า)
✓ ต้องการ parallel operations บน partitions
✓ Index บน partition key เดี่ยวใหญ่เกินไป
✓ Maintenance (VACUUM) บน full table ช้ามาก
```

### Partitioning ไม่ช่วยเมื่อ:

```
✗ Query ไม่ filter ตาม partition key (ต้อง scan ทุก partition)
✗ ตารางเล็ก (< 1GB)
✗ Cross-partition joins บ่อย (overhead)
✗ Partition key มี high write contention (hotspot)
✗ ต้องการ foreign keys ข้าม partitions (PostgreSQL ไม่รองรับ FK บน partitioned tables ในบางกรณี)
```

---

## 10. Complete Example: Time-series Data (Log System)

```sql
-- ========================================
-- ระบบ Log ขนาดใหญ่ด้วย Range Partitioning
-- ========================================

-- Step 1: สร้าง partitioned table
CREATE TABLE application_logs (
    log_id      BIGSERIAL,
    log_level   VARCHAR(10) NOT NULL,  -- DEBUG, INFO, WARN, ERROR, FATAL
    service     VARCHAR(50) NOT NULL,
    message     TEXT,
    context     JSONB,
    created_at  TIMESTAMP NOT NULL DEFAULT NOW(),
    PRIMARY KEY (log_id, created_at)
) PARTITION BY RANGE (created_at);

-- Step 2: สร้าง monthly partitions สำหรับ 2024
DO $$
DECLARE
    v_month INT;
    v_start DATE;
    v_end DATE;
BEGIN
    FOR v_month IN 1..12 LOOP
        v_start := make_date(2024, v_month, 1);
        v_end := v_start + INTERVAL '1 month';
        EXECUTE format(
            'CREATE TABLE application_logs_%s_%s PARTITION OF application_logs 
             FOR VALUES FROM (%L) TO (%L)',
            2024, LPAD(v_month::text, 2, '0'), v_start, v_end
        );
    END LOOP;
END $$;

-- Step 3: สร้าง indexes (สร้างบน parent → apply ทุก partitions)
CREATE INDEX idx_app_logs_service_time 
ON application_logs(service, created_at DESC);

CREATE INDEX idx_app_logs_level_time 
ON application_logs(log_level, created_at DESC) 
WHERE log_level IN ('ERROR', 'FATAL');

CREATE INDEX idx_app_logs_context 
ON application_logs USING GIN(context);

-- Step 4: Insert data (ไปที่ partition ที่ถูกต้องอัตโนมัติ)
INSERT INTO application_logs (log_level, service, message, context)
VALUES ('ERROR', 'payment-service', 'Payment failed', '{"order_id": 12345}');
-- PostgreSQL routes ไปยัง partition ที่ created_at ตรง

-- Step 5: Query examples
-- Query 1: ดู errors ล่าสุดของ service (ใช้ partition pruning + index)
SELECT * FROM application_logs
WHERE service = 'payment-service'
  AND log_level = 'ERROR'
  AND created_at >= NOW() - INTERVAL '24 hours'
ORDER BY created_at DESC
LIMIT 100;

-- Query 2: Count errors per service (ดึงข้อมูลเฉพาะ partition เดือนนี้)
SELECT service, COUNT(*) AS error_count
FROM application_logs
WHERE log_level IN ('ERROR', 'FATAL')
  AND created_at >= DATE_TRUNC('month', NOW())
GROUP BY service
ORDER BY error_count DESC;

-- Step 6: Partition maintenance
-- ดูขนาดแต่ละ partition
SELECT 
    child.relname AS partition_name,
    pg_size_pretty(pg_relation_size(child.oid)) AS partition_size,
    pg_size_pretty(pg_total_relation_size(child.oid)) AS total_size
FROM pg_class parent
JOIN pg_inherits ON pg_inherits.inhparent = parent.oid
JOIN pg_class child ON child.oid = pg_inherits.inhrelid
WHERE parent.relname = 'application_logs'
ORDER BY child.relname;

-- Archive + Drop old partitions (เก็บไว้ 6 เดือน)
-- ตรวจสอบก่อน:
SELECT relname FROM pg_class 
WHERE relname LIKE 'application_logs_2023%';

-- Archive ด้วย DETACH:
ALTER TABLE application_logs DETACH PARTITION application_logs_2023_01;
-- pg_dump application_logs_2023_01 > archive/logs_2023_01.sql
-- DROP TABLE application_logs_2023_01;
```

---

## 11. Complete Example: E-commerce Orders (Large Table)

```sql
-- ========================================
-- ระบบ E-commerce Orders พร้อม Sub-partitions
-- ========================================

-- ตาราง orders ที่มีทั้ง range และ list partitioning
CREATE TABLE orders (
    order_id        BIGSERIAL,
    customer_id     INT NOT NULL,
    region          VARCHAR(20) NOT NULL,
    order_date      DATE NOT NULL,
    total_amount    DECIMAL(12,2),
    status          VARCHAR(20) DEFAULT 'pending',
    PRIMARY KEY (order_id, order_date, region)
) PARTITION BY RANGE (order_date);

-- Year-level partitions
CREATE TABLE orders_2023 PARTITION OF orders
    FOR VALUES FROM ('2023-01-01') TO ('2024-01-01')
    PARTITION BY LIST (region);  -- Sub-partition!

CREATE TABLE orders_2024 PARTITION OF orders
    FOR VALUES FROM ('2024-01-01') TO ('2025-01-01')
    PARTITION BY LIST (region);  -- Sub-partition!

-- Sub-partitions สำหรับ 2024 ตาม region
CREATE TABLE orders_2024_central PARTITION OF orders_2024
    FOR VALUES IN ('central', 'east', 'west');

CREATE TABLE orders_2024_north PARTITION OF orders_2024
    FOR VALUES IN ('north', 'north-east');

CREATE TABLE orders_2024_south PARTITION OF orders_2024
    FOR VALUES IN ('south', 'south-east');

CREATE TABLE orders_2024_other PARTITION OF orders_2024 DEFAULT;

-- Indexes
CREATE INDEX idx_orders_customer ON orders(customer_id, order_date DESC);
CREATE INDEX idx_orders_status ON orders(status, order_date) 
WHERE status IN ('pending', 'processing');

-- Query ที่ใช้ 2-level pruning:
SELECT * FROM orders
WHERE order_date >= '2024-01-01' AND order_date < '2025-01-01'
  AND region = 'central'
  AND status = 'pending';
-- Scan เฉพาะ orders_2024_central partition!
```

---

## 12. Monitoring Partitioned Tables

```sql
-- ดูสถิติการใช้งานแต่ละ partition
SELECT 
    c.relname AS partition_name,
    s.seq_scan,
    s.seq_tup_read,
    s.idx_scan,
    s.idx_tup_fetch,
    s.n_live_tup,
    s.n_dead_tup,
    pg_size_pretty(pg_total_relation_size(c.oid)) AS total_size
FROM pg_stat_user_tables s
JOIN pg_class c ON c.relname = s.relname
WHERE s.relname LIKE 'orders_%'
ORDER BY c.relname;

-- ดูว่า partition pruning ทำงานหรือเปล่า
EXPLAIN (ANALYZE, FORMAT TEXT)
SELECT * FROM orders 
WHERE order_date = '2024-06-15';
-- ดู: จำนวน "Partitions excluded" ใน plan
```

---

## แบบฝึกหัด (10 ข้อ)

**ข้อ 1:** อธิบายความแตกต่างระหว่าง Range, List, และ Hash partitioning พร้อมตัวอย่าง use case

**เฉลยข้อ 1:**
- **Range**: แบ่งตาม range ของค่า เช่น date ranges → ดีสำหรับ time-series, logs, transactions
- **List**: แบ่งตาม discrete values เช่น region, category → ดีสำหรับ categorical data ที่ query แยกกัน
- **Hash**: กระจาย evenly ตาม hash → ดีสำหรับ write scalability โดยไม่มี natural partition key

---

**ข้อ 2:** Partition Pruning คืออะไรและทดสอบได้อย่างไร?

**เฉลยข้อ 2:**
Partition Pruning คือกระบวนการที่ Planner ข้าม partitions ที่แน่นอนว่าไม่มีข้อมูลตาม WHERE clause ทดสอบด้วย EXPLAIN:
```sql
EXPLAIN SELECT * FROM orders WHERE order_date >= '2024-01-01';
-- ดูว่า partitions ก่อนปี 2024 ไม่ปรากฏใน plan
```

---

**ข้อ 3:** ออกแบบ partition strategy สำหรับตาราง `sensor_data` ที่มี 10 ล้านแถวต่อวัน

**เฉลยข้อ 3:**
```sql
CREATE TABLE sensor_data (
    reading_id  BIGSERIAL,
    sensor_id   INT,
    value       FLOAT,
    recorded_at TIMESTAMP NOT NULL,
    PRIMARY KEY (reading_id, recorded_at)
) PARTITION BY RANGE (recorded_at);

-- Monthly partitions (10M/day × 30 = 300M/month)
-- สร้าง function เพื่อ auto-create
CREATE OR REPLACE FUNCTION ensure_monthly_partition(p_date DATE)
RETURNS VOID AS $$
BEGIN
    EXECUTE format(
        'CREATE TABLE IF NOT EXISTS sensor_data_%s_%s 
         PARTITION OF sensor_data 
         FOR VALUES FROM (%L) TO (%L)',
        EXTRACT(YEAR FROM p_date),
        LPAD(EXTRACT(MONTH FROM p_date)::text, 2, '0'),
        DATE_TRUNC('month', p_date),
        DATE_TRUNC('month', p_date) + INTERVAL '1 month'
    );
END;
$$ LANGUAGE plpgsql;

-- Cron job รัน daily เพื่อสร้าง partition ล่วงหน้า:
-- SELECT ensure_monthly_partition(CURRENT_DATE + 32);
```

---

**ข้อ 4:** DROP PARTITION เร็วกว่า DELETE ทำไม?

**เฉลยข้อ 4:**
- **DROP TABLE/PARTITION**: ลบ data files ออกจาก disk ทันที, ไม่ต้องสร้าง WAL records สำหรับแต่ละ row, ไม่ต้องอัพเดต indexes, instant (milliseconds)
- **DELETE**: ต้องสร้าง WAL records ทุก row, อัพเดต indexes ทุก row, mark rows เป็น "dead" (ต้อง VACUUM), ช้ามากสำหรับ large tables (minutes to hours)

---

**ข้อ 5:** Partition key ต้องอยู่ใน Primary Key ใน PostgreSQL เพราะอะไร?

**เฉลยข้อ 5:**
PostgreSQL ต้องการให้ partition key เป็นส่วนหนึ่งของ PK เพราะ:
- Row ต้องถูก route ไปยัง partition เดียวเท่านั้น
- ถ้า PK ไม่มี partition key Uniqueness ไม่สามารถ enforce ได้ข้าม partitions
- เช่น PK = order_id เท่านั้น → row ที่มี order_id เดียวกันอาจอยู่คนละ partition!

ดังนั้น PK ต้องเป็น (order_id, partition_key) เสมอ

---

**ข้อ 6:** เมื่อใดที่ partitioning ไม่ช่วยและอาจทำให้ช้าลง?

**เฉลยข้อ 6:**
1. **Query ไม่มี partition key ใน WHERE**: ต้อง scan ทุก partition = overhead
2. **Cross-partition queries**: Planner ต้อง merge results จากหลาย partitions
3. **Hotspot partition**: rows ใหม่ทั้งหมดไปที่ partition เดียว (เช่น partition วันนี้) → lock contention
4. **ตารางเล็ก**: Overhead ของ partition routing > benefit
5. **Foreign Keys**: PostgreSQL ไม่รองรับ FK กลับไปยัง partitioned table ในบางกรณี

---

**ข้อ 7:** เขียน script เพื่อ monitor ขนาดของแต่ละ partition ใน PostgreSQL

**เฉลยข้อ 7:**
```sql
-- ดูขนาด partitions ของตาราง orders
SELECT 
    child.relname AS partition_name,
    pg_size_pretty(pg_relation_size(child.oid)) AS data_size,
    pg_size_pretty(pg_indexes_size(child.oid)) AS index_size,
    pg_size_pretty(pg_total_relation_size(child.oid)) AS total_size,
    s.n_live_tup AS row_count,
    pg_get_expr(child.relpartbound, child.oid, true) AS bounds
FROM pg_class parent
JOIN pg_inherits ON pg_inherits.inhparent = parent.oid
JOIN pg_class child ON child.oid = pg_inherits.inhrelid
LEFT JOIN pg_stat_user_tables s ON s.relname = child.relname
WHERE parent.relname = 'orders'
ORDER BY child.relname;
```

---

**ข้อ 8:** อธิบาย DETACH PARTITION และ ATTACH PARTITION ใช้เมื่อไหร่?

**เฉลยข้อ 8:**
- **DETACH**: แยก partition ออกจาก parent table → partition กลายเป็นตารางอิสระ ใช้สำหรับ: archiving เก่า, maintenance, ก่อน DROP
- **ATTACH**: นำตาราง existing เข้าเป็น partition ใช้สำหรับ: bulk load data เข้า staging table ก่อน → แล้ว ATTACH ทีเดียว (เร็วกว่า INSERT, ไม่กระทบ production)

```sql
-- Bulk load workflow:
CREATE TABLE orders_2025_staging (LIKE orders INCLUDING ALL);
COPY orders_2025_staging FROM '/path/to/data.csv';
-- ตรวจสอบข้อมูล...
ALTER TABLE orders ATTACH PARTITION orders_2025_staging
    FOR VALUES FROM ('2025-01-01') TO ('2026-01-01');
```

---

**ข้อ 9:** MySQL Partitioning ต่างจาก PostgreSQL อย่างไร?

**เฉลยข้อ 9:**
- **MySQL**: Partition key ต้องเป็นส่วนหนึ่งของ PRIMARY KEY หรือ UNIQUE KEY ทุกตัว, ไม่รองรับ sub-partitioning ซับซ้อน, ต้องใช้ INT expression สำหรับ RANGE (ไม่ใช่ DATE โดยตรง → ต้องใช้ YEAR() หรือ TO_DAYS())
- **PostgreSQL**: Declarative partitioning ยืดหยุ่นกว่า, รองรับ sub-partitioning, ATTACH/DETACH, partition key ต้องอยู่ใน PK, index สร้างบน parent แล้ว propagate อัตโนมัติ

---

**ข้อ 10:** ออกแบบ complete partition strategy สำหรับ financial transactions table (10M rows/day, เก็บ 3 ปี, query ส่วนใหญ่ดูข้อมูล 30 วันล่าสุด)

**เฉลยข้อ 10:**
```sql
-- Strategy: Monthly partitions, keep 36 partitions (3 years)
CREATE TABLE financial_transactions (
    txn_id      BIGSERIAL,
    account_id  INT NOT NULL,
    txn_type    VARCHAR(20),
    amount      DECIMAL(15,2),
    currency    CHAR(3) DEFAULT 'THB',
    created_at  TIMESTAMP NOT NULL DEFAULT NOW(),
    status      VARCHAR(20) DEFAULT 'pending',
    PRIMARY KEY (txn_id, created_at)
) PARTITION BY RANGE (created_at);

-- สร้าง function สำหรับ auto-manage
CREATE OR REPLACE FUNCTION manage_partitions()
RETURNS VOID AS $$
DECLARE
    v_future_date DATE := DATE_TRUNC('month', CURRENT_DATE) + INTERVAL '2 months';
    v_drop_date DATE := DATE_TRUNC('month', CURRENT_DATE) - INTERVAL '36 months';
BEGIN
    -- สร้าง partition ล่วงหน้า 2 เดือน
    PERFORM create_monthly_partition('financial_transactions', 
        EXTRACT(YEAR FROM v_future_date)::int,
        EXTRACT(MONTH FROM v_future_date)::int);
    
    -- Archive และลบ partition เก่ากว่า 3 ปี
    -- (Logic สำหรับ archive และ DROP old partition)
    RAISE NOTICE 'Partitions managed for %', CURRENT_DATE;
END;
$$ LANGUAGE plpgsql;

-- Indexes ที่จำเป็น:
CREATE INDEX idx_txn_account_date 
ON financial_transactions(account_id, created_at DESC);
CREATE INDEX idx_txn_status_date 
ON financial_transactions(status, created_at DESC)
WHERE status IN ('pending', 'processing');

-- Schedule: รัน manage_partitions() ทุกต้นเดือน
-- ผลลัพธ์:
-- Query 30 วัน: scan 1-2 partitions (เร็วมาก)
-- Archive: DROP partition เก่า (instant)
-- Maintenance: VACUUM/ANALYZE partition เดียวได้
```

---

## สรุป

ใน Part 069 เราได้เรียนรู้:

1. **Table Partitioning** คืออะไรและทำงานอย่างไร
2. **Range Partitioning** - สำหรับ date/time data
3. **List Partitioning** - สำหรับ categorical data
4. **Hash Partitioning** - สำหรับ even distribution
5. **PostgreSQL Declarative Partitioning** - สร้าง partitioned tables
6. **MySQL Partitioning** - ข้อแตกต่างและ syntax
7. **Partition Pruning** - ทดสอบด้วย EXPLAIN
8. **Partition Indexes** - Index บน partitioned tables
9. **Maintenance** - ATTACH/DETACH, DROP, Archive
10. **Complete Examples** - Time-series logs, E-commerce orders, Financial transactions

ใน Part 070 เราจะเรียนรู้เกี่ยวกับ Performance Testing และ Benchmarking
