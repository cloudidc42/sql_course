# Part 119: IoT and Time-Series Data

## บทนำ (Introduction)

IoT (Internet of Things) และ Time-Series Data เป็นหนึ่งในสาขาที่เติบโตเร็วที่สุดในวงการ Data Engineering ข้อมูล IoT มีลักษณะพิเศษ: ปริมาณมาก, เข้ามาต่อเนื่อง, และต้องการ Query แบบช่วงเวลา บทนี้ครอบคลุม Schema Design, Downsampling, Gap Filling, Rolling Windows และ Anomaly Detection

## 1. IoT Database Schema

```sql
-- =========================================
-- IoT TIME-SERIES DATABASE - COMPLETE DDL
-- =========================================

CREATE DATABASE iot_db
    CHARACTER SET utf8mb4
    COLLATE utf8mb4_unicode_ci;

USE iot_db;

-- =========================================
-- SECTION 1: DEVICE MANAGEMENT
-- =========================================

CREATE TABLE device_types (
    type_id         INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    type_code       VARCHAR(50) NOT NULL UNIQUE,
    name            VARCHAR(200) NOT NULL,
    description     TEXT,
    manufacturer    VARCHAR(200),
    protocol        ENUM('mqtt','http','coap','modbus','opc_ua') DEFAULT 'mqtt',
    data_interval_sec INT DEFAULT 60 COMMENT 'Expected reading interval'
) ENGINE=InnoDB;

CREATE TABLE locations (
    location_id     INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    parent_id       INT UNSIGNED,
    name            VARCHAR(200) NOT NULL,
    location_type   ENUM('building','floor','room','outdoor','zone') DEFAULT 'room',
    latitude        DECIMAL(10,7),
    longitude       DECIMAL(10,7),
    timezone        VARCHAR(50) DEFAULT 'Asia/Bangkok',
    FOREIGN KEY (parent_id) REFERENCES locations(location_id) ON DELETE SET NULL
) ENGINE=InnoDB;

CREATE TABLE devices (
    device_id       INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    device_eui      VARCHAR(50) NOT NULL UNIQUE COMMENT 'Unique hardware identifier',
    type_id         INT UNSIGNED NOT NULL,
    location_id     INT UNSIGNED,
    name            VARCHAR(200),
    firmware_version VARCHAR(20),
    hardware_version VARCHAR(20),
    install_date    DATE,
    battery_pct     TINYINT UNSIGNED,
    signal_strength INT COMMENT 'dBm',
    status          ENUM('online','offline','maintenance','decommissioned') DEFAULT 'offline',
    last_seen       DATETIME,
    config          JSON COMMENT 'Device-specific configuration',
    FOREIGN KEY (type_id) REFERENCES device_types(type_id),
    FOREIGN KEY (location_id) REFERENCES locations(location_id) ON DELETE SET NULL,
    INDEX idx_type (type_id),
    INDEX idx_location (location_id),
    INDEX idx_last_seen (last_seen)
) ENGINE=InnoDB;

CREATE TABLE sensor_metrics (
    metric_id       INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    type_id         INT UNSIGNED NOT NULL,
    metric_code     VARCHAR(50) NOT NULL,
    name            VARCHAR(200),
    unit            VARCHAR(20),
    data_type       ENUM('float','integer','boolean','string') DEFAULT 'float',
    min_value       DECIMAL(15,4),
    max_value       DECIMAL(15,4),
    FOREIGN KEY (type_id) REFERENCES device_types(type_id),
    UNIQUE KEY uk_type_metric (type_id, metric_code)
) ENGINE=InnoDB;

-- =========================================
-- SECTION 2: TIME-SERIES READINGS
-- =========================================

-- Raw Readings (Partitioned by month)
CREATE TABLE sensor_readings (
    reading_id      BIGINT UNSIGNED AUTO_INCREMENT,
    device_id       INT UNSIGNED NOT NULL,
    metric_code     VARCHAR(50) NOT NULL,
    ts              DATETIME(3) NOT NULL COMMENT 'Timestamp with millisecond precision',
    value_float     DECIMAL(15,4),
    value_int       BIGINT,
    value_bool      BOOLEAN,
    value_text      VARCHAR(500),
    quality         TINYINT DEFAULT 100 COMMENT '0-100 quality score',
    PRIMARY KEY (reading_id, ts),
    FOREIGN KEY (device_id) REFERENCES devices(device_id),
    INDEX idx_device_metric_ts (device_id, metric_code, ts),
    INDEX idx_ts (ts)
) ENGINE=InnoDB
PARTITION BY RANGE (YEAR(ts) * 100 + MONTH(ts)) (
    PARTITION p_2024_01 VALUES LESS THAN (202402),
    PARTITION p_2024_02 VALUES LESS THAN (202403),
    PARTITION p_2024_03 VALUES LESS THAN (202404),
    PARTITION p_2024_04 VALUES LESS THAN (202405),
    PARTITION p_2024_05 VALUES LESS THAN (202406),
    PARTITION p_2024_06 VALUES LESS THAN (202407),
    PARTITION p_future VALUES LESS THAN MAXVALUE
);

-- Downsampled: hourly aggregates
CREATE TABLE sensor_readings_hourly (
    device_id       INT UNSIGNED NOT NULL,
    metric_code     VARCHAR(50) NOT NULL,
    bucket_hour     DATETIME NOT NULL COMMENT 'Hour start time',
    reading_count   INT UNSIGNED DEFAULT 0,
    value_min       DECIMAL(15,4),
    value_max       DECIMAL(15,4),
    value_avg       DECIMAL(15,4),
    value_sum       DECIMAL(20,4),
    value_first     DECIMAL(15,4),
    value_last      DECIMAL(15,4),
    stddev_value    DECIMAL(15,4),
    PRIMARY KEY (device_id, metric_code, bucket_hour),
    FOREIGN KEY (device_id) REFERENCES devices(device_id),
    INDEX idx_device_hour (device_id, bucket_hour)
) ENGINE=InnoDB;

-- Downsampled: daily aggregates
CREATE TABLE sensor_readings_daily (
    device_id       INT UNSIGNED NOT NULL,
    metric_code     VARCHAR(50) NOT NULL,
    bucket_date     DATE NOT NULL,
    reading_count   INT UNSIGNED DEFAULT 0,
    value_min       DECIMAL(15,4),
    value_max       DECIMAL(15,4),
    value_avg       DECIMAL(15,4),
    value_sum       DECIMAL(20,4),
    PRIMARY KEY (device_id, metric_code, bucket_date),
    FOREIGN KEY (device_id) REFERENCES devices(device_id)
) ENGINE=InnoDB;

-- =========================================
-- SECTION 3: ALERTS AND EVENTS
-- =========================================

CREATE TABLE alert_rules (
    rule_id         INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    device_id       INT UNSIGNED,
    type_id         INT UNSIGNED COMMENT 'Apply to all devices of this type',
    metric_code     VARCHAR(50) NOT NULL,
    rule_name       VARCHAR(200) NOT NULL,
    condition_type  ENUM('above','below','equal','outside_range','rate_of_change') NOT NULL,
    threshold_low   DECIMAL(15,4),
    threshold_high  DECIMAL(15,4),
    severity        ENUM('info','warning','critical','emergency') DEFAULT 'warning',
    is_active       BOOLEAN DEFAULT TRUE,
    FOREIGN KEY (device_id) REFERENCES devices(device_id) ON DELETE CASCADE,
    FOREIGN KEY (type_id) REFERENCES device_types(type_id) ON DELETE CASCADE
) ENGINE=InnoDB;

CREATE TABLE alerts (
    alert_id        BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    rule_id         INT UNSIGNED,
    device_id       INT UNSIGNED NOT NULL,
    metric_code     VARCHAR(50),
    severity        ENUM('info','warning','critical','emergency'),
    message         TEXT,
    trigger_value   DECIMAL(15,4),
    started_at      DATETIME NOT NULL,
    resolved_at     DATETIME,
    acknowledged_by INT UNSIGNED,
    acknowledged_at DATETIME,
    FOREIGN KEY (rule_id) REFERENCES alert_rules(rule_id) ON DELETE SET NULL,
    FOREIGN KEY (device_id) REFERENCES devices(device_id),
    INDEX idx_device (device_id, started_at),
    INDEX idx_severity (severity, resolved_at)
) ENGINE=InnoDB;

-- =========================================
-- SECTION 4: ENERGY METERING
-- =========================================

CREATE TABLE energy_meters (
    meter_id        INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    device_id       INT UNSIGNED NOT NULL UNIQUE,
    meter_number    VARCHAR(50),
    circuit_name    VARCHAR(200),
    location_id     INT UNSIGNED,
    tariff_type     ENUM('T1','T2','T3') DEFAULT 'T1' COMMENT 'Thai electricity tariff type',
    contracted_demand_kva DECIMAL(8,2),
    FOREIGN KEY (device_id) REFERENCES devices(device_id),
    FOREIGN KEY (location_id) REFERENCES locations(location_id)
) ENGINE=InnoDB;

CREATE TABLE energy_readings (
    reading_id      BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    meter_id        INT UNSIGNED NOT NULL,
    ts              DATETIME NOT NULL,
    active_power_kw DECIMAL(10,3) COMMENT 'kW',
    reactive_power_kvar DECIMAL(10,3) COMMENT 'kVAR',
    apparent_power_kva DECIMAL(10,3) COMMENT 'kVA',
    power_factor    DECIMAL(4,3),
    voltage_v       DECIMAL(7,2),
    current_a       DECIMAL(8,3),
    energy_kwh      DECIMAL(15,3) COMMENT 'Cumulative kWh meter reading',
    FOREIGN KEY (meter_id) REFERENCES energy_meters(meter_id),
    INDEX idx_meter_ts (meter_id, ts)
) ENGINE=InnoDB;
```

## 2. Sample Data

```sql
-- Device Types
INSERT INTO device_types (type_id, type_code, name, protocol, data_interval_sec) VALUES
(1, 'TEMP_HUM', 'Temperature & Humidity Sensor', 'mqtt', 60),
(2, 'CO2_AIR', 'CO2 Air Quality Sensor', 'mqtt', 300),
(3, 'ENERGY_METER', 'Smart Energy Meter', 'modbus', 900),
(4, 'WATER_METER', 'Water Flow Meter', 'modbus', 3600),
(5, 'MOTION_PIR', 'PIR Motion Sensor', 'mqtt', 0),
(6, 'SOLAR_INV', 'Solar Inverter', 'modbus', 300);

-- Sensor Metrics
INSERT INTO sensor_metrics (type_id, metric_code, name, unit, data_type, min_value, max_value) VALUES
(1, 'temp', 'Temperature', '°C', 'float', -40, 80),
(1, 'humidity', 'Relative Humidity', '%', 'float', 0, 100),
(2, 'co2', 'CO2 Concentration', 'ppm', 'integer', 400, 5000),
(2, 'tvoc', 'Total VOC', 'ppb', 'float', 0, 1000),
(3, 'active_power', 'Active Power', 'kW', 'float', 0, 1000),
(3, 'energy_kwh', 'Energy kWh', 'kWh', 'float', 0, 9999999),
(6, 'dc_power', 'DC Power', 'W', 'float', 0, 50000),
(6, 'ac_power', 'AC Power', 'W', 'float', 0, 50000),
(6, 'daily_yield', 'Daily Yield', 'kWh', 'float', 0, 500);

-- Locations
INSERT INTO locations (location_id, parent_id, name, location_type, latitude, longitude) VALUES
(1, NULL, 'Office Building A', 'building', 13.7563, 100.5018),
(2, 1, 'Floor 1', 'floor', NULL, NULL),
(3, 1, 'Floor 2', 'floor', NULL, NULL),
(4, 2, 'Meeting Room 101', 'room', NULL, NULL),
(5, 2, 'Office Area 1', 'room', NULL, NULL),
(6, 3, 'Server Room', 'room', NULL, NULL),
(7, NULL, 'Rooftop', 'outdoor', 13.7564, 100.5019);

-- Devices
INSERT INTO devices (device_id, device_eui, type_id, location_id, name, firmware_version, status, last_seen, battery_pct) VALUES
(1, 'TH001234AABBCCDD', 1, 4, 'Temp Sensor - Meeting Room 101', '2.1.0', 'online', NOW(), 85),
(2, 'TH001234AABBCCEE', 1, 5, 'Temp Sensor - Office Area 1', '2.1.0', 'online', NOW(), 72),
(3, 'TH001234AABBCCFF', 1, 6, 'Temp Sensor - Server Room', '2.1.0', 'online', NOW(), 91),
(4, 'CO2001234AABB01', 2, 4, 'Air Quality - Meeting Room 101', '1.5.2', 'online', NOW(), 78),
(5, 'CO2001234AABB02', 2, 5, 'Air Quality - Office Area 1', '1.5.2', 'online', NOW(), 65),
(6, 'EM001234AABB01', 3, 1, 'Main Energy Meter', '3.0.1', 'online', NOW(), NULL),
(7, 'SI001234SOLAR01', 6, 7, 'Solar Inverter - Rooftop', '4.2.0', 'online', NOW(), NULL);

-- Sensor Readings (simulate data for Feb 1-7, 2024)
INSERT INTO sensor_readings (device_id, metric_code, ts, value_float, quality) VALUES
-- Temperature in meeting room (rises during work hours, cooler at night)
(1, 'temp', '2024-02-01 00:00:00', 24.5, 100),
(1, 'temp', '2024-02-01 06:00:00', 23.8, 100),
(1, 'temp', '2024-02-01 08:00:00', 25.2, 100),
(1, 'temp', '2024-02-01 09:00:00', 26.8, 100),
(1, 'temp', '2024-02-01 12:00:00', 27.5, 100),
(1, 'temp', '2024-02-01 14:00:00', 28.1, 100),
(1, 'temp', '2024-02-01 18:00:00', 26.3, 100),
(1, 'temp', '2024-02-01 22:00:00', 24.9, 100),
(1, 'temp', '2024-02-02 09:00:00', 26.2, 100),
(1, 'temp', '2024-02-02 14:00:00', 27.8, 100),
-- Humidity
(1, 'humidity', '2024-02-01 00:00:00', 62.3, 100),
(1, 'humidity', '2024-02-01 09:00:00', 55.4, 100),
(1, 'humidity', '2024-02-01 14:00:00', 52.1, 100),
(1, 'humidity', '2024-02-01 18:00:00', 58.7, 100),
-- CO2 in meeting room
(4, 'co2', '2024-02-01 08:00:00', 420, 100),
(4, 'co2', '2024-02-01 09:00:00', 650, 100),
(4, 'co2', '2024-02-01 10:00:00', 980, 100),
(4, 'co2', '2024-02-01 11:00:00', 1250, 100),
(4, 'co2', '2024-02-01 12:00:00', 890, 100),
(4, 'co2', '2024-02-01 14:00:00', 1450, 95),
(4, 'co2', '2024-02-01 15:00:00', 720, 100),
(4, 'co2', '2024-02-01 18:00:00', 430, 100),
-- Server Room temperature (high, AC running)
(3, 'temp', '2024-02-01 00:00:00', 21.2, 100),
(3, 'temp', '2024-02-01 06:00:00', 20.8, 100),
(3, 'temp', '2024-02-01 12:00:00', 22.1, 100),
(3, 'temp', '2024-02-01 18:00:00', 21.9, 100),
-- Anomaly: server room spike
(3, 'temp', '2024-02-03 14:00:00', 35.6, 100),
(3, 'temp', '2024-02-03 15:00:00', 38.2, 100),
(3, 'temp', '2024-02-03 16:00:00', 22.3, 100);

-- Hourly aggregates (pre-computed)
INSERT INTO sensor_readings_hourly (device_id, metric_code, bucket_hour, reading_count, value_min, value_max, value_avg, value_first, value_last) VALUES
(1, 'temp', '2024-02-01 08:00:00', 12, 24.8, 26.5, 25.6, 24.8, 26.5),
(1, 'temp', '2024-02-01 09:00:00', 12, 25.9, 27.2, 26.5, 25.9, 27.2),
(1, 'temp', '2024-02-01 10:00:00', 12, 26.8, 28.1, 27.4, 26.8, 28.1),
(1, 'temp', '2024-02-01 14:00:00', 12, 27.5, 29.0, 28.2, 27.5, 29.0);

-- Energy readings
INSERT INTO energy_readings (meter_id, ts, active_power_kw, reactive_power_kvar, power_factor, voltage_v, energy_kwh) VALUES
(1, '2024-02-01 00:00:00', 45.2, 12.3, 0.965, 220.1, 12500.123),
(1, '2024-02-01 01:00:00', 42.1, 11.8, 0.963, 219.8, 12545.323),
(1, '2024-02-01 08:00:00', 125.6, 35.2, 0.962, 219.5, 12895.644),
(1, '2024-02-01 09:00:00', 185.3, 52.1, 0.963, 219.3, 13020.900),
(1, '2024-02-01 10:00:00', 195.8, 55.3, 0.963, 219.6, 13216.700),
(1, '2024-02-01 18:00:00', 165.2, 45.6, 0.964, 220.2, 14489.500),
(1, '2024-02-01 23:00:00', 52.3, 14.2, 0.965, 220.5, 14776.100),
(1, '2024-02-02 00:00:00', 44.8, 12.1, 0.965, 220.3, 14820.900);

-- Alert Rules
INSERT INTO alert_rules (rule_id, type_id, metric_code, rule_name, condition_type, threshold_high, severity) VALUES
(1, 1, 'temp', 'High Temperature Warning', 'above', 30, 'warning'),
(2, 1, 'temp', 'Critical Temperature Alert', 'above', 35, 'critical'),
(3, 2, 'co2', 'High CO2 Warning', 'above', 1000, 'warning'),
(4, 2, 'co2', 'Critical CO2 Alert', 'above', 1500, 'critical'),
(5, 1, 'humidity', 'High Humidity Alert', 'above', 80, 'warning');

-- Alerts
INSERT INTO alerts (rule_id, device_id, metric_code, severity, message, trigger_value, started_at, resolved_at) VALUES
(2, 3, 'temp', 'critical', 'Server room temperature exceeded 35°C! Check cooling system.', 35.6, '2024-02-03 14:00:00', '2024-02-03 16:30:00'),
(3, 4, 'co2', 'warning', 'CO2 level in Meeting Room 101 exceeded 1000 ppm', 1250, '2024-02-01 11:00:00', '2024-02-01 12:30:00');
```

## 3. IoT Time-Series Queries

### Query 1: Real-Time Device Status Dashboard

```sql
-- Dashboard สถานะ Devices ทั้งหมด
SELECT 
    d.device_id,
    d.name AS device_name,
    dt.name AS device_type,
    l.name AS location,
    d.status,
    d.battery_pct,
    d.signal_strength,
    d.last_seen,
    TIMESTAMPDIFF(MINUTE, d.last_seen, NOW()) AS minutes_since_last_seen,
    -- Latest readings for each metric
    recent.metric_code,
    recent.value_float AS latest_value,
    recent.ts AS reading_time,
    -- Health check
    CASE 
        WHEN d.status = 'offline' THEN 'OFFLINE'
        WHEN TIMESTAMPDIFF(MINUTE, d.last_seen, NOW()) > dt.data_interval_sec / 60 * 3 THEN 'STALE DATA'
        WHEN d.battery_pct < 15 THEN 'LOW BATTERY'
        ELSE 'HEALTHY'
    END AS health_status
FROM devices d
JOIN device_types dt ON d.type_id = dt.type_id
LEFT JOIN locations l ON d.location_id = l.location_id
LEFT JOIN (
    -- Latest reading per device per metric
    SELECT device_id, metric_code, value_float, ts,
           ROW_NUMBER() OVER (PARTITION BY device_id, metric_code ORDER BY ts DESC) AS rn
    FROM sensor_readings
    WHERE ts >= DATE_SUB(NOW(), INTERVAL 2 HOUR)
) recent ON d.device_id = recent.device_id AND recent.rn = 1
ORDER BY health_status DESC, d.device_id;
```

---

### Query 2: Time-Series Downsampling

```sql
-- Downsample raw readings เป็นช่วง 15 นาที (สำหรับ Chart)
SELECT 
    device_id,
    metric_code,
    -- แบ่ง timestamp เป็น 15-minute buckets
    FROM_UNIXTIME(
        FLOOR(UNIX_TIMESTAMP(ts) / 900) * 900
    ) AS bucket_15min,
    COUNT(*) AS reading_count,
    ROUND(MIN(value_float), 2) AS min_val,
    ROUND(MAX(value_float), 2) AS max_val,
    ROUND(AVG(value_float), 2) AS avg_val,
    -- First and last value in bucket
    SUBSTRING_INDEX(GROUP_CONCAT(value_float ORDER BY ts ASC SEPARATOR ','), ',', 1) AS first_val,
    SUBSTRING_INDEX(GROUP_CONCAT(value_float ORDER BY ts DESC SEPARATOR ','), ',', 1) AS last_val
FROM sensor_readings
WHERE device_id = 1
  AND metric_code = 'temp'
  AND ts BETWEEN '2024-02-01 00:00:00' AND '2024-02-01 23:59:59'
GROUP BY device_id, metric_code, FLOOR(UNIX_TIMESTAMP(ts) / 900)
ORDER BY bucket_15min ASC;
```

---

### Query 3: Gap Filling (Time-Series Without Gaps)

```sql
-- Gap Filling: เติมช่วงเวลาที่ไม่มีข้อมูล
WITH RECURSIVE time_series AS (
    -- Generate hourly timestamps
    SELECT '2024-02-01 00:00:00' AS hour_slot
    UNION ALL
    SELECT TIMESTAMPADD(HOUR, 1, hour_slot)
    FROM time_series
    WHERE hour_slot < '2024-02-01 23:00:00'
),
actual_data AS (
    SELECT 
        DATE_FORMAT(ts, '%Y-%m-%d %H:00:00') AS hour_slot,
        ROUND(AVG(value_float), 2) AS avg_temp
    FROM sensor_readings
    WHERE device_id = 1
      AND metric_code = 'temp'
      AND DATE(ts) = '2024-02-01'
    GROUP BY DATE_FORMAT(ts, '%Y-%m-%d %H:00:00')
)
SELECT 
    ts.hour_slot,
    ad.avg_temp,
    CASE WHEN ad.avg_temp IS NULL THEN 'MISSING' ELSE 'OK' END AS data_status,
    -- Forward fill: use last known value
    LAST_VALUE(ad.avg_temp IGNORE NULLS) OVER (
        ORDER BY ts.hour_slot
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    ) AS temp_forward_fill
FROM time_series ts
LEFT JOIN actual_data ad ON ts.hour_slot = ad.hour_slot
ORDER BY ts.hour_slot;
```

---

### Query 4: Rolling Window Statistics

```sql
-- Rolling Window: 1-hour rolling average, min, max
SELECT 
    device_id,
    ts,
    ROUND(value_float, 2) AS temperature,
    -- 1-hour rolling window (assumes data every 5 minutes = 12 readings)
    ROUND(AVG(value_float) OVER (
        PARTITION BY device_id 
        ORDER BY ts 
        RANGE BETWEEN INTERVAL 1 HOUR PRECEDING AND CURRENT ROW
    ), 2) AS rolling_1h_avg,
    ROUND(MIN(value_float) OVER (
        PARTITION BY device_id 
        ORDER BY ts 
        RANGE BETWEEN INTERVAL 1 HOUR PRECEDING AND CURRENT ROW
    ), 2) AS rolling_1h_min,
    ROUND(MAX(value_float) OVER (
        PARTITION BY device_id 
        ORDER BY ts 
        RANGE BETWEEN INTERVAL 1 HOUR PRECEDING AND CURRENT ROW
    ), 2) AS rolling_1h_max,
    -- Standard deviation over 2 hours
    ROUND(STDDEV(value_float) OVER (
        PARTITION BY device_id 
        ORDER BY ts 
        RANGE BETWEEN INTERVAL 2 HOUR PRECEDING AND CURRENT ROW
    ), 3) AS rolling_2h_stddev,
    -- Rate of change
    ROUND(value_float - LAG(value_float) OVER (
        PARTITION BY device_id ORDER BY ts
    ), 2) AS delta_from_prev
FROM sensor_readings
WHERE device_id = 1
  AND metric_code = 'temp'
  AND ts BETWEEN '2024-02-01 00:00:00' AND '2024-02-02 00:00:00'
ORDER BY ts;
```

---

### Query 5: Anomaly Detection (Z-Score Method)

```sql
-- Anomaly Detection โดยใช้ Z-Score
WITH baseline AS (
    SELECT 
        device_id,
        metric_code,
        AVG(value_float) AS mean_val,
        STDDEV(value_float) AS stddev_val
    FROM sensor_readings
    WHERE ts BETWEEN '2024-02-01 00:00:00' AND '2024-02-02 23:59:59'
      AND quality >= 80
    GROUP BY device_id, metric_code
)
SELECT 
    sr.device_id,
    d.name AS device_name,
    sm.name AS metric_name,
    sm.unit,
    sr.ts,
    ROUND(sr.value_float, 2) AS value,
    ROUND(b.mean_val, 2) AS baseline_mean,
    ROUND(b.stddev_val, 3) AS baseline_stddev,
    -- Z-Score
    ROUND(ABS(sr.value_float - b.mean_val) / NULLIF(b.stddev_val, 0), 2) AS z_score,
    -- Flag anomalies
    CASE 
        WHEN ABS(sr.value_float - b.mean_val) / NULLIF(b.stddev_val, 0) > 3 THEN 'SEVERE ANOMALY (3σ)'
        WHEN ABS(sr.value_float - b.mean_val) / NULLIF(b.stddev_val, 0) > 2 THEN 'ANOMALY (2σ)'
        ELSE 'NORMAL'
    END AS anomaly_status
FROM sensor_readings sr
JOIN baseline b ON sr.device_id = b.device_id AND sr.metric_code = b.metric_code
JOIN devices d ON sr.device_id = d.device_id
JOIN sensor_metrics sm ON d.type_id = sm.type_id AND sr.metric_code = sm.metric_code
WHERE ABS(sr.value_float - b.mean_val) / NULLIF(b.stddev_val, 0) > 2
ORDER BY z_score DESC;
```

---

### Query 6: Energy Consumption Analysis

```sql
-- วิเคราะห์การใช้พลังงานรายชั่วโมง (kWh)
WITH hourly_consumption AS (
    SELECT 
        meter_id,
        DATE_FORMAT(ts, '%Y-%m-%d %H:00:00') AS hour_bucket,
        HOUR(ts) AS hour_of_day,
        DAYOFWEEK(ts) AS day_of_week,
        -- Calculate consumption = difference in cumulative kWh
        energy_kwh - LAG(energy_kwh) OVER (
            PARTITION BY meter_id ORDER BY ts
        ) AS consumption_kwh,
        active_power_kw,
        power_factor
    FROM energy_readings
    WHERE meter_id = 1
)
SELECT 
    hour_bucket,
    hour_of_day,
    CASE day_of_week
        WHEN 1 THEN 'Sunday'
        WHEN 2 THEN 'Monday'
        WHEN 3 THEN 'Tuesday'
        WHEN 4 THEN 'Wednesday'
        WHEN 5 THEN 'Thursday'
        WHEN 6 THEN 'Friday'
        WHEN 7 THEN 'Saturday'
    END AS day_name,
    ROUND(consumption_kwh, 3) AS consumption_kwh,
    ROUND(active_power_kw, 2) AS peak_power_kw,
    ROUND(power_factor, 3) AS avg_power_factor,
    -- Thai electricity cost (T1 tariff, peak hours 9-22)
    CASE 
        WHEN hour_of_day BETWEEN 9 AND 21 THEN ROUND(consumption_kwh * 4.18, 2)
        ELSE ROUND(consumption_kwh * 2.56, 2)
    END AS estimated_cost_thb,
    -- Usage category
    CASE 
        WHEN active_power_kw > 180 THEN 'PEAK'
        WHEN active_power_kw > 100 THEN 'HIGH'
        WHEN active_power_kw > 50 THEN 'NORMAL'
        ELSE 'LOW'
    END AS usage_level
FROM hourly_consumption
WHERE consumption_kwh > 0
ORDER BY hour_bucket;
```

---

### Query 7: Heatmap Data (Hour x Day of Week)

```sql
-- สร้างข้อมูลสำหรับ Heatmap: Temperature ตาม Hour x Day
SELECT 
    HOUR(ts) AS hour_of_day,
    DAYNAME(ts) AS day_of_week,
    DAYOFWEEK(ts) AS day_num,
    COUNT(*) AS reading_count,
    ROUND(AVG(value_float), 2) AS avg_temperature,
    ROUND(MIN(value_float), 2) AS min_temp,
    ROUND(MAX(value_float), 2) AS max_temp,
    -- For heatmap color intensity (0-100%)
    ROUND(
        (AVG(value_float) - MIN(AVG(value_float)) OVER ()) * 100.0 / 
        NULLIF(MAX(AVG(value_float)) OVER () - MIN(AVG(value_float)) OVER (), 0),
        1
    ) AS intensity_pct
FROM sensor_readings
WHERE device_id = 1
  AND metric_code = 'temp'
  AND ts >= DATE_SUB(CURDATE(), INTERVAL 30 DAY)
GROUP BY HOUR(ts), DAYNAME(ts), DAYOFWEEK(ts)
ORDER BY day_num, hour_of_day;
```

---

### Query 8: Predictive Maintenance Score

```sql
-- คำนวณ Predictive Maintenance Score สำหรับแต่ละ Device
WITH device_health AS (
    SELECT 
        d.device_id,
        d.name AS device_name,
        d.install_date,
        DATEDIFF(CURDATE(), d.install_date) AS age_days,
        d.battery_pct,
        d.signal_strength,
        -- Count alerts in last 30 days
        COALESCE(alert_count.cnt, 0) AS alerts_30d,
        -- Data quality score (% of expected readings received)
        COALESCE(quality_score.avg_quality, 100) AS avg_data_quality,
        -- Temperature trend (server rooms)
        COALESCE(temp_trend.avg_temp, 0) AS recent_avg_temp
    FROM devices d
    LEFT JOIN (
        SELECT device_id, COUNT(*) AS cnt
        FROM alerts
        WHERE started_at >= DATE_SUB(NOW(), INTERVAL 30 DAY)
        GROUP BY device_id
    ) alert_count ON d.device_id = alert_count.device_id
    LEFT JOIN (
        SELECT device_id, AVG(quality) AS avg_quality
        FROM sensor_readings
        WHERE ts >= DATE_SUB(NOW(), INTERVAL 7 DAY)
        GROUP BY device_id
    ) quality_score ON d.device_id = quality_score.device_id
    LEFT JOIN (
        SELECT device_id, AVG(value_float) AS avg_temp
        FROM sensor_readings
        WHERE metric_code = 'temp'
          AND ts >= DATE_SUB(NOW(), INTERVAL 24 HOUR)
        GROUP BY device_id
    ) temp_trend ON d.device_id = temp_trend.device_id
    WHERE d.status != 'decommissioned'
)
SELECT 
    device_id,
    device_name,
    age_days,
    battery_pct,
    alerts_30d,
    ROUND(avg_data_quality, 1) AS data_quality_pct,
    -- Health Score (0-100, higher = better)
    GREATEST(0, ROUND(
        100
        - LEAST(50, age_days / 730 * 30)  -- Age penalty: up to 30 pts for 2yr+
        - CASE WHEN battery_pct < 15 THEN 25 
               WHEN battery_pct < 30 THEN 15
               WHEN battery_pct < 50 THEN 5 
               ELSE 0 END  -- Battery penalty
        - LEAST(30, alerts_30d * 5)  -- Alert penalty
        - GREATEST(0, 100 - avg_data_quality) * 0.5  -- Data quality penalty
    , 0)) AS health_score,
    -- Maintenance recommendation
    CASE 
        WHEN battery_pct < 15 THEN 'URGENT: Replace battery'
        WHEN alerts_30d > 5 THEN 'URGENT: Investigate frequent alerts'
        WHEN age_days > 1095 THEN 'SCHEDULED: 3-year maintenance due'
        WHEN battery_pct < 30 THEN 'Plan battery replacement soon'
        ELSE 'Normal monitoring'
    END AS maintenance_action
FROM device_health
ORDER BY health_score ASC;
```

---

### Query 9: CO2 Occupancy Inference

```sql
-- อนุมาน Occupancy จาก CO2 Level (>700 ppm = occupied)
WITH co2_with_occupancy AS (
    SELECT 
        device_id,
        ts,
        ROUND(value_float, 0) AS co2_ppm,
        LAG(ROUND(value_float, 0)) OVER (PARTITION BY device_id ORDER BY ts) AS prev_co2,
        -- Classify occupancy
        CASE 
            WHEN value_float > 1200 THEN 'CROWDED'
            WHEN value_float > 900 THEN 'FULL'
            WHEN value_float > 700 THEN 'OCCUPIED'
            WHEN value_float > 500 THEN 'PARTIAL'
            ELSE 'EMPTY'
        END AS occupancy_level,
        -- Ventilation needed
        CASE WHEN value_float > 1000 THEN TRUE ELSE FALSE END AS needs_ventilation
    FROM sensor_readings
    WHERE device_id = 4
      AND metric_code = 'co2'
      AND ts BETWEEN '2024-02-01 00:00:00' AND '2024-02-01 23:59:59'
),
occupancy_events AS (
    SELECT 
        ts,
        co2_ppm,
        occupancy_level,
        needs_ventilation,
        -- Detect state changes
        CASE WHEN occupancy_level != LAG(occupancy_level) OVER (ORDER BY ts) 
             THEN 1 ELSE 0 END AS state_changed
    FROM co2_with_occupancy
)
SELECT 
    ts,
    co2_ppm,
    occupancy_level,
    needs_ventilation,
    -- Time in each state
    TIMESTAMPDIFF(MINUTE, ts, 
        LEAD(ts) OVER (ORDER BY ts)
    ) AS duration_min
FROM occupancy_events
ORDER BY ts;
```

---

### Query 10: Energy Peak Demand Alert

```sql
-- ตรวจสอบ Demand Charges: Peak 15-minute demand
WITH demand_15min AS (
    SELECT 
        meter_id,
        FROM_UNIXTIME(FLOOR(UNIX_TIMESTAMP(ts) / 900) * 900) AS bucket_15min,
        ROUND(AVG(active_power_kw), 2) AS avg_demand_kw,
        ROUND(MAX(active_power_kw), 2) AS peak_demand_kw
    FROM energy_readings
    WHERE meter_id = 1
      AND HOUR(ts) BETWEEN 9 AND 21  -- Peak hours
      AND DAYOFWEEK(ts) BETWEEN 2 AND 6  -- Mon-Fri
    GROUP BY meter_id, FLOOR(UNIX_TIMESTAMP(ts) / 900)
),
monthly_peak AS (
    SELECT 
        meter_id,
        DATE_FORMAT(bucket_15min, '%Y-%m') AS month,
        MAX(avg_demand_kw) AS monthly_peak_demand_kw
    FROM demand_15min
    GROUP BY meter_id, DATE_FORMAT(bucket_15min, '%Y-%m')
)
SELECT 
    d15.meter_id,
    d15.bucket_15min AS time_slot,
    d15.avg_demand_kw,
    d15.peak_demand_kw,
    mp.monthly_peak_demand_kw,
    em.contracted_demand_kva,
    -- Demand charge (excess over contracted)
    CASE 
        WHEN d15.avg_demand_kw > em.contracted_demand_kva THEN
            ROUND((d15.avg_demand_kw - em.contracted_demand_kva) * 132.93, 2)
        ELSE 0
    END AS excess_demand_charge_thb,
    -- % of contracted demand
    ROUND(d15.avg_demand_kw * 100.0 / NULLIF(em.contracted_demand_kva, 0), 1) AS demand_utilization_pct
FROM demand_15min d15
JOIN monthly_peak mp ON d15.meter_id = mp.meter_id
    AND DATE_FORMAT(d15.bucket_15min, '%Y-%m') = mp.month
JOIN energy_meters em ON d15.meter_id = em.meter_id
ORDER BY d15.avg_demand_kw DESC
LIMIT 20;
```

---

## แบบฝึกหัด (Challenge Exercises)

1. **Event Detection**: เขียน Query ตรวจจับ Events เช่น "ประตูเปิด-ปิด" จาก PIR Sensor โดยนับ State Changes ในช่วงเวลา และ Duration ของแต่ละ Event

2. **Fleet Management**: ออกแบบ Schema สำหรับ GPS Tracking ของรถยนต์ แล้วเขียน Query คำนวณ Distance traveled, Average speed, Idling time ต่อวัน

3. **Solar Performance Ratio**: เขียน Query คำนวณ Performance Ratio ของ Solar Panel = Actual Energy / (Irradiance * System Size) และเปรียบเทียบกับ Expected PR

4. **Water Leak Detection**: ออกแบบ Algorithm ตรวจจับน้ำรั่วจาก Water Meter readings โดยดู Night Flow Rate ที่ไม่ควรมีการใช้น้ำ

5. **Time-Series Compression**: เขียน Query ทำ Ramer-Douglas-Peucker Line Simplification แบบง่าย บน Time-Series Data เพื่อลด Storage โดยเก็บเฉพาะ Inflection Points

6. **Cross-Sensor Correlation**: วิเคราะห์ Correlation ระหว่าง CO2 levels กับ Temperature ในห้องเดียวกัน โดยใช้ Pearson Correlation Coefficient

7. **Predictive Temperature**: สร้าง Simple Linear Regression ใน SQL เพื่อทำนาย Temperature ในชั่วโมงถัดไป จาก Current Temperature และ Rate of Change

8. **Energy Benchmarking**: เปรียบเทียบ Energy Intensity (kWh/m²) ของแต่ละ Floor กับ Industry Benchmark สำหรับ Office Buildings ในประเทศไทย

9. **Occupancy Heatmap**: สร้าง Occupancy Pattern Heatmap (Hour x Day) จาก CO2 Sensor Data สำหรับวางแผน AC Scheduling

10. **Retention and Data Lifecycle**: เขียน Stored Procedure สำหรับ Data Retention Policy: ย้ายข้อมูล raw readings ที่เก่ากว่า 90 วัน ไปยัง Summary Table แล้ว DELETE ออกจาก raw table
