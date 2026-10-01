# Part 090: Database Events and Scheduled Jobs

## บทนำ

Scheduled Jobs หรือ Database Events คือความสามารถในการรัน SQL/Procedures อัตโนมัติตามเวลาที่กำหนด ใช้สำหรับ maintenance tasks, data archival, report generation, และงาน automation ต่างๆ ที่ต้องทำซ้ำๆ ตามตารางเวลา

---

## 1. PostgreSQL pg_cron Extension

```sql
-- ตัวอย่างที่ 1: ติดตั้ง pg_cron
-- ต้องเพิ่มใน postgresql.conf:
-- shared_preload_libraries = 'pg_cron'
-- cron.database_name = 'your_database'

CREATE EXTENSION IF NOT EXISTS pg_cron;

-- ตรวจสอบ extension
SELECT * FROM pg_extension WHERE extname = 'pg_cron';

-- ตัวอย่างที่ 2: Cron Expression Format
-- ┌───────────── minute (0 - 59)
-- │ ┌───────────── hour (0 - 23)
-- │ │ ┌───────────── day of the month (1 - 31)
-- │ │ │ ┌───────────── month (1 - 12)
-- │ │ │ │ ┌───────────── day of the week (0 - 7, Sunday = 0 or 7)
-- │ │ │ │ │
-- * * * * *

-- ตัวอย่าง Cron Expressions:
-- '* * * * *'      ทุกนาที
-- '0 * * * *'      ทุกชั่วโมง (ที่นาทีที่ 0)
-- '0 2 * * *'      ทุกวันตอนตี 2
-- '0 2 * * 0'      ทุกวันอาทิตย์ตอนตี 2
-- '0 2 1 * *'      วันที่ 1 ของทุกเดือน ตอนตี 2
-- '0 2 1 1 *'      วันที่ 1 มกราคมทุกปี ตอนตี 2
-- '*/5 * * * *'    ทุก 5 นาที
-- '0 8-18 * * 1-5' ทุกชั่วโมงระหว่าง 8am-6pm วันจันทร์-ศุกร์

-- ตัวอย่างที่ 3: สร้าง Scheduled Job พื้นฐาน
-- Refresh Materialized View ทุกชั่วโมง
SELECT cron.schedule(
    'refresh-mv-product-sales',        -- job name
    '0 * * * *',                       -- cron expression: ทุกชั่วโมง
    'REFRESH MATERIALIZED VIEW CONCURRENTLY mv_product_sales'
);

-- VACUUM ทุกคืน
SELECT cron.schedule(
    'nightly-vacuum',
    '0 3 * * *',  -- ตี 3 ทุกวัน
    'VACUUM ANALYZE'
);

-- ตัวอย่างที่ 4: Schedule Stored Procedure
SELECT cron.schedule(
    'cleanup-expired-sessions',
    '*/30 * * * *',  -- ทุก 30 นาที
    'CALL sp_cleanup_expired_sessions()'
);

-- ตัวอย่างที่ 5: Schedule SQL Block
SELECT cron.schedule(
    'archive-old-orders',
    '0 2 * * 0',  -- ทุกวันอาทิตย์ตี 2
    $$
    INSERT INTO orders_archive
    SELECT * FROM orders
    WHERE order_date < CURRENT_DATE - INTERVAL '1 year'
      AND status IN ('completed', 'cancelled');
    
    DELETE FROM orders
    WHERE order_date < CURRENT_DATE - INTERVAL '1 year'
      AND status IN ('completed', 'cancelled');
    $$
);
```

---

## 2. Managing pg_cron Jobs

```sql
-- ตัวอย่างที่ 6: ดูรายการ Jobs
SELECT 
    jobid,
    jobname,
    schedule,
    command,
    nodename,
    nodeport,
    database,
    username,
    active
FROM cron.job
ORDER BY jobname;

-- ตัวอย่างที่ 7: ดู Job Run History
SELECT 
    j.jobname,
    r.start_time,
    r.end_time,
    r.status,
    r.return_message,
    EXTRACT(EPOCH FROM (r.end_time - r.start_time)) AS duration_seconds
FROM cron.job_run_details r
JOIN cron.job j ON r.jobid = j.jobid
ORDER BY r.start_time DESC
LIMIT 50;

-- ตัวอย่างที่ 8: ยกเลิก / Enable / Disable Jobs
-- ยกเลิก Job
SELECT cron.unschedule('nightly-vacuum');
SELECT cron.unschedule(1);  -- โดย job ID

-- Disable Job (ไม่รัน แต่ยังเก็บไว้)
UPDATE cron.job SET active = FALSE WHERE jobname = 'cleanup-expired-sessions';

-- Enable Job กลับมา
UPDATE cron.job SET active = TRUE WHERE jobname = 'cleanup-expired-sessions';

-- เปลี่ยน Schedule
UPDATE cron.job SET schedule = '0 4 * * *' WHERE jobname = 'nightly-vacuum';

-- ตัวอย่างที่ 9: Job ที่รันบน Database อื่น
SELECT cron.schedule_in_database(
    'refresh-reporting-views',
    '0 1 * * *',
    'CALL sp_refresh_all_reporting_views()',
    'reporting_db'  -- ชื่อ database
);
```

---

## 3. MySQL Event Scheduler

```sql
-- ตัวอย่างที่ 10: เปิด Event Scheduler
SET GLOBAL event_scheduler = ON;
-- หรือใน my.cnf: event_scheduler = ON

-- ตรวจสอบสถานะ
SHOW VARIABLES LIKE 'event_scheduler';
SELECT @@event_scheduler;

-- ตัวอย่างที่ 11: สร้าง Event พื้นฐาน
CREATE EVENT IF NOT EXISTS ev_daily_cleanup
ON SCHEDULE EVERY 1 DAY
STARTS '2024-01-01 02:00:00'
COMMENT 'Daily cleanup of expired data'
DO
    CALL sp_cleanup_expired_sessions();

-- ตัวอย่างที่ 12: Event ที่รันครั้งเดียว (one-time)
CREATE EVENT ev_one_time_migration
ON SCHEDULE AT '2024-06-15 00:00:00'
DO
BEGIN
    CALL sp_migrate_legacy_data();
    INSERT INTO migration_log (migration_name, completed_at)
    VALUES ('legacy_data_migration', NOW());
END;

-- ตัวอย่างที่ 13: Event ที่รันทุก N ชั่วโมง/นาที
CREATE EVENT ev_hourly_stats
ON SCHEDULE EVERY 1 HOUR
STARTS NOW()
DO
    CALL sp_update_hourly_statistics();

-- ทุก 15 นาที
CREATE EVENT ev_session_cleanup
ON SCHEDULE EVERY 15 MINUTE
DO
    DELETE FROM user_sessions WHERE expires_at < NOW();

-- ตัวอย่างที่ 14: Event ที่มี END time
CREATE EVENT ev_temporary_promotion
ON SCHEDULE EVERY 1 HOUR
STARTS '2024-12-01 00:00:00'
ENDS '2024-12-31 23:59:59'
DO
    CALL sp_apply_holiday_discounts();

-- ตัวอย่างที่ 15: Event พร้อม BEGIN/END block
DELIMITER //

CREATE EVENT ev_comprehensive_maintenance
ON SCHEDULE EVERY 1 DAY
STARTS '2024-01-01 03:00:00'
DO
BEGIN
    -- Step 1: Archive old data
    INSERT INTO orders_archive 
    SELECT * FROM orders 
    WHERE order_date < DATE_SUB(NOW(), INTERVAL 1 YEAR);
    
    DELETE FROM orders 
    WHERE order_date < DATE_SUB(NOW(), INTERVAL 1 YEAR)
      AND status IN ('completed', 'cancelled');
    
    -- Step 2: Update statistics
    CALL sp_update_daily_statistics();
    
    -- Step 3: Cleanup
    DELETE FROM audit_log WHERE created_at < DATE_SUB(NOW(), INTERVAL 90 DAY);
    DELETE FROM user_sessions WHERE expires_at < NOW();
    
    -- Step 4: Log completion
    INSERT INTO maintenance_log (task, completed_at, status)
    VALUES ('daily_maintenance', NOW(), 'completed');
END //

DELIMITER ;
```

---

## 4. Managing MySQL Events

```sql
-- ตัวอย่างที่ 16: ดูรายการ Events
SHOW EVENTS;
SHOW EVENTS FROM your_database;
SHOW EVENTS LIKE 'ev_%';

-- ดูรายละเอียดสมบูรณ์
SELECT 
    EVENT_SCHEMA,
    EVENT_NAME,
    EVENT_TYPE,
    EXECUTE_AT,
    INTERVAL_VALUE,
    INTERVAL_FIELD,
    STATUS,
    LAST_EXECUTED,
    STARTS,
    ENDS,
    EVENT_COMMENT
FROM information_schema.EVENTS
WHERE EVENT_SCHEMA = DATABASE()
ORDER BY EVENT_NAME;

-- ตัวอย่างที่ 17: Modify Event
ALTER EVENT ev_daily_cleanup
ON SCHEDULE EVERY 1 DAY
STARTS '2024-01-01 04:00:00'
COMMENT 'Updated schedule';

-- Disable Event
ALTER EVENT ev_daily_cleanup DISABLE;

-- Enable Event
ALTER EVENT ev_daily_cleanup ENABLE;

-- Drop Event
DROP EVENT IF EXISTS ev_one_time_migration;
```

---

## 5. SQL Server SQL Agent Jobs

```sql
-- ตัวอย่างที่ 18: SQL Server Agent Job (T-SQL)
-- ต้องมี SQL Server Agent service running

-- สร้าง Job
USE msdb;
GO

EXEC sp_add_job
    @job_name = N'Daily Database Maintenance',
    @enabled = 1,
    @description = N'Daily maintenance including backup, index rebuild, statistics update',
    @category_name = N'Database Maintenance';

-- เพิ่ม Job Step
EXEC sp_add_jobstep
    @job_name = N'Daily Database Maintenance',
    @step_name = N'Step 1: Update Statistics',
    @step_id = 1,
    @command = N'
        EXEC sp_updatestats;
        PRINT ''Statistics updated at '' + CONVERT(VARCHAR, GETDATE());
    ',
    @on_success_action = 3,  -- Go to next step
    @on_fail_action = 2;     -- Quit with failure

EXEC sp_add_jobstep
    @job_name = N'Daily Database Maintenance',
    @step_name = N'Step 2: Rebuild Indexes',
    @step_id = 2,
    @command = N'
        -- Rebuild all fragmented indexes
        EXEC dbo.sp_rebuild_fragmented_indexes;
    ',
    @on_success_action = 3,
    @on_fail_action = 2;

EXEC sp_add_jobstep
    @job_name = N'Daily Database Maintenance',
    @step_name = N'Step 3: Cleanup Old Logs',
    @step_id = 3,
    @command = N'
        DELETE FROM application_logs WHERE log_date < DATEADD(DAY, -90, GETDATE());
        DELETE FROM audit_trail WHERE created_at < DATEADD(DAY, -365, GETDATE());
    ',
    @on_success_action = 1,  -- Quit with success
    @on_fail_action = 2;

-- เพิ่ม Schedule
EXEC sp_add_schedule
    @schedule_name = N'Daily 2AM Schedule',
    @freq_type = 4,            -- Daily
    @freq_interval = 1,        -- Every 1 day
    @active_start_time = 20000; -- 02:00:00 AM

-- Attach Schedule to Job
EXEC sp_attach_schedule
    @job_name = N'Daily Database Maintenance',
    @schedule_name = N'Daily 2AM Schedule';

-- Add Job to Server
EXEC sp_add_jobserver
    @job_name = N'Daily Database Maintenance',
    @server_name = N'(local)';
```

---

## 6. Maintenance Jobs

```sql
-- ตัวอย่างที่ 19: PostgreSQL - VACUUM Automation
-- สร้าง Function สำหรับ Smart VACUUM
CREATE OR REPLACE PROCEDURE sp_smart_vacuum()
LANGUAGE plpgsql
AS $$
DECLARE
    v_table RECORD;
    v_bloat_ratio FLOAT;
BEGIN
    FOR v_table IN
        SELECT 
            schemaname,
            tablename,
            n_dead_tup,
            n_live_tup,
            last_vacuum,
            last_autovacuum
        FROM pg_stat_user_tables
        WHERE n_live_tup > 0
    LOOP
        -- คำนวณ bloat ratio
        v_bloat_ratio := v_table.n_dead_tup::FLOAT / v_table.n_live_tup;
        
        -- VACUUM ถ้ามี dead tuples มากกว่า 10%
        IF v_bloat_ratio > 0.1 OR v_table.n_dead_tup > 10000 THEN
            EXECUTE format('VACUUM ANALYZE %I.%I', v_table.schemaname, v_table.tablename);
            RAISE NOTICE 'Vacuumed %.% (bloat: %)', 
                v_table.schemaname, v_table.tablename,
                ROUND(v_bloat_ratio * 100) || '%';
        END IF;
    END LOOP;
END;
$$;

-- Schedule ด้วย pg_cron
SELECT cron.schedule('smart-vacuum', '0 4 * * *', 'CALL sp_smart_vacuum()');

-- ตัวอย่างที่ 20: Index Rebuild Job
CREATE OR REPLACE PROCEDURE sp_rebuild_fragmented_indexes()
LANGUAGE plpgsql
AS $$
DECLARE
    v_index RECORD;
    v_threshold_pct FLOAT := 30.0;  -- Rebuild ถ้า fragmented มากกว่า 30%
BEGIN
    FOR v_index IN
        SELECT 
            schemaname,
            tablename,
            indexname
        FROM pg_stat_user_indexes
        WHERE idx_scan > 0  -- Index ที่ถูกใช้จริง
        ORDER BY tablename, indexname
    LOOP
        BEGIN
            EXECUTE format('REINDEX INDEX CONCURRENTLY %I.%I', 
                v_index.schemaname, v_index.indexname);
            RAISE NOTICE 'Rebuilt index: %.%', v_index.schemaname, v_index.indexname;
        EXCEPTION WHEN OTHERS THEN
            RAISE WARNING 'Failed to rebuild index %.%: %', 
                v_index.schemaname, v_index.indexname, SQLERRM;
        END;
    END LOOP;
    
    RAISE NOTICE 'Index rebuild completed';
END;
$$;

-- ตัวอย่างที่ 21: Statistics Update Job
CREATE OR REPLACE PROCEDURE sp_update_all_statistics()
LANGUAGE plpgsql
AS $$
DECLARE
    v_table RECORD;
    v_start TIMESTAMP;
BEGIN
    v_start := NOW();
    
    FOR v_table IN
        SELECT schemaname, tablename
        FROM pg_stat_user_tables
        ORDER BY schemaname, tablename
    LOOP
        EXECUTE format('ANALYZE %I.%I', v_table.schemaname, v_table.tablename);
    END LOOP;
    
    RAISE NOTICE 'Statistics update completed in % seconds',
        EXTRACT(EPOCH FROM (NOW() - v_start))::INT;
END;
$$;
```

---

## 7. Data Archival Jobs

```sql
-- ตัวอย่างที่ 22: Data Archival - PostgreSQL
CREATE TABLE orders_archive (LIKE orders);
ALTER TABLE orders_archive ADD COLUMN archived_at TIMESTAMP DEFAULT NOW();

CREATE OR REPLACE PROCEDURE sp_archive_old_orders(
    p_months_old INT DEFAULT 12,
    p_batch_size INT DEFAULT 1000
)
LANGUAGE plpgsql
AS $$
DECLARE
    v_cutoff_date DATE;
    v_archived INT := 0;
    v_batch_archived INT;
BEGIN
    v_cutoff_date := CURRENT_DATE - (p_months_old || ' months')::INTERVAL;
    
    LOOP
        WITH archived AS (
            DELETE FROM orders
            WHERE order_id IN (
                SELECT order_id FROM orders
                WHERE order_date < v_cutoff_date
                  AND status IN ('completed', 'cancelled')
                LIMIT p_batch_size
            )
            RETURNING *
        )
        INSERT INTO orders_archive SELECT *, NOW() FROM archived;
        
        GET DIAGNOSTICS v_batch_archived = ROW_COUNT;
        v_archived := v_archived + v_batch_archived;
        
        EXIT WHEN v_batch_archived = 0;
        
        COMMIT;
        RAISE NOTICE 'Archived % orders so far...', v_archived;
    END LOOP;
    
    RAISE NOTICE 'Total orders archived: %', v_archived;
    
    -- Log archival
    INSERT INTO archival_log (task_name, records_archived, archived_before, completed_at)
    VALUES ('orders_archival', v_archived, v_cutoff_date, NOW());
END;
$$;

-- ตัวอย่างที่ 23: Partition-based Archival (PostgreSQL)
-- สำหรับ partitioned tables
CREATE OR REPLACE PROCEDURE sp_detach_old_partition(p_table TEXT, p_months INT DEFAULT 24)
LANGUAGE plpgsql
AS $$
DECLARE
    v_partition_name TEXT;
    v_cutoff_year INT;
    v_cutoff_month INT;
BEGIN
    v_cutoff_year := EXTRACT(YEAR FROM CURRENT_DATE - (p_months || ' months')::INTERVAL)::INT;
    v_cutoff_month := EXTRACT(MONTH FROM CURRENT_DATE - (p_months || ' months')::INTERVAL)::INT;
    
    v_partition_name := p_table || '_' || v_cutoff_year || '_' || LPAD(v_cutoff_month::TEXT, 2, '0');
    
    IF EXISTS (
        SELECT 1 FROM pg_tables 
        WHERE tablename = v_partition_name AND schemaname = 'public'
    ) THEN
        -- Detach partition
        EXECUTE format('ALTER TABLE %I DETACH PARTITION %I', p_table, v_partition_name);
        
        -- Optionally: move to archive schema
        EXECUTE format('ALTER TABLE %I SET SCHEMA archive', v_partition_name);
        
        RAISE NOTICE 'Detached partition: %', v_partition_name;
    END IF;
END;
$$;

-- MySQL Data Archival Event
DELIMITER //

CREATE EVENT ev_archive_old_data
ON SCHEDULE EVERY 1 WEEK
STARTS '2024-01-07 02:00:00'
DO
BEGIN
    DECLARE v_cutoff_date DATE;
    DECLARE v_archived INT DEFAULT 0;
    
    SET v_cutoff_date = DATE_SUB(CURDATE(), INTERVAL 1 YEAR);
    
    -- Archive orders
    INSERT INTO orders_archive 
    SELECT *, NOW() AS archived_at
    FROM orders 
    WHERE order_date < v_cutoff_date 
      AND status IN ('completed', 'cancelled');
    
    SET v_archived = ROW_COUNT();
    
    -- Delete archived orders
    DELETE FROM orders 
    WHERE order_date < v_cutoff_date 
      AND status IN ('completed', 'cancelled');
    
    -- Log
    INSERT INTO maintenance_log (task, records_affected, completed_at)
    VALUES (CONCAT('archive_orders_before_', v_cutoff_date), v_archived, NOW());
END //

DELIMITER ;
```

---

## 8. Report Generation Jobs

```sql
-- ตัวอย่างที่ 24: Daily Sales Report Generation
CREATE OR REPLACE PROCEDURE sp_generate_daily_sales_report(p_date DATE DEFAULT CURRENT_DATE - 1)
LANGUAGE plpgsql
AS $$
DECLARE
    v_total_orders INT;
    v_total_revenue DECIMAL(15,2);
    v_new_customers INT;
    v_top_product TEXT;
BEGIN
    -- สรุปยอดขาย
    SELECT COUNT(*), SUM(total_amount)
    INTO v_total_orders, v_total_revenue
    FROM orders
    WHERE DATE(order_date) = p_date AND status = 'completed';
    
    -- ลูกค้าใหม่
    SELECT COUNT(*) INTO v_new_customers
    FROM customers
    WHERE DATE(created_at) = p_date;
    
    -- สินค้าขายดี
    SELECT p.product_name INTO v_top_product
    FROM order_details od
    JOIN products p ON od.product_id = p.product_id
    JOIN orders o ON od.order_id = o.order_id
    WHERE DATE(o.order_date) = p_date AND o.status = 'completed'
    GROUP BY p.product_id, p.product_name
    ORDER BY SUM(od.quantity) DESC
    LIMIT 1;
    
    -- บันทึก report
    INSERT INTO daily_reports (
        report_date,
        total_orders,
        total_revenue,
        new_customers,
        top_product,
        generated_at
    ) VALUES (
        p_date,
        COALESCE(v_total_orders, 0),
        COALESCE(v_total_revenue, 0),
        COALESCE(v_new_customers, 0),
        v_top_product,
        NOW()
    ) ON CONFLICT (report_date) DO UPDATE
    SET 
        total_orders = EXCLUDED.total_orders,
        total_revenue = EXCLUDED.total_revenue,
        new_customers = EXCLUDED.new_customers,
        top_product = EXCLUDED.top_product,
        generated_at = EXCLUDED.generated_at;
    
    RAISE NOTICE 'Daily report generated for %: % orders, % revenue', 
        p_date, v_total_orders, v_total_revenue;
END;
$$;

-- Schedule ทุกเช้า
SELECT cron.schedule(
    'generate-daily-report',
    '5 1 * * *',  -- ตี 1 ห้านาที ทุกวัน
    'CALL sp_generate_daily_sales_report()'
);

-- ตัวอย่างที่ 25: Monthly Report
CREATE OR REPLACE PROCEDURE sp_generate_monthly_report(
    p_year INT DEFAULT EXTRACT(YEAR FROM CURRENT_DATE)::INT,
    p_month INT DEFAULT EXTRACT(MONTH FROM CURRENT_DATE - INTERVAL '1 month')::INT
)
LANGUAGE plpgsql
AS $$
BEGIN
    -- สร้าง/อัพเดท Monthly Report
    INSERT INTO monthly_reports (
        year, month,
        total_orders,
        total_revenue,
        new_customers,
        repeat_customers,
        avg_order_value,
        top_category,
        generated_at
    )
    WITH monthly_data AS (
        SELECT 
            COUNT(DISTINCT o.order_id) AS total_orders,
            SUM(o.total_amount) AS total_revenue,
            AVG(o.total_amount) AS avg_value,
            COUNT(DISTINCT CASE WHEN c.created_at >= DATE_TRUNC('month', NOW() - INTERVAL '1 month') THEN c.customer_id END) AS new_custs,
            COUNT(DISTINCT CASE WHEN c.created_at < DATE_TRUNC('month', NOW() - INTERVAL '1 month') THEN o.customer_id END) AS repeat_custs
        FROM orders o
        JOIN customers c ON o.customer_id = c.customer_id
        WHERE EXTRACT(YEAR FROM o.order_date) = p_year
          AND EXTRACT(MONTH FROM o.order_date) = p_month
          AND o.status = 'completed'
    ),
    top_cat AS (
        SELECT c.category_name
        FROM order_details od
        JOIN products p ON od.product_id = p.product_id
        JOIN categories c ON p.category_id = c.category_id
        JOIN orders o ON od.order_id = o.order_id
        WHERE EXTRACT(YEAR FROM o.order_date) = p_year
          AND EXTRACT(MONTH FROM o.order_date) = p_month
          AND o.status = 'completed'
        GROUP BY c.category_id, c.category_name
        ORDER BY SUM(od.quantity * od.unit_price) DESC
        LIMIT 1
    )
    SELECT 
        p_year, p_month,
        md.total_orders,
        md.total_revenue,
        md.new_custs,
        md.repeat_custs,
        md.avg_value,
        tc.category_name,
        NOW()
    FROM monthly_data md, top_cat tc
    ON CONFLICT (year, month) DO UPDATE
    SET total_orders = EXCLUDED.total_orders,
        total_revenue = EXCLUDED.total_revenue,
        generated_at = EXCLUDED.generated_at;
    
    RAISE NOTICE 'Monthly report generated for %/%', p_year, p_month;
END;
$$;
```

---

## 9. Event Monitoring

```sql
-- ตัวอย่างที่ 26: สร้าง Job Monitor
CREATE TABLE scheduled_job_monitor (
    monitor_id BIGSERIAL PRIMARY KEY,
    job_name TEXT,
    last_success TIMESTAMP,
    last_failure TIMESTAMP,
    failure_count INT DEFAULT 0,
    last_error TEXT,
    status TEXT DEFAULT 'OK',
    checked_at TIMESTAMP DEFAULT NOW()
);

CREATE OR REPLACE PROCEDURE sp_check_job_health()
LANGUAGE plpgsql
AS $$
DECLARE
    v_job RECORD;
    v_last_run TIMESTAMP;
    v_expected_interval INTERVAL;
BEGIN
    -- ตรวจสอบ pg_cron jobs
    FOR v_job IN
        SELECT 
            j.jobname,
            j.schedule,
            r.start_time AS last_run,
            r.status AS last_status,
            r.return_message
        FROM cron.job j
        LEFT JOIN LATERAL (
            SELECT start_time, status, return_message
            FROM cron.job_run_details
            WHERE jobid = j.jobid
            ORDER BY start_time DESC
            LIMIT 1
        ) r ON TRUE
        WHERE j.active = TRUE
    LOOP
        -- ตรวจสอบว่า job รันล่าสุดเมื่อไหร่
        IF v_job.last_run < NOW() - INTERVAL '2 hours' THEN
            INSERT INTO scheduled_job_monitor (job_name, last_failure, failure_count, last_error, status)
            VALUES (v_job.jobname, NOW(), 1, 'Job has not run in 2+ hours', 'WARNING')
            ON CONFLICT (job_name) DO UPDATE
            SET failure_count = scheduled_job_monitor.failure_count + 1,
                last_failure = EXCLUDED.last_failure,
                status = 'WARNING',
                checked_at = NOW();
        ELSIF v_job.last_status = 'failed' THEN
            INSERT INTO scheduled_job_monitor (job_name, last_failure, failure_count, last_error, status)
            VALUES (v_job.jobname, NOW(), 1, v_job.return_message, 'ERROR')
            ON CONFLICT (job_name) DO UPDATE
            SET failure_count = scheduled_job_monitor.failure_count + 1,
                last_failure = EXCLUDED.last_failure,
                status = 'ERROR',
                last_error = EXCLUDED.last_error,
                checked_at = NOW();
        ELSE
            INSERT INTO scheduled_job_monitor (job_name, last_success, status)
            VALUES (v_job.jobname, NOW(), 'OK')
            ON CONFLICT (job_name) DO UPDATE
            SET last_success = EXCLUDED.last_success,
                status = 'OK',
                failure_count = 0,
                checked_at = NOW();
        END IF;
    END LOOP;
END;
$$;

-- ตัวอย่างที่ 27: Alert สำหรับ Failed Jobs
CREATE OR REPLACE PROCEDURE sp_alert_failed_jobs()
LANGUAGE plpgsql
AS $$
DECLARE
    v_failed RECORD;
BEGIN
    FOR v_failed IN
        SELECT job_name, failure_count, last_error, last_failure
        FROM scheduled_job_monitor
        WHERE status IN ('ERROR', 'WARNING')
          AND failure_count >= 3  -- alert เมื่อ fail ติดต่อกัน 3 ครั้ง
    LOOP
        -- ส่ง notification (simulated)
        INSERT INTO notifications (
            notification_type,
            title,
            body,
            created_at,
            is_sent
        ) VALUES (
            'job_failure',
            'Scheduled Job Failed: ' || v_failed.job_name,
            'Job has failed ' || v_failed.failure_count || ' times. Last error: ' || v_failed.last_error,
            NOW(),
            FALSE
        );
        
        RAISE WARNING 'ALERT: Job % failed % times! Last error: %',
            v_failed.job_name, v_failed.failure_count, v_failed.last_error;
    END LOOP;
END;
$$;
```

---

## 10. SQLite - No Built-in Scheduler

```sql
-- SQLite ไม่มี built-in scheduler
-- ต้องใช้ external scheduler เช่น:
-- - Cron (Linux/Mac)
-- - Task Scheduler (Windows)
-- - Application-level scheduling (APScheduler ใน Python, node-cron ใน Node.js)

-- ตัวอย่างที่ 28: SQLite - Workarounds using Application Code
-- Python example using APScheduler:
-- 
-- from apscheduler.schedulers.background import BackgroundScheduler
-- import sqlite3
-- 
-- def cleanup_sessions():
--     with sqlite3.connect('database.db') as conn:
--         conn.execute("DELETE FROM sessions WHERE expires_at < datetime('now')")
--         conn.commit()
-- 
-- def generate_daily_report():
--     with sqlite3.connect('database.db') as conn:
--         # Your reporting logic here
--         pass
-- 
-- scheduler = BackgroundScheduler()
-- scheduler.add_job(cleanup_sessions, 'interval', minutes=30)
-- scheduler.add_job(generate_daily_report, 'cron', hour=2, minute=0)
-- scheduler.start()

-- SQLite Triggers เป็นทางเลือกสำหรับ event-driven automation
-- ตัวอย่างที่ 29: SQLite Trigger (ทำงาน event-driven ไม่ใช่ time-based)
-- SQLite syntax
CREATE TRIGGER IF NOT EXISTS trg_update_timestamp
AFTER UPDATE ON products
FOR EACH ROW
BEGIN
    UPDATE products SET updated_at = datetime('now') WHERE product_id = NEW.product_id;
END;

CREATE TRIGGER IF NOT EXISTS trg_auto_archive
AFTER INSERT ON orders
FOR EACH ROW
WHEN NEW.order_date < date('now', '-1 year')
BEGIN
    INSERT INTO orders_archive SELECT * FROM orders WHERE order_id = NEW.order_id;
    DELETE FROM orders WHERE order_id = NEW.order_id;
END;
```

---

## 11. Complete Maintenance Automation Example

```sql
-- ตัวอย่างที่ 30: Comprehensive Database Maintenance Schedule
-- PostgreSQL + pg_cron

-- 1. ทุก 15 นาที: ล้าง expired sessions
SELECT cron.schedule('clean-sessions', '*/15 * * * *',
    'DELETE FROM user_sessions WHERE expires_at < NOW()');

-- 2. ทุกชั่วโมง: Refresh Materialized Views
SELECT cron.schedule('refresh-mvs', '5 * * * *',
    $$CALL refresh_all_materialized_views()$$);

-- 3. ทุกคืนตี 2: Generate daily reports
SELECT cron.schedule('daily-reports', '0 2 * * *',
    'CALL sp_generate_daily_sales_report()');

-- 4. ทุกคืนตี 3: VACUUM และ Statistics
SELECT cron.schedule('nightly-vacuum', '0 3 * * *',
    $$VACUUM ANALYZE; SELECT sp_update_all_statistics()$$);

-- 5. ทุกอาทิตย์ตี 2: Archive old data
SELECT cron.schedule('weekly-archive', '0 2 * * 0',
    'CALL sp_archive_old_orders(12, 5000)');

-- 6. ทุกสัปดาห์ตี 4 วันเสาร์: Rebuild indexes
SELECT cron.schedule('weekly-reindex', '0 4 * * 6',
    'CALL sp_rebuild_fragmented_indexes()');

-- 7. ทุกวันที่ 1 ของเดือน: Monthly report
SELECT cron.schedule('monthly-report', '0 1 1 * *',
    'CALL sp_generate_monthly_report()');

-- 8. ทุกปีวันที่ 1 มกราคม: Year-end processing
SELECT cron.schedule('yearly-processing', '0 0 1 1 *',
    'CALL sp_year_end_processing()');

-- ดูรายการทั้งหมด
SELECT jobname, schedule, active FROM cron.job ORDER BY jobname;

-- ตัวอย่างที่ 31: MySQL - Complete Maintenance Schedule
-- เปิด Event Scheduler
SET GLOBAL event_scheduler = ON;

-- 15 นาที: Session cleanup
CREATE EVENT ev_session_cleanup ON SCHEDULE EVERY 15 MINUTE
DO DELETE FROM user_sessions WHERE expires_at < NOW();

-- ทุกชั่วโมง: Cache refresh
CREATE EVENT ev_cache_refresh ON SCHEDULE EVERY 1 HOUR
DO CALL sp_refresh_product_cache();

-- ทุกคืนตี 2: Daily report
CREATE EVENT ev_daily_report ON SCHEDULE EVERY 1 DAY STARTS '2024-01-01 02:00:00'
DO CALL sp_generate_daily_report();

-- ทุกสัปดาห์: Archive
CREATE EVENT ev_weekly_archive ON SCHEDULE EVERY 1 WEEK STARTS '2024-01-07 03:00:00'
DO CALL sp_archive_old_orders();

-- ทุกเดือน: Monthly report
CREATE EVENT ev_monthly_report 
ON SCHEDULE EVERY 1 MONTH
STARTS '2024-02-01 01:00:00'
DO CALL sp_generate_monthly_report();
```

---

## แบบฝึกหัด (Exercises)

**ข้อ 1:** สร้าง pg_cron job สำหรับ VACUUM ANALYZE ทุกคืนตี 3

**คำตอบข้อ 1:**
```sql
SELECT cron.schedule(
    'nightly-vacuum-analyze',
    '0 3 * * *',
    'VACUUM ANALYZE'
);
-- ตรวจสอบ
SELECT jobname, schedule, active FROM cron.job WHERE jobname = 'nightly-vacuum-analyze';
```

**ข้อ 2:** สร้าง MySQL Event ที่ลบ session หมดอายุทุก 30 นาที

**คำตอบข้อ 2:**
```sql
CREATE EVENT IF NOT EXISTS ev_cleanup_sessions
ON SCHEDULE EVERY 30 MINUTE STARTS NOW()
COMMENT 'Remove expired user sessions'
DO DELETE FROM user_sessions WHERE expires_at < NOW();
SHOW EVENTS LIKE 'ev_cleanup_%';
```

**ข้อ 3:** สร้าง pg_cron job สำหรับ generate daily report ทุกเช้า

**คำตอบข้อ 3:**
```sql
SELECT cron.schedule('daily-sales-report', '5 1 * * *',
    'CALL sp_generate_daily_sales_report()');
-- จะรันทุกวันตี 1 ห้านาที
```

**ข้อ 4:** ดู Job Run History ของ pg_cron ล่าสุด 20 records

**คำตอบข้อ 4:**
```sql
SELECT j.jobname, r.start_time, r.end_time, r.status, r.return_message,
       EXTRACT(EPOCH FROM (r.end_time - r.start_time)) AS duration_sec
FROM cron.job_run_details r JOIN cron.job j ON r.jobid = j.jobid
ORDER BY r.start_time DESC LIMIT 20;
```

**ข้อ 5:** สร้าง MySQL Event สำหรับ Archive ข้อมูลเก่าทุกสัปดาห์

**คำตอบข้อ 5:**
```sql
DELIMITER //
CREATE EVENT ev_weekly_data_archive ON SCHEDULE EVERY 1 WEEK STARTS '2024-01-07 02:00:00'
DO BEGIN
    INSERT INTO audit_log_archive SELECT *, NOW() FROM audit_log
    WHERE created_at < DATE_SUB(NOW(), INTERVAL 90 DAY);
    DELETE FROM audit_log WHERE created_at < DATE_SUB(NOW(), INTERVAL 90 DAY);
    INSERT INTO maintenance_log (task, completed_at) VALUES ('weekly_archive', NOW());
END //
DELIMITER ;
```

**ข้อ 6:** สร้าง SQL Server Agent Job สำหรับ Update Statistics ทุกวัน

**คำตอบข้อ 6:**
```sql
USE msdb;
EXEC sp_add_job @job_name = N'Daily Statistics Update', @enabled = 1;
EXEC sp_add_jobstep @job_name = N'Daily Statistics Update',
    @step_name = N'Update Stats', @command = N'EXEC sp_updatestats;';
EXEC sp_add_schedule @schedule_name = N'Daily 2AM',
    @freq_type = 4, @freq_interval = 1, @active_start_time = 20000;
EXEC sp_attach_schedule @job_name = N'Daily Statistics Update', @schedule_name = N'Daily 2AM';
EXEC sp_add_jobserver @job_name = N'Daily Statistics Update', @server_name = N'(local)';
```

**ข้อ 7:** สร้าง Procedure สำหรับ Smart Vacuum ที่ vacuum เฉพาะ table ที่มี dead tuples มาก

**คำตอบข้อ 7:**
```sql
CREATE OR REPLACE PROCEDURE sp_targeted_vacuum(p_threshold_pct FLOAT DEFAULT 10.0)
LANGUAGE plpgsql AS $$
DECLARE v_tbl RECORD;
BEGIN
    FOR v_tbl IN
        SELECT schemaname, tablename FROM pg_stat_user_tables
        WHERE n_live_tup > 0 AND (n_dead_tup::FLOAT / n_live_tup * 100) > p_threshold_pct
    LOOP
        EXECUTE format('VACUUM ANALYZE %I.%I', v_tbl.schemaname, v_tbl.tablename);
        RAISE NOTICE 'Vacuumed %.%', v_tbl.schemaname, v_tbl.tablename;
    END LOOP;
END; $$;

SELECT cron.schedule('smart-vacuum', '0 4 * * *', 'CALL sp_targeted_vacuum(15.0)');
```

**ข้อ 8:** Disable และ Enable pg_cron job

**คำตอบข้อ 8:**
```sql
-- Disable
UPDATE cron.job SET active = FALSE WHERE jobname = 'nightly-vacuum-analyze';

-- Enable
UPDATE cron.job SET active = TRUE WHERE jobname = 'nightly-vacuum-analyze';

-- ตรวจสอบ
SELECT jobname, active FROM cron.job WHERE jobname = 'nightly-vacuum-analyze';
```

**ข้อ 9:** สร้าง MySQL Event ที่รันครั้งเดียว (one-time) เพื่อทำ data migration

**คำตอบข้อ 9:**
```sql
CREATE EVENT ev_one_time_data_fix
ON SCHEDULE AT DATE_ADD(NOW(), INTERVAL 1 HOUR)  -- รัน 1 ชั่วโมงหลังจากนี้
ON COMPLETION NOT PRESERVE  -- ลบ event หลังรัน
DO BEGIN
    UPDATE customers SET tier = 'Bronze' WHERE tier IS NULL;
    UPDATE products SET discontinued = 0 WHERE discontinued IS NULL;
    INSERT INTO migration_log(name, completed_at) VALUES('fix_nulls', NOW());
END;
```

**ข้อ 10:** สร้าง Monitoring Query ที่ดูว่า Jobs ทำงานปกติหรือไม่

**คำตอบข้อ 10:**
```sql
-- PostgreSQL pg_cron monitor
SELECT 
    j.jobname,
    j.schedule,
    j.active,
    r.start_time AS last_run,
    r.status AS last_status,
    CASE WHEN r.start_time < NOW() - INTERVAL '25 hours' THEN 'OVERDUE'
         WHEN r.status = 'failed' THEN 'FAILED'
         ELSE 'OK' END AS health
FROM cron.job j
LEFT JOIN LATERAL (
    SELECT start_time, status FROM cron.job_run_details
    WHERE jobid = j.jobid ORDER BY start_time DESC LIMIT 1
) r ON TRUE
WHERE j.active = TRUE
ORDER BY j.jobname;

-- MySQL Events monitor
SELECT EVENT_NAME, LAST_EXECUTED, STATUS, 
       CASE WHEN LAST_EXECUTED < DATE_SUB(NOW(), INTERVAL 25 HOUR) THEN 'OVERDUE' ELSE 'OK' END AS health
FROM information_schema.EVENTS WHERE EVENT_SCHEMA = DATABASE() AND STATUS = 'ENABLED';
```

---

## สรุป (Summary)

| Feature | PostgreSQL | MySQL | SQL Server | SQLite |
|---------|------------|-------|------------|--------|
| Scheduler | pg_cron (extension) | Event Scheduler (built-in) | SQL Agent | ไม่มี |
| Install | `CREATE EXTENSION pg_cron` | `SET GLOBAL event_scheduler = ON` | Service | N/A |
| Create Job | `cron.schedule()` | `CREATE EVENT` | `sp_add_job` | External |
| View Jobs | `cron.job` | `information_schema.EVENTS` | `msdb.dbo.sysjobs` | N/A |
| Run History | `cron.job_run_details` | ไม่มี built-in | `msdb.dbo.sysjobhistory` | N/A |
| Min Interval | 1 นาที | 1 วินาที | 1 นาที | ขึ้นอยู่กับ external scheduler |

---

*จบ Part 090: Database Events and Scheduled Jobs*

---

## จบ SQL Course Parts 081-090

ยินดีด้วย! คุณได้เรียนรู้หัวข้อขั้นสูงครบทั้ง 10 บท:

- **Part 081**: Views - Virtual Tables
- **Part 082**: Updatable Views  
- **Part 083**: Materialized Views
- **Part 084**: Stored Procedures - Introduction
- **Part 085**: PL/pgSQL - PostgreSQL
- **Part 086**: MySQL Stored Procedures
- **Part 087**: User-Defined Functions
- **Part 088**: Triggers
- **Part 089**: Dynamic SQL (พร้อม Security Warnings)
- **Part 090**: Database Events and Scheduled Jobs
