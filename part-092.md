# ส่วนที่ 92: Recursive CTEs

## บทนำ

Recursive CTE เป็นหนึ่งในฟีเจอร์ที่ทรงพลังที่สุดของ SQL สมัยใหม่ ช่วยให้เราสามารถ traverse ข้อมูลแบบลำดับชั้น (hierarchical data) ได้อย่างสง่างาม ไม่ว่าจะเป็น org charts, category trees, bill of materials, network graphs, หรือแม้แต่การสร้าง sequences

---

## 92.1 Recursive CTE Syntax

### โครงสร้างพื้นฐาน

```sql
WITH RECURSIVE cte_name AS (
    -- Anchor member (base case): ไม่อ้างอิงตัวเอง
    SELECT ...
    
    UNION ALL
    
    -- Recursive member: อ้างอิง cte_name
    SELECT ...
    FROM cte_name
    WHERE -- เงื่อนไขหยุด recursion
)
SELECT * FROM cte_name;
```

### วิธีที่ Recursion ทำงาน

```
1. Execute Anchor member → ได้ชุดข้อมูลเริ่มต้น
2. Execute Recursive member โดยใช้ผลลัพธ์จากรอบก่อน
3. ทำซ้ำจนกว่า Recursive member ไม่ return rows ใหม่
4. รวมผลลัพธ์ทั้งหมดด้วย UNION ALL
```

---

## 92.2 ตัวอย่างพื้นฐาน: Counting Sequences

### ตัวอย่างที่ 1: สร้างตัวเลข 1-10

```sql
-- นับ 1 ถึง 10 ด้วย Recursive CTE
WITH RECURSIVE counter AS (
    -- Anchor: เริ่มที่ 1
    SELECT 1 AS n
    
    UNION ALL
    
    -- Recursive: เพิ่มทีละ 1
    SELECT n + 1
    FROM counter
    WHERE n < 10  -- เงื่อนไขหยุด
)
SELECT n FROM counter;
```

### ตัวอย่างที่ 2: สร้าง Fibonacci sequence

```sql
-- Fibonacci: 1, 1, 2, 3, 5, 8, 13, 21, ...
WITH RECURSIVE fibonacci AS (
    -- Anchor: สองตัวแรก
    SELECT 
        1 AS position,
        0 AS fib_prev,
        1 AS fib_current
    
    UNION ALL
    
    -- Recursive: คำนวณตัวถัดไป
    SELECT 
        position + 1,
        fib_current,
        fib_prev + fib_current
    FROM fibonacci
    WHERE position < 15  -- แสดง 15 ตัว
)
SELECT position, fib_current AS fibonacci_number
FROM fibonacci;
```

### ตัวอย่างที่ 3: สร้าง Date Sequence

```sql
-- สร้างวันที่ทุกวันใน 3 เดือน
WITH RECURSIVE date_series AS (
    SELECT '2024-01-01'::date AS dt
    
    UNION ALL
    
    SELECT dt + INTERVAL '1 day'
    FROM date_series
    WHERE dt < '2024-03-31'
)
SELECT 
    dt,
    TO_CHAR(dt, 'Day') AS day_name,
    EXTRACT(DOW FROM dt) IN (0, 6) AS is_weekend
FROM date_series;
```

---

## 92.3 Hierarchical Data: Org Chart

### ตัวอย่างที่ 4: โครงสร้างองค์กร

```sql
-- สร้างตาราง org chart
CREATE TABLE org_chart (
    emp_id      INT PRIMARY KEY,
    name        VARCHAR(100),
    title       VARCHAR(100),
    manager_id  INT REFERENCES org_chart(emp_id)
);

INSERT INTO org_chart VALUES
(1,  'สมเกียรติ วงศ์ใหญ่',  'CEO',              NULL),
(2,  'วิภา พัฒนกิจ',        'CTO',              1),
(3,  'สุชาติ มั่งมี',        'CFO',              1),
(4,  'รัตนา ฝีมือดี',        'VP Engineering',   2),
(5,  'ชัยชนะ นำโชค',        'VP Finance',       3),
(6,  'ธนา รุ่งเรือง',        'Senior Dev',       4),
(7,  'พิมล แสนดี',           'Senior Dev',       4),
(8,  'ลำยอง งามวิไล',        'Junior Dev',       6),
(9,  'ศิริ ใจดีมาก',         'Accountant',       5),
(10, 'กมล เจริญทรัพย์',      'Junior Dev',       7);

-- ดึง org chart ทั้งหมดพร้อม level
WITH RECURSIVE org_tree AS (
    -- Anchor: CEO (ไม่มี manager)
    SELECT 
        emp_id,
        name,
        title,
        manager_id,
        0 AS level,
        name::TEXT AS path
    FROM org_chart
    WHERE manager_id IS NULL
    
    UNION ALL
    
    -- Recursive: พนักงานที่มี manager ใน CTE
    SELECT 
        e.emp_id,
        e.name,
        e.title,
        e.manager_id,
        t.level + 1,
        t.path || ' → ' || e.name
    FROM org_chart e
    JOIN org_tree t ON e.manager_id = t.emp_id
)
SELECT 
    emp_id,
    REPEAT('  ', level) || name AS indented_name,  -- เยื้องตาม level
    title,
    level,
    path
FROM org_tree
ORDER BY path;
```

### ตัวอย่างที่ 5: หา subordinates ทั้งหมดของผู้จัดการ

```sql
-- หาพนักงานทั้งหมดที่อยู่ใต้ CTO (emp_id = 2)
WITH RECURSIVE subordinates AS (
    -- Anchor: CTO
    SELECT emp_id, name, title, 0 AS level
    FROM org_chart
    WHERE emp_id = 2
    
    UNION ALL
    
    -- Recursive: ลูกน้องทุกระดับ
    SELECT e.emp_id, e.name, e.title, s.level + 1
    FROM org_chart e
    JOIN subordinates s ON e.manager_id = s.emp_id
)
SELECT 
    level,
    name,
    title
FROM subordinates
ORDER BY level, name;
```

### ตัวอย่างที่ 6: หา management chain ของพนักงาน

```sql
-- หา chain of command ของพนักงาน emp_id = 8 ขึ้นไปถึง CEO
WITH RECURSIVE management_chain AS (
    -- Anchor: พนักงานที่ต้องการ
    SELECT emp_id, name, title, manager_id, 0 AS level
    FROM org_chart
    WHERE emp_id = 8
    
    UNION ALL
    
    -- Recursive: หา manager ขึ้นไปเรื่อยๆ
    SELECT e.emp_id, e.name, e.title, e.manager_id, mc.level + 1
    FROM org_chart e
    JOIN management_chain mc ON e.emp_id = mc.manager_id
)
SELECT level, name, title
FROM management_chain
ORDER BY level;
```

### ตัวอย่างที่ 7: นับจำนวนลูกน้องทุกระดับ

```sql
WITH RECURSIVE all_reports AS (
    SELECT 
        m.emp_id AS manager_id,
        e.emp_id AS report_id
    FROM org_chart m
    JOIN org_chart e ON e.manager_id = m.emp_id
    
    UNION ALL
    
    SELECT ar.manager_id, e.emp_id
    FROM all_reports ar
    JOIN org_chart e ON e.manager_id = ar.report_id
)
SELECT 
    oc.name,
    oc.title,
    COUNT(ar.report_id) AS total_reports
FROM org_chart oc
LEFT JOIN all_reports ar ON oc.emp_id = ar.manager_id
GROUP BY oc.emp_id, oc.name, oc.title
ORDER BY total_reports DESC;
```

---

## 92.4 Category Trees

### ตัวอย่างที่ 8: โครงสร้างหมวดหมู่สินค้า

```sql
CREATE TABLE categories (
    cat_id      INT PRIMARY KEY,
    cat_name    VARCHAR(100),
    parent_id   INT REFERENCES categories(cat_id)
);

INSERT INTO categories VALUES
(1,  'Electronics',    NULL),
(2,  'Computers',      1),
(3,  'Mobile',         1),
(4,  'Laptops',        2),
(5,  'Desktops',       2),
(6,  'Smartphones',    3),
(7,  'Tablets',        3),
(8,  'Gaming Laptops', 4),
(9,  'Ultrabooks',     4),
(10, 'Android',        6),
(11, 'iPhone',         6);

-- ดึง category tree ทั้งหมด
WITH RECURSIVE cat_tree AS (
    -- Root categories
    SELECT 
        cat_id,
        cat_name,
        parent_id,
        0 AS depth,
        cat_name::VARCHAR(500) AS full_path,
        ARRAY[cat_id] AS path_ids
    FROM categories
    WHERE parent_id IS NULL
    
    UNION ALL
    
    -- Children
    SELECT 
        c.cat_id,
        c.cat_name,
        c.parent_id,
        ct.depth + 1,
        (ct.full_path || ' > ' || c.cat_name)::VARCHAR(500),
        ct.path_ids || c.cat_id
    FROM categories c
    JOIN cat_tree ct ON c.parent_id = ct.cat_id
)
SELECT 
    REPEAT('  ', depth) || cat_name AS tree_view,
    full_path,
    depth
FROM cat_tree
ORDER BY full_path;
```

### ตัวอย่างที่ 9: หา leaf categories (categories ที่ไม่มีลูก)

```sql
WITH RECURSIVE cat_hierarchy AS (
    SELECT cat_id, cat_name, parent_id, 0 AS depth
    FROM categories
    WHERE parent_id IS NULL
    
    UNION ALL
    
    SELECT c.cat_id, c.cat_name, c.parent_id, h.depth + 1
    FROM categories c
    JOIN cat_hierarchy h ON c.parent_id = h.cat_id
),
has_children AS (
    SELECT DISTINCT parent_id FROM categories WHERE parent_id IS NOT NULL
)
SELECT ch.cat_id, ch.cat_name, ch.depth
FROM cat_hierarchy ch
WHERE ch.cat_id NOT IN (SELECT parent_id FROM has_children)
ORDER BY ch.depth, ch.cat_name;
```

### ตัวอย่างที่ 10: นับสินค้าในแต่ละ category รวมถึง subcategories

```sql
CREATE TABLE products (
    product_id  INT PRIMARY KEY,
    name        VARCHAR(100),
    cat_id      INT REFERENCES categories(cat_id)
);

INSERT INTO products VALUES
(1, 'ASUS Gaming', 8), (2, 'Dell XPS', 9),
(3, 'MacBook Air', 9), (4, 'Samsung S24', 10),
(5, 'iPhone 15', 11), (6, 'iPad Pro', 7),
(7, 'Lenovo Gaming', 8);

-- นับสินค้าในแต่ละ category รวม subcategories ด้วย
WITH RECURSIVE category_path AS (
    -- แต่ละ category เริ่มต้นด้วยตัวเอง
    SELECT cat_id AS root_id, cat_id AS member_id
    FROM categories
    
    UNION ALL
    
    -- เพิ่ม children
    SELECT cp.root_id, c.cat_id
    FROM category_path cp
    JOIN categories c ON c.parent_id = cp.member_id
)
SELECT 
    c.cat_name,
    COUNT(DISTINCT p.product_id) AS product_count
FROM categories c
JOIN category_path cp ON c.cat_id = cp.root_id
LEFT JOIN products p ON cp.member_id = p.cat_id
GROUP BY c.cat_id, c.cat_name
ORDER BY product_count DESC;
```

---

## 92.5 Bill of Materials (BOM)

### ตัวอย่างที่ 11: โครงสร้าง BOM

```sql
CREATE TABLE bom (
    component_id    INT,
    parent_id       INT,    -- NULL = finished product
    component_name  VARCHAR(100),
    quantity        DECIMAL(10,3),
    unit_cost       DECIMAL(10,2)
);

INSERT INTO bom VALUES
(1,  NULL, 'Bicycle',          1, 0),
(2,  1,    'Frame',            1, 5000),
(3,  1,    'Wheel Assembly',   2, 0),
(4,  1,    'Drivetrain',       1, 0),
(5,  3,    'Rim',              1, 800),
(6,  3,    'Tire',             1, 350),
(7,  3,    'Spoke',            36, 15),
(8,  3,    'Hub',              1, 500),
(9,  4,    'Chain',            1, 200),
(10, 4,    'Sprocket',         1, 300),
(11, 4,    'Pedal',            2, 400),
(12, 4,    'Crank',            1, 600);

-- ดึง BOM ทั้งหมดพร้อมต้นทุนสะสม
WITH RECURSIVE bom_tree AS (
    -- ตัวสินค้าหลัก
    SELECT 
        component_id,
        component_name,
        parent_id,
        quantity,
        unit_cost,
        0 AS level,
        component_name::VARCHAR(500) AS path,
        unit_cost * quantity AS total_cost
    FROM bom
    WHERE parent_id IS NULL
    
    UNION ALL
    
    -- ส่วนประกอบ
    SELECT 
        b.component_id,
        b.component_name,
        b.parent_id,
        b.quantity * bt.quantity AS quantity,  -- ปริมาณสะสม
        b.unit_cost,
        bt.level + 1,
        bt.path || ' > ' || b.component_name,
        b.unit_cost * b.quantity * bt.quantity AS total_cost
    FROM bom b
    JOIN bom_tree bt ON b.parent_id = bt.component_id
)
SELECT 
    REPEAT('  ', level) || component_name AS structure,
    quantity,
    unit_cost,
    total_cost,
    level
FROM bom_tree
ORDER BY path;

-- รวมต้นทุนทั้งหมด
WITH RECURSIVE bom_tree AS (
    SELECT component_id, component_name, parent_id, quantity, unit_cost, 0 AS level
    FROM bom WHERE parent_id IS NULL
    
    UNION ALL
    
    SELECT b.component_id, b.component_name, b.parent_id,
           b.quantity * bt.quantity, b.unit_cost, bt.level + 1
    FROM bom b JOIN bom_tree bt ON b.parent_id = bt.component_id
)
SELECT 
    SUM(CASE WHEN level > 0 THEN unit_cost * quantity ELSE 0 END) AS total_material_cost
FROM bom_tree;
```

---

## 92.6 Cycle Detection

### ตัวอย่างที่ 12: ตรวจจับ Cycle ใน Graph

```sql
-- สร้าง graph ที่อาจมี cycle
CREATE TABLE graph_edges (
    from_node INT,
    to_node   INT
);

INSERT INTO graph_edges VALUES
(1, 2), (2, 3), (3, 4),
(4, 2),  -- Cycle: 2 → 3 → 4 → 2
(5, 6), (6, 7);

-- Traverse พร้อม cycle detection ด้วย ARRAY
WITH RECURSIVE path_traverse AS (
    SELECT 
        from_node AS start,
        to_node AS current,
        ARRAY[from_node, to_node] AS visited,
        FALSE AS has_cycle
    FROM graph_edges
    WHERE from_node = 1
    
    UNION ALL
    
    SELECT 
        pt.start,
        ge.to_node,
        pt.visited || ge.to_node,
        ge.to_node = ANY(pt.visited)  -- ตรวจ cycle
    FROM graph_edges ge
    JOIN path_traverse pt ON ge.from_node = pt.current
    WHERE NOT pt.has_cycle    -- หยุดเมื่อเจอ cycle
      AND NOT ge.to_node = ANY(pt.visited[1:ARRAY_LENGTH(pt.visited,1)-1])
)
SELECT start, current, visited, has_cycle
FROM path_traverse;
```

### ตัวอย่างที่ 13: Safe Traversal ด้วย CYCLE clause (PostgreSQL 14+)

```sql
-- PostgreSQL 14+ รองรับ CYCLE clause โดยตรง
WITH RECURSIVE org_traverse AS (
    SELECT emp_id, name, manager_id
    FROM org_chart
    WHERE manager_id IS NULL
    
    UNION ALL
    
    SELECT e.emp_id, e.name, e.manager_id
    FROM org_chart e
    JOIN org_traverse ot ON e.manager_id = ot.emp_id
)
CYCLE emp_id
SET is_cycle
USING path_column
SELECT * FROM org_traverse;
```

---

## 92.7 Path Building

### ตัวอย่างที่ 14: สร้าง URL Path จาก Category Tree

```sql
-- สร้าง breadcrumb paths สำหรับ category
WITH RECURSIVE category_path AS (
    SELECT 
        cat_id,
        cat_name,
        parent_id,
        LOWER(REPLACE(cat_name, ' ', '-')) AS slug,
        LOWER(REPLACE(cat_name, ' ', '-'))::VARCHAR(500) AS url_path
    FROM categories
    WHERE parent_id IS NULL
    
    UNION ALL
    
    SELECT 
        c.cat_id,
        c.cat_name,
        c.parent_id,
        LOWER(REPLACE(c.cat_name, ' ', '-')),
        (cp.url_path || '/' || LOWER(REPLACE(c.cat_name, ' ', '-')))::VARCHAR(500)
    FROM categories c
    JOIN category_path cp ON c.parent_id = cp.cat_id
)
SELECT 
    cat_id,
    cat_name,
    '/' || url_path AS full_url
FROM category_path
ORDER BY full_url;
```

### ตัวอย่างที่ 15: Build Path String สำหรับ File System

```sql
CREATE TABLE file_system (
    node_id     INT PRIMARY KEY,
    name        VARCHAR(100),
    parent_id   INT REFERENCES file_system(node_id),
    is_file     BOOLEAN
);

INSERT INTO file_system VALUES
(1, '/',            NULL,  false),
(2, 'home',         1,     false),
(3, 'user',         2,     false),
(4, 'documents',    3,     false),
(5, 'report.pdf',   4,     true),
(6, 'thesis.docx',  4,     true),
(7, 'pictures',     3,     false),
(8, 'photo.jpg',    7,     true);

WITH RECURSIVE full_path AS (
    -- Root
    SELECT node_id, name, parent_id, is_file, 
           name::VARCHAR(500) AS path, 0 AS depth
    FROM file_system
    WHERE parent_id IS NULL
    
    UNION ALL
    
    -- Children
    SELECT 
        f.node_id, f.name, f.parent_id, f.is_file,
        CASE 
            WHEN fp.path = '/' THEN '/' || f.name
            ELSE fp.path || '/' || f.name
        END::VARCHAR(500),
        fp.depth + 1
    FROM file_system f
    JOIN full_path fp ON f.parent_id = fp.node_id
)
SELECT 
    path,
    CASE WHEN is_file THEN '📄' ELSE '📁' END AS type
FROM full_path
WHERE node_id != 1  -- ไม่แสดง root
ORDER BY path;
```

---

## 92.8 Depth-limited Recursion

### ตัวอย่างที่ 16: จำกัดความลึกของ Recursion

```sql
-- ดึง org chart แค่ 2 ระดับแรก
WITH RECURSIVE limited_org AS (
    SELECT emp_id, name, title, manager_id, 0 AS level
    FROM org_chart
    WHERE manager_id IS NULL
    
    UNION ALL
    
    SELECT e.emp_id, e.name, e.title, e.manager_id, lo.level + 1
    FROM org_chart e
    JOIN limited_org lo ON e.manager_id = lo.emp_id
    WHERE lo.level < 2  -- จำกัดที่ 2 ระดับ
)
SELECT level, name, title
FROM limited_org
ORDER BY level, name;
```

### ตัวอย่างที่ 17: หา Shortest Path ระหว่างสอง Node

```sql
-- หา shortest path ระหว่าง node 1 และ 7 ใน graph
CREATE TABLE connections (
    node_a INT,
    node_b INT,
    distance INT
);

INSERT INTO connections VALUES
(1, 2, 4), (1, 3, 2), (2, 4, 5),
(3, 4, 1), (3, 5, 8), (4, 6, 3),
(5, 6, 2), (4, 7, 1), (6, 7, 4);

WITH RECURSIVE shortest_path AS (
    -- เริ่มจาก node 1
    SELECT 
        node_b AS current_node,
        distance AS total_distance,
        ARRAY[1, node_b] AS path
    FROM connections
    WHERE node_a = 1
    
    UNION ALL
    
    -- ขยาย path
    SELECT 
        c.node_b,
        sp.total_distance + c.distance,
        sp.path || c.node_b
    FROM connections c
    JOIN shortest_path sp ON c.node_a = sp.current_node
    WHERE c.node_b != ALL(sp.path)  -- ไม่วนซ้ำ
      AND sp.total_distance + c.distance < 999  -- จำกัด
)
SELECT path, total_distance
FROM shortest_path
WHERE current_node = 7  -- ปลายทาง
ORDER BY total_distance
LIMIT 1;  -- เส้นทางสั้นที่สุด
```

---

## 92.9 ตัวอย่างขั้นสูง

### ตัวอย่างที่ 18: Family Tree / Genealogy

```sql
CREATE TABLE family_tree (
    person_id   INT PRIMARY KEY,
    name        VARCHAR(100),
    birth_year  INT,
    parent1_id  INT,    -- พ่อ
    parent2_id  INT     -- แม่
);

INSERT INTO family_tree VALUES
(1, 'ปู่สมศักดิ์',  1945, NULL, NULL),
(2, 'ย่าประภา',    1948, NULL, NULL),
(3, 'พ่อวิชัย',    1970, 1, 2),
(4, 'แม่สมหญิง',   1972, NULL, NULL),
(5, 'ลูกมานพ',     1995, 3, 4),
(6, 'ลูกมาลี',     1998, 3, 4);

-- หา ancestors ทั้งหมดของ มานพ (5)
WITH RECURSIVE ancestors AS (
    SELECT person_id, name, birth_year, parent1_id, parent2_id, 0 AS generation
    FROM family_tree
    WHERE person_id = 5
    
    UNION ALL
    
    -- พ่อ
    SELECT f.person_id, f.name, f.birth_year, f.parent1_id, f.parent2_id, a.generation + 1
    FROM family_tree f
    JOIN ancestors a ON f.person_id = a.parent1_id
    
    UNION ALL
    
    -- แม่
    SELECT f.person_id, f.name, f.birth_year, f.parent1_id, f.parent2_id, a.generation + 1
    FROM family_tree f
    JOIN ancestors a ON f.person_id = a.parent2_id
)
SELECT 
    generation,
    CASE generation
        WHEN 0 THEN 'ตัวเอง'
        WHEN 1 THEN 'พ่อ/แม่'
        WHEN 2 THEN 'ปู่/ย่า/ตา/ยาย'
        ELSE 'บรรพบุรุษ gen ' || generation
    END AS relation,
    name,
    birth_year
FROM ancestors
ORDER BY generation, name;
```

### ตัวอย่างที่ 19: Network Topology Traversal

```sql
-- traverse network dependencies
CREATE TABLE service_deps (
    service     VARCHAR(50),
    depends_on  VARCHAR(50)
);

INSERT INTO service_deps VALUES
('webapp', 'api-gateway'),
('webapp', 'cdn'),
('api-gateway', 'auth-service'),
('api-gateway', 'database'),
('auth-service', 'database'),
('auth-service', 'redis'),
('database', 'storage'),
('redis', 'memory-server');

-- หา dependencies ทั้งหมดของ webapp
WITH RECURSIVE all_deps AS (
    -- Direct dependencies
    SELECT service, depends_on, 1 AS depth
    FROM service_deps
    WHERE service = 'webapp'
    
    UNION ALL
    
    -- Transitive dependencies
    SELECT sd.service, sd.depends_on, ad.depth + 1
    FROM service_deps sd
    JOIN all_deps ad ON sd.service = ad.depends_on
)
SELECT DISTINCT depends_on AS dependency, MIN(depth) AS min_depth
FROM all_deps
GROUP BY depends_on
ORDER BY min_depth, depends_on;
```

### ตัวอย่างที่ 20: Closure Table Pattern

```sql
-- Closure table: เก็บ relationship ระหว่างทุกคู่ ancestor-descendant
CREATE TABLE category_closure (
    ancestor_id   INT,
    descendant_id INT,
    depth         INT
);

-- Generate closure table จาก categories
WITH RECURSIVE closure AS (
    -- Self-relationships (depth = 0)
    SELECT cat_id AS ancestor_id, cat_id AS descendant_id, 0 AS depth
    FROM categories
    
    UNION ALL
    
    -- ขยาย ancestors
    SELECT c.ancestor_id, cat.cat_id, c.depth + 1
    FROM closure c
    JOIN categories cat ON cat.parent_id = c.descendant_id
    WHERE c.depth < 10  -- ป้องกัน infinite loop
)
INSERT INTO category_closure
SELECT DISTINCT ancestor_id, descendant_id, depth FROM closure;

-- ใช้ closure table หาสินค้าทุกชิ้นใน Electronics (cat_id = 1)
SELECT p.name, c.cat_name
FROM category_closure cc
JOIN categories c ON cc.descendant_id = c.cat_id
JOIN products p ON p.cat_id = cc.descendant_id
WHERE cc.ancestor_id = 1
ORDER BY c.cat_name, p.name;
```

### ตัวอย่างที่ 21: Recursive String Splitting

```sql
-- แยก comma-separated values ด้วย Recursive CTE
WITH RECURSIVE split_csv AS (
    SELECT 
        'apple,banana,cherry,date,elderberry' AS remaining,
        NULL::TEXT AS current_value,
        0 AS position
    
    UNION ALL
    
    SELECT 
        CASE 
            WHEN POSITION(',' IN remaining) > 0 
            THEN SUBSTRING(remaining FROM POSITION(',' IN remaining) + 1)
            ELSE NULL
        END AS remaining,
        CASE 
            WHEN POSITION(',' IN remaining) > 0
            THEN SUBSTRING(remaining FROM 1 FOR POSITION(',' IN remaining) - 1)
            ELSE remaining
        END AS current_value,
        position + 1
    FROM split_csv
    WHERE remaining IS NOT NULL
)
SELECT position, current_value
FROM split_csv
WHERE current_value IS NOT NULL
ORDER BY position;
```

### ตัวอย่างที่ 22: Power Set Generation

```sql
-- สร้าง power set ของ set {1, 2, 3}
WITH RECURSIVE power_set AS (
    -- Empty set
    SELECT ARRAY[]::INT[] AS subset, 0 AS next_element
    
    UNION ALL
    
    -- เพิ่มแต่ละ element
    SELECT subset || next_element, next_element + 1
    FROM power_set
    WHERE next_element <= 3
    
    UNION ALL
    
    -- ไม่เพิ่ม element (skip)
    SELECT subset, next_element + 1
    FROM power_set
    WHERE next_element <= 3
)
SELECT DISTINCT subset
FROM power_set
WHERE ARRAY_LENGTH(subset, 1) IS NOT NULL OR TRUE
ORDER BY ARRAY_LENGTH(subset, 1) NULLS FIRST, subset;
```

---

## 92.10 Performance Considerations

### ตัวอย่างที่ 23: ใช้ SEARCH clause (PostgreSQL 14+)

```sql
-- PostgreSQL 14+: ควบคุม traversal order
WITH RECURSIVE org_search AS (
    SELECT emp_id, name, manager_id
    FROM org_chart
    WHERE manager_id IS NULL
    
    UNION ALL
    
    SELECT e.emp_id, e.name, e.manager_id
    FROM org_chart e
    JOIN org_search s ON e.manager_id = s.emp_id
)
SEARCH DEPTH FIRST BY emp_id SET ordercol
SELECT emp_id, name, ordercol
FROM org_search
ORDER BY ordercol;

-- Breadth-first search
WITH RECURSIVE org_bfs AS (
    SELECT emp_id, name, manager_id
    FROM org_chart
    WHERE manager_id IS NULL
    
    UNION ALL
    
    SELECT e.emp_id, e.name, e.manager_id
    FROM org_chart e
    JOIN org_bfs s ON e.manager_id = s.emp_id
)
SEARCH BREADTH FIRST BY emp_id SET ordercol
SELECT emp_id, name, ordercol
FROM org_bfs
ORDER BY ordercol;
```

### ตัวอย่างที่ 24: Materialized Recursive CTE

```sql
-- สำหรับ recursive CTE ที่ถูกใช้หลายครั้ง ให้ materialize
WITH RECURSIVE org_full AS MATERIALIZED (
    SELECT emp_id, name, manager_id, 0 AS level
    FROM org_chart
    WHERE manager_id IS NULL
    
    UNION ALL
    
    SELECT e.emp_id, e.name, e.manager_id, o.level + 1
    FROM org_chart e
    JOIN org_full o ON e.manager_id = o.emp_id
)
SELECT level, COUNT(*) AS employee_count
FROM org_full
GROUP BY level
ORDER BY level;
```

### ตัวอย่างที่ 25: Recursive CTE vs Iterative approach

```sql
-- Recursive CTE สำหรับ hierarchical sum
WITH RECURSIVE dept_hierarchy AS (
    -- Leaf departments
    SELECT d.dept_id, d.dept_name, d.parent_dept_id,
           COALESCE(SUM(e.salary), 0) AS direct_salary_cost
    FROM departments d
    LEFT JOIN employees e ON e.dept_id = d.dept_id
    GROUP BY d.dept_id, d.dept_name, d.parent_dept_id
),
dept_total AS (
    -- Base: leaf nodes
    SELECT dept_id, dept_name, parent_dept_id, direct_salary_cost
    FROM dept_hierarchy
    WHERE dept_id NOT IN (SELECT DISTINCT parent_dept_id FROM departments WHERE parent_dept_id IS NOT NULL)
    
    UNION ALL
    
    -- Roll up to parent
    SELECT dh.dept_id, dh.dept_name, dh.parent_dept_id,
           dh.direct_salary_cost + SUM(dt.direct_salary_cost) OVER (PARTITION BY dh.dept_id)
    FROM dept_hierarchy dh
    JOIN dept_total dt ON dt.parent_dept_id = dh.dept_id
)
SELECT * FROM dept_total;
```

---

## แบบฝึกหัดท้ายบท

### แบบฝึกหัดที่ 1
เขียน Recursive CTE เพื่อสร้างตัวเลข 1 ถึง 100 และคำนวณ sum สะสม

**คำตอบ:**
```sql
WITH RECURSIVE numbers AS (
    SELECT 1 AS n, 1 AS running_sum
    
    UNION ALL
    
    SELECT n + 1, running_sum + (n + 1)
    FROM numbers
    WHERE n < 100
)
SELECT n, running_sum
FROM numbers
ORDER BY n;
```

### แบบฝึกหัดที่ 2
ใช้ Recursive CTE เพื่อหาพนักงานทุกคนที่อยู่ใต้ผู้จัดการ manager_id = 2 (ทุกระดับ)

**คำตอบ:**
```sql
WITH RECURSIVE team AS (
    -- Base: direct reports ของ manager 2
    SELECT emp_id, name, title, manager_id, 1 AS level
    FROM org_chart
    WHERE manager_id = 2
    
    UNION ALL
    
    -- Recursive: reports ของ reports
    SELECT e.emp_id, e.name, e.title, e.manager_id, t.level + 1
    FROM org_chart e
    JOIN team t ON e.manager_id = t.emp_id
)
SELECT level, emp_id, name, title
FROM team
ORDER BY level, name;
```

### แบบฝึกหัดที่ 3
เขียน Recursive CTE เพื่อสร้าง multiplication table 1-10

**คำตอบ:**
```sql
WITH RECURSIVE rows AS (
    SELECT 1 AS i
    UNION ALL
    SELECT i + 1 FROM rows WHERE i < 10
),
cols AS (
    SELECT 1 AS j
    UNION ALL
    SELECT j + 1 FROM cols WHERE j < 10
)
SELECT 
    r.i,
    c.j,
    r.i * c.j AS product
FROM rows r
CROSS JOIN cols c
ORDER BY r.i, c.j;
```

### แบบฝึกหัดที่ 4
ใช้ Recursive CTE เพื่อหา full path ของ category ที่มี product_id = 5

**คำตอบ:**
```sql
WITH RECURSIVE cat_path AS (
    -- เริ่มจาก category ของ product
    SELECT c.cat_id, c.cat_name, c.parent_id
    FROM products p
    JOIN categories c ON p.cat_id = c.cat_id
    WHERE p.product_id = 5
    
    UNION ALL
    
    -- ขึ้นไปหา parent
    SELECT c.cat_id, c.cat_name, c.parent_id
    FROM categories c
    JOIN cat_path cp ON c.cat_id = cp.parent_id
),
ordered AS (
    SELECT cat_name, ROW_NUMBER() OVER () AS rn,
           COUNT(*) OVER () AS total
    FROM cat_path
)
SELECT STRING_AGG(cat_name, ' > ' ORDER BY rn DESC) AS full_path
FROM ordered;
```

### แบบฝึกหัดที่ 5
เขียน Recursive CTE ที่แสดง org chart แบบ indented tree พร้อมตำแหน่งและเงินเดือน

**คำตอบ:**
```sql
WITH RECURSIVE org_display AS (
    SELECT 
        emp_id, name, title, salary, manager_id,
        0 AS level,
        name::VARCHAR(500) AS sort_path
    FROM employees
    WHERE manager_id IS NULL
    
    UNION ALL
    
    SELECT 
        e.emp_id, e.name, e.title, e.salary, e.manager_id,
        od.level + 1,
        (od.sort_path || ' | ' || e.name)::VARCHAR(500)
    FROM employees e
    JOIN org_display od ON e.manager_id = od.emp_id
)
SELECT 
    REPEAT('  ', level) || '- ' || name AS tree_node,
    title,
    salary
FROM org_display
ORDER BY sort_path;
```

### แบบฝึกหัดที่ 6
ใช้ Recursive CTE เพื่อคำนวณต้นทุนรวมของ BOM รวมถึงส่วนประกอบทุกระดับ

**คำตอบ:**
```sql
WITH RECURSIVE bom_costs AS (
    SELECT 
        component_id,
        component_name,
        parent_id,
        quantity,
        unit_cost,
        quantity * unit_cost AS item_cost
    FROM bom
    WHERE parent_id IS NULL
    
    UNION ALL
    
    SELECT 
        b.component_id,
        b.component_name,
        b.parent_id,
        b.quantity * bc.quantity,
        b.unit_cost,
        b.unit_cost * b.quantity * bc.quantity
    FROM bom b
    JOIN bom_costs bc ON b.parent_id = bc.component_id
)
SELECT 
    SUM(item_cost) AS total_cost
FROM bom_costs
WHERE parent_id IS NOT NULL;  -- ไม่รวม root
```

### แบบฝึกหัดที่ 7
เขียน Recursive CTE ที่สร้าง calendar สำหรับปี 2024 พร้อมระบุว่าเป็นวันหยุดสุดสัปดาห์หรือไม่

**คำตอบ:**
```sql
WITH RECURSIVE year_calendar AS (
    SELECT '2024-01-01'::date AS cal_date
    UNION ALL
    SELECT cal_date + 1
    FROM year_calendar
    WHERE cal_date < '2024-12-31'
)
SELECT 
    cal_date,
    TO_CHAR(cal_date, 'Day')                   AS day_name,
    TO_CHAR(cal_date, 'Mon')                   AS month_abbr,
    EXTRACT(WEEK FROM cal_date)                AS week_number,
    EXTRACT(DOW FROM cal_date) IN (0, 6)       AS is_weekend
FROM year_calendar
ORDER BY cal_date;
```

### แบบฝึกหัดที่ 8
ใช้ Recursive CTE เพื่อหา all paths ที่สั้นกว่า 3 hops ระหว่าง service dependencies

**คำตอบ:**
```sql
WITH RECURSIVE short_paths AS (
    SELECT 
        service AS start,
        depends_on AS current,
        ARRAY[service, depends_on] AS path,
        1 AS hops
    FROM service_deps
    
    UNION ALL
    
    SELECT 
        sp.start,
        sd.depends_on,
        sp.path || sd.depends_on,
        sp.hops + 1
    FROM service_deps sd
    JOIN short_paths sp ON sd.service = sp.current
    WHERE sp.hops < 3
      AND NOT sd.depends_on = ANY(sp.path)
)
SELECT start, current AS end_service, path, hops
FROM short_paths
ORDER BY start, hops;
```

### แบบฝึกหัดที่ 9
เขียน Recursive CTE เพื่อแปลง binary tree เป็น list (in-order traversal)

**คำตอบ:**
```sql
CREATE TABLE binary_tree (
    node_id     INT PRIMARY KEY,
    value       INT,
    left_child  INT,
    right_child INT
);

INSERT INTO binary_tree VALUES
(1, 5, 2, 3),
(2, 3, 4, 5),
(3, 8, 6, 7),
(4, 1, NULL, NULL),
(5, 4, NULL, NULL),
(6, 7, NULL, NULL),
(7, 9, NULL, NULL);

-- In-order traversal: left → root → right
WITH RECURSIVE inorder AS (
    SELECT node_id, value, left_child, right_child, 
           ARRAY[node_id] AS path, 'root' AS direction
    FROM binary_tree
    WHERE node_id = 1  -- เริ่มที่ root
    
    UNION ALL
    
    SELECT b.node_id, b.value, b.left_child, b.right_child,
           i.path || b.node_id, 'visit'
    FROM binary_tree b
    JOIN inorder i ON b.node_id = i.left_child OR b.node_id = i.right_child
    WHERE NOT b.node_id = ANY(i.path)
)
SELECT value, path
FROM inorder
ORDER BY path;
```

### แบบฝึกหัดที่ 10
สร้าง Recursive CTE ที่ generate prime numbers ตั้งแต่ 2 ถึง 50 (โดยใช้ Sieve of Eratosthenes แนวคิด)

**คำตอบ:**
```sql
-- สร้างตัวเลข 2-50 ก่อน
WITH RECURSIVE candidates AS (
    SELECT 2 AS n
    UNION ALL
    SELECT n + 1 FROM candidates WHERE n < 50
),
-- หาตัวเลขที่หารลงตัวได้ (ไม่ใช่ prime)
composites AS (
    SELECT DISTINCT a.n
    FROM candidates a
    JOIN candidates b ON b.n < a.n AND b.n >= 2
    WHERE a.n % b.n = 0
)
SELECT n AS prime_number
FROM candidates
WHERE n NOT IN (SELECT n FROM composites)
ORDER BY n;
```

---

## สรุปบทที่ 92

Recursive CTEs เปิดโลกใหม่ให้กับ SQL:
- **Hierarchical Data**: traverse org charts, category trees, file systems
- **Graph Traversal**: หา paths, shortest paths, connected components
- **Sequence Generation**: สร้างตัวเลข, วันที่, patterns
- **Transitive Closure**: หา indirect relationships ทั้งหมด
- **BOM Processing**: คำนวณต้นทุนสะสมจากโครงสร้าง nested

สิ่งที่ต้องระวัง:
- ต้องมีเงื่อนไขหยุด recursion เสมอ
- ระวัง infinite loops และ cycles
- ใช้ depth limits เพื่อป้องกัน performance issues
- PostgreSQL 14+ รองรับ CYCLE และ SEARCH clauses

ในบทถัดไป เราจะเข้าสู่โลกของ Window Functions ซึ่งเป็นอาวุธลับสำหรับการวิเคราะห์ข้อมูลระดับสูง
