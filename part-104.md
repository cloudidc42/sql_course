# ตอนที่ 104: SQL Server (T-SQL) Advanced Features

## บทนำ

Microsoft SQL Server เป็นระบบฐานข้อมูลระดับ enterprise ที่ใช้ภาษา T-SQL (Transact-SQL) ซึ่งเป็นการขยาย SQL standard ของ Microsoft ในบทนี้เราจะศึกษาคุณสมบัติขั้นสูงของ T-SQL ที่จำเป็นสำหรับการพัฒนาระบบระดับ enterprise

---

## 1. T-SQL Specifics

### 1.1 TOP Clause

```sql
-- TOP กับจำนวนแถว
SELECT TOP 10 *
FROM orders
ORDER BY order_date DESC;

-- TOP กับ percentage
SELECT TOP 10 PERCENT *
FROM products
ORDER BY price DESC;

-- TOP WITH TIES: รวมแถวที่มีค่าเท่ากับแถวสุดท้าย
SELECT TOP 5 WITH TIES
    salesperson,
    SUM(amount) AS total_sales
FROM sales
GROUP BY salesperson
ORDER BY total_sales DESC;
-- ถ้าตำแหน่ง 5 และ 6 มียอดขายเท่ากัน ทั้งคู่จะถูกรวม

-- TOP ใน UPDATE/DELETE
UPDATE TOP (100) customer_scores
SET score = score * 1.05
WHERE last_purchase > GETDATE() - 30;

DELETE TOP (1000) FROM audit_logs
WHERE created_at < DATEADD(MONTH, -6, GETDATE());
```

### 1.2 NOLOCK Hint

```sql
-- NOLOCK: อ่านข้อมูลโดยไม่ต้อง acquire shared lock
-- เร็วกว่าแต่อาจอ่านข้อมูลที่ยังไม่ commit (dirty reads)
SELECT * FROM large_table WITH (NOLOCK)
WHERE status = 'active';

-- ใช้สำหรับ reporting queries ที่ไม่ต้องการความ consistent เป๊ะ ๆ
SELECT 
    COUNT(*) AS total_orders,
    SUM(amount) AS total_revenue
FROM orders WITH (NOLOCK)
WHERE order_date >= '2024-01-01';

-- READPAST: ข้าม rows ที่ถูก lock
SELECT TOP 10 *
FROM job_queue WITH (READPAST)
WHERE status = 'pending'
ORDER BY priority DESC;

-- UPDLOCK: lock สำหรับ update (ป้องกัน deadlock)
BEGIN TRANSACTION;
SELECT * FROM inventory WITH (UPDLOCK)
WHERE product_id = 101;
-- ทำการตรวจสอบ...
UPDATE inventory SET quantity = quantity - 1 WHERE product_id = 101;
COMMIT;

-- ROWLOCK: lock ระดับ row แทน page lock
SELECT * FROM products WITH (ROWLOCK, UPDLOCK)
WHERE id = 500;

-- XLOCK: exclusive lock
SELECT * FROM critical_data WITH (XLOCK)
WHERE id = 1;

-- TABLOCK: lock ทั้ง table
SELECT * FROM reports WITH (TABLOCK);

-- PAGLOCK: page-level lock
INSERT INTO big_table WITH (PAGLOCK) VALUES (1, 'data');
```

### 1.3 Query Hints อื่น ๆ

```sql
-- OPTION clause สำหรับ query hints
SELECT *
FROM orders o
JOIN customers c ON o.customer_id = c.id
OPTION (
    HASH JOIN,           -- บังคับใช้ hash join
    MAXDOP 4,            -- maximum degree of parallelism
    OPTIMIZE FOR (@id = 100),  -- optimize สำหรับค่าที่ระบุ
    RECOMPILE            -- compile ใหม่ทุกครั้ง
);

-- Optimizer hints
SELECT *
FROM employees
WHERE department = 'IT'
OPTION (INDEX(idx_department));  -- บังคับใช้ index นี้

-- FORCESEEK: บังคับ index seek
SELECT * FROM orders WITH (FORCESEEK)
WHERE customer_id = 123;

-- FORCESCAN: บังคับ table scan
SELECT * FROM small_lookup WITH (FORCESCAN);
```

---

## 2. SQL Server System Tables และ DMVs

### 2.1 System Catalog Views

```sql
-- ดูตารางทั้งหมดใน database
SELECT 
    t.name AS table_name,
    s.name AS schema_name,
    t.create_date,
    t.modify_date,
    p.rows AS row_count
FROM sys.tables t
JOIN sys.schemas s ON t.schema_id = s.schema_id
JOIN sys.partitions p ON t.object_id = p.object_id
WHERE p.index_id IN (0, 1)
ORDER BY p.rows DESC;

-- ดู columns ของตาราง
SELECT 
    c.name AS column_name,
    tp.name AS data_type,
    c.max_length,
    c.precision,
    c.scale,
    c.is_nullable,
    c.is_identity
FROM sys.columns c
JOIN sys.types tp ON c.user_type_id = tp.user_type_id
JOIN sys.tables t ON c.object_id = t.object_id
WHERE t.name = 'orders'
ORDER BY c.column_id;

-- ดู indexes
SELECT 
    t.name AS table_name,
    i.name AS index_name,
    i.type_desc AS index_type,
    i.is_unique,
    i.is_primary_key,
    STRING_AGG(c.name, ', ') WITHIN GROUP (ORDER BY ic.key_ordinal) AS columns
FROM sys.indexes i
JOIN sys.tables t ON i.object_id = t.object_id
JOIN sys.index_columns ic ON i.object_id = ic.object_id AND i.index_id = ic.index_id
JOIN sys.columns c ON ic.object_id = c.object_id AND ic.column_id = c.column_id
WHERE t.is_ms_shipped = 0
GROUP BY t.name, i.name, i.type_desc, i.is_unique, i.is_primary_key
ORDER BY t.name, i.name;

-- ดู stored procedures
SELECT 
    ROUTINE_NAME,
    ROUTINE_TYPE,
    CREATED,
    LAST_ALTERED
FROM INFORMATION_SCHEMA.ROUTINES
WHERE ROUTINE_TYPE = 'PROCEDURE'
ORDER BY LAST_ALTERED DESC;

-- ดู foreign keys
SELECT 
    fk.name AS constraint_name,
    tp.name AS parent_table,
    tr.name AS referenced_table,
    STRING_AGG(c_parent.name, ', ') AS parent_columns,
    STRING_AGG(c_ref.name, ', ') AS referenced_columns
FROM sys.foreign_keys fk
JOIN sys.tables tp ON fk.parent_object_id = tp.object_id
JOIN sys.tables tr ON fk.referenced_object_id = tr.object_id
JOIN sys.foreign_key_columns fkc ON fk.object_id = fkc.constraint_object_id
JOIN sys.columns c_parent ON fkc.parent_object_id = c_parent.object_id 
    AND fkc.parent_column_id = c_parent.column_id
JOIN sys.columns c_ref ON fkc.referenced_object_id = c_ref.object_id 
    AND fkc.referenced_column_id = c_ref.column_id
GROUP BY fk.name, tp.name, tr.name;
```

### 2.2 Dynamic Management Views (DMVs)

```sql
-- Active connections
SELECT 
    s.session_id,
    s.login_name,
    s.status,
    s.cpu_time,
    s.memory_usage,
    s.total_elapsed_time / 1000 AS elapsed_seconds,
    r.command,
    r.sql_handle,
    r.wait_type,
    t.text AS query_text
FROM sys.dm_exec_sessions s
LEFT JOIN sys.dm_exec_requests r ON s.session_id = r.session_id
OUTER APPLY sys.dm_exec_sql_text(r.sql_handle) t
WHERE s.is_user_process = 1
ORDER BY s.cpu_time DESC;

-- Missing indexes suggestion
SELECT 
    mig.equality_columns,
    mig.inequality_columns,
    mig.included_columns,
    migs.avg_total_user_cost * migs.avg_user_impact * (migs.user_seeks + migs.user_scans) AS improvement_measure,
    'CREATE INDEX [IX_' + OBJECT_NAME(mid.object_id) + '_' + 
        REPLACE(REPLACE(mig.equality_columns, '[', ''), ']', '') + 
        '] ON ' + OBJECT_NAME(mid.object_id) + 
        ' (' + mig.equality_columns + 
        ISNULL(', ' + mig.inequality_columns, '') + ')' +
        ISNULL(' INCLUDE (' + mig.included_columns + ')', '') AS create_index_statement
FROM sys.dm_db_missing_index_details mid
JOIN sys.dm_db_missing_index_groups mig ON mid.index_handle = mig.index_handle
JOIN sys.dm_db_missing_index_group_stats migs ON mig.index_group_handle = migs.group_handle
WHERE mid.database_id = DB_ID()
ORDER BY improvement_measure DESC;

-- Index usage statistics
SELECT 
    OBJECT_NAME(i.object_id) AS table_name,
    i.name AS index_name,
    ius.user_seeks,
    ius.user_scans,
    ius.user_lookups,
    ius.user_updates,
    ius.last_user_seek,
    ius.last_user_scan
FROM sys.indexes i
LEFT JOIN sys.dm_db_index_usage_stats ius 
    ON i.object_id = ius.object_id 
    AND i.index_id = ius.index_id
    AND ius.database_id = DB_ID()
WHERE OBJECTPROPERTY(i.object_id, 'IsUserTable') = 1
ORDER BY ius.user_seeks + ius.user_scans DESC;

-- Query performance stats
SELECT TOP 20
    qs.total_elapsed_time / qs.execution_count AS avg_elapsed_ms,
    qs.execution_count,
    qs.total_logical_reads / qs.execution_count AS avg_reads,
    qs.total_cpu_time / qs.execution_count AS avg_cpu,
    SUBSTRING(qt.text, (qs.statement_start_offset/2) + 1,
        ((CASE qs.statement_end_offset WHEN -1 THEN DATALENGTH(qt.text)
        ELSE qs.statement_end_offset END - qs.statement_start_offset)/2) + 1) AS query_text
FROM sys.dm_exec_query_stats qs
CROSS APPLY sys.dm_exec_sql_text(qs.sql_handle) qt
ORDER BY avg_elapsed_ms DESC;
```

---

## 3. sp_executesql - Dynamic SQL

### 3.1 ความปลอดภัยด้วย sp_executesql

```sql
-- ไม่ปลอดภัย: SQL Injection susceptible
DECLARE @TableName NVARCHAR(100) = 'users';
EXEC('SELECT * FROM ' + @TableName);

-- ปลอดภัย: ใช้ sp_executesql กับ parameters
DECLARE @SQL NVARCHAR(500);
DECLARE @UserId INT = 101;
DECLARE @Status NVARCHAR(20) = 'active';

SET @SQL = N'SELECT * FROM users WHERE id = @id AND status = @status';

EXEC sp_executesql 
    @SQL,
    N'@id INT, @status NVARCHAR(20)',
    @id = @UserId,
    @status = @Status;

-- sp_executesql กับ OUTPUT parameters
DECLARE @Count INT;
SET @SQL = N'SELECT @cnt = COUNT(*) FROM orders WHERE customer_id = @cid';

EXEC sp_executesql 
    @SQL,
    N'@cid INT, @cnt INT OUTPUT',
    @cid = 101,
    @cnt = @Count OUTPUT;

PRINT 'Order count: ' + CAST(@Count AS NVARCHAR(10));
```

### 3.2 Dynamic Pivot

```sql
-- Dynamic PIVOT: columns จาก data
DECLARE @Columns NVARCHAR(MAX);
DECLARE @SQL NVARCHAR(MAX);

-- หา column names จาก data
SELECT @Columns = STRING_AGG(QUOTENAME(CONVERT(NVARCHAR(7), month_date, 120)), ', ')
FROM (
    SELECT DISTINCT DATETRUNC(MONTH, order_date) AS month_date
    FROM orders
    WHERE order_date >= DATEADD(YEAR, -1, GETDATE())
) AS months;

-- สร้าง dynamic PIVOT query
SET @SQL = N'
SELECT product_name, ' + @Columns + '
FROM (
    SELECT 
        p.name AS product_name,
        CONVERT(NVARCHAR(7), o.order_date, 120) AS order_month,
        oi.quantity
    FROM order_items oi
    JOIN orders o ON oi.order_id = o.id
    JOIN products p ON oi.product_id = p.id
) AS source
PIVOT (
    SUM(quantity) FOR order_month IN (' + @Columns + ')
) AS pivot_table
ORDER BY product_name;';

EXEC sp_executesql @SQL;
```

---

## 4. TRY/CATCH ใน T-SQL

### 4.1 Error Handling Pattern

```sql
-- Basic TRY/CATCH
BEGIN TRY
    BEGIN TRANSACTION;
    
    INSERT INTO orders (customer_id, product_id, quantity, amount)
    VALUES (101, 1, 5, 2500.00);
    
    UPDATE inventory
    SET quantity = quantity - 5
    WHERE product_id = 1;
    
    IF (SELECT quantity FROM inventory WHERE product_id = 1) < 0
    BEGIN
        RAISERROR('Insufficient inventory', 16, 1);
    END;
    
    COMMIT TRANSACTION;
    PRINT 'Order placed successfully';
    
END TRY
BEGIN CATCH
    IF @@TRANCOUNT > 0
        ROLLBACK TRANSACTION;
    
    DECLARE @ErrorMessage NVARCHAR(4000) = ERROR_MESSAGE();
    DECLARE @ErrorSeverity INT = ERROR_SEVERITY();
    DECLARE @ErrorState INT = ERROR_STATE();
    DECLARE @ErrorLine INT = ERROR_LINE();
    DECLARE @ErrorProcedure NVARCHAR(200) = ERROR_PROCEDURE();
    
    -- Log error
    INSERT INTO error_log (
        error_message, error_severity, error_state,
        error_line, error_procedure, error_time
    )
    VALUES (
        @ErrorMessage, @ErrorSeverity, @ErrorState,
        @ErrorLine, @ErrorProcedure, GETDATE()
    );
    
    -- Re-throw error
    THROW;
    -- หรือ RAISERROR เดิม:
    -- RAISERROR(@ErrorMessage, @ErrorSeverity, @ErrorState);
END CATCH;

-- Nested TRY/CATCH
BEGIN TRY
    -- Outer operation
    BEGIN TRY
        -- Inner risky operation
        EXEC risky_stored_procedure;
    END TRY
    BEGIN CATCH
        -- Handle inner error, maybe ignore
        PRINT 'Inner error handled: ' + ERROR_MESSAGE();
    END CATCH;
    
    -- Continue with outer operation
    INSERT INTO processed_records VALUES (1, 'done');
END TRY
BEGIN CATCH
    PRINT 'Outer error: ' + ERROR_MESSAGE();
    THROW;
END CATCH;
```

### 4.2 Custom Error Messages

```sql
-- สร้าง custom error messages
EXEC sp_addmessage 
    @msgnum = 50001,
    @severity = 16,
    @msgtext = 'Product %s is out of stock. Available: %d units.';

-- ใช้ custom message
RAISERROR(50001, 16, 1, 'iPhone 15', 0);

-- THROW (SQL Server 2012+): ง่ายกว่า
THROW 50001, 'Custom error message here', 1;

-- ใน TRY/CATCH: re-throw ด้วย THROW ไม่มี parameters
BEGIN TRY
    SELECT 1/0;
END TRY
BEGIN CATCH
    -- ทำ cleanup...
    THROW;  -- re-throw error เดิม
END CATCH;
```

---

## 5. Cursors ใน T-SQL

### 5.1 การใช้งาน Cursors

```sql
-- Cursors ช้าและควรหลีกเลี่ยงถ้าทำได้ด้วย set-based operations
-- แต่บางครั้งจำเป็น เช่น row-by-row processing

DECLARE @CustomerID INT;
DECLARE @CustomerName NVARCHAR(100);
DECLARE @OrderCount INT;

-- สร้าง cursor
DECLARE customer_cursor CURSOR
    LOCAL                   -- local ใน scope นี้
    FAST_FORWARD            -- forward-only, read-only
    FOR
    SELECT id, name
    FROM customers
    WHERE status = 'active'
    ORDER BY id;

OPEN customer_cursor;

FETCH NEXT FROM customer_cursor INTO @CustomerID, @CustomerName;

WHILE @@FETCH_STATUS = 0
BEGIN
    -- Process each customer
    SELECT @OrderCount = COUNT(*)
    FROM orders
    WHERE customer_id = @CustomerID;
    
    IF @OrderCount > 10
    BEGIN
        UPDATE customers
        SET loyalty_tier = 'Gold'
        WHERE id = @CustomerID;
        
        PRINT 'Upgraded ' + @CustomerName + ' to Gold tier';
    END;
    
    FETCH NEXT FROM customer_cursor INTO @CustomerID, @CustomerName;
END;

CLOSE customer_cursor;
DEALLOCATE customer_cursor;

-- STATIC cursor: snapshot ของข้อมูล (ไม่เห็นการเปลี่ยนแปลง)
DECLARE static_cursor CURSOR STATIC FOR
SELECT * FROM products;

-- KEYSET cursor: เห็น updates แต่ไม่เห็น inserts/deletes
DECLARE keyset_cursor CURSOR KEYSET FOR
SELECT id, name FROM products;

-- DYNAMIC cursor: เห็นทุกการเปลี่ยนแปลง (ช้าที่สุด)
DECLARE dynamic_cursor CURSOR DYNAMIC FOR
SELECT id, stock FROM inventory;

-- UPDATE ผ่าน cursor
DECLARE update_cursor CURSOR
    FOR SELECT id, price FROM products FOR UPDATE;

OPEN update_cursor;
FETCH NEXT FROM update_cursor;

WHILE @@FETCH_STATUS = 0
BEGIN
    UPDATE products
    SET price = price * 1.05
    WHERE CURRENT OF update_cursor;  -- update current row
    
    FETCH NEXT FROM update_cursor;
END;

CLOSE update_cursor;
DEALLOCATE update_cursor;
```

### 5.2 Cursor Alternatives (แนะนำ)

```sql
-- แทน cursor ด้วย set-based operation
-- ไม่แนะนำ: cursor
DECLARE @id INT;
DECLARE cur CURSOR FOR SELECT id FROM customers;
OPEN cur;
FETCH NEXT FROM cur INTO @id;
WHILE @@FETCH_STATUS = 0
BEGIN
    UPDATE customers SET total_orders = (SELECT COUNT(*) FROM orders WHERE customer_id = @id)
    WHERE id = @id;
    FETCH NEXT FROM cur INTO @id;
END;
CLOSE cur; DEALLOCATE cur;

-- แนะนำ: set-based UPDATE
UPDATE customers
SET total_orders = (
    SELECT COUNT(*) FROM orders WHERE orders.customer_id = customers.id
);

-- หรือใช้ UPDATE with JOIN
UPDATE c
SET c.total_orders = o.order_count
FROM customers c
JOIN (
    SELECT customer_id, COUNT(*) AS order_count
    FROM orders
    GROUP BY customer_id
) o ON c.id = o.customer_id;
```

---

## 6. Table Variables vs Temp Tables

### 6.1 Table Variables

```sql
-- Table variable: เก็บใน memory (mostly)
DECLARE @OrderSummary TABLE (
    customer_id INT,
    total_orders INT,
    total_amount DECIMAL(12,2),
    last_order_date DATE
);

INSERT INTO @OrderSummary
SELECT 
    customer_id,
    COUNT(*) AS total_orders,
    SUM(amount) AS total_amount,
    MAX(order_date) AS last_order_date
FROM orders
WHERE order_date >= DATEADD(YEAR, -1, GETDATE())
GROUP BY customer_id;

-- ใช้ table variable ใน query
SELECT 
    c.name,
    os.total_orders,
    os.total_amount
FROM customers c
JOIN @OrderSummary os ON c.id = os.customer_id
WHERE os.total_amount > 10000
ORDER BY os.total_amount DESC;

-- ข้อจำกัดของ table variable:
-- 1. ไม่มี statistics -> query planner ประมาณ 1 row เสมอ
-- 2. ไม่สามารถ CREATE INDEX (ยกเว้น PRIMARY KEY และ UNIQUE constraints)
-- 3. ไม่ rollback เมื่อ transaction rollback
```

### 6.2 Temporary Tables

```sql
-- Local temp table: ใช้ได้เฉพาะ session นี้
CREATE TABLE #TempOrders (
    customer_id INT,
    total_orders INT,
    total_amount DECIMAL(12,2),
    last_order_date DATE,
    INDEX idx_customer (customer_id)
);

INSERT INTO #TempOrders
SELECT 
    customer_id,
    COUNT(*) AS total_orders,
    SUM(amount) AS total_amount,
    MAX(order_date) AS last_order_date
FROM orders
WHERE order_date >= DATEADD(YEAR, -1, GETDATE())
GROUP BY customer_id;

-- สร้าง index เพิ่มเติม
CREATE INDEX idx_amount ON #TempOrders (total_amount DESC);

-- ดี: มี statistics -> query plan ถูกต้องกว่า
UPDATE STATISTICS #TempOrders;

EXPLAIN 
SELECT * FROM customers c
JOIN #TempOrders t ON c.id = t.customer_id;

DROP TABLE IF EXISTS #TempOrders;

-- Global temp table: ใช้ได้ทุก session
CREATE TABLE ##GlobalTemp (
    data NVARCHAR(MAX)
);

-- เมื่อ session ที่สร้างปิด, ##GlobalTemp จะถูกลบอัตโนมัติ
```

### 6.3 เปรียบเทียบ

```sql
-- ใช้ Table Variable เมื่อ:
-- - ข้อมูลน้อย (< 1000 rows)
-- - ต้องการ automatic cleanup
-- - ใช้ใน stored procedures ที่ call บ่อย

-- ใช้ Temp Tables เมื่อ:
-- - ข้อมูลมาก (> 1000 rows)
-- - ต้องการ indexes
-- - ต้องการ statistics ที่ถูกต้อง
-- - reuse ใน scope เดียวกัน

-- Performance comparison
SET STATISTICS TIME ON;
SET STATISTICS IO ON;

-- ทดสอบ table variable
DECLARE @tv TABLE (id INT, val NVARCHAR(100));
INSERT INTO @tv SELECT TOP 10000 id, name FROM big_table;
SELECT COUNT(*) FROM @tv WHERE val LIKE 'A%';

-- ทดสอบ temp table  
CREATE TABLE #tt (id INT, val NVARCHAR(100));
INSERT INTO #tt SELECT TOP 10000 id, name FROM big_table;
CREATE INDEX idx ON #tt (val);
SELECT COUNT(*) FROM #tt WHERE val LIKE 'A%';
DROP TABLE #tt;

SET STATISTICS TIME OFF;
SET STATISTICS IO OFF;
```

---

## 7. Common T-SQL Patterns

### 7.1 Upsert Pattern (MERGE)

```sql
-- MERGE statement
MERGE INTO product_inventory AS target
USING (
    SELECT product_id, quantity_change, warehouse_id
    FROM incoming_shipments
    WHERE processed = 0
) AS source
ON target.product_id = source.product_id 
AND target.warehouse_id = source.warehouse_id
WHEN MATCHED THEN
    UPDATE SET 
        quantity = target.quantity + source.quantity_change,
        last_updated = GETDATE()
WHEN NOT MATCHED BY TARGET THEN
    INSERT (product_id, warehouse_id, quantity, last_updated)
    VALUES (source.product_id, source.warehouse_id, source.quantity_change, GETDATE())
WHEN NOT MATCHED BY SOURCE AND target.quantity = 0 THEN
    DELETE
OUTPUT 
    $action AS merge_action,
    INSERTED.product_id,
    INSERTED.quantity,
    DELETED.quantity AS old_quantity
INTO merge_log (action, product_id, new_quantity, old_quantity);

-- Mark as processed
UPDATE incoming_shipments SET processed = 1
WHERE processed = 0;
```

### 7.2 Pagination Pattern

```sql
-- เก่า: ROW_NUMBER
WITH PagedResults AS (
    SELECT 
        *,
        ROW_NUMBER() OVER (ORDER BY created_at DESC) AS RowNum
    FROM orders
    WHERE customer_id = 101
)
SELECT * FROM PagedResults
WHERE RowNum BETWEEN 11 AND 20;  -- หน้า 2, 10 rows per page

-- ใหม่ (SQL Server 2012+): OFFSET FETCH
SELECT *
FROM orders
WHERE customer_id = 101
ORDER BY created_at DESC
OFFSET 10 ROWS           -- ข้าม 10 rows แรก
FETCH NEXT 10 ROWS ONLY; -- ดึง 10 rows ถัดไป

-- Stored procedure สำหรับ pagination
CREATE OR ALTER PROCEDURE sp_GetOrdersPaged
    @CustomerId INT,
    @PageNumber INT = 1,
    @PageSize INT = 20,
    @SortColumn NVARCHAR(50) = 'created_at',
    @SortDirection NVARCHAR(4) = 'DESC'
AS
BEGIN
    SET NOCOUNT ON;
    
    DECLARE @Offset INT = (@PageNumber - 1) * @PageSize;
    DECLARE @SQL NVARCHAR(MAX);
    
    -- Count total
    SELECT COUNT(*) AS TotalCount
    FROM orders
    WHERE customer_id = @CustomerId;
    
    -- Get page
    SET @SQL = N'
    SELECT *
    FROM orders
    WHERE customer_id = @cid
    ORDER BY ' + QUOTENAME(@SortColumn) + ' ' + @SortDirection + '
    OFFSET @offset ROWS
    FETCH NEXT @pageSize ROWS ONLY';
    
    EXEC sp_executesql 
        @SQL,
        N'@cid INT, @offset INT, @pageSize INT',
        @cid = @CustomerId,
        @offset = @Offset,
        @pageSize = @PageSize;
END;
```

### 7.3 String Aggregation

```sql
-- STRING_AGG (SQL Server 2017+)
SELECT 
    department,
    STRING_AGG(employee_name, ', ') WITHIN GROUP (ORDER BY employee_name) AS employees
FROM employees
GROUP BY department;

-- STUFF + FOR XML PATH (เก่ากว่า)
SELECT DISTINCT
    department,
    STUFF((
        SELECT ', ' + e2.employee_name
        FROM employees e2
        WHERE e2.department = e.department
        ORDER BY e2.employee_name
        FOR XML PATH(''), TYPE
    ).value('.', 'NVARCHAR(MAX)'), 1, 2, '') AS employees
FROM employees e
ORDER BY department;
```

---

## 8. SQL Server-specific Functions

### 8.1 ISNULL, IIF, CHOOSE

```sql
-- ISNULL: แทน NULL ด้วยค่าที่ระบุ (SQL Server specific)
SELECT ISNULL(discount, 0) AS discount FROM orders;
-- COALESCE เป็น SQL standard ที่ยืดหยุ่นกว่า:
SELECT COALESCE(discount, promo_discount, 0) AS discount FROM orders;

-- IIF: inline IF-ELSE (SQL Server 2012+)
SELECT 
    product_name,
    price,
    IIF(price > 10000, 'Premium', 'Standard') AS tier
FROM products;

-- เทียบกับ CASE:
SELECT 
    product_name,
    CASE WHEN price > 10000 THEN 'Premium' ELSE 'Standard' END AS tier
FROM products;

-- CHOOSE: เลือกค่าจาก list ตาม index
SELECT CHOOSE(MONTH(GETDATE()), 
    'January', 'February', 'March', 'April',
    'May', 'June', 'July', 'August',
    'September', 'October', 'November', 'December'
) AS current_month;

-- ใช้ CHOOSE กับ quarter
SELECT 
    order_id,
    CHOOSE(DATEPART(QUARTER, order_date), 'Q1', 'Q2', 'Q3', 'Q4') AS quarter
FROM orders;

-- TRY_CAST, TRY_CONVERT: safe type conversion
SELECT TRY_CAST('123abc' AS INT);           -- NULL ไม่ error
SELECT TRY_CAST('2024-01-01' AS DATE);      -- 2024-01-01
SELECT TRY_CONVERT(INT, '123.45');          -- 123
SELECT TRY_CONVERT(DATE, '01/15/2024', 101); -- 2024-01-15

-- EOMONTH: วันสุดท้ายของเดือน
SELECT EOMONTH(GETDATE()) AS last_day_of_month;
SELECT EOMONTH('2024-02-01') AS feb_last_day;  -- 2024-02-29 (leap year)
SELECT EOMONTH('2024-02-01', 1) AS next_month_last;  -- 2024-03-31

-- DATEFROMPARTS, DATETIMEFROMPARTS (SQL Server 2012+)
SELECT DATEFROMPARTS(2024, 3, 15);               -- 2024-03-15
SELECT DATETIMEFROMPARTS(2024, 3, 15, 9, 30, 0, 0);  -- 2024-03-15 09:30:00

-- FORMAT function (ช้า - ใช้ CONVERT แทนถ้าเป็นไปได้)
SELECT FORMAT(GETDATE(), 'dd/MM/yyyy', 'th-TH');  -- Thai format
SELECT FORMAT(1234567.89, 'N2');                   -- 1,234,567.89
SELECT FORMAT(0.1567, 'P');                        -- 15.67%

-- DATEDIFF_BIG (SQL Server 2016+)
SELECT DATEDIFF_BIG(SECOND, '2000-01-01', GETDATE()) AS seconds_since_2000;

-- AT TIME ZONE (SQL Server 2016+)
SELECT GETDATE() AT TIME ZONE 'SE Asia Standard Time' AS bangkok_time;
SELECT GETUTCDATE() AT TIME ZONE 'UTC' AT TIME ZONE 'SE Asia Standard Time' AS local_time;
```

---

## 9. FOR JSON PATH

### 9.1 JSON Output

```sql
-- FOR JSON PATH: แปลง result เป็น JSON
SELECT 
    id,
    name,
    email
FROM customers
WHERE status = 'active'
FOR JSON PATH;
-- [{"id":1,"name":"Alice","email":"alice@example.com"},...]

-- JSON_F52E2B61-18A1-11d1-B105-00805F49916B path alias
SELECT 
    c.id,
    c.name AS [customer.name],     -- nested object
    c.email AS [customer.email],
    o.id AS [orders[0].id],        -- array (ต้องใช้ wrapper)
    o.amount AS [orders[0].amount]
FROM customers c
JOIN orders o ON c.id = o.customer_id
FOR JSON PATH;

-- ROOT: ใส่ root element
SELECT id, name FROM customers
FOR JSON PATH, ROOT('customers');
-- {"customers": [...]}

-- INCLUDE_NULL_VALUES: รวม NULL values
SELECT id, name, phone FROM customers
FOR JSON PATH, INCLUDE_NULL_VALUES;

-- WITHOUT_ARRAY_WRAPPER: สำหรับ single row
SELECT TOP 1 id, name FROM customers
FOR JSON PATH, WITHOUT_ARRAY_WRAPPER;
-- {"id":1,"name":"Alice"}

-- FOR JSON AUTO: automatic JSON structure
SELECT 
    c.id,
    c.name,
    o.id AS order_id,
    o.amount
FROM customers c
JOIN orders o ON c.id = o.customer_id
FOR JSON AUTO;

-- Complex JSON with subqueries
SELECT 
    c.id,
    c.name,
    c.email,
    (
        SELECT 
            o.id,
            o.amount,
            o.status,
            (
                SELECT oi.product_name, oi.quantity, oi.price
                FROM order_items oi
                WHERE oi.order_id = o.id
                FOR JSON PATH
            ) AS items
        FROM orders o
        WHERE o.customer_id = c.id
        FOR JSON PATH
    ) AS orders
FROM customers c
WHERE c.id = 101
FOR JSON PATH, WITHOUT_ARRAY_WRAPPER;

-- JSON_MODIFY: แก้ไข JSON
DECLARE @json NVARCHAR(MAX) = '{"name": "Alice", "age": 30}';
SELECT JSON_MODIFY(@json, '$.age', 31);
SELECT JSON_MODIFY(@json, '$.city', 'Bangkok');
SELECT JSON_MODIFY(@json, '$.hobbies', JSON_QUERY('["reading","coding"]'));

-- OPENJSON: แปลง JSON เป็น rows
DECLARE @data NVARCHAR(MAX) = '[{"id":1,"name":"Alice"},{"id":2,"name":"Bob"}]';

SELECT *
FROM OPENJSON(@data)
WITH (
    id INT '$.id',
    name NVARCHAR(100) '$.name'
);

-- OPENJSON สำหรับ complex nested JSON
SELECT 
    customer.*,
    orders.[order_id],
    orders.[order_amount]
FROM OPENJSON(@complex_json, '$.customers')
WITH (
    id INT '$.id',
    name NVARCHAR(100) '$.name',
    orders NVARCHAR(MAX) '$.orders' AS JSON
) AS customer
CROSS APPLY OPENJSON(customer.orders)
WITH (
    order_id INT '$.id',
    order_amount DECIMAL(10,2) '$.amount'
) AS orders;
```

---

## 10. FOR XML

### 10.1 XML Output

```sql
-- FOR XML RAW: แต่ละ row เป็น <row> element
SELECT id, name FROM customers
FOR XML RAW;
-- <row id="1" name="Alice" /><row id="2" name="Bob" />

-- FOR XML AUTO
SELECT c.id, c.name, o.id, o.amount
FROM customers c JOIN orders o ON c.id = o.customer_id
FOR XML AUTO;

-- FOR XML PATH: ควบคุม structure ได้มากที่สุด
SELECT 
    c.id AS '@id',              -- attribute
    c.name AS 'name',           -- element
    (
        SELECT o.id AS '@id', o.amount AS 'amount'
        FROM orders o
        WHERE o.customer_id = c.id
        FOR XML PATH('order'), TYPE
    ) AS 'orders'
FROM customers c
FOR XML PATH('customer'), ROOT('customers');

-- OUTPUT:
-- <customers>
--   <customer id="1">
--     <name>Alice</name>
--     <orders>
--       <order id="10"><amount>1500.00</amount></order>
--     </orders>
--   </customer>
-- </customers>

-- EXPLICIT mode (complex but flexible)
SELECT 
    1 AS Tag,
    NULL AS Parent,
    c.id AS [customer!1!id],
    c.name AS [customer!1!name],
    NULL AS [order!2!id],
    NULL AS [order!2!amount]
FROM customers c
UNION ALL
SELECT 
    2, 1,
    c.id, c.name,
    o.id, o.amount
FROM customers c JOIN orders o ON c.id = o.customer_id
FOR XML EXPLICIT;
```

---

## 11. SQL Server Agent

### 11.1 Job Management

```sql
-- ดู jobs
SELECT 
    j.name AS job_name,
    j.enabled,
    js.last_run_date,
    js.last_run_time,
    CASE js.last_run_outcome
        WHEN 0 THEN 'Failed'
        WHEN 1 THEN 'Succeeded'
        WHEN 2 THEN 'Retry'
        WHEN 3 THEN 'Cancelled'
        ELSE 'Unknown'
    END AS last_outcome
FROM msdb.dbo.sysjobs j
JOIN msdb.dbo.sysjobservers js ON j.job_id = js.job_id
ORDER BY j.name;

-- ดู job history
SELECT TOP 100
    j.name AS job_name,
    h.step_name,
    h.run_date,
    h.run_time,
    CASE h.run_status
        WHEN 0 THEN 'Failed'
        WHEN 1 THEN 'Succeeded'
        WHEN 2 THEN 'Retry'
        WHEN 3 THEN 'Cancelled'
        WHEN 4 THEN 'In Progress'
    END AS status,
    h.message
FROM msdb.dbo.sysjobs j
JOIN msdb.dbo.sysjobhistory h ON j.job_id = h.job_id
ORDER BY h.run_date DESC, h.run_time DESC;

-- สร้าง SQL Agent Job ด้วย T-SQL
USE msdb;
EXEC sp_add_job 
    @job_name = 'Daily Database Maintenance',
    @enabled = 1,
    @description = 'Runs nightly maintenance tasks';

EXEC sp_add_jobstep 
    @job_name = 'Daily Database Maintenance',
    @step_name = 'Update Statistics',
    @command = 'EXEC sp_updatestats',
    @database_name = 'ProductionDB';

EXEC sp_add_jobstep 
    @job_name = 'Daily Database Maintenance',
    @step_name = 'Rebuild Fragmented Indexes',
    @command = '
    EXEC sys.sp_MSforeachtable 
        @command1 = "IF EXISTS (SELECT * FROM sys.indexes WHERE object_id = OBJECT_ID(''?'') AND type_desc != ''HEAP'' AND avg_fragmentation_in_percent > 30)
                     ALTER INDEX ALL ON ? REBUILD WITH (ONLINE = ON)"',
    @database_name = 'ProductionDB';

EXEC sp_add_schedule 
    @schedule_name = 'Nightly 2AM',
    @freq_type = 4,         -- daily
    @freq_interval = 1,
    @active_start_time = 020000;  -- 2:00 AM

EXEC sp_attach_schedule 
    @job_name = 'Daily Database Maintenance',
    @schedule_name = 'Nightly 2AM';

EXEC sp_add_jobserver 
    @job_name = 'Daily Database Maintenance';
```

---

## 12. Linked Servers

### 12.1 การใช้งาน Linked Servers

```sql
-- สร้าง linked server ไปยัง SQL Server อื่น
EXEC sp_addlinkedserver 
    @server = 'REMOTE_SQL_SERVER',
    @srvproduct = 'SQL Server';

EXEC sp_addlinkedsrvlogin 
    @rmtsrvname = 'REMOTE_SQL_SERVER',
    @useself = 'FALSE',
    @locallogin = NULL,
    @rmtuser = 'remote_user',
    @rmtpassword = 'remote_password';

-- Query ผ่าน linked server
-- [ServerName].[DatabaseName].[SchemaName].[TableName]
SELECT * FROM [REMOTE_SQL_SERVER].[RemoteDB].[dbo].[Customers];

-- สร้าง linked server ไปยัง Oracle
EXEC sp_addlinkedserver
    @server = 'ORACLE_SERVER',
    @srvproduct = 'Oracle',
    @provider = 'OraOLEDB.Oracle',
    @datasrc = 'ORACLEDB';

-- ดู linked servers
SELECT * FROM sys.servers WHERE is_linked = 1;
EXEC sp_linkedservers;

-- Distributed transaction
BEGIN DISTRIBUTED TRANSACTION;
    INSERT INTO [LOCAL_DB].[dbo].[orders] SELECT * FROM [REMOTE_SQL_SERVER].[RemoteDB].[dbo].[new_orders];
    DELETE FROM [REMOTE_SQL_SERVER].[RemoteDB].[dbo].[new_orders];
COMMIT;

-- OPENQUERY: pass query ไปรัน remote
SELECT * FROM OPENQUERY([REMOTE_SQL_SERVER], 
    'SELECT id, name FROM RemoteDB.dbo.customers WHERE active = 1');

-- OPENROWSET: ad-hoc distributed query
SELECT *
FROM OPENROWSET(
    'SQLNCLI',
    'Server=REMOTE_SERVER;Trusted_Connection=yes;',
    'SELECT * FROM RemoteDB.dbo.orders'
);
```

---

## 13. SQL Server CLR

### 13.1 CLR Integration

```sql
-- เปิด CLR integration
EXEC sp_configure 'show advanced options', 1;
RECONFIGURE;
EXEC sp_configure 'clr enabled', 1;
RECONFIGURE;

-- ดู CLR assemblies
SELECT * FROM sys.assemblies;

-- สร้าง assembly จาก DLL
CREATE ASSEMBLY [StringUtilities]
FROM 'C:\Assemblies\StringUtilities.dll'
WITH PERMISSION_SET = SAFE;  -- SAFE, EXTERNAL_ACCESS, UNSAFE

-- สร้าง function จาก CLR method
CREATE FUNCTION dbo.RegexMatch
(
    @input NVARCHAR(MAX),
    @pattern NVARCHAR(MAX)
)
RETURNS BIT
EXTERNAL NAME [StringUtilities].[StringUtilities.Functions].[RegexMatch];

-- ใช้งาน CLR function
SELECT name FROM products
WHERE dbo.RegexMatch(email, '^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$') = 1;

-- CLR aggregate function
-- ตัวอย่าง: Concatenate strings (ก่อน STRING_AGG)
SELECT 
    department,
    dbo.Concatenate(employee_name) AS employee_list
FROM employees
GROUP BY department;
```

---

## 14. ตัวอย่างเพิ่มเติม

```sql
-- Example 1: PIVOT table
SELECT *
FROM (
    SELECT 
        salesperson,
        DATENAME(QUARTER, sale_date) AS quarter,
        amount
    FROM sales
    WHERE YEAR(sale_date) = 2024
) AS source
PIVOT (
    SUM(amount) FOR quarter IN ([Q1], [Q2], [Q3], [Q4])
) AS pivot_table;

-- Example 2: Recursive CTE กับ path
WITH RECURSIVE_CTE AS (
    SELECT id, name, parent_id, 
           CAST(name AS NVARCHAR(MAX)) AS path,
           0 AS depth
    FROM categories WHERE parent_id IS NULL
    UNION ALL
    SELECT c.id, c.name, c.parent_id,
           CAST(r.path + ' > ' + c.name AS NVARCHAR(MAX)),
           r.depth + 1
    FROM categories c
    JOIN RECURSIVE_CTE r ON c.parent_id = r.id
)
SELECT 
    REPLICATE('  ', depth) + name AS category_tree,
    path, depth
FROM RECURSIVE_CTE
ORDER BY path;

-- Example 3: Window functions - running balance
SELECT 
    transaction_date,
    description,
    amount,
    SUM(amount) OVER (
        ORDER BY transaction_date, id
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    ) AS running_balance
FROM bank_transactions
WHERE account_id = 123
ORDER BY transaction_date, id;

-- Example 4: Apply operators
-- CROSS APPLY
SELECT c.id, c.name, recent.order_date, recent.amount
FROM customers c
CROSS APPLY (
    SELECT TOP 3 order_date, amount
    FROM orders
    WHERE customer_id = c.id
    ORDER BY order_date DESC
) AS recent;

-- OUTER APPLY (รวม customers ที่ไม่มี orders)
SELECT c.id, c.name, recent.order_date, recent.amount
FROM customers c
OUTER APPLY (
    SELECT TOP 1 order_date, amount
    FROM orders
    WHERE customer_id = c.id
    ORDER BY order_date DESC
) AS recent;

-- Example 5: Filtered Index
CREATE INDEX idx_active_orders
ON orders (customer_id, created_at)
INCLUDE (amount, status)
WHERE status = 'pending';  -- filtered index

-- Query ที่ใช้ filtered index
SELECT customer_id, SUM(amount)
FROM orders
WHERE status = 'pending'
GROUP BY customer_id;
```

---

## แบบฝึกหัด

### ข้อที่ 1: Dynamic SQL Pivot Report
สร้าง stored procedure ที่ generate monthly sales report แบบ pivot ด้วย dynamic SQL

**เฉลย:**
```sql
CREATE OR ALTER PROCEDURE sp_MonthlySalesPivot
    @Year INT = NULL,
    @Region NVARCHAR(50) = NULL
AS
BEGIN
    SET NOCOUNT ON;
    
    IF @Year IS NULL SET @Year = YEAR(GETDATE());
    
    DECLARE @Months NVARCHAR(MAX) = '';
    DECLARE @SQL NVARCHAR(MAX);
    
    -- สร้าง month columns
    SELECT @Months = @Months + ', [' + DATENAME(MONTH, DATEFROMPARTS(@Year, n, 1)) + ']'
    FROM (VALUES(1),(2),(3),(4),(5),(6),(7),(8),(9),(10),(11),(12)) AS months(n);
    
    SET @Months = STUFF(@Months, 1, 2, '');  -- ลบ leading comma
    
    SET @SQL = N'
    SELECT salesperson, ' + @Months + ', 
           (SELECT SUM(amount) FROM sales WHERE salesperson = s.salesperson AND YEAR(sale_date) = @yr) AS YearTotal
    FROM (
        SELECT 
            salesperson,
            DATENAME(MONTH, sale_date) AS sale_month,
            amount
        FROM sales
        WHERE YEAR(sale_date) = @yr
        AND (@region IS NULL OR region = @region)
    ) AS s
    PIVOT (
        SUM(amount) FOR sale_month IN (' + @Months + ')
    ) AS pivot_table
    ORDER BY YearTotal DESC';
    
    EXEC sp_executesql @SQL, N'@yr INT, @region NVARCHAR(50)', @yr = @Year, @region = @Region;
END;

EXEC sp_MonthlySalesPivot @Year = 2024, @Region = 'North';
```

### ข้อที่ 2: Error Handling Pattern ที่สมบูรณ์

**เฉลย:**
```sql
CREATE OR ALTER PROCEDURE sp_PlaceOrder
    @CustomerId INT,
    @ProductId INT,
    @Quantity INT,
    @OrderId INT OUTPUT
AS
BEGIN
    SET NOCOUNT ON;
    
    DECLARE @AvailableStock INT;
    DECLARE @UnitPrice DECIMAL(10,2);
    
    BEGIN TRY
        BEGIN TRANSACTION;
        
        -- ตรวจสอบ customer
        IF NOT EXISTS (SELECT 1 FROM customers WHERE id = @CustomerId AND status = 'active')
        BEGIN
            THROW 50001, 'Customer not found or inactive', 1;
        END;
        
        -- ตรวจสอบ product และ lock
        SELECT @AvailableStock = stock_quantity,
               @UnitPrice = price
        FROM products WITH (UPDLOCK, ROWLOCK)
        WHERE id = @ProductId AND is_available = 1;
        
        IF @UnitPrice IS NULL
        BEGIN
            THROW 50002, 'Product not found or unavailable', 1;
        END;
        
        IF @AvailableStock < @Quantity
        BEGIN
            DECLARE @Msg NVARCHAR(200) = FORMATMESSAGE(
                'Insufficient stock. Available: %d, Requested: %d',
                @AvailableStock, @Quantity
            );
            THROW 50003, @Msg, 1;
        END;
        
        -- Create order
        INSERT INTO orders (customer_id, status, created_at)
        VALUES (@CustomerId, 'pending', GETDATE());
        
        SET @OrderId = SCOPE_IDENTITY();
        
        -- Create order item
        INSERT INTO order_items (order_id, product_id, quantity, unit_price, total_price)
        VALUES (@OrderId, @ProductId, @Quantity, @UnitPrice, @UnitPrice * @Quantity);
        
        -- Update stock
        UPDATE products
        SET stock_quantity = stock_quantity - @Quantity
        WHERE id = @ProductId;
        
        -- Update order total
        UPDATE orders
        SET total_amount = @UnitPrice * @Quantity
        WHERE id = @OrderId;
        
        COMMIT TRANSACTION;
        
    END TRY
    BEGIN CATCH
        IF @@TRANCOUNT > 0 ROLLBACK TRANSACTION;
        
        -- Log error
        INSERT INTO error_log (
            procedure_name, error_number, error_message, 
            error_severity, error_state, created_at,
            context_data
        )
        VALUES (
            'sp_PlaceOrder',
            ERROR_NUMBER(), ERROR_MESSAGE(),
            ERROR_SEVERITY(), ERROR_STATE(), GETDATE(),
            (SELECT @CustomerId AS customer_id, @ProductId AS product_id, 
                    @Quantity AS quantity FOR JSON PATH, WITHOUT_ARRAY_WRAPPER)
        );
        
        THROW;
    END CATCH;
END;
```

### ข้อที่ 3: Audit System ด้วย Triggers

**เฉลย:**
```sql
CREATE TABLE audit_log (
    id BIGINT IDENTITY(1,1) PRIMARY KEY,
    table_name NVARCHAR(128),
    operation NVARCHAR(10),
    primary_key NVARCHAR(255),
    old_values NVARCHAR(MAX),  -- JSON
    new_values NVARCHAR(MAX),  -- JSON
    changed_by NVARCHAR(128) DEFAULT SYSTEM_USER,
    changed_at DATETIME2 DEFAULT SYSDATETIME(),
    INDEX idx_audit_table (table_name, changed_at DESC)
);

-- Trigger สำหรับ customers table
CREATE OR ALTER TRIGGER trg_customers_audit
ON customers
AFTER INSERT, UPDATE, DELETE
AS
BEGIN
    SET NOCOUNT ON;
    
    IF EXISTS (SELECT 1 FROM INSERTED) AND EXISTS (SELECT 1 FROM DELETED)
    BEGIN
        -- UPDATE
        INSERT INTO audit_log (table_name, operation, primary_key, old_values, new_values)
        SELECT 'customers', 'UPDATE', 
               CAST(d.id AS NVARCHAR(255)),
               (SELECT * FROM DELETED d2 WHERE d2.id = d.id FOR JSON PATH, WITHOUT_ARRAY_WRAPPER),
               (SELECT * FROM INSERTED i WHERE i.id = d.id FOR JSON PATH, WITHOUT_ARRAY_WRAPPER)
        FROM DELETED d;
    END
    ELSE IF EXISTS (SELECT 1 FROM INSERTED)
    BEGIN
        -- INSERT
        INSERT INTO audit_log (table_name, operation, primary_key, new_values)
        SELECT 'customers', 'INSERT',
               CAST(id AS NVARCHAR(255)),
               (SELECT * FROM INSERTED i2 WHERE i2.id = i.id FOR JSON PATH, WITHOUT_ARRAY_WRAPPER)
        FROM INSERTED i;
    END
    ELSE
    BEGIN
        -- DELETE
        INSERT INTO audit_log (table_name, operation, primary_key, old_values)
        SELECT 'customers', 'DELETE',
               CAST(id AS NVARCHAR(255)),
               (SELECT * FROM DELETED d2 WHERE d2.id = d.id FOR JSON PATH, WITHOUT_ARRAY_WRAPPER)
        FROM DELETED d;
    END;
END;
```

### ข้อที่ 4: Cursor vs Set-based Solution

**เฉลย:**
```sql
-- บัญชีธนาคาร: คำนวณ running balance

-- Cursor approach (ช้า)
DECLARE @AccountId INT, @TransDate DATE, @Amount DECIMAL(10,2), @Balance DECIMAL(12,2);
DECLARE @RunningBalance DECIMAL(12,2) = 0;

DECLARE trans_cursor CURSOR FOR
SELECT account_id, trans_date, amount FROM transactions
WHERE account_id = 12345
ORDER BY trans_date, id;

OPEN trans_cursor;
FETCH NEXT FROM trans_cursor INTO @AccountId, @TransDate, @Amount;

WHILE @@FETCH_STATUS = 0
BEGIN
    SET @RunningBalance += @Amount;
    UPDATE transactions 
    SET running_balance = @RunningBalance
    WHERE account_id = @AccountId AND trans_date = @TransDate;
    
    FETCH NEXT FROM trans_cursor INTO @AccountId, @TransDate, @Amount;
END;

CLOSE trans_cursor; DEALLOCATE trans_cursor;

-- Set-based approach (เร็ว - แนะนำ)
WITH running_totals AS (
    SELECT 
        id,
        trans_date,
        amount,
        SUM(amount) OVER (
            PARTITION BY account_id
            ORDER BY trans_date, id
            ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
        ) AS running_balance
    FROM transactions
    WHERE account_id = 12345
)
UPDATE t
SET running_balance = rt.running_balance
FROM transactions t
JOIN running_totals rt ON t.id = rt.id;
```

### ข้อที่ 5: FOR JSON สร้าง API Response

**เฉลย:**
```sql
CREATE OR ALTER FUNCTION fn_GetCustomerDetailsJson
(
    @CustomerId INT
)
RETURNS NVARCHAR(MAX)
AS
BEGIN
    RETURN (
        SELECT 
            c.id,
            c.name,
            c.email,
            c.phone,
            addr.street AS [address.street],
            addr.city AS [address.city],
            addr.postal_code AS [address.postalCode],
            (
                SELECT TOP 5
                    o.id,
                    o.order_date AS orderDate,
                    o.total_amount AS totalAmount,
                    o.status,
                    (
                        SELECT 
                            oi.product_name AS productName,
                            oi.quantity,
                            oi.unit_price AS unitPrice
                        FROM order_items oi
                        WHERE oi.order_id = o.id
                        FOR JSON PATH
                    ) AS items
                FROM orders o
                WHERE o.customer_id = c.id
                ORDER BY o.order_date DESC
                FOR JSON PATH
            ) AS recentOrders
        FROM customers c
        LEFT JOIN addresses addr ON c.id = addr.customer_id AND addr.is_primary = 1
        WHERE c.id = @CustomerId
        FOR JSON PATH, WITHOUT_ARRAY_WRAPPER
    );
END;

-- ใช้งาน
SELECT dbo.fn_GetCustomerDetailsJson(101);
```

### ข้อที่ 6: DMV Monitoring Dashboard

**เฉลย:**
```sql
CREATE OR ALTER PROCEDURE sp_DatabaseHealthCheck
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Active sessions
    SELECT 'Active Sessions' AS [Check], COUNT(*) AS [Value]
    FROM sys.dm_exec_sessions WHERE is_user_process = 1;
    
    -- Blocking chains
    SELECT 
        'Blocking Chains' AS [Metric],
        blocking_session_id AS [Blocker],
        session_id AS [Blocked],
        wait_type AS [WaitType],
        wait_time / 1000.0 AS [WaitSeconds]
    FROM sys.dm_exec_requests
    WHERE blocking_session_id > 0;
    
    -- Top 10 slow queries
    SELECT TOP 10
        qs.total_elapsed_time / qs.execution_count / 1000.0 AS [AvgDurationMs],
        qs.execution_count AS [Executions],
        qs.total_logical_reads / qs.execution_count AS [AvgReads],
        SUBSTRING(qt.text, 1, 200) AS [QueryText]
    FROM sys.dm_exec_query_stats qs
    CROSS APPLY sys.dm_exec_sql_text(qs.sql_handle) qt
    WHERE qs.execution_count > 10
    ORDER BY [AvgDurationMs] DESC;
    
    -- Index fragmentation
    SELECT 
        OBJECT_NAME(ips.object_id) AS [Table],
        i.name AS [Index],
        ROUND(ips.avg_fragmentation_in_percent, 2) AS [Fragmentation%],
        ips.page_count AS [Pages],
        CASE 
            WHEN ips.avg_fragmentation_in_percent > 30 THEN 'REBUILD'
            WHEN ips.avg_fragmentation_in_percent > 10 THEN 'REORGANIZE'
            ELSE 'OK'
        END AS [Action]
    FROM sys.dm_db_index_physical_stats(DB_ID(), NULL, NULL, NULL, 'SAMPLED') ips
    JOIN sys.indexes i ON ips.object_id = i.object_id AND ips.index_id = i.index_id
    WHERE ips.avg_fragmentation_in_percent > 5
    AND ips.page_count > 100
    ORDER BY ips.avg_fragmentation_in_percent DESC;
END;

EXEC sp_DatabaseHealthCheck;
```

### ข้อที่ 7: Table Partitioning

**เฉลย:**
```sql
-- สร้าง partition function
CREATE PARTITION FUNCTION pf_OrderDate (DATE)
AS RANGE RIGHT FOR VALUES (
    '2021-01-01', '2022-01-01', '2023-01-01', '2024-01-01'
);

-- สร้าง partition scheme
CREATE PARTITION SCHEME ps_OrderDate
AS PARTITION pf_OrderDate
TO (
    [PRIMARY],    -- before 2021
    [FG_2021],    -- 2021
    [FG_2022],    -- 2022
    [FG_2023],    -- 2023
    [FG_2024]     -- 2024 and after
);

-- สร้างตาราง partitioned
CREATE TABLE orders_partitioned (
    id BIGINT IDENTITY(1,1),
    order_date DATE NOT NULL,
    customer_id INT,
    amount DECIMAL(10,2),
    CONSTRAINT PK_orders_partitioned PRIMARY KEY CLUSTERED (id, order_date)
) ON ps_OrderDate(order_date);

-- ดู partition info
SELECT 
    p.partition_number,
    p.rows,
    prv.value AS boundary_value,
    fg.name AS filegroup
FROM sys.partitions p
JOIN sys.indexes i ON p.object_id = i.object_id AND p.index_id = i.index_id
JOIN sys.partition_schemes ps ON i.data_space_id = ps.data_space_id
JOIN sys.partition_range_values prv ON ps.function_id = prv.function_id
    AND p.partition_number = prv.boundary_id + 1
JOIN sys.destination_data_spaces dds ON ps.data_space_id = dds.partition_scheme_id
    AND p.partition_number = dds.destination_id
JOIN sys.filegroups fg ON dds.data_space_id = fg.data_space_id
WHERE OBJECT_NAME(p.object_id) = 'orders_partitioned';
```

### ข้อที่ 8: CLR Function สำหรับ Complex String Operations

**เฉลย:**
```csharp
// C# code สำหรับ CLR function (ใน Visual Studio)
using System;
using System.Text.RegularExpressions;
using Microsoft.SqlServer.Server;

public class StringFunctions
{
    [SqlFunction(IsDeterministic = true, IsPrecise = true)]
    public static bool RegexMatch(string input, string pattern)
    {
        if (input == null || pattern == null) return false;
        return Regex.IsMatch(input, pattern);
    }
    
    [SqlFunction(IsDeterministic = true, IsPrecise = true)]
    public static string RegexReplace(string input, string pattern, string replacement)
    {
        if (input == null) return null;
        return Regex.Replace(input, pattern, replacement ?? "");
    }
}
```

```sql
-- SQL สำหรับ register CLR
CREATE ASSEMBLY [StringFunctions]
FROM 'C:\CLR\StringFunctions.dll'
WITH PERMISSION_SET = SAFE;

CREATE FUNCTION dbo.RegexMatch(@input NVARCHAR(MAX), @pattern NVARCHAR(MAX))
RETURNS BIT
AS EXTERNAL NAME [StringFunctions].[StringFunctions].[RegexMatch];

CREATE FUNCTION dbo.RegexReplace(@input NVARCHAR(MAX), @pattern NVARCHAR(MAX), @replacement NVARCHAR(MAX))
RETURNS NVARCHAR(MAX)
AS EXTERNAL NAME [StringFunctions].[StringFunctions].[RegexReplace];

-- ใช้งาน
SELECT name FROM products
WHERE dbo.RegexMatch(sku, '^[A-Z]{2}-\d{4}$') = 1;

SELECT dbo.RegexReplace(phone, '[^0-9]', '') AS clean_phone
FROM customers;
```

### ข้อที่ 9: Temporal Tables (System-Versioned)

**เฉลย:**
```sql
-- SQL Server Temporal Table
CREATE TABLE employee_history (
    id INT PRIMARY KEY,
    name NVARCHAR(100),
    department NVARCHAR(50),
    salary DECIMAL(10,2),
    manager_id INT,
    valid_from DATETIME2 GENERATED ALWAYS AS ROW START,
    valid_to DATETIME2 GENERATED ALWAYS AS ROW END,
    PERIOD FOR SYSTEM_TIME (valid_from, valid_to)
)
WITH (SYSTEM_VERSIONING = ON (HISTORY_TABLE = dbo.employee_history_archive));

-- Insert และ Update data
INSERT INTO employee_history (id, name, department, salary, manager_id)
VALUES (1, 'Alice', 'Engineering', 80000, NULL);

-- Salary increase
UPDATE employee_history SET salary = 90000 WHERE id = 1;
UPDATE employee_history SET department = 'Senior Engineering' WHERE id = 1;

-- ดูประวัติทั้งหมด
SELECT *, valid_from, valid_to
FROM employee_history
FOR SYSTEM_TIME ALL
WHERE id = 1
ORDER BY valid_from;

-- ดูข้อมูล ณ วันที่ระบุ
SELECT *
FROM employee_history
FOR SYSTEM_TIME AS OF '2024-01-15 10:00:00'
WHERE id = 1;

-- ดูการเปลี่ยนแปลงในช่วงเวลา
SELECT *
FROM employee_history
FOR SYSTEM_TIME BETWEEN '2024-01-01' AND '2024-12-31'
WHERE id = 1
ORDER BY valid_from;
```

### ข้อที่ 10: Full-Text Search ด้วย CONTAINS และ FREETEXT

**เฉลย:**
```sql
-- สร้าง Full-Text Index
CREATE FULLTEXT CATALOG ft_catalog AS DEFAULT;

CREATE FULLTEXT INDEX ON articles (
    title LANGUAGE 'English',
    content LANGUAGE 'English',
    author
)
KEY INDEX PK_articles
ON ft_catalog
WITH CHANGE_TRACKING AUTO;

-- CONTAINS: precise search
SELECT title, author
FROM articles
WHERE CONTAINS(content, '"database performance"');  -- exact phrase

-- Boolean operators
SELECT title FROM articles
WHERE CONTAINS((title, content), 'SQL AND (server OR database)');

-- Proximity search (NEAR)
SELECT title FROM articles
WHERE CONTAINS(content, 'NEAR((database, performance), 3)');  -- ห่างกันไม่เกิน 3 words

-- Thesaurus และ inflections
SELECT title FROM articles
WHERE CONTAINS(content, 'FORMSOF(INFLECTIONAL, optimize)');  -- optimizes, optimized, optimization, etc.

-- FREETEXT: คล้าย natural language search
SELECT title, author
FROM articles
WHERE FREETEXT(content, 'improve database query performance');

-- CONTAINSTABLE: return relevance rank
SELECT a.title, a.author, ct.[RANK]
FROM articles a
INNER JOIN CONTAINSTABLE(articles, content, 'database performance') ct
    ON a.id = ct.[KEY]
ORDER BY ct.[RANK] DESC;

-- FREETEXTTABLE
SELECT a.title, ft.[RANK]
FROM articles a
INNER JOIN FREETEXTTABLE(articles, *, 'optimize SQL queries') ft
    ON a.id = ft.[KEY]
ORDER BY ft.[RANK] DESC;
```

---

## สรุป

ในบทนี้เราได้เรียนรู้คุณสมบัติขั้นสูงของ SQL Server (T-SQL):

1. **T-SQL Hints** - NOLOCK, ROWLOCK, UPDLOCK สำหรับ concurrency control
2. **System Tables/DMVs** - ตรวจสอบ performance และ metadata
3. **sp_executesql** - Dynamic SQL ที่ปลอดภัย
4. **TRY/CATCH** - Error handling ที่ครบวงจร
5. **Cursors** - row-by-row processing (เมื่อจำเป็น)
6. **Table Variables vs Temp Tables** - เลือกให้ถูกต้อง
7. **Common Patterns** - MERGE, pagination, string aggregation
8. **SQL Server Functions** - ISNULL, IIF, CHOOSE, FORMAT
9. **FOR JSON/XML** - export data ในรูปแบบต่าง ๆ
10. **SQL Server Agent** - automated jobs
11. **Linked Servers** - distributed queries
12. **CLR Integration** - .NET code ใน SQL Server

SQL Server และ T-SQL มีความสามารถที่ครบครันสำหรับ enterprise applications ทั้ง performance, security, และ manageability
