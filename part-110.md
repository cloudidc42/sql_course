# ตอนที่ 110: Database Backup, Recovery, and High Availability

## บทนำ

การสำรองข้อมูลและการกู้คืนเป็นเรื่องที่ไม่มีนักพัฒนาคนไหนอยากทดสอบจริง แต่ทุกคนต้องเตรียมพร้อม บทนี้ครอบคลุมตั้งแต่กลยุทธ์การ backup, PITR, Streaming Replication, High Availability ไปจนถึง Disaster Recovery Planning

---

## 1. Backup Concepts

### 1.1 RPO และ RTO

```
RPO (Recovery Point Objective): ข้อมูลสูงสุดที่ยอมให้หายไปได้
  - RPO = 0:    ไม่ยอมให้ข้อมูลหาย (synchronous replication)
  - RPO = 1h:   ยอมให้ข้อมูลหายได้ไม่เกิน 1 ชั่วโมง (hourly backup)
  - RPO = 24h:  ยอมให้ข้อมูลหายได้ไม่เกิน 1 วัน (daily backup)

RTO (Recovery Time Objective): เวลาสูงสุดที่ยอมให้ระบบ downtime ได้
  - RTO = 0:    ต้องพร้อมใช้งานตลอดเวลา (HA cluster)
  - RTO = 15m:  กู้คืนได้ภายใน 15 นาที (hot standby)
  - RTO = 1h:   กู้คืนได้ภายใน 1 ชั่วโมง (warm standby)
  - RTO = 4h:   กู้คืนได้ภายใน 4 ชั่วโมง (cold standby/restore from backup)

ประเภท Backup:
- Full Backup: สำรองทุกอย่าง (ช้า แต่ง่ายต่อการกู้คืน)
- Incremental Backup: สำรองเฉพาะส่วนที่เปลี่ยนแปลงจาก backup ล่าสุด
- Differential Backup: สำรองเฉพาะส่วนที่เปลี่ยนแปลงจาก full backup ล่าสุด
- PITR (Point-in-Time Recovery): กู้คืนไปยังจุดเวลาใด ๆ
```

---

## 2. PostgreSQL Backup

### 2.1 pg_dump

```bash
#!/bin/bash
# backup_postgresql.sh

set -euo pipefail

# Configuration
DB_HOST="localhost"
DB_PORT="5432"
DB_NAME="myapp"
DB_USER="postgres"
BACKUP_DIR="/var/backups/postgresql"
RETENTION_DAYS=30
DATE=$(date +%Y%m%d_%H%M%S)

# Create backup directory
mkdir -p "${BACKUP_DIR}/daily"
mkdir -p "${BACKUP_DIR}/weekly"
mkdir -p "${BACKUP_DIR}/monthly"

# Full database backup (custom format - compressed, parallel restore)
pg_dump \
    --host="${DB_HOST}" \
    --port="${DB_PORT}" \
    --username="${DB_USER}" \
    --format=custom \
    --compress=9 \
    --no-acl \
    --no-owner \
    --verbose \
    --file="${BACKUP_DIR}/daily/${DB_NAME}_${DATE}.dump" \
    "${DB_NAME}"

echo "Backup created: ${BACKUP_DIR}/daily/${DB_NAME}_${DATE}.dump"

# Schema-only backup (สำหรับ documentation)
pg_dump \
    --host="${DB_HOST}" \
    --username="${DB_USER}" \
    --schema-only \
    --file="${BACKUP_DIR}/daily/${DB_NAME}_schema_${DATE}.sql" \
    "${DB_NAME}"

# Backup specific tables only
pg_dump \
    --host="${DB_HOST}" \
    --username="${DB_USER}" \
    --format=custom \
    --table=customers \
    --table=orders \
    --table=order_items \
    --file="${BACKUP_DIR}/daily/${DB_NAME}_critical_${DATE}.dump" \
    "${DB_NAME}"

# All databases backup
pg_dumpall \
    --host="${DB_HOST}" \
    --username="${DB_USER}" \
    --globals-only \
    --file="${BACKUP_DIR}/daily/globals_${DATE}.sql"

# Upload to S3
if command -v aws &> /dev/null; then
    aws s3 cp \
        "${BACKUP_DIR}/daily/${DB_NAME}_${DATE}.dump" \
        "s3://my-db-backups/${DB_NAME}/daily/${DATE}.dump" \
        --storage-class STANDARD_IA
    
    echo "Backup uploaded to S3"
fi

# Cleanup old backups
find "${BACKUP_DIR}/daily" -name "*.dump" -mtime +${RETENTION_DAYS} -delete
echo "Old backups cleaned up (older than ${RETENTION_DAYS} days)"

# Verify backup
pg_restore --list "${BACKUP_DIR}/daily/${DB_NAME}_${DATE}.dump" > /dev/null
echo "Backup verification: OK"
```

### 2.2 pg_restore

```bash
#!/bin/bash
# restore_postgresql.sh

BACKUP_FILE=$1
DB_NAME=${2:-"myapp_restored"}
DB_USER="postgres"

if [ -z "$BACKUP_FILE" ]; then
    echo "Usage: $0 <backup_file> [database_name]"
    exit 1
fi

echo "Starting restore from: $BACKUP_FILE"
echo "Target database: $DB_NAME"

# Create target database
createdb -U "${DB_USER}" "${DB_NAME}"

# Restore with parallel jobs (faster)
pg_restore \
    --username="${DB_USER}" \
    --dbname="${DB_NAME}" \
    --jobs=4 \
    --verbose \
    --no-acl \
    --no-owner \
    "${BACKUP_FILE}"

echo "Restore completed successfully"

# Verify row counts
psql -U "${DB_USER}" "${DB_NAME}" -c "
SELECT 
    schemaname,
    tablename,
    n_live_tup AS estimated_rows
FROM pg_stat_user_tables
ORDER BY n_live_tup DESC;
"

# Restore specific tables only
pg_restore \
    --username="${DB_USER}" \
    --dbname="${DB_NAME}" \
    --table=customers \
    --table=orders \
    "${BACKUP_FILE}"
```

---

## 3. PostgreSQL PITR (Point-in-Time Recovery)

### 3.1 WAL Archiving Setup

```bash
# postgresql.conf configuration
wal_level = replica
archive_mode = on
archive_command = 'test ! -f /var/lib/postgresql/wal_archive/%f && cp %p /var/lib/postgresql/wal_archive/%f'

# หรือส่ง WAL ไป S3 โดยตรง
archive_command = 'aws s3 cp %p s3://my-wal-archive/%f'

# Compression
archive_command = 'gzip -c %p > /wal_archive/%f.gz'

# wal-g (popular WAL archiver)
archive_command = 'wal-g wal-push %p'
```

```bash
#!/bin/bash
# setup_basebackup.sh - สร้าง base backup สำหรับ PITR

# Base backup
pg_basebackup \
    --host=localhost \
    --username=replication_user \
    --pgdata=/var/backups/postgresql/basebackup \
    --format=tar \
    --gzip \
    --compress=9 \
    --checkpoint=fast \
    --progress \
    --label="base_backup_$(date +%Y%m%d)"

# wal-g full backup
wal-g backup-push /var/lib/postgresql/data
```

### 3.2 PITR Recovery

```bash
#!/bin/bash
# pitr_recovery.sh - Point-in-Time Recovery

TARGET_TIME="2024-06-15 14:30:00+07"
BACKUP_DIR="/var/lib/postgresql/data_recovery"
WAL_ARCHIVE="/var/lib/postgresql/wal_archive"

echo "Starting PITR recovery to: ${TARGET_TIME}"

# Stop PostgreSQL ถ้ายังรันอยู่
pg_ctlcluster 16 main stop

# Restore base backup
tar -xzf /var/backups/basebackup/base.tar.gz -C "${BACKUP_DIR}"

# Create recovery configuration
cat > "${BACKUP_DIR}/postgresql.conf" << EOF
# Recovery settings
restore_command = 'cp ${WAL_ARCHIVE}/%f %p'
recovery_target_time = '${TARGET_TIME}'
recovery_target_action = 'promote'
EOF

# หรือใช้ recovery.signal (PG 12+)
touch "${BACKUP_DIR}/recovery.signal"

# Start PostgreSQL in recovery mode
pg_ctlcluster 16 main start

echo "Recovery started. Monitor logs for completion."
echo "tail -f /var/log/postgresql/postgresql-16-main.log"
```

---

## 4. MySQL Backup

### 4.1 mysqldump

```bash
#!/bin/bash
# mysql_backup.sh

DB_HOST="localhost"
DB_USER="backup_user"
DB_PASS="${MYSQL_BACKUP_PASSWORD}"  # From environment variable
DB_NAME="myapp"
BACKUP_DIR="/var/backups/mysql"
DATE=$(date +%Y%m%d_%H%M%S)

# Single database backup
mysqldump \
    --host="${DB_HOST}" \
    --user="${DB_USER}" \
    --password="${DB_PASS}" \
    --single-transaction \
    --routines \
    --triggers \
    --events \
    --hex-blob \
    --set-gtid-purged=OFF \
    "${DB_NAME}" | gzip > "${BACKUP_DIR}/${DB_NAME}_${DATE}.sql.gz"

# All databases
mysqldump \
    --host="${DB_HOST}" \
    --user="${DB_USER}" \
    --password="${DB_PASS}" \
    --all-databases \
    --single-transaction \
    --routines \
    --triggers \
    --events \
    --flush-privileges \
    | gzip > "${BACKUP_DIR}/all_databases_${DATE}.sql.gz"

# Schema only
mysqldump \
    --host="${DB_HOST}" \
    --user="${DB_USER}" \
    --password="${DB_PASS}" \
    --no-data \
    "${DB_NAME}" > "${BACKUP_DIR}/${DB_NAME}_schema_${DATE}.sql"

echo "MySQL backup completed: ${DB_NAME}_${DATE}.sql.gz"

# Restore
gunzip -c "${BACKUP_DIR}/${DB_NAME}_${DATE}.sql.gz" | \
    mysql -h "${DB_HOST}" -u "${DB_USER}" -p"${DB_PASS}" "${DB_NAME}"

echo "Restore completed"
```

### 4.2 MySQL Binary Log (PITR)

```bash
# my.cnf configuration
[mysqld]
server-id = 1
log_bin = /var/log/mysql/mysql-bin.log
binlog_format = ROW
expire_logs_days = 7
max_binlog_size = 100M
sync_binlog = 1  # Flush to disk on every write (safest, slower)

# GTID (Global Transaction Identifiers)
gtid_mode = ON
enforce_gtid_consistency = ON

# MySQL PITR Recovery
# 1. Restore full backup
gunzip -c full_backup.sql.gz | mysql -u root -p

# 2. Apply binary logs up to target time
mysqlbinlog \
    --start-datetime="2024-06-15 00:00:00" \
    --stop-datetime="2024-06-15 14:30:00" \
    /var/log/mysql/mysql-bin.000001 \
    /var/log/mysql/mysql-bin.000002 \
    | mysql -u root -p

# หรือใช้ GTID
mysqlbinlog \
    --include-gtids="uuid:1-1000" \
    /var/log/mysql/mysql-bin.000001 \
    | mysql -u root -p
```

---

## 5. Streaming Replication

### 5.1 PostgreSQL Streaming Replication

```bash
# === Primary Server Setup ===

# postgresql.conf
wal_level = replica
max_wal_senders = 10
wal_keep_size = 1024  # MB
hot_standby = on

# pg_hba.conf - allow replication from standby
host replication replication_user 10.0.1.2/32 scram-sha-256

# Create replication user
psql -c "CREATE USER replication_user REPLICATION LOGIN PASSWORD 'replication_pass';"

# === Standby Server Setup ===
# 1. Stop standby
pg_ctlcluster 16 main stop

# 2. Backup from primary
pg_basebackup \
    --host=10.0.1.1 \
    --username=replication_user \
    --pgdata=/var/lib/postgresql/16/main \
    --wal-method=stream \
    --checkpoint=fast \
    --progress

# 3. Create standby.signal
touch /var/lib/postgresql/16/main/standby.signal

# postgresql.conf on standby
primary_conninfo = 'host=10.0.1.1 port=5432 user=replication_user password=replication_pass'
hot_standby = on
hot_standby_feedback = on

# 4. Start standby
pg_ctlcluster 16 main start

# Monitor replication status
# On primary:
SELECT 
    client_addr,
    state,
    sent_lsn,
    write_lsn,
    flush_lsn,
    replay_lsn,
    write_lag,
    flush_lag,
    replay_lag,
    sync_state
FROM pg_stat_replication;

# On standby:
SELECT 
    status,
    receive_start_lsn,
    written_lsn,
    flushed_lsn,
    replayed_lsn,
    last_msg_receipt_time
FROM pg_stat_wal_receiver;
```

### 5.2 Read Replica สำหรับ Load Distribution

```python
# Python: Read/Write Splitting

import random
from sqlalchemy import create_engine
from sqlalchemy.pool import QueuePool

class DatabaseRouter:
    def __init__(self):
        # Write to primary
        self.primary = create_engine(
            "postgresql://user:pass@primary:5432/myapp",
            pool_size=10,
            max_overflow=20,
            pool_timeout=30,
            pool_pre_ping=True
        )
        
        # Read from replicas (load balanced)
        self.replicas = [
            create_engine(
                f"postgresql://user:pass@replica{i}:5432/myapp",
                pool_size=5,
                max_overflow=10,
                pool_pre_ping=True
            )
            for i in range(1, 4)  # 3 replicas
        ]
    
    def get_write_engine(self):
        return self.primary
    
    def get_read_engine(self):
        # Round-robin load balancing
        return random.choice(self.replicas)
    
    def execute_read(self, query, params=None):
        engine = self.get_read_engine()
        with engine.connect() as conn:
            return conn.execute(query, params or {}).fetchall()
    
    def execute_write(self, query, params=None):
        engine = self.get_write_engine()
        with engine.begin() as conn:
            return conn.execute(query, params or {})

# Usage
db = DatabaseRouter()

# Reads go to replica
products = db.execute_read("SELECT * FROM products WHERE is_active = TRUE")

# Writes go to primary
db.execute_write("INSERT INTO orders (customer_id, status) VALUES (:cid, :status)",
                 {'cid': 42, 'status': 'pending'})
```

---

## 6. Failover และ High Availability

### 6.1 Patroni (PostgreSQL HA)

```yaml
# patroni.yml - Patroni configuration

scope: postgres-ha
namespace: /service/
name: postgresql-1

restapi:
  listen: 0.0.0.0:8008
  connect_address: 10.0.1.1:8008

etcd3:
  hosts: 10.0.10.1:2379,10.0.10.2:2379,10.0.10.3:2379

bootstrap:
  dcs:
    ttl: 30
    loop_wait: 10
    retry_timeout: 10
    maximum_lag_on_failover: 1048576  # 1MB
    synchronous_mode: false
    
    postgresql:
      use_pg_rewind: true
      use_slots: true
      parameters:
        wal_level: replica
        hot_standby: "on"
        max_connections: 200
        max_wal_senders: 10
        wal_keep_size: 1024
        
  initdb:
  - encoding: UTF8
  - data-checksums
  
  pg_hba:
  - host replication replicator 10.0.1.0/24 md5
  - host all all 0.0.0.0/0 md5

postgresql:
  listen: 0.0.0.0:5432
  connect_address: 10.0.1.1:5432
  data_dir: /var/lib/postgresql/16/main
  bin_dir: /usr/lib/postgresql/16/bin
  
  authentication:
    replication:
      username: replicator
      password: replication_password
    superuser:
      username: postgres
      password: postgres_password
  
  parameters:
    max_connections: 200
    shared_buffers: 256MB

watchdog:
  mode: automatic
  device: /dev/watchdog
  safety_margin: 5
```

```bash
# Patroni commands
patronictl -c /etc/patroni/patroni.yml list
patronictl -c /etc/patroni/patroni.yml switchover
patronictl -c /etc/patroni/patroni.yml failover
patronictl -c /etc/patroni/patroni.yml restart postgresql-1

# Manual failover
patronictl -c /etc/patroni/patroni.yml failover --master postgresql-1 --candidate postgresql-2 --force
```

### 6.2 PgBouncer (Connection Pooling)

```ini
# pgbouncer.ini

[databases]
myapp = host=10.0.1.1 port=5432 dbname=myapp
myapp_read = host=10.0.1.2 port=5432 dbname=myapp  ; Read replica

[pgbouncer]
listen_addr = 0.0.0.0
listen_port = 6432

auth_type = scram-sha-256
auth_file = /etc/pgbouncer/userlist.txt

pool_mode = transaction  ; session, transaction, or statement

; Connection limits
max_client_conn = 1000
default_pool_size = 25
min_pool_size = 5
reserve_pool_size = 5
reserve_pool_timeout = 3

; Timeouts
server_connect_timeout = 10
server_idle_timeout = 600
query_timeout = 300

; Logging
log_connections = 1
log_disconnections = 1
log_pooler_errors = 1

; Admin
admin_users = postgres
stats_users = stats_user

; TLS
server_tls_sslmode = require
server_tls_ca_file = /etc/pgbouncer/ca.crt
```

```sql
-- Monitor PgBouncer
-- Connect to pgbouncer admin console
SHOW POOLS;
SHOW STATS;
SHOW CLIENTS;
SHOW SERVERS;
SHOW DATABASES;
```

---

## 7. Backup Verification

```bash
#!/bin/bash
# verify_backup.sh - Automated backup verification

BACKUP_FILE=$1
TEST_DB="backup_test_$(date +%s)"
PG_USER="postgres"

echo "=== Backup Verification ==="
echo "File: ${BACKUP_FILE}"
echo "Test DB: ${TEST_DB}"

# 1. Check backup file integrity
if ! pg_restore --list "${BACKUP_FILE}" > /dev/null 2>&1; then
    echo "FAIL: Backup file is corrupted"
    exit 1
fi

echo "PASS: Backup file integrity OK"

# 2. Create test database and restore
createdb -U "${PG_USER}" "${TEST_DB}"

if ! pg_restore \
    -U "${PG_USER}" \
    -d "${TEST_DB}" \
    --jobs=2 \
    "${BACKUP_FILE}" 2>/dev/null; then
    echo "FAIL: Restore failed"
    dropdb -U "${PG_USER}" "${TEST_DB}"
    exit 1
fi

echo "PASS: Restore completed"

# 3. Run validation queries
RESULT=$(psql -U "${PG_USER}" -d "${TEST_DB}" -t -c "
    SELECT json_build_object(
        'tables', (SELECT COUNT(*) FROM information_schema.tables WHERE table_schema = 'public'),
        'customers', (SELECT COUNT(*) FROM customers),
        'orders', (SELECT COUNT(*) FROM orders),
        'products', (SELECT COUNT(*) FROM products)
    )
")

echo "Validation results: ${RESULT}"

# 4. Compare with expected counts (from reference file)
if [ -f "/var/backups/expected_counts.json" ]; then
    EXPECTED=$(cat /var/backups/expected_counts.json)
    echo "Expected: ${EXPECTED}"
    echo "Actual: ${RESULT}"
fi

# 5. Cleanup
dropdb -U "${PG_USER}" "${TEST_DB}"

echo "=== Verification Complete ==="
```

---

## 8. Monitoring Queries

```sql
-- Database Health Monitoring

-- 1. Replication lag monitoring
SELECT 
    client_addr AS replica,
    state,
    sent_lsn - replay_lsn AS replication_lag_bytes,
    EXTRACT(EPOCH FROM replay_lag) AS replication_lag_seconds,
    CASE 
        WHEN EXTRACT(EPOCH FROM replay_lag) > 60 THEN 'CRITICAL'
        WHEN EXTRACT(EPOCH FROM replay_lag) > 10 THEN 'WARNING'
        ELSE 'OK'
    END AS status
FROM pg_stat_replication;

-- 2. Database size monitoring
SELECT 
    datname AS database,
    pg_size_pretty(pg_database_size(datname)) AS size,
    pg_database_size(datname) AS size_bytes
FROM pg_database
WHERE datname NOT LIKE 'template%'
ORDER BY size_bytes DESC;

-- 3. Table sizes
SELECT 
    tablename,
    pg_size_pretty(pg_total_relation_size(tablename::TEXT)) AS total_size,
    pg_size_pretty(pg_relation_size(tablename::TEXT)) AS table_size,
    pg_size_pretty(pg_indexes_size(tablename::TEXT)) AS index_size
FROM pg_tables
WHERE schemaname = 'public'
ORDER BY pg_total_relation_size(tablename::TEXT) DESC
LIMIT 20;

-- 4. Long running queries
SELECT 
    pid,
    now() - query_start AS duration,
    state,
    wait_event_type,
    wait_event,
    LEFT(query, 100) AS query_preview
FROM pg_stat_activity
WHERE state != 'idle'
AND query_start < NOW() - INTERVAL '5 minutes'
ORDER BY duration DESC;

-- 5. Lock monitoring
SELECT 
    blocked.pid AS blocked_pid,
    blocked.usename AS blocked_user,
    blocking.pid AS blocking_pid,
    blocking.usename AS blocking_user,
    now() - blocked.query_start AS blocked_duration,
    LEFT(blocked.query, 100) AS blocked_query,
    LEFT(blocking.query, 100) AS blocking_query
FROM pg_stat_activity blocked
JOIN pg_stat_activity blocking 
    ON blocking.pid = ANY(pg_blocking_pids(blocked.pid))
WHERE blocked.cardinality(pg_blocking_pids(blocked.pid)) > 0;

-- 6. Cache hit ratio
SELECT 
    'index hit rate' AS name,
    ROUND(100.0 * sum(idx_blks_hit) / NULLIF(sum(idx_blks_hit) + sum(idx_blks_read), 0), 2) AS ratio
FROM pg_statio_user_indexes
UNION ALL
SELECT 
    'table hit rate',
    ROUND(100.0 * sum(heap_blks_hit) / NULLIF(sum(heap_blks_hit) + sum(heap_blks_read), 0), 2)
FROM pg_statio_user_tables;

-- 7. Vacuum and analyze status
SELECT 
    schemaname,
    tablename,
    last_vacuum,
    last_autovacuum,
    last_analyze,
    last_autoanalyze,
    n_dead_tup,
    n_live_tup,
    ROUND(100.0 * n_dead_tup / NULLIF(n_live_tup + n_dead_tup, 0), 2) AS dead_pct
FROM pg_stat_user_tables
ORDER BY n_dead_tup DESC
LIMIT 20;

-- 8. Connection pool status
SELECT 
    datname,
    count(*) AS connections,
    count(*) FILTER (WHERE state = 'active') AS active,
    count(*) FILTER (WHERE state = 'idle') AS idle,
    count(*) FILTER (WHERE state = 'idle in transaction') AS idle_in_tx
FROM pg_stat_activity
GROUP BY datname
ORDER BY connections DESC;
```

---

## 9. Automated Backup System

```python
#!/usr/bin/env python3
# backup_manager.py - Automated PostgreSQL Backup Manager

import subprocess
import os
import boto3
import logging
from datetime import datetime, timedelta
from pathlib import Path

logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(levelname)s - %(message)s'
)
logger = logging.getLogger(__name__)


class PostgresBackupManager:
    def __init__(self, config: dict):
        self.db_host = config['db_host']
        self.db_name = config['db_name']
        self.db_user = config['db_user']
        self.backup_dir = Path(config['backup_dir'])
        self.s3_bucket = config.get('s3_bucket')
        self.retention_days = config.get('retention_days', 30)
        
        self.backup_dir.mkdir(parents=True, exist_ok=True)
        
        if self.s3_bucket:
            self.s3 = boto3.client('s3')
    
    def create_backup(self, backup_type: str = 'full') -> Path:
        timestamp = datetime.now().strftime('%Y%m%d_%H%M%S')
        filename = f"{self.db_name}_{backup_type}_{timestamp}.dump"
        backup_path = self.backup_dir / filename
        
        logger.info(f"Starting {backup_type} backup: {backup_path}")
        
        cmd = [
            'pg_dump',
            f'--host={self.db_host}',
            f'--username={self.db_user}',
            '--format=custom',
            '--compress=9',
            f'--file={backup_path}',
            self.db_name
        ]
        
        result = subprocess.run(cmd, capture_output=True, text=True)
        
        if result.returncode != 0:
            logger.error(f"Backup failed: {result.stderr}")
            raise RuntimeError(f"pg_dump failed: {result.stderr}")
        
        size_mb = backup_path.stat().st_size / (1024 * 1024)
        logger.info(f"Backup created: {backup_path} ({size_mb:.1f} MB)")
        
        return backup_path
    
    def verify_backup(self, backup_path: Path) -> bool:
        logger.info(f"Verifying backup: {backup_path}")
        
        result = subprocess.run(
            ['pg_restore', '--list', str(backup_path)],
            capture_output=True
        )
        
        if result.returncode != 0:
            logger.error("Backup verification failed!")
            return False
        
        # Count objects in backup
        objects = len(result.stdout.decode().strip().split('\n'))
        logger.info(f"Backup contains {objects} objects")
        
        return True
    
    def upload_to_s3(self, backup_path: Path) -> str:
        if not self.s3_bucket:
            return None
        
        s3_key = f"{self.db_name}/{datetime.now().strftime('%Y/%m/%d')}/{backup_path.name}"
        
        logger.info(f"Uploading to S3: s3://{self.s3_bucket}/{s3_key}")
        
        self.s3.upload_file(
            str(backup_path),
            self.s3_bucket,
            s3_key,
            ExtraArgs={
                'StorageClass': 'STANDARD_IA',
                'ServerSideEncryption': 'AES256'
            }
        )
        
        s3_url = f"s3://{self.s3_bucket}/{s3_key}"
        logger.info(f"Uploaded: {s3_url}")
        return s3_url
    
    def cleanup_old_backups(self):
        cutoff = datetime.now() - timedelta(days=self.retention_days)
        
        for backup_file in self.backup_dir.glob('*.dump'):
            file_mtime = datetime.fromtimestamp(backup_file.stat().st_mtime)
            
            if file_mtime < cutoff:
                backup_file.unlink()
                logger.info(f"Deleted old backup: {backup_file}")
    
    def run_backup(self):
        try:
            backup_path = self.create_backup()
            
            if not self.verify_backup(backup_path):
                raise RuntimeError("Backup verification failed!")
            
            if self.s3_bucket:
                self.upload_to_s3(backup_path)
            
            self.cleanup_old_backups()
            
            logger.info("Backup process completed successfully")
            return True
            
        except Exception as e:
            logger.error(f"Backup process failed: {e}")
            # Send alert
            self.send_alert(f"Database backup failed: {e}")
            return False
    
    def send_alert(self, message: str):
        # Integration กับ Slack, PagerDuty, etc.
        logger.critical(f"ALERT: {message}")


if __name__ == '__main__':
    config = {
        'db_host': os.getenv('DB_HOST', 'localhost'),
        'db_name': os.getenv('DB_NAME', 'myapp'),
        'db_user': os.getenv('DB_USER', 'postgres'),
        'backup_dir': '/var/backups/postgresql',
        's3_bucket': os.getenv('S3_BACKUP_BUCKET'),
        'retention_days': 30
    }
    
    manager = PostgresBackupManager(config)
    success = manager.run_backup()
    exit(0 if success else 1)
```

---

## 10. Disaster Recovery Plan

```markdown
# Disaster Recovery Plan

## 1. Recovery Scenarios

### Scenario A: Data Corruption (single table)
- Detection: Application errors, data validation failures
- RTO: < 1 hour | RPO: < 5 minutes (with PITR)
- Steps:
  1. Identify affected table and time of corruption
  2. Create fresh database from latest base backup
  3. Apply WAL logs up to point before corruption
  4. Export specific table data
  5. Import to production

### Scenario B: Database Server Failure
- Detection: Monitoring alerts, application unavailability
- RTO: < 15 minutes (with Patroni) | RPO: < 1 second
- Steps:
  1. Patroni automatic failover to standby
  2. Update HAProxy/load balancer if needed
  3. Verify application connectivity
  4. Provision new standby server

### Scenario C: Data Center Failure
- Detection: Multiple monitoring alerts
- RTO: < 1 hour | RPO: < 5 minutes
- Steps:
  1. Activate DR site in secondary region
  2. Point DNS to DR site
  3. Verify data consistency
  4. Notify stakeholders

## 2. Recovery Runbook

### ขั้นตอนทั่วไป
1. ตรวจสอบสถานการณ์และระดับความเสียหาย
2. ประกาศ incident และแจ้งทีมที่เกี่ยวข้อง
3. เริ่มกระบวนการกู้คืนตาม scenario
4. ทดสอบการทำงานก่อน resume traffic
5. บันทึกเหตุการณ์และ lessons learned
```

```bash
#!/bin/bash
# dr_test.sh - DR drill script

echo "=== Disaster Recovery Test ==="
echo "Started: $(date)"

# 1. Create point-in-time snapshot
echo "1. Creating PITR snapshot..."
psql -c "SELECT pg_start_backup('dr_test_$(date +%s)', true);"

# 2. Simulate failure (promote standby)
echo "2. Testing failover..."
patronictl -c /etc/patroni/patroni.yml failover --force

# 3. Verify standby became primary
echo "3. Verifying new primary..."
sleep 10
psql -h pgbouncer -c "SELECT pg_is_in_recovery();"
# Should return 'f' (false = primary)

# 4. Run application health checks
echo "4. Running health checks..."
curl -sf http://localhost:8000/health || echo "HEALTH CHECK FAILED"

# 5. Test data integrity
echo "5. Testing data integrity..."
psql -c "
    SELECT 
        COUNT(*) AS customers,
        (SELECT COUNT(*) FROM orders) AS orders,
        (SELECT COUNT(*) FROM products) AS products;
"

echo "=== DR Test Complete: $(date) ==="
```

---

## แบบฝึกหัด

### ข้อที่ 1: Backup Script

**เฉลย:**
```bash
#!/bin/bash
# complete_backup.sh

DB_NAME=${1:-myapp}
BACKUP_BASE="/var/backups/postgresql"
DATE=$(date +%Y%m%d_%H%M%S)

mkdir -p "${BACKUP_BASE}/daily" "${BACKUP_BASE}/weekly" "${BACKUP_BASE}/monthly"

# Determine backup type
if [ $(date +%d) = "01" ]; then
    TYPE="monthly"
elif [ $(date +%u) = "7" ]; then
    TYPE="weekly"  
else
    TYPE="daily"
fi

BACKUP_FILE="${BACKUP_BASE}/${TYPE}/${DB_NAME}_${DATE}.dump"

# Run backup
pg_dump --format=custom --compress=9 --file="${BACKUP_FILE}" "${DB_NAME}"
echo "Backup: ${BACKUP_FILE} ($(du -sh ${BACKUP_FILE} | cut -f1))"

# Verify
pg_restore --list "${BACKUP_FILE}" > /dev/null && echo "VERIFY: OK" || echo "VERIFY: FAILED"

# Cleanup
case $TYPE in
    daily) find "${BACKUP_BASE}/daily" -mtime +7 -delete ;;
    weekly) find "${BACKUP_BASE}/weekly" -mtime +30 -delete ;;
    monthly) find "${BACKUP_BASE}/monthly" -mtime +365 -delete ;;
esac
```

### ข้อที่ 2-10 (เฉลย ย่อ)

```bash
# ข้อที่ 2: MySQL binary log PITR
# 1. Restore full backup
gunzip -c full_backup.sql.gz | mysql -u root -p myapp

# 2. Apply binary logs up to target time
mysqlbinlog --start-datetime="2024-06-15 00:00:00" \
            --stop-datetime="2024-06-15 14:30:00" \
            /var/log/mysql/mysql-bin.* | mysql -u root -p

# ข้อที่ 3: Replication setup
# Primary: wal_level=replica, max_wal_senders=10, archive_mode=on
# Standby: pg_basebackup then touch standby.signal
# Monitor: SELECT * FROM pg_stat_replication;

# ข้อที่ 4: Connection pooling
cat > /etc/pgbouncer/pgbouncer.ini << 'EOF'
[databases]
myapp = host=primary port=5432 dbname=myapp
myapp_ro = host=replica port=5432 dbname=myapp

[pgbouncer]
pool_mode = transaction
max_client_conn = 500
default_pool_size = 20
EOF
```

```sql
-- ข้อที่ 5: Health monitoring
SELECT 
    COUNT(*) FILTER (WHERE state = 'active') AS active_queries,
    COUNT(*) FILTER (WHERE state = 'idle in transaction') AS stuck_txns,
    MAX(EXTRACT(EPOCH FROM (NOW() - query_start))) FILTER (WHERE state = 'active') AS longest_query_sec,
    pg_size_pretty(pg_database_size(current_database())) AS db_size
FROM pg_stat_activity;

-- ข้อที่ 6: Find and kill long queries
SELECT pg_terminate_backend(pid)
FROM pg_stat_activity
WHERE state = 'active'
AND query_start < NOW() - INTERVAL '10 minutes'
AND query NOT ILIKE '%vacuum%';

-- ข้อที่ 7: Bloat detection
SELECT tablename, 
       n_dead_tup,
       n_live_tup,
       ROUND(100.0 * n_dead_tup / NULLIF(n_live_tup + n_dead_tup, 0), 1) AS dead_pct
FROM pg_stat_user_tables
WHERE n_dead_tup > 10000
ORDER BY dead_pct DESC;

-- ข้อที่ 8: Backup retention audit
SELECT 
    name AS backup_file,
    size,
    modification_time,
    NOW() - modification_time::TIMESTAMP AS age
FROM pg_ls_dir('/var/backups/postgresql/daily') AS name
CROSS JOIN LATERAL (
    SELECT size, modification_time 
    FROM pg_stat_file('/var/backups/postgresql/daily/' || name)
) f
ORDER BY modification_time;
```

```python
# ข้อที่ 9: Automated restore test
def test_restore(backup_file: str, test_db: str = 'restore_test') -> bool:
    import subprocess
    
    try:
        # Create test DB
        subprocess.run(['createdb', test_db], check=True)
        
        # Restore
        subprocess.run([
            'pg_restore', '-d', test_db, '--jobs=2', backup_file
        ], check=True)
        
        # Verify
        result = subprocess.run([
            'psql', '-d', test_db, '-c',
            'SELECT COUNT(*) FROM customers;'
        ], capture_output=True, text=True, check=True)
        
        count = int(result.stdout.split()[2])
        print(f"Restore verified: {count} customers")
        return count > 0
        
    finally:
        subprocess.run(['dropdb', '--if-exists', test_db])

# ข้อที่ 10: DR runbook
RUNBOOK = {
    'data_corruption': {
        'detection': 'Application errors or data validation failures',
        'rto': '< 1 hour',
        'rpo': '< 5 minutes',
        'steps': [
            'Identify corruption scope and timing',
            'Create fresh DB from latest base backup',
            'Apply WAL to point before corruption',
            'Export clean data',
            'Import to production'
        ]
    },
    'server_failure': {
        'detection': 'Monitoring alert, app unavailable',
        'rto': '< 15 minutes',
        'rpo': '< 1 second',
        'steps': [
            'Patroni auto-failover',
            'Verify new primary',
            'Update connection config if needed',
            'Provision new standby'
        ]
    }
}
```

---

## สรุป

บทนี้ครอบคลุม Database Backup, Recovery, และ High Availability อย่างครบถ้วน:

1. **RPO/RTO** - การวางแผน recovery objectives
2. **pg_dump/pg_restore** - Full backups, selective restore
3. **WAL Archiving + PITR** - Point-in-time recovery
4. **MySQL Backup** - mysqldump, binary log PITR
5. **Streaming Replication** - Primary-standby setup
6. **Read Replicas** - Load distribution
7. **Failover** - Patroni HA cluster
8. **Connection Pooling** - PgBouncer
9. **Monitoring** - Replication lag, locks, cache hits
10. **DR Planning** - Runbooks, DR drills

สิ่งสำคัญที่สุด: **ทดสอบ restore เสมอ!** Backup ที่ไม่เคย restore คือ backup ที่ไม่มีคุณค่า ควร schedule automated restore tests ทุกสัปดาห์

---

## สิ้นสุดตอนที่ 101-110

ขอแสดงความยินดี! คุณได้ศึกษาครบทั้ง 10 บทของ Advanced SQL Series:

- **101**: PostgreSQL Advanced Features
- **102**: MySQL/MariaDB Advanced Features  
- **103**: SQLite - Lightweight Powerhouse
- **104**: SQL Server (T-SQL) Advanced Features
- **105**: SQL with Python
- **106**: SQL with Node.js/JavaScript
- **107**: SQL in Web Applications
- **108**: SQL for Data Analysis
- **109**: Database Security
- **110**: Backup, Recovery, and High Availability
