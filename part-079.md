# Part 79: Connection Pooling and Concurrency Patterns

## บทนำ

Database connections มีต้นทุนสูง: การสร้าง connection ใหม่ใช้เวลา 20-100ms และทรัพยากรหน่วยความจำ การใช้ Connection Pooling ช่วยลดต้นทุนเหล่านี้โดยการ **นำ connections กลับมาใช้ซ้ำ** แทนที่จะสร้างใหม่ทุกครั้ง

---

## 1. PostgreSQL Connection Architecture

```
Client Application
       |
  [Connection]  ← expensive to create!
       |
 PostgreSQL Backend Process
   - Allocates ~5-10 MB RAM per connection
   - Creates OS process (fork)
   - Authenticates user
   - Sets up session state

Without Connection Pool:
App → create_conn() → [5ms-100ms] → query → close_conn() → repeat...

With Connection Pool:
App → get_conn_from_pool() → [microseconds] → query → return_to_pool()
```

```sql
-- ดู connections ปัจจุบัน
SELECT 
    state,
    COUNT(*) AS count,
    MAX(now() - state_change) AS max_age
FROM pg_stat_activity
WHERE datname = current_database()
GROUP BY state
ORDER BY count DESC;

-- ดู max connections setting
SHOW max_connections;

-- ดู connections per user
SELECT 
    usename,
    COUNT(*) AS connections,
    COUNT(*) FILTER (WHERE state = 'active') AS active,
    COUNT(*) FILTER (WHERE state = 'idle') AS idle,
    COUNT(*) FILTER (WHERE state = 'idle in transaction') AS idle_in_tx
FROM pg_stat_activity
WHERE datname = current_database()
GROUP BY usename
ORDER BY connections DESC;

-- ดู reserved connections สำหรับ superuser
SHOW superuser_reserved_connections;
-- default = 3 (จาก max_connections ทั้งหมด)

-- คำนวณ available connections
SELECT 
    current_setting('max_connections')::INT AS max_connections,
    current_setting('superuser_reserved_connections')::INT AS reserved,
    current_setting('max_connections')::INT - 
    current_setting('superuser_reserved_connections')::INT AS available_for_users,
    (SELECT COUNT(*) FROM pg_stat_activity WHERE datname = current_database()) AS current_connections
;
```

---

## 2. Connection Pooling Concepts

### Pool Types

```
Session Pooling (สำหรับ PgBouncer):
- 1 client ← พัง → 1 server connection ตลอด session
- ไม่มีประสิทธิภาพกว่า Transaction Pooling แต่รองรับ session-level features

Transaction Pooling (แนะนำ):
- 1 server connection ถูก share ระหว่างหลาย clients
- Assign connection ให้ client เฉพาะตอนที่ client กำลัง execute transaction
- ไม่รองรับ session-level settings, prepared statements (บางโหมด), LISTEN/NOTIFY

Statement Pooling:
- Aggressive: share connection ระหว่าง statements
- ไม่รองรับ multi-statement transactions
- ใช้กรณีพิเศษเท่านั้น
```

```sql
-- สิ่งที่ใช้ได้กับ Transaction Pooling:
-- ✓ BEGIN/COMMIT/ROLLBACK
-- ✓ Prepared statements (with prepare on connection)
-- ✓ Most SQL queries

-- สิ่งที่ใช้ไม่ได้กับ Transaction Pooling:
-- ✗ SET statement ที่ต้องการ persist across transactions
-- ✗ LISTEN/NOTIFY
-- ✗ Advisory locks (session-level)
-- ✗ WITH HOLD cursors
-- ✗ PREPARE without deallocate

-- วิธีแก้ปัญหา SET persistence ใน transaction pooling:
-- ใช้ SET LOCAL แทน SET (จะ reset เมื่อ transaction จบ)
BEGIN;
SET LOCAL search_path = 'myschema';
SELECT * FROM mytable;  -- ใช้ myschema
COMMIT;
-- search_path กลับเป็น default หลัง COMMIT
```

---

## 3. PgBouncer Setup และ Configuration

```ini
# pgbouncer.ini
[databases]
# Database alias
myapp = host=localhost port=5432 dbname=myapp_db

# ชี้ไปหลาย databases
myapp_read = host=replica1 port=5432 dbname=myapp_db
myapp_write = host=primary port=5432 dbname=myapp_db

# ใช้ connection string
analytics = host=analytics-server port=5432 dbname=analytics_db user=analytics_user

[pgbouncer]
# Network settings
listen_port = 6432
listen_addr = 0.0.0.0

# Authentication
auth_type = md5
auth_file = /etc/pgbouncer/userlist.txt

# Pool mode
pool_mode = transaction  # แนะนำสำหรับ web applications

# Pool sizes
default_pool_size = 20      # connections per database-user pair
min_pool_size = 5           # minimum idle connections
reserve_pool_size = 5       # extra connections when pool full
reserve_pool_timeout = 5    # seconds to wait before using reserve

# Limits
max_client_conn = 1000      # maximum client connections
max_db_connections = 50     # maximum connections to a single database
max_user_connections = 0    # 0 = unlimited per user

# Timeouts
server_idle_timeout = 600   # close idle server connections after 600s
client_idle_timeout = 0     # 0 = no timeout
query_timeout = 0           # 0 = no timeout
query_wait_timeout = 120    # max time to wait for connection
client_login_timeout = 60   # login timeout
idle_transaction_timeout = 0 # 0 = disable (careful: idle in tx blocks server conn)

# Logging
log_connections = 0
log_disconnections = 0
log_pooler_errors = 1
stats_period = 60

# Admin
admin_users = pgbouncer_admin
stats_users = pgbouncer_stats

# TLS (production)
# server_tls_sslmode = require
# server_tls_ca_file = /etc/ssl/certs/ca-bundle.crt
```

```
# userlist.txt (passwords are MD5 hashed)
"myapp_user" "md5<hash>"
"pgbouncer_admin" "md5<hash>"
```

```sql
-- ดู PgBouncer stats (connect ผ่าน admin port)
-- psql -p 6432 -U pgbouncer_admin pgbouncer

-- Pool status
SHOW POOLS;
/*
database | user   | cl_active | cl_waiting | sv_active | sv_idle | sv_used | maxwait
myapp    | myapp  | 15        | 3          | 18        | 2       | 0       | 0.5
*/

-- Active connections
SHOW CLIENTS;
SHOW SERVERS;

-- Statistics
SHOW STATS;
/*
database | total_xact_count | total_query_count | total_received | total_sent |
         | total_xact_time  | total_query_time  | total_wait_time | avg_xact_count
*/

-- Config
SHOW CONFIG;

-- Pause/Resume
PAUSE myapp;    -- หยุด new queries (สำหรับ maintenance)
RESUME myapp;   -- เริ่มใหม่

-- Kill specific client
KILL myapp;    -- kill all connections to database myapp

-- Reload config
RELOAD;
```

---

## 4. Pool Sizing Formulas

```
หลักการของ PostgreSQL (Neil Stopford / Percona recommendations):

Total PostgreSQL Connections = (Core Count * 2) + Effective Spindle Count

ตัวอย่าง: 4-core server, SSDs (spindle = 1)
= (4 * 2) + 1 = 9 connections

สำหรับ web applications:
Optimal Pool Size = (Active Threads * Average Query Time) / Response Time Target

ตัวอย่าง:
- 100 concurrent threads
- Average query: 10ms
- Target response: 100ms
= (100 * 0.01) / 0.1 = 10 connections

HikariCP Formula (HikariCP team):
connections = ((core_count * 2) + effective_spindle_count)

เพิ่ม headroom เล็กน้อย:
max_pool_size = formula_result * 1.2
```

```sql
-- คำนวณ recommended pool size สำหรับ server นี้
CREATE OR REPLACE FUNCTION recommend_pool_size()
RETURNS TABLE(
    metric      TEXT,
    value       TEXT,
    notes       TEXT
) AS $$
DECLARE
    v_max_conn    INT;
    v_cpu_count   INT;
    v_reserved    INT;
    v_recommended INT;
BEGIN
    v_max_conn  := current_setting('max_connections')::INT;
    v_reserved  := current_setting('superuser_reserved_connections')::INT;
    -- ดึงจำนวน CPU (PostgreSQL ไม่มี built-in function นี้ แต่ simulate ได้)
    v_cpu_count := 4;  -- Replace with actual CPU count
    
    v_recommended := LEAST(
        (v_cpu_count * 2) + 1,
        v_max_conn - v_reserved - 10  -- เหลือ 10 สำหรับ admin
    );
    
    metric := 'Max PostgreSQL Connections';
    value  := v_max_conn::TEXT;
    notes  := 'Current max_connections setting';
    RETURN NEXT;
    
    metric := 'Available for Apps';
    value  := (v_max_conn - v_reserved)::TEXT;
    notes  := 'After superuser_reserved_connections';
    RETURN NEXT;
    
    metric := 'Recommended Pool Size (per app instance)';
    value  := v_recommended::TEXT;
    notes  := '(cpu_count * 2) + spindle_count, but tuned down for shared pool';
    RETURN NEXT;
    
    metric := 'PgBouncer default_pool_size suggestion';
    value  := LEAST(v_recommended, 20)::TEXT;
    notes  := 'Start conservative, increase based on monitoring';
    RETURN NEXT;
    
    metric := 'Max App Instances at this Pool Size';
    value  := ((v_max_conn - v_reserved - 10) / v_recommended)::TEXT;
    notes  := 'Number of app instances that can safely connect';
    RETURN NEXT;
END;
$$ LANGUAGE plpgsql;

SELECT * FROM recommend_pool_size();
```

---

## 5. HikariCP (Java) Configuration

```java
// HikariCP configuration (Java/Spring Boot)
HikariConfig config = new HikariConfig();

// Connection details
config.setJdbcUrl("jdbc:postgresql://localhost:5432/myapp_db");
config.setUsername("myapp_user");
config.setPassword("password");

// Pool sizing
config.setMinimumIdle(5);          // Minimum idle connections
config.setMaximumPoolSize(20);     // Maximum pool size

// Timeouts
config.setConnectionTimeout(30000);        // 30s to get connection from pool
config.setIdleTimeout(600000);             // 10min: close idle connections
config.setMaxLifetime(1800000);            // 30min: max connection age
config.setKeepaliveTime(300000);           // 5min: send keepalive

// Connection testing
config.setConnectionTestQuery("SELECT 1"); // Test query (optional for PG)
config.setValidationTimeout(5000);         // 5s timeout for validation

// Performance
config.addDataSourceProperty("cachePrepStmts", "true");
config.addDataSourceProperty("prepStmtCacheSize", "250");
config.addDataSourceProperty("prepStmtCacheSqlLimit", "2048");
config.addDataSourceProperty("useServerPrepStmts", "true");

// Pool name (for monitoring)
config.setPoolName("MyApp-DB-Pool");

HikariDataSource ds = new HikariDataSource(config);
```

```yaml
# Spring Boot application.yml
spring:
  datasource:
    url: jdbc:postgresql://localhost:6432/myapp  # ผ่าน PgBouncer
    username: myapp_user
    password: ${DB_PASSWORD}
    hikari:
      minimum-idle: 5
      maximum-pool-size: 20
      connection-timeout: 30000
      idle-timeout: 600000
      max-lifetime: 1800000
      pool-name: "MyAppPool"
      # Connection test
      connection-test-query: "SELECT 1"
      # Properties
      data-source-properties:
        cachePrepStmts: true
        prepStmtCacheSize: 250
```

---

## 6. Monitoring Connection Pool Health

```sql
-- Dashboard สำหรับ monitoring connections
CREATE OR REPLACE VIEW connection_pool_dashboard AS
SELECT
    -- Overall stats
    COUNT(*) AS total_connections,
    COUNT(*) FILTER (WHERE state = 'active') AS active,
    COUNT(*) FILTER (WHERE state = 'idle') AS idle,
    COUNT(*) FILTER (WHERE state = 'idle in transaction') AS idle_in_transaction,
    COUNT(*) FILTER (WHERE state = 'idle in transaction (aborted)') AS idle_in_tx_aborted,
    COUNT(*) FILTER (WHERE wait_event IS NOT NULL AND state = 'active') AS waiting,
    
    -- Capacity
    current_setting('max_connections')::INT AS max_connections,
    ROUND(COUNT(*)::NUMERIC / current_setting('max_connections')::INT * 100, 1) AS utilization_pct,
    
    -- Age of oldest connection
    MAX(now() - backend_start) AS max_connection_age,
    MAX(now() - state_change) FILTER (WHERE state = 'idle in transaction') AS max_idle_in_tx_age,
    
    -- Alerts
    CASE 
        WHEN COUNT(*) > current_setting('max_connections')::INT * 0.9 
        THEN 'CRITICAL: Near max connections!'
        WHEN COUNT(*) > current_setting('max_connections')::INT * 0.7 
        THEN 'WARNING: High connection count'
        ELSE 'OK'
    END AS status
FROM pg_stat_activity
WHERE datname = current_database();

SELECT * FROM connection_pool_dashboard;

-- หา long-running idle in transaction connections
SELECT 
    pid,
    usename,
    application_name,
    client_addr,
    state,
    ROUND(EXTRACT(EPOCH FROM (NOW() - state_change))::NUMERIC, 2) AS idle_seconds,
    LEFT(query, 100) AS last_query
FROM pg_stat_activity
WHERE state = 'idle in transaction'
  AND state_change < NOW() - INTERVAL '60 seconds'
ORDER BY idle_seconds DESC;

-- Kill idle-in-transaction connections (ใช้ด้วยความระมัดระวัง)
SELECT pg_terminate_backend(pid)
FROM pg_stat_activity
WHERE state = 'idle in transaction'
  AND state_change < NOW() - INTERVAL '5 minutes'
  AND datname = current_database();

-- Connection wait analysis
SELECT 
    wait_event_type,
    wait_event,
    COUNT(*) AS waiting_connections,
    STRING_AGG(pid::TEXT, ', ' ORDER BY pid) AS pids
FROM pg_stat_activity
WHERE wait_event IS NOT NULL
  AND state = 'active'
GROUP BY wait_event_type, wait_event
ORDER BY waiting_connections DESC;

-- Connection history (ต้อง enable pg_stat_statements)
SELECT 
    calls,
    total_exec_time / calls AS avg_ms,
    rows / calls AS avg_rows,
    LEFT(query, 100) AS query
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 20;
```

---

## 7. PostgreSQL Connection Settings Tuning

```sql
-- postgresql.conf settings ที่เกี่ยวข้อง
-- (ต้อง superuser หรือแก้ file โดยตรง)

-- Maximum connections
ALTER SYSTEM SET max_connections = 200;

-- Memory per connection
ALTER SYSTEM SET work_mem = '4MB';  -- Per sort/hash operation, per connection!
-- Warning: work_mem * max_connections * sorts_per_query = TOTAL MEMORY

-- Idle transaction timeout (สำคัญมาก!)
ALTER SYSTEM SET idle_in_transaction_session_timeout = '5min';  -- Kill idle-in-tx > 5min
ALTER SYSTEM SET idle_session_timeout = '30min';                  -- Kill idle > 30min (pg14+)

-- Statement timeout
ALTER SYSTEM SET statement_timeout = '30s';  -- Kill queries > 30s
-- Set per-user
ALTER USER myapp_user SET statement_timeout = '60s';

-- Lock timeouts
ALTER SYSTEM SET lock_timeout = '30s';
ALTER SYSTEM SET deadlock_timeout = '1s';

SELECT pg_reload_conf();

-- ดู current settings
SELECT name, setting, unit, context, source 
FROM pg_settings
WHERE name IN (
    'max_connections',
    'work_mem',
    'idle_in_transaction_session_timeout',
    'statement_timeout',
    'lock_timeout'
);
```

---

## 8. Connection Pooling สำหรับ Read Replicas

```
Architecture:
                    ┌─ PgBouncer Write ──→ Primary DB
Application ──→────┤
                    └─ PgBouncer Read ───→ Replica 1
                                        → Replica 2

Load Balancing: HAProxy หรือ Patroni ช่วย route traffic
```

```sql
-- สร้าง function ที่รู้ว่าอยู่บน primary หรือ replica
CREATE OR REPLACE FUNCTION is_primary() RETURNS BOOLEAN AS $$
BEGIN
    RETURN NOT pg_is_in_recovery();
END;
$$ LANGUAGE plpgsql;

-- Check replication lag
SELECT 
    client_addr,
    state,
    sent_lsn,
    write_lsn,
    flush_lsn,
    replay_lsn,
    ROUND((sent_lsn - replay_lsn)::NUMERIC / 1024 / 1024, 2) AS lag_mb,
    write_lag,
    flush_lag,
    replay_lag
FROM pg_stat_replication;

-- Application-level read/write splitting
-- (ตัวอย่างใน Python psycopg2)
/*
import psycopg2

class DBRouter:
    def __init__(self):
        self.write_pool = PgConnectionPool(host='primary:6432')
        self.read_pool = PgConnectionPool(host='pgbouncer-read:6432')
    
    def get_write_conn(self):
        return self.write_pool.get_conn()
    
    def get_read_conn(self):
        # ถ้า replica lag มากเกิน ใช้ primary แทน
        return self.read_pool.get_conn()

# ใช้งาน:
db = DBRouter()

# Writes ไปที่ primary
with db.get_write_conn() as conn:
    conn.execute("INSERT INTO orders ...")

# Reads ไปที่ replica
with db.get_read_conn() as conn:
    results = conn.execute("SELECT * FROM products")
*/
```

---

## 9. Detecting and Preventing Connection Leaks

```sql
-- Connection leak = connection ที่ไม่ถูก return คืน pool

-- หา connections ที่ค้างนาน
CREATE OR REPLACE VIEW suspicious_connections AS
SELECT 
    pid,
    usename,
    application_name,
    client_addr,
    backend_start,
    state,
    state_change,
    NOW() - backend_start AS connection_age,
    NOW() - state_change AS state_duration,
    LEFT(query, 200) AS last_query,
    CASE
        WHEN state = 'idle in transaction' AND state_change < NOW() - INTERVAL '5 minutes'
            THEN 'LEAK: Idle in transaction too long'
        WHEN state = 'active' AND state_change < NOW() - INTERVAL '30 minutes'
            THEN 'SLOW: Query running too long'
        WHEN backend_start < NOW() - INTERVAL '1 day'
            THEN 'OLD: Connection open for more than 1 day'
        ELSE 'OK'
    END AS concern
FROM pg_stat_activity
WHERE pid != pg_backend_pid()
  AND datname = current_database();

SELECT * FROM suspicious_connections WHERE concern != 'OK';

-- Auto-cleanup function (เรียกจาก cron)
CREATE OR REPLACE FUNCTION cleanup_stale_connections(
    p_idle_in_tx_timeout INTERVAL DEFAULT '10 minutes',
    p_long_query_timeout INTERVAL DEFAULT '1 hour'
) RETURNS TABLE(
    action  TEXT,
    pid     INT,
    reason  TEXT
) AS $$
DECLARE
    v_conn RECORD;
BEGIN
    FOR v_conn IN 
        SELECT pid, state, state_change, query
        FROM pg_stat_activity
        WHERE pid != pg_backend_pid()
          AND datname = current_database()
    LOOP
        IF v_conn.state = 'idle in transaction' 
           AND v_conn.state_change < NOW() - p_idle_in_tx_timeout THEN
            PERFORM pg_terminate_backend(v_conn.pid);
            action := 'TERMINATED';
            pid    := v_conn.pid;
            reason := format('Idle in transaction for %s', 
                            NOW() - v_conn.state_change);
            RETURN NEXT;
        ELSIF v_conn.state = 'active'
              AND v_conn.state_change < NOW() - p_long_query_timeout THEN
            PERFORM pg_terminate_backend(v_conn.pid);
            action := 'TERMINATED';
            pid    := v_conn.pid;
            reason := format('Long running query: %s', 
                            NOW() - v_conn.state_change);
            RETURN NEXT;
        END IF;
    END LOOP;
END;
$$ LANGUAGE plpgsql;

-- ทดสอบ (dry run - ดูก่อนว่าจะ kill อะไร)
SELECT * FROM suspicious_connections WHERE concern != 'OK';

-- Execute cleanup
BEGIN;
SELECT * FROM cleanup_stale_connections('5 minutes', '30 minutes');
COMMIT;
```

---

## 10. Connection Pooling Patterns สำหรับ Serverless

```sql
-- Serverless functions สร้าง connection ใหม่ทุก invocation
-- ถ้าไม่มี connection pooler → database จะ overloaded

-- Architecture สำหรับ AWS Lambda / Serverless:
/*
Lambda Function (many instances)
    → AWS RDS Proxy (connection pooler)
    → RDS PostgreSQL

หรือ:
Lambda Functions
    → PgBouncer (self-managed, on EC2/ECS)
    → PostgreSQL
*/

-- ตั้งค่า PostgreSQL สำหรับ serverless workload:
ALTER SYSTEM SET idle_in_transaction_session_timeout = '60s';
ALTER SYSTEM SET statement_timeout = '30s';

-- Connection-aware query pattern สำหรับ serverless:
-- (code ตัวอย่างใน Node.js)
/*
const { Pool } = require('pg');

// สร้าง pool นอก handler (reused ระหว่าง invocations)
let pool;

const getPool = () => {
  if (!pool) {
    pool = new Pool({
      host: process.env.PGHOST,
      database: process.env.PGDATABASE,
      user: process.env.PGUSER,
      password: process.env.PGPASSWORD,
      max: 2,              // Lambda: ใช้ connections น้อย
      idleTimeoutMillis: 30000,
      connectionTimeoutMillis: 5000,
    });
  }
  return pool;
};

exports.handler = async (event) => {
  const client = await getPool().connect();
  try {
    const result = await client.query('SELECT * FROM products WHERE id = $1', [event.id]);
    return result.rows[0];
  } finally {
    client.release();  // คืน connection กลับ pool
  }
};
*/

-- ดู connections จาก serverless
SELECT 
    application_name,
    client_addr,
    COUNT(*) AS connections
FROM pg_stat_activity
WHERE datname = current_database()
GROUP BY application_name, client_addr
ORDER BY connections DESC
LIMIT 20;
```

---

## 11. Advanced Patterns: Connection Multiplexing

```sql
-- Queue-based Connection Management
-- แทนที่จะให้ทุก request รอ connection ตรงๆ ใช้ queue

-- Work Queue pattern
CREATE TABLE work_queue (
    job_id      BIGSERIAL PRIMARY KEY,
    job_type    VARCHAR(50) NOT NULL,
    payload     JSONB NOT NULL,
    status      VARCHAR(20) DEFAULT 'pending',
    created_at  TIMESTAMPTZ DEFAULT NOW(),
    started_at  TIMESTAMPTZ,
    completed_at TIMESTAMPTZ,
    error       TEXT
);

CREATE INDEX ix_wq_status_created ON work_queue(status, created_at)
WHERE status = 'pending';

-- Worker picks up jobs using SKIP LOCKED
CREATE OR REPLACE FUNCTION process_next_job(
    p_worker_id TEXT DEFAULT 'worker-1'
) RETURNS JSONB AS $$
DECLARE
    v_job RECORD;
BEGIN
    -- Get next pending job without blocking other workers
    SELECT * INTO v_job
    FROM work_queue
    WHERE status = 'pending'
    ORDER BY created_at
    LIMIT 1
    FOR UPDATE SKIP LOCKED;
    
    IF NOT FOUND THEN
        RETURN jsonb_build_object('result', 'no_work');
    END IF;
    
    -- Mark as processing
    UPDATE work_queue
    SET status = 'processing', started_at = NOW()
    WHERE job_id = v_job.job_id;
    
    -- Process (simulate)
    PERFORM pg_sleep(0.1);  -- actual work here
    
    -- Mark as complete
    UPDATE work_queue
    SET status = 'completed', completed_at = NOW()
    WHERE job_id = v_job.job_id;
    
    RETURN jsonb_build_object(
        'result', 'processed',
        'job_id', v_job.job_id,
        'worker', p_worker_id
    );
END;
$$ LANGUAGE plpgsql;

-- แทรกงาน
INSERT INTO work_queue (job_type, payload) VALUES
    ('email', '{"to": "user@example.com", "subject": "Test"}'),
    ('report', '{"report_id": 123}'),
    ('sync', '{"entity": "orders", "since": "2024-01-01"}');

-- Process jobs (รัน concurrent workers)
BEGIN; SELECT * FROM process_next_job('worker-1'); COMMIT;
BEGIN; SELECT * FROM process_next_job('worker-2'); COMMIT;

-- Monitor queue
SELECT status, COUNT(*), AVG(EXTRACT(EPOCH FROM (NOW() - created_at))) AS avg_wait_sec
FROM work_queue
GROUP BY status
ORDER BY status;
```

---

## 12. Connection Pool Monitoring Dashboard

```sql
-- Comprehensive monitoring view
CREATE OR REPLACE VIEW pool_monitoring AS
WITH conn_stats AS (
    SELECT 
        datname,
        state,
        COUNT(*) AS cnt,
        MAX(NOW() - state_change) AS max_duration
    FROM pg_stat_activity
    WHERE datname IS NOT NULL
    GROUP BY datname, state
),
db_totals AS (
    SELECT 
        datname,
        SUM(cnt) AS total_connections,
        SUM(cnt) FILTER (WHERE state = 'active') AS active,
        SUM(cnt) FILTER (WHERE state = 'idle') AS idle,
        SUM(cnt) FILTER (WHERE state = 'idle in transaction') AS idle_in_tx,
        MAX(max_duration) FILTER (WHERE state = 'idle in transaction') AS max_idle_in_tx_time
    FROM conn_stats
    GROUP BY datname
)
SELECT 
    dt.datname,
    dt.total_connections,
    dt.active,
    dt.idle,
    dt.idle_in_tx,
    dt.max_idle_in_tx_time,
    current_setting('max_connections')::INT AS pg_max_conn,
    ROUND(dt.total_connections::NUMERIC / current_setting('max_connections')::INT * 100, 1) AS conn_utilization_pct,
    CASE 
        WHEN dt.idle_in_tx > 5 THEN 'ALERT: Multiple idle-in-transaction connections'
        WHEN dt.total_connections > current_setting('max_connections')::INT * 0.8 
        THEN 'WARNING: High connection utilization'
        ELSE 'OK'
    END AS health_status
FROM db_totals dt
ORDER BY dt.total_connections DESC;

SELECT * FROM pool_monitoring;

-- Hourly connection usage report
CREATE TABLE connection_usage_history (
    recorded_at          TIMESTAMPTZ PRIMARY KEY,
    total_connections    INT,
    active_connections   INT,
    idle_connections     INT,
    idle_in_tx           INT,
    max_conn             INT,
    utilization_pct      NUMERIC
);

-- บันทึก snapshot (เรียกจาก pg_cron ทุกชั่วโมง)
INSERT INTO connection_usage_history
SELECT 
    NOW(),
    COUNT(*),
    COUNT(*) FILTER (WHERE state = 'active'),
    COUNT(*) FILTER (WHERE state = 'idle'),
    COUNT(*) FILTER (WHERE state = 'idle in transaction'),
    current_setting('max_connections')::INT,
    ROUND(COUNT(*)::NUMERIC / current_setting('max_connections')::INT * 100, 1)
FROM pg_stat_activity
WHERE datname = current_database();

-- Trend analysis
SELECT 
    DATE_TRUNC('hour', recorded_at) AS hour,
    AVG(total_connections) AS avg_connections,
    MAX(total_connections) AS peak_connections,
    AVG(idle_in_tx) AS avg_idle_in_tx,
    AVG(utilization_pct) AS avg_utilization
FROM connection_usage_history
WHERE recorded_at > NOW() - INTERVAL '7 days'
GROUP BY DATE_TRUNC('hour', recorded_at)
ORDER BY hour DESC;
```

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: Connection Analysis

```sql
-- คำตอบ
CREATE OR REPLACE FUNCTION analyze_connections()
RETURNS TABLE(
    finding   TEXT,
    severity  TEXT,
    detail    TEXT,
    action    TEXT
) AS $$
BEGIN
    -- Check total utilization
    IF (SELECT COUNT(*)::FLOAT / current_setting('max_connections')::INT 
        FROM pg_stat_activity) > 0.8 THEN
        finding  := 'High Connection Utilization';
        severity := 'WARNING';
        detail   := format('%s/%s connections used', 
            (SELECT COUNT(*) FROM pg_stat_activity),
            current_setting('max_connections'));
        action   := 'Consider increasing max_connections or adding PgBouncer';
        RETURN NEXT;
    END IF;
    
    -- Check idle in transaction
    IF (SELECT COUNT(*) FROM pg_stat_activity 
        WHERE state = 'idle in transaction'
        AND state_change < NOW() - INTERVAL '5 minutes') > 0 THEN
        finding  := 'Stale Idle-in-Transaction Connections';
        severity := 'CRITICAL';
        detail   := format('%s connections idle in transaction > 5 min',
            (SELECT COUNT(*) FROM pg_stat_activity 
             WHERE state = 'idle in transaction' 
             AND state_change < NOW() - INTERVAL '5 minutes'));
        action   := 'Terminate stale connections, check application code for missing COMMIT/ROLLBACK';
        RETURN NEXT;
    END IF;
    
    -- Check long-running queries
    IF (SELECT COUNT(*) FROM pg_stat_activity 
        WHERE state = 'active' 
        AND state_change < NOW() - INTERVAL '30 minutes') > 0 THEN
        finding  := 'Long Running Queries';
        severity := 'WARNING';
        detail   := 'Queries running > 30 minutes detected';
        action   := 'Review and potentially terminate long queries';
        RETURN NEXT;
    END IF;
    
    -- If all OK
    IF NOT FOUND THEN
        finding  := 'Connection Health';
        severity := 'OK';
        detail   := 'No concerning connections detected';
        action   := 'Continue monitoring';
        RETURN NEXT;
    END IF;
END;
$$ LANGUAGE plpgsql;

SELECT * FROM analyze_connections();
```

### แบบฝึกหัดที่ 2: Pool Configuration Calculator

```sql
-- คำตอบ
CREATE OR REPLACE FUNCTION calculate_pool_config(
    p_cpu_cores         INT,
    p_total_memory_gb   INT,
    p_app_instances     INT,
    p_avg_query_ms      INT DEFAULT 50,
    p_target_response_ms INT DEFAULT 200
) RETURNS TABLE(
    setting TEXT,
    recommended_value TEXT,
    reasoning TEXT
) AS $$
DECLARE
    v_pg_max_conn     INT;
    v_per_instance    INT;
    v_pool_size       INT;
    v_work_mem_mb     INT;
BEGIN
    -- PostgreSQL max connections
    v_pg_max_conn := LEAST((p_cpu_cores * 2) + 1 + 20, 500);  -- +20 for admin, max 500
    
    -- Pool size per app instance
    v_per_instance := LEAST(
        GREATEST((p_cpu_cores * 2) + 1, 10),
        (v_pg_max_conn - 20) / GREATEST(p_app_instances, 1)
    );
    
    -- PgBouncer pool size
    v_pool_size := v_per_instance * p_app_instances;
    
    -- work_mem calculation (conservative)
    v_work_mem_mb := GREATEST(4, p_total_memory_gb * 1024 / v_pg_max_conn / 4);
    
    setting           := 'max_connections (PostgreSQL)';
    recommended_value := v_pg_max_conn::TEXT;
    reasoning         := format('(cpu_cores * 2) + 1 + 20 admin buffer');
    RETURN NEXT;
    
    setting           := 'default_pool_size (PgBouncer)';
    recommended_value := v_pool_size::TEXT;
    reasoning         := format('%s instances × %s connections each', p_app_instances, v_per_instance);
    RETURN NEXT;
    
    setting           := 'min_pool_size (PgBouncer)';
    recommended_value := GREATEST(5, v_pool_size / 4)::TEXT;
    reasoning         := '25% of max pool, minimum 5';
    RETURN NEXT;
    
    setting           := 'max_client_conn (PgBouncer)';
    recommended_value := (v_pool_size * 5)::TEXT;
    reasoning         := '5× pool size for client connections';
    RETURN NEXT;
    
    setting           := 'work_mem (PostgreSQL)';
    recommended_value := v_work_mem_mb || 'MB';
    reasoning         := format('RAM(%sGB) / max_conn / 4 parallel ops', p_total_memory_gb);
    RETURN NEXT;
    
    setting           := 'idle_in_transaction_session_timeout';
    recommended_value := '300000';  -- 5 minutes in ms
    reasoning         := '5 minutes: kills stuck idle-in-transaction connections';
    RETURN NEXT;
END;
$$ LANGUAGE plpgsql;

-- ตัวอย่าง: 8-core server, 32GB RAM, 4 app instances
SELECT * FROM calculate_pool_config(8, 32, 4);
```

### แบบฝึกหัดที่ 3: Connection Leak Detector

```sql
-- คำตอบ
CREATE TABLE connection_leak_log (
    log_id      SERIAL PRIMARY KEY,
    pid         INT,
    usename     TEXT,
    app_name    TEXT,
    client_addr INET,
    state       TEXT,
    idle_time   INTERVAL,
    last_query  TEXT,
    logged_at   TIMESTAMPTZ DEFAULT NOW()
);

CREATE OR REPLACE FUNCTION detect_and_log_leaks(
    p_idle_in_tx_threshold INTERVAL DEFAULT '5 minutes',
    p_should_kill          BOOLEAN DEFAULT FALSE
) RETURNS INT AS $$
DECLARE
    v_conn  RECORD;
    v_count INT := 0;
BEGIN
    FOR v_conn IN
        SELECT 
            pid, usename, application_name, client_addr, state,
            NOW() - state_change AS idle_time,
            LEFT(query, 500) AS last_query
        FROM pg_stat_activity
        WHERE state = 'idle in transaction'
          AND state_change < NOW() - p_idle_in_tx_threshold
          AND pid != pg_backend_pid()
    LOOP
        -- Log the leak
        INSERT INTO connection_leak_log 
            (pid, usename, app_name, client_addr, state, idle_time, last_query)
        VALUES 
            (v_conn.pid, v_conn.usename, v_conn.application_name, 
             v_conn.client_addr, v_conn.state, v_conn.idle_time, v_conn.last_query);
        
        -- Optionally kill
        IF p_should_kill THEN
            PERFORM pg_terminate_backend(v_conn.pid);
        END IF;
        
        v_count := v_count + 1;
    END LOOP;
    
    RETURN v_count;
END;
$$ LANGUAGE plpgsql;

-- Run detection
BEGIN;
SELECT detect_and_log_leaks('2 minutes', FALSE);
COMMIT;

-- View leak log
SELECT app_name, COUNT(*) AS leak_count, AVG(idle_time) AS avg_idle
FROM connection_leak_log
WHERE logged_at > NOW() - INTERVAL '1 hour'
GROUP BY app_name
ORDER BY leak_count DESC;
```

### แบบฝึกหัดที่ 4: PgBouncer Stats Simulation

```sql
-- คำตอบ
-- จำลอง PgBouncer stats ใน PostgreSQL
CREATE OR REPLACE VIEW pgbouncer_simulated_stats AS
SELECT 
    current_database() AS database,
    COUNT(*) AS cl_active,
    0 AS cl_waiting,
    COUNT(*) FILTER (WHERE state = 'active') AS sv_active,
    COUNT(*) FILTER (WHERE state = 'idle') AS sv_idle,
    0 AS sv_used,
    current_setting('max_connections')::INT AS max_conn,
    -- Pool utilization
    ROUND(COUNT(*)::NUMERIC / current_setting('max_connections')::INT * 100, 1) AS pool_utilization_pct,
    -- Estimated pool health
    CASE
        WHEN COUNT(*) FILTER (WHERE state = 'idle') < 
             current_setting('max_connections')::INT * 0.2
        THEN 'POOL EXHAUSTED RISK'
        ELSE 'HEALTHY'
    END AS pool_health
FROM pg_stat_activity
WHERE datname = current_database();

SELECT * FROM pgbouncer_simulated_stats;
```

### แบบฝึกหัดที่ 5: Connection Pool Stress Test

```sql
-- คำตอบ
CREATE OR REPLACE FUNCTION connection_pool_stress_test(
    p_concurrent_queries INT DEFAULT 20,
    p_query_duration_ms  INT DEFAULT 100
) RETURNS TABLE(
    metric TEXT,
    value  TEXT
) AS $$
DECLARE
    v_start    TIMESTAMPTZ := clock_timestamp();
    v_pid_list INT[];
    i          INT;
BEGIN
    -- Record initial state
    metric := 'Initial Connections';
    value  := (SELECT COUNT(*)::TEXT FROM pg_stat_activity WHERE datname = current_database());
    RETURN NEXT;
    
    metric := 'Max Allowed Connections';
    value  := current_setting('max_connections');
    RETURN NEXT;
    
    -- Simulate concurrent workload
    FOR i IN 1..LEAST(p_concurrent_queries, 100) LOOP
        BEGIN
            PERFORM pg_sleep(p_query_duration_ms / 1000.0 * random());
        EXCEPTION WHEN OTHERS THEN
            NULL;
        END;
    END LOOP;
    
    metric := 'Test Duration';
    value  := ROUND(EXTRACT(EPOCH FROM (clock_timestamp() - v_start)) * 1000)::TEXT || 'ms';
    RETURN NEXT;
    
    metric := 'Active During Test';
    value  := (SELECT COUNT(*)::TEXT FROM pg_stat_activity 
               WHERE datname = current_database() AND state = 'active');
    RETURN NEXT;
    
    metric := 'Test Result';
    value  := 'Completed without connection exhaustion';
    RETURN NEXT;
END;
$$ LANGUAGE plpgsql;

BEGIN;
SELECT * FROM connection_pool_stress_test(10, 50);
COMMIT;
```

### แบบฝึกหัดที่ 6: Idle Connection Timeout Monitor

```sql
-- คำตอบ
CREATE OR REPLACE FUNCTION monitor_idle_connections(
    p_warn_after  INTERVAL DEFAULT '10 minutes',
    p_kill_after  INTERVAL DEFAULT '30 minutes'
) RETURNS TABLE(
    action  TEXT,
    pid     INT,
    usename TEXT,
    idle_for INTERVAL,
    terminated BOOLEAN
) AS $$
DECLARE
    v_conn RECORD;
BEGIN
    FOR v_conn IN
        SELECT 
            pid, usename, state, state_change,
            NOW() - state_change AS idle_duration
        FROM pg_stat_activity
        WHERE pid != pg_backend_pid()
          AND state IN ('idle', 'idle in transaction')
          AND state_change < NOW() - p_warn_after
    LOOP
        IF v_conn.idle_duration > p_kill_after THEN
            PERFORM pg_terminate_backend(v_conn.pid);
            action := 'TERMINATED';
            terminated := TRUE;
        ELSE
            action := 'WARNING';
            terminated := FALSE;
        END IF;
        
        pid     := v_conn.pid;
        usename := v_conn.usename;
        idle_for := v_conn.idle_duration;
        RETURN NEXT;
    END LOOP;
END;
$$ LANGUAGE plpgsql;

SELECT * FROM monitor_idle_connections('5 minutes', '15 minutes');
```

### แบบฝึกหัดที่ 7: Work Queue Implementation

```sql
-- คำตอบ
CREATE TABLE job_queue (
    id          BIGSERIAL PRIMARY KEY,
    type        VARCHAR(50) NOT NULL,
    payload     JSONB NOT NULL,
    priority    INT DEFAULT 5,
    status      VARCHAR(20) DEFAULT 'pending',
    attempts    INT DEFAULT 0,
    max_attempts INT DEFAULT 3,
    next_attempt TIMESTAMPTZ DEFAULT NOW(),
    created_at  TIMESTAMPTZ DEFAULT NOW(),
    updated_at  TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX ix_jq_pick ON job_queue(priority DESC, next_attempt) 
WHERE status = 'pending';

-- Worker function
CREATE OR REPLACE FUNCTION dequeue_job(p_worker TEXT)
RETURNS JSONB AS $$
DECLARE
    v_job RECORD;
BEGIN
    SELECT * INTO v_job
    FROM job_queue
    WHERE status = 'pending'
      AND attempts < max_attempts
      AND next_attempt <= NOW()
    ORDER BY priority DESC, next_attempt
    LIMIT 1
    FOR UPDATE SKIP LOCKED;
    
    IF NOT FOUND THEN RETURN NULL; END IF;
    
    UPDATE job_queue 
    SET status = 'processing', 
        attempts = attempts + 1,
        updated_at = NOW()
    WHERE id = v_job.id;
    
    RETURN jsonb_build_object(
        'id', v_job.id, 
        'type', v_job.type, 
        'payload', v_job.payload,
        'attempt', v_job.attempts + 1
    );
END;
$$ LANGUAGE plpgsql;

-- Complete or fail job
CREATE OR REPLACE FUNCTION complete_job(p_job_id BIGINT, p_success BOOLEAN, p_error TEXT DEFAULT NULL)
RETURNS VOID AS $$
BEGIN
    IF p_success THEN
        UPDATE job_queue SET status = 'done', updated_at = NOW() WHERE id = p_job_id;
    ELSE
        UPDATE job_queue 
        SET status = CASE WHEN attempts >= max_attempts THEN 'failed' ELSE 'pending' END,
            next_attempt = NOW() + (INTERVAL '1 minute' * POWER(2, attempts - 1)),
            updated_at = NOW()
        WHERE id = p_job_id;
    END IF;
END;
$$ LANGUAGE plpgsql;

-- Test queue
INSERT INTO job_queue (type, payload, priority) VALUES
    ('email', '{"to": "user@example.com"}', 8),
    ('report', '{"id": 1}', 5),
    ('cleanup', '{}', 1);

BEGIN;
SELECT dequeue_job('worker-1');
COMMIT;
```

### แบบฝึกหัดที่ 8: Connection Pool Health Report

```sql
-- คำตอบ
CREATE OR REPLACE FUNCTION generate_pool_health_report()
RETURNS TEXT AS $$
DECLARE
    v_total      INT;
    v_active     INT;
    v_idle       INT;
    v_idle_tx    INT;
    v_max        INT;
    v_utilization NUMERIC;
    v_report     TEXT;
BEGIN
    SELECT 
        COUNT(*),
        COUNT(*) FILTER (WHERE state = 'active'),
        COUNT(*) FILTER (WHERE state = 'idle'),
        COUNT(*) FILTER (WHERE state = 'idle in transaction'),
        current_setting('max_connections')::INT
    INTO v_total, v_active, v_idle, v_idle_tx, v_max
    FROM pg_stat_activity
    WHERE datname = current_database();
    
    v_utilization := ROUND(v_total::NUMERIC / v_max * 100, 1);
    
    v_report := format(
        E'=== Connection Pool Health Report ===\n' ||
        E'Timestamp: %s\n' ||
        E'Database: %s\n\n' ||
        E'Connections:\n' ||
        E'  Total: %s / %s (%.1f%% utilized)\n' ||
        E'  Active: %s\n' ||
        E'  Idle: %s\n' ||
        E'  Idle in Transaction: %s\n\n' ||
        E'Status: %s\n',
        NOW()::TEXT,
        current_database(),
        v_total, v_max, v_utilization,
        v_active, v_idle, v_idle_tx,
        CASE 
            WHEN v_utilization > 90 THEN 'CRITICAL'
            WHEN v_idle_tx > 5 THEN 'WARNING: Idle-in-tx connections'
            WHEN v_utilization > 70 THEN 'WARNING'
            ELSE 'OK'
        END
    );
    
    RETURN v_report;
END;
$$ LANGUAGE plpgsql;

SELECT generate_pool_health_report();
```

### แบบฝึกหัดที่ 9: Per-Application Connection Limits

```sql
-- คำตอบ
-- สร้าง limits per application
CREATE TABLE app_connection_limits (
    app_name    TEXT PRIMARY KEY,
    max_conn    INT NOT NULL,
    warn_at     INT NOT NULL,
    description TEXT
);

INSERT INTO app_connection_limits VALUES
    ('myapp-api', 50, 40, 'Main API server'),
    ('myapp-workers', 20, 15, 'Background job workers'),
    ('analytics', 10, 8, 'Analytics queries');

-- ตรวจสอบ limits
CREATE OR REPLACE VIEW app_connection_status AS
SELECT 
    acl.app_name,
    acl.max_conn,
    acl.warn_at,
    COUNT(psa.pid) AS current_conn,
    acl.max_conn - COUNT(psa.pid) AS remaining,
    ROUND(COUNT(psa.pid)::NUMERIC / acl.max_conn * 100, 1) AS utilization_pct,
    CASE 
        WHEN COUNT(psa.pid) >= acl.max_conn THEN 'CRITICAL: At limit'
        WHEN COUNT(psa.pid) >= acl.warn_at THEN 'WARNING: Near limit'
        ELSE 'OK'
    END AS status
FROM app_connection_limits acl
LEFT JOIN pg_stat_activity psa ON psa.application_name = acl.app_name
GROUP BY acl.app_name, acl.max_conn, acl.warn_at;

SELECT * FROM app_connection_status;
```

### แบบฝึกหัดที่ 10: Complete Pool Tuning Worksheet

```sql
-- คำตอบ
CREATE OR REPLACE FUNCTION pool_tuning_worksheet()
RETURNS TABLE(
    phase       TEXT,
    question    TEXT,
    current_val TEXT,
    recommended TEXT,
    action      TEXT
) AS $$
DECLARE
    v_max_conn    INT := current_setting('max_connections')::INT;
    v_total_conn  INT;
    v_idle_tx     INT;
    v_active      INT;
BEGIN
    SELECT 
        COUNT(*),
        COUNT(*) FILTER (WHERE state = 'idle in transaction'),
        COUNT(*) FILTER (WHERE state = 'active')
    INTO v_total_conn, v_idle_tx, v_active
    FROM pg_stat_activity WHERE datname = current_database();
    
    phase       := '1. Capacity';
    question    := 'Are we near max_connections?';
    current_val := format('%s/%s (%.0f%%)', v_total_conn, v_max_conn, 
                         v_total_conn::FLOAT/v_max_conn*100);
    recommended := '< 70% utilization';
    action      := CASE WHEN v_total_conn > v_max_conn * 0.7 
                   THEN 'Add PgBouncer or increase max_connections'
                   ELSE 'OK' END;
    RETURN NEXT;
    
    phase       := '2. Leaks';
    question    := 'Idle-in-transaction connections?';
    current_val := v_idle_tx::TEXT;
    recommended := '0 long-term';
    action      := CASE WHEN v_idle_tx > 0 
                   THEN 'Set idle_in_transaction_session_timeout'
                   ELSE 'OK' END;
    RETURN NEXT;
    
    phase       := '3. Timeouts';
    question    := 'idle_in_transaction_session_timeout set?';
    current_val := current_setting('idle_in_transaction_session_timeout');
    recommended := '300000 (5 min)';
    action      := CASE WHEN current_setting('idle_in_transaction_session_timeout') = '0'
                   THEN 'SET idle_in_transaction_session_timeout = ''5min'''
                   ELSE 'OK' END;
    RETURN NEXT;
    
    phase       := '4. Workload';
    question    := 'Active query ratio';
    current_val := format('%s active of %s total', v_active, v_total_conn);
    recommended := 'Active > 50% during peak';
    action      := 'Monitor during peak hours';
    RETURN NEXT;
END;
$$ LANGUAGE plpgsql;

SELECT * FROM pool_tuning_worksheet();
```

---

## สรุป

ใน Part นี้เราได้เรียนรู้:

1. **Connection Cost**: สร้าง connection ใหม่ใช้เวลา 20-100ms
2. **PgBouncer**: Connection pooler ที่ช่วยลด connections ที่ PostgreSQL เห็น
3. **Pool Modes**: Session, Transaction (แนะนำ), Statement
4. **Pool Sizing**: `(cpu_cores * 2) + 1` เป็น starting point
5. **HikariCP**: Java connection pool ที่นิยมใช้กับ Spring Boot
6. **Monitoring**: `pg_stat_activity` เป็น view หลักในการ monitor
7. **Connection Leaks**: Idle-in-transaction connections เป็นปัญหาหลัก
8. **Timeouts**: `idle_in_transaction_session_timeout` ควรตั้งเสมอ
9. **Work Queue**: SKIP LOCKED pattern สำหรับ concurrent workers
10. **Serverless**: ต้องใช้ RDS Proxy หรือ PgBouncer เพิ่มเติม

ใน Part 80 เราจะเรียนรู้เรื่อง Distributed Transactions ซึ่งเป็นหัวข้อสุดท้ายของ Chapter นี้
