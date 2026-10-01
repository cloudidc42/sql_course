# ส่วนที่ 98: Full-Text Search ใน SQL

## บทนำ

Full-Text Search (FTS) คือการค้นหาข้อความที่เข้าใจความหมาย ไม่ใช่แค่ pattern matching เหมือน LIKE ระบบ FTS สามารถ:
- ค้นหาคำที่มีรูปแบบต่างกัน (running = run = ran)
- ค้นหาตาม relevance ranking
- รองรับ boolean operators (AND, OR, NOT)
- จัดการ stop words และ stemming

---

## 98.1 Full-Text Search ใน PostgreSQL

### ตัวอย่างที่ 1: tsvector และ tsquery พื้นฐาน

```sql
-- tsvector: document representation (preprocessed text)
-- tsquery: query terms

-- สร้าง tsvector
SELECT to_tsvector('english', 'The quick brown fox jumps over the lazy dog');
-- 'brown':3 'dog':9 'fox':4 'jump':5 'lazi':8 'quick':2

-- สร้าง tsquery
SELECT to_tsquery('english', 'quick & fox');
-- 'quick' & 'fox'

-- ตรวจสอบว่า document match query
SELECT to_tsvector('english', 'The quick brown fox') @@ to_tsquery('english', 'quick & fox');
-- true

SELECT to_tsvector('english', 'The quick brown fox') @@ to_tsquery('english', 'quick & cat');
-- false
```

### ตัวอย่างที่ 2: สร้างตารางสำหรับ FTS

```sql
-- สร้างตาราง articles
CREATE TABLE articles (
    article_id  SERIAL PRIMARY KEY,
    title       VARCHAR(500),
    body        TEXT,
    author      VARCHAR(200),
    category    VARCHAR(50),
    tags        TEXT[],
    published_at DATE,
    search_vector TSVECTOR   -- precomputed search vector
);

INSERT INTO articles (title, body, author, category, tags, published_at) VALUES
(1, 'Introduction to PostgreSQL', 
 'PostgreSQL is a powerful open-source relational database management system. It supports advanced data types and performance optimization.',
 'John Smith', 'Database', ARRAY['postgresql', 'database', 'tutorial'], '2024-01-01'),
(2, 'Advanced SQL Window Functions',
 'Window functions in SQL allow you to perform calculations across related rows. They are essential for analytics and reporting.',
 'Jane Doe', 'SQL', ARRAY['sql', 'analytics', 'window-functions'], '2024-01-15'),
(3, 'Database Performance Optimization',
 'Optimizing database queries involves proper indexing, query planning, and understanding execution plans.',
 'Bob Johnson', 'Performance', ARRAY['optimization', 'index', 'performance'], '2024-02-01'),
(4, 'Introduction to JSON in SQL',
 'Modern databases support JSON data types for storing semi-structured data. PostgreSQL jsonb offers powerful query capabilities.',
 'Alice Brown', 'Database', ARRAY['json', 'postgresql', 'nosql'], '2024-02-15'),
(5, 'Full-Text Search Techniques',
 'Full-text search enables efficient searching through large text documents. PostgreSQL provides tsvector and tsquery for this purpose.',
 'Charlie Wilson', 'Search', ARRAY['search', 'fulltext', 'postgresql'], '2024-03-01');
```

### ตัวอย่างที่ 3: to_tsvector ด้วยหลาย columns

```sql
-- สร้าง search vector จากหลาย columns (title มี weight สูงกว่า)
UPDATE articles
SET search_vector = 
    setweight(to_tsvector('english', COALESCE(title, '')), 'A') ||
    setweight(to_tsvector('english', COALESCE(body, '')), 'B') ||
    setweight(to_tsvector('english', COALESCE(author, '')), 'C');

-- ค้นหาด้วย tsvector
SELECT title, author
FROM articles
WHERE search_vector @@ to_tsquery('english', 'postgresql & database');
```

---

## 98.2 GIN Index สำหรับ Full-Text Search

### ตัวอย่างที่ 4: สร้าง GIN Index

```sql
-- GIN index สำหรับ tsvector column
CREATE INDEX idx_articles_search ON articles USING GIN (search_vector);

-- ค้นหาเร็วด้วย index
SELECT title, published_at
FROM articles
WHERE search_vector @@ to_tsquery('english', 'database')
ORDER BY published_at DESC;

-- Index บน computed tsvector (ไม่ต้องมี stored column)
CREATE INDEX idx_articles_fts ON articles 
USING GIN (to_tsvector('english', title || ' ' || body));
```

---

## 98.3 Query Types ใน PostgreSQL

### ตัวอย่างที่ 5: to_tsquery - Boolean Operators

```sql
-- AND: & (ต้องมีทั้งสองคำ)
SELECT title FROM articles
WHERE search_vector @@ to_tsquery('english', 'postgresql & database');

-- OR: | (มีอย่างน้อยหนึ่งคำ)
SELECT title FROM articles
WHERE search_vector @@ to_tsquery('english', 'postgresql | mysql');

-- NOT: ! (ต้องไม่มีคำนี้)
SELECT title FROM articles
WHERE search_vector @@ to_tsquery('english', 'database & !mysql');

-- ลำดับ (<-): ต้องอยู่ติดกัน
SELECT title FROM articles
WHERE search_vector @@ to_tsquery('english', 'window <-> function');

-- Distance (n): ห่างกัน n คำ
SELECT title FROM articles
WHERE search_vector @@ to_tsquery('english', 'text <2> search');
```

### ตัวอย่างที่ 6: plainto_tsquery - Natural Language Query

```sql
-- plainto_tsquery: แปลง plain text เป็น query (AND by default)
SELECT plainto_tsquery('english', 'database optimization');
-- 'databas' & 'optim'

SELECT title FROM articles
WHERE search_vector @@ plainto_tsquery('english', 'database performance optimization');
-- ค้นหาแบบ natural language
```

### ตัวอย่างที่ 7: phraseto_tsquery - Exact Phrase

```sql
-- phraseto_tsquery: ค้นหาวลีที่แน่นอน
SELECT phraseto_tsquery('english', 'open source database');
-- 'open' <-> 'sourc' <-> 'databas'

SELECT title FROM articles
WHERE search_vector @@ phraseto_tsquery('english', 'window functions');
```

### ตัวอย่างที่ 8: websearch_to_tsquery (PostgreSQL 11+)

```sql
-- websearch_to_tsquery: รูปแบบเหมือน Google search
-- "exact phrase" → phraseto
-- -word → NOT
-- word1 word2 → AND
-- word1 OR word2 → OR

SELECT websearch_to_tsquery('english', '"postgresql" performance -mysql');
-- 'postgresql' & 'perform' & !'mysql'

SELECT title FROM articles
WHERE search_vector @@ websearch_to_tsquery('english', 'postgresql performance -mysql');
```

---

## 98.4 ts_rank - Relevance Scoring

### ตัวอย่างที่ 9: ts_rank พื้นฐาน

```sql
-- ts_rank: คำนวณ relevance score
SELECT 
    title,
    ts_rank(search_vector, to_tsquery('english', 'postgresql')) AS rank
FROM articles
WHERE search_vector @@ to_tsquery('english', 'postgresql')
ORDER BY rank DESC;
```

### ตัวอย่างที่ 10: ts_rank_cd (Cover Density)

```sql
-- ts_rank_cd: คำนึงถึงความหนาแน่นของคำค้นหา
SELECT 
    title,
    ts_rank(search_vector, query) AS rank,
    ts_rank_cd(search_vector, query) AS rank_cd
FROM articles,
     to_tsquery('english', 'database | postgresql') AS query
WHERE search_vector @@ query
ORDER BY rank_cd DESC;
```

### ตัวอย่างที่ 11: Ranking พร้อม Normalization

```sql
-- ts_rank normalization options
-- 0 (default): ignores document length
-- 1: divides by 1 + log(document length)
-- 2: divides by document length
-- 4: divides by mean harmonic distance
-- 8: divides by number of unique words
-- 16: divides by 1 + log(unique words)
-- 32: divides by itself + 1

SELECT 
    title,
    ts_rank(search_vector, query, 1) AS rank_normalized
FROM articles,
     to_tsquery('english', 'sql | database') AS query
WHERE search_vector @@ query
ORDER BY rank_normalized DESC;
```

---

## 98.5 Text Highlighting

### ตัวอย่างที่ 12: ts_headline - Highlight Search Terms

```sql
-- ts_headline: แสดง excerpt พร้อม highlight
SELECT 
    title,
    ts_headline(
        'english',
        body,
        to_tsquery('english', 'database'),
        'MaxWords=30, MinWords=15, StartSel=<b>, StopSel=</b>'
    ) AS excerpt
FROM articles
WHERE search_vector @@ to_tsquery('english', 'database');
```

### ตัวอย่างที่ 13: ts_headline พร้อม Options

```sql
-- Options สำหรับ ts_headline
SELECT 
    article_id,
    title,
    ts_headline(
        'english',
        body,
        websearch_to_tsquery('english', 'postgresql performance'),
        'StartSel=[, StopSel=], MaxFragments=3, FragmentDelimiter=...'
    ) AS highlighted_body
FROM articles
WHERE search_vector @@ websearch_to_tsquery('english', 'postgresql performance')
ORDER BY ts_rank(search_vector, websearch_to_tsquery('english', 'postgresql performance')) DESC;
```

---

## 98.6 Multi-Language Search

### ตัวอย่างที่ 14: ค้นหาหลายภาษา

```sql
-- PostgreSQL รองรับหลาย language configurations
-- english, spanish, french, german, portuguese, russian, etc.

-- สร้าง search vector สำหรับภาษาไทย (simple config)
SELECT to_tsvector('simple', 'ระบบฐานข้อมูลเชิงสัมพันธ์');
-- 'ระบบฐานข้อมูลเชิงสัมพันธ์':1

-- สำหรับภาษาไทย ใช้ 'simple' config (no stemming)
CREATE TABLE thai_content (
    content_id  SERIAL PRIMARY KEY,
    title       TEXT,
    body        TEXT,
    search_vec  TSVECTOR GENERATED ALWAYS AS (
        to_tsvector('simple', COALESCE(title, '') || ' ' || COALESCE(body, ''))
    ) STORED
);

CREATE INDEX idx_thai_content ON thai_content USING GIN (search_vec);

INSERT INTO thai_content (title, body) VALUES
('บทนำ SQL', 'SQL เป็นภาษาสำหรับจัดการฐานข้อมูล สามารถ query update delete ข้อมูลได้'),
('Window Functions', 'Window functions ช่วยวิเคราะห์ข้อมูลโดยไม่ต้องรวม rows'),
('JSON ใน PostgreSQL', 'PostgreSQL รองรับ JSONB สำหรับข้อมูล semi-structured');

-- ค้นหาภาษาไทย
SELECT title, body
FROM thai_content
WHERE search_vec @@ to_tsquery('simple', 'SQL & ฐานข้อมูล');
```

---

## 98.7 Automatic tsvector Update

### ตัวอย่างที่ 15: Generated Column (PostgreSQL 12+)

```sql
-- Generated column อัพเดตอัตโนมัติ
CREATE TABLE blog_posts (
    post_id     SERIAL PRIMARY KEY,
    title       TEXT,
    content     TEXT,
    author      TEXT,
    search_vec  TSVECTOR GENERATED ALWAYS AS (
        setweight(to_tsvector('english', COALESCE(title, '')), 'A') ||
        setweight(to_tsvector('english', COALESCE(content, '')), 'B') ||
        setweight(to_tsvector('english', COALESCE(author, '')), 'C')
    ) STORED
);

CREATE INDEX idx_blog_search ON blog_posts USING GIN (search_vec);

INSERT INTO blog_posts (title, content, author) VALUES
('SQL Tips and Tricks', 'Learn advanced SQL techniques for better performance', 'Jane Doe'),
('Database Design Patterns', 'Best practices for relational database schema design', 'John Smith');

-- search_vec อัพเดตอัตโนมัติเมื่อ INSERT หรือ UPDATE
SELECT title, ts_rank(search_vec, query) AS rank
FROM blog_posts, to_tsquery('english', 'database & design') AS query
WHERE search_vec @@ query
ORDER BY rank DESC;
```

### ตัวอย่างที่ 16: Trigger-based Update (สำหรับเวอร์ชันเก่า)

```sql
-- ใช้ trigger สำหรับ PostgreSQL < 12
CREATE OR REPLACE FUNCTION update_search_vector()
RETURNS TRIGGER AS $$
BEGIN
    NEW.search_vector = 
        setweight(to_tsvector('english', COALESCE(NEW.title, '')), 'A') ||
        setweight(to_tsvector('english', COALESCE(NEW.body, '')), 'B');
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER articles_search_update
BEFORE INSERT OR UPDATE ON articles
FOR EACH ROW EXECUTE FUNCTION update_search_vector();
```

---

## 98.8 Full-Text Search ใน MySQL

### ตัวอย่างที่ 17: MySQL FULLTEXT Index

```sql
-- MySQL FULLTEXT Index (InnoDB: MySQL 5.6+)
CREATE TABLE articles_mysql (
    article_id  INT PRIMARY KEY AUTO_INCREMENT,
    title       VARCHAR(500),
    body        TEXT,
    author      VARCHAR(200),
    FULLTEXT idx_search (title, body)
);

INSERT INTO articles_mysql (title, body, author) VALUES
('Introduction to MySQL', 'MySQL is a popular open-source relational database. It is widely used for web applications.', 'John Smith'),
('MySQL Performance Tips', 'Optimizing MySQL queries with proper indexing and query analysis helps improve performance.', 'Jane Doe'),
('Advanced MySQL Features', 'MySQL supports stored procedures, triggers, and JSON data types for modern applications.', 'Bob Johnson');
```

### ตัวอย่างที่ 18: MATCH...AGAINST ใน MySQL

```sql
-- Natural Language Mode (default)
SELECT title, body,
       MATCH(title, body) AGAINST('mysql performance') AS relevance
FROM articles_mysql
WHERE MATCH(title, body) AGAINST('mysql performance')
ORDER BY relevance DESC;

-- Boolean Mode
SELECT title,
       MATCH(title, body) AGAINST('+mysql -oracle' IN BOOLEAN MODE) AS relevance
FROM articles_mysql
WHERE MATCH(title, body) AGAINST('+mysql -oracle' IN BOOLEAN MODE);
-- + = must contain, - = must not contain

-- Query Expansion Mode
SELECT title
FROM articles_mysql
WHERE MATCH(title, body) AGAINST('database' WITH QUERY EXPANSION);
-- ค้นหาเพิ่มเติมจาก results ของ round แรก
```

### ตัวอย่างที่ 19: MySQL Boolean Mode Operators

```sql
-- Boolean Mode ใน MySQL:
-- +word    : word ต้องอยู่
-- -word    : word ต้องไม่อยู่
-- >word    : เพิ่ม rank
-- <word    : ลด rank
-- (...)    : grouping
-- "phrase" : exact phrase
-- word*    : wildcard prefix
-- ~word    : นับเป็น negative แต่ไม่ exclude

SELECT title, 
       MATCH(title, body) AGAINST('+mysql +performance -slow' IN BOOLEAN MODE) AS score
FROM articles_mysql
WHERE MATCH(title, body) AGAINST('+mysql +performance -slow' IN BOOLEAN MODE)
ORDER BY score DESC;

-- Phrase search
SELECT title
FROM articles_mysql
WHERE MATCH(title, body) AGAINST('"open source relational"' IN BOOLEAN MODE);

-- Wildcard
SELECT title
FROM articles_mysql
WHERE MATCH(title, body) AGAINST('optim*' IN BOOLEAN MODE);
-- matches: optimize, optimizing, optimization
```

---

## 98.9 Full-Text Search ใน SQL Server

### ตัวอย่างที่ 20: CONTAINS และ FREETEXT

```sql
-- SQL Server Full-Text Search

-- สร้าง full-text catalog
-- (ต้องทำผ่าน SQL Server Management Studio หรือ T-SQL)
CREATE FULLTEXT CATALOG ft_catalog AS DEFAULT;

CREATE TABLE articles_ss (
    article_id  INT PRIMARY KEY,
    title       NVARCHAR(500),
    body        NVARCHAR(MAX),
    author      NVARCHAR(200)
);

-- สร้าง full-text index
CREATE FULLTEXT INDEX ON articles_ss (title, body, author)
KEY INDEX PK_articles_ss ON ft_catalog;

-- CONTAINS: precise search
SELECT title
FROM articles_ss
WHERE CONTAINS(body, 'database');

-- CONTAINS กับ boolean
SELECT title
FROM articles_ss
WHERE CONTAINS((title, body), '"full text" AND search');

-- CONTAINS กับ NEAR
SELECT title
FROM articles_ss
WHERE CONTAINS(body, 'NEAR((database, performance), 5)');
-- database และ performance ต้องอยู่ห่างกันไม่เกิน 5 คำ

-- FREETEXT: natural language search
SELECT title
FROM articles_ss
WHERE FREETEXT(body, 'database management systems');
-- ค้นหาตาม concept ไม่ใช่ exact match
```

### ตัวอย่างที่ 21: CONTAINSTABLE และ FREETEXTTABLE

```sql
-- CONTAINSTABLE: returns table with relevance
SELECT a.title, ct.RANK AS relevance
FROM articles_ss a
INNER JOIN CONTAINSTABLE(articles_ss, body, 'database') AS ct
    ON a.article_id = ct.[KEY]
ORDER BY ct.RANK DESC;

-- FREETEXTTABLE: natural language + ranking
SELECT a.title, ft.RANK
FROM articles_ss a
INNER JOIN FREETEXTTABLE(articles_ss, (title, body), 'database performance') AS ft
    ON a.article_id = ft.[KEY]
ORDER BY ft.RANK DESC;
```

---

## 98.10 Full-Text Search Patterns ขั้นสูง

### ตัวอย่างที่ 22: Search with Filtering

```sql
-- ค้นหาพร้อม filter อื่นๆ
SELECT 
    article_id,
    title,
    category,
    published_at,
    ts_rank(search_vector, query) AS relevance
FROM articles,
     to_tsquery('english', 'database & performance') AS query
WHERE search_vector @@ query
  AND category IN ('Database', 'Performance')
  AND published_at >= '2024-01-01'
ORDER BY relevance DESC, published_at DESC;
```

### ตัวอย่างที่ 23: Search API Pattern

```sql
-- Search function ที่ยืดหยุ่น
CREATE OR REPLACE FUNCTION search_articles(
    p_query TEXT,
    p_category VARCHAR DEFAULT NULL,
    p_limit INT DEFAULT 10,
    p_offset INT DEFAULT 0
)
RETURNS TABLE (
    article_id  INT,
    title       TEXT,
    excerpt     TEXT,
    author      TEXT,
    category    TEXT,
    relevance   REAL
) AS $$
DECLARE
    v_query TSQUERY;
BEGIN
    -- แปลง query string เป็น tsquery
    v_query = websearch_to_tsquery('english', p_query);
    
    RETURN QUERY
    SELECT 
        a.article_id::INT,
        a.title::TEXT,
        ts_headline(
            'english',
            a.body,
            v_query,
            'MaxWords=50, MinWords=20'
        )::TEXT AS excerpt,
        a.author::TEXT,
        a.category::TEXT,
        ts_rank(a.search_vector, v_query)::REAL AS relevance
    FROM articles a
    WHERE a.search_vector @@ v_query
      AND (p_category IS NULL OR a.category = p_category)
    ORDER BY relevance DESC
    LIMIT p_limit
    OFFSET p_offset;
END;
$$ LANGUAGE plpgsql;

-- ใช้งาน
SELECT * FROM search_articles('postgresql database', NULL, 5, 0);
```

### ตัวอย่างที่ 24: Faceted Search

```sql
-- Search พร้อม facets (aggregation สำหรับ filters)
WITH search_results AS (
    SELECT 
        article_id,
        title,
        category,
        author,
        tags,
        ts_rank(search_vector, query) AS rank
    FROM articles,
         to_tsquery('english', 'database') AS query
    WHERE search_vector @@ query
),
facets AS (
    SELECT 
        'category' AS facet_type,
        category AS facet_value,
        COUNT(*) AS count
    FROM search_results
    GROUP BY category
    
    UNION ALL
    
    SELECT 
        'author',
        author,
        COUNT(*)
    FROM search_results
    GROUP BY author
)
SELECT 
    jsonb_build_object(
        'total', (SELECT COUNT(*) FROM search_results),
        'results', (
            SELECT jsonb_agg(jsonb_build_object(
                'id', article_id,
                'title', title,
                'relevance', rank
            ) ORDER BY rank DESC)
            FROM search_results LIMIT 5
        ),
        'facets', (
            SELECT jsonb_object_agg(facet_type, 
                jsonb_agg(jsonb_build_object('value', facet_value, 'count', count) ORDER BY count DESC)
            )
            FROM facets
        )
    ) AS search_response;
```

### ตัวอย่างที่ 25: Spell-checking / Did You Mean

```sql
-- ใช้ pg_trgm extension สำหรับ fuzzy search
CREATE EXTENSION IF NOT EXISTS pg_trgm;

-- Trigram similarity
SELECT title,
       similarity(title, 'postgresl') AS sim  -- typo: postgresl
FROM articles
ORDER BY sim DESC
LIMIT 5;

-- รวม full-text search กับ trigram similarity
SELECT title,
       ts_rank(search_vector, to_tsquery('english', 'postgresl')) AS fts_rank,
       similarity(title, 'postgresl') AS trgm_sim
FROM articles
WHERE title % 'postgresl'  -- trigram match (threshold = 0.3 default)
   OR search_vector @@ to_tsquery('english', 'postgresl')
ORDER BY trgm_sim + ts_rank(search_vector, to_tsquery('english', 'postgresl')) DESC;
```

### ตัวอย่างที่ 26: Autocomplete ด้วย prefix search

```sql
-- Prefix search สำหรับ autocomplete
SELECT DISTINCT 
    word
FROM ts_stat('SELECT search_vector FROM articles')
WHERE word LIKE 'data%'
ORDER BY nentry DESC, ndoc DESC, word
LIMIT 10;

-- หรือใช้ GIN index กับ trgm
CREATE INDEX idx_title_trgm ON articles USING GIN (title gin_trgm_ops);
SELECT title
FROM articles
WHERE title ILIKE 'postgre%'
ORDER BY title;
```

---

## 98.11 Full-Text Index Maintenance

### ตัวอย่างที่ 27: ตรวจสอบ Index Statistics

```sql
-- ตรวจสอบคำที่มีใน index
SELECT word, ndoc, nentry
FROM ts_stat('SELECT search_vector FROM articles')
ORDER BY nentry DESC
LIMIT 20;

-- ตรวจสอบขนาด index
SELECT 
    schemaname,
    tablename,
    indexname,
    pg_size_pretty(pg_relation_size(indexrelid)) AS index_size
FROM pg_stat_user_indexes
WHERE tablename = 'articles';
```

### ตัวอย่างที่ 28: Rebuild Search Vectors

```sql
-- Rebuild search vectors ทั้งหมด
UPDATE articles
SET search_vector = 
    setweight(to_tsvector('english', COALESCE(title, '')), 'A') ||
    setweight(to_tsvector('english', COALESCE(body, '')), 'B') ||
    setweight(to_tsvector('english', COALESCE(author, '')), 'C');

-- REINDEX ถ้า index เสียหาย
REINDEX INDEX idx_articles_search;
```

---

## 98.12 Custom Dictionary และ Synonyms

### ตัวอย่างที่ 29: Simple Dictionary สำหรับ Synonyms

```sql
-- สร้าง text configuration ที่รองรับ synonyms
-- (ต้องมี dictionary files บน server)

-- ตัวอย่าง: ค้นหา 'DB' ให้ match 'database'
-- ต้องสร้าง synonym file และ dictionary

-- Workaround ใน SQL: expand query manually
CREATE OR REPLACE FUNCTION expand_search_query(p_query TEXT)
RETURNS TSQUERY AS $$
DECLARE
    v_expanded TEXT;
BEGIN
    v_expanded = p_query;
    -- แทนที่ abbreviations
    v_expanded = REPLACE(v_expanded, 'DB', 'database');
    v_expanded = REPLACE(v_expanded, 'PG', 'postgresql');
    
    RETURN websearch_to_tsquery('english', v_expanded);
END;
$$ LANGUAGE plpgsql;

SELECT title FROM articles
WHERE search_vector @@ expand_search_query('PG performance');
```

### ตัวอย่างที่ 30: Stop Words Configuration

```sql
-- ตรวจสอบว่าคำไหนถูกตัดออกเป็น stop words
SELECT * FROM ts_debug('english', 'The quick brown fox jumps over the lazy dog');
-- แสดง alias, description, token, dictionaries, dictionary, lexemes

-- ค้นหาด้วยคำที่ไม่ใช่ stop words เท่านั้น
SELECT 
    to_tsvector('english', 'The quick brown fox') AS vector,
    to_tsquery('english', 'quick & fox') AS query;
-- 'the' ถูกตัดออก (stop word)
```

---

## 98.13 การผสม FTS กับ Other Search Types

### ตัวอย่างที่ 31: FTS + LIKE + Pattern Matching

```sql
-- ค้นหาแบบ hybrid: full-text + exact match
SELECT 
    article_id,
    title,
    ts_rank(search_vector, fts_query) AS fts_score,
    -- เพิ่ม score ถ้า title มี exact match
    CASE WHEN title ILIKE '%' || 'postgresql' || '%' THEN 0.5 ELSE 0 END AS exact_bonus
FROM articles,
     to_tsquery('english', 'postgresql') AS fts_query
WHERE 
    search_vector @@ fts_query
    OR title ILIKE '%postgresql%'  -- รวม results ที่ไม่ผ่าน FTS
ORDER BY fts_score + CASE WHEN title ILIKE '%postgresql%' THEN 0.5 ELSE 0 END DESC;
```

---

## แบบฝึกหัดท้ายบท

### แบบฝึกหัดที่ 1
สร้าง full-text search index บนตาราง articles และค้นหาบทความที่เกี่ยวกับ "database"

**คำตอบ:**
```sql
-- อัพเดต search_vector
UPDATE articles
SET search_vector = 
    setweight(to_tsvector('english', COALESCE(title, '')), 'A') ||
    setweight(to_tsvector('english', COALESCE(body, '')), 'B');

-- สร้าง index
CREATE INDEX IF NOT EXISTS idx_articles_search ON articles USING GIN (search_vector);

-- ค้นหา
SELECT article_id, title, 
       ts_rank(search_vector, to_tsquery('english', 'database')) AS rank
FROM articles
WHERE search_vector @@ to_tsquery('english', 'database')
ORDER BY rank DESC;
```

### แบบฝึกหัดที่ 2
ค้นหาบทความที่มีคำว่า "sql" และ "performance" แต่ไม่มี "slow"

**คำตอบ:**
```sql
SELECT article_id, title,
       ts_rank(search_vector, query) AS rank
FROM articles,
     to_tsquery('english', 'sql & performance & !slow') AS query
WHERE search_vector @@ query
ORDER BY rank DESC;
```

### แบบฝึกหัดที่ 3
สร้าง function ที่รับ search query และ return results พร้อม highlighted excerpts

**คำตอบ:**
```sql
CREATE OR REPLACE FUNCTION full_text_search(
    p_query TEXT,
    p_limit INT DEFAULT 10
)
RETURNS TABLE(
    article_id INT,
    title TEXT,
    excerpt TEXT,
    rank REAL
) AS $$
DECLARE
    v_tsquery TSQUERY;
BEGIN
    v_tsquery = websearch_to_tsquery('english', p_query);
    
    RETURN QUERY
    SELECT 
        a.article_id::INT,
        a.title::TEXT,
        ts_headline(
            'english', a.body, v_tsquery,
            'StartSel=<<, StopSel=>>, MaxWords=40'
        )::TEXT,
        ts_rank(a.search_vector, v_tsquery)::REAL
    FROM articles a
    WHERE a.search_vector @@ v_tsquery
    ORDER BY ts_rank(a.search_vector, v_tsquery) DESC
    LIMIT p_limit;
END;
$$ LANGUAGE plpgsql;

SELECT * FROM full_text_search('postgresql database');
```

### แบบฝึกหัดที่ 4
Implement search pagination ด้วย ts_rank

**คำตอบ:**
```sql
CREATE OR REPLACE FUNCTION paginated_search(
    p_query TEXT,
    p_page INT DEFAULT 1,
    p_page_size INT DEFAULT 5
)
RETURNS TABLE(
    article_id INT,
    title TEXT,
    category TEXT,
    rank REAL,
    total_count BIGINT
) AS $$
DECLARE
    v_query TSQUERY;
    v_offset INT;
BEGIN
    v_query = websearch_to_tsquery('english', p_query);
    v_offset = (p_page - 1) * p_page_size;
    
    RETURN QUERY
    SELECT 
        a.article_id::INT,
        a.title::TEXT,
        a.category::TEXT,
        ts_rank(a.search_vector, v_query)::REAL,
        COUNT(*) OVER ()::BIGINT AS total_count
    FROM articles a
    WHERE a.search_vector @@ v_query
    ORDER BY ts_rank(a.search_vector, v_query) DESC
    LIMIT p_page_size
    OFFSET v_offset;
END;
$$ LANGUAGE plpgsql;

SELECT * FROM paginated_search('database', 1, 3);
```

### แบบฝึกหัดที่ 5
เขียน MySQL query สำหรับ full-text search ด้วย boolean mode ค้นหา articles เกี่ยวกับ "mysql" แต่ไม่ใช่ "oracle"

**คำตอบ:**
```sql
SELECT 
    article_id,
    title,
    MATCH(title, body) AGAINST('+mysql -oracle' IN BOOLEAN MODE) AS relevance
FROM articles_mysql
WHERE MATCH(title, body) AGAINST('+mysql -oracle' IN BOOLEAN MODE)
ORDER BY relevance DESC;
```

### แบบฝึกหัดที่ 6
สร้าง autocomplete suggestion โดยใช้ pg_trgm

**คำตอบ:**
```sql
CREATE EXTENSION IF NOT EXISTS pg_trgm;
CREATE INDEX IF NOT EXISTS idx_title_trgm ON articles USING GIN (title gin_trgm_ops);

CREATE OR REPLACE FUNCTION suggest_titles(p_prefix TEXT, p_limit INT DEFAULT 5)
RETURNS TABLE(suggestion TEXT, similarity REAL) AS $$
BEGIN
    RETURN QUERY
    SELECT DISTINCT
        title::TEXT,
        similarity(title, p_prefix)::REAL
    FROM articles
    WHERE title ILIKE p_prefix || '%'
    ORDER BY similarity(title, p_prefix) DESC, title
    LIMIT p_limit;
END;
$$ LANGUAGE plpgsql;

SELECT * FROM suggest_titles('Post');
```

### แบบฝึกหัดที่ 7
วิเคราะห์ word frequency ใน search index

**คำตอบ:**
```sql
SELECT 
    word,
    ndoc AS documents_containing,
    nentry AS total_occurrences,
    ROUND(nentry::DECIMAL / ndoc, 2) AS avg_per_document
FROM ts_stat('SELECT search_vector FROM articles')
WHERE length(word) > 3  -- ตัด short words ออก
ORDER BY nentry DESC
LIMIT 20;
```

### แบบฝึกหัดที่ 8
ค้นหา articles ที่เกี่ยวกับ phrase "window function" (ต้องอยู่ติดกัน)

**คำตอบ:**
```sql
SELECT article_id, title,
       ts_rank(search_vector, query) AS rank
FROM articles,
     phraseto_tsquery('english', 'window function') AS query
WHERE search_vector @@ query
ORDER BY rank DESC;
```

### แบบฝึกหัดที่ 9
สร้าง faceted search ที่แสดง results พร้อม count ตาม category

**คำตอบ:**
```sql
WITH search_base AS (
    SELECT 
        article_id,
        title,
        category,
        author,
        ts_rank(search_vector, query) AS rank
    FROM articles,
         websearch_to_tsquery('english', 'database performance') AS query
    WHERE search_vector @@ query
)
SELECT 
    jsonb_build_object(
        'results', (
            SELECT jsonb_agg(jsonb_build_object(
                'id', article_id,
                'title', title,
                'category', category,
                'rank', ROUND(rank::NUMERIC, 4)
            ) ORDER BY rank DESC)
            FROM search_base LIMIT 5
        ),
        'facets', jsonb_build_object(
            'by_category', (
                SELECT jsonb_object_agg(category, cnt)
                FROM (
                    SELECT category, COUNT(*) AS cnt
                    FROM search_base
                    GROUP BY category
                ) cats
            ),
            'by_author', (
                SELECT jsonb_object_agg(author, cnt)
                FROM (
                    SELECT author, COUNT(*) AS cnt
                    FROM search_base
                    GROUP BY author
                ) auths
            )
        ),
        'total', (SELECT COUNT(*) FROM search_base)
    ) AS search_response;
```

### แบบฝึกหัดที่ 10
สร้าง "Did You Mean?" feature โดยใช้ trigram similarity

**คำตอบ:**
```sql
-- ค้นหาและถ้าไม่เจอ suggest คำที่ใกล้เคียง
CREATE OR REPLACE FUNCTION smart_search(p_query TEXT)
RETURNS TABLE(
    result_type TEXT,
    article_id INT,
    title TEXT,
    suggestion TEXT
) AS $$
DECLARE
    v_tsquery TSQUERY;
    v_count INT;
BEGIN
    BEGIN
        v_tsquery = websearch_to_tsquery('english', p_query);
    EXCEPTION WHEN OTHERS THEN
        v_tsquery = plainto_tsquery('english', p_query);
    END;
    
    SELECT COUNT(*) INTO v_count
    FROM articles WHERE search_vector @@ v_tsquery;
    
    IF v_count > 0 THEN
        -- Return results
        RETURN QUERY
        SELECT 
            'result'::TEXT,
            a.article_id::INT,
            a.title::TEXT,
            NULL::TEXT
        FROM articles a
        WHERE a.search_vector @@ v_tsquery
        ORDER BY ts_rank(a.search_vector, v_tsquery) DESC
        LIMIT 5;
    ELSE
        -- Suggest similar words
        RETURN QUERY
        SELECT 
            'suggestion'::TEXT,
            NULL::INT,
            NULL::TEXT,
            word::TEXT AS suggestion
        FROM ts_stat('SELECT search_vector FROM articles'),
             to_tsvector('simple', p_query) query_vec
        WHERE similarity(word, p_query) > 0.3
        ORDER BY similarity(word, p_query) DESC
        LIMIT 3;
    END IF;
END;
$$ LANGUAGE plpgsql;

SELECT * FROM smart_search('postgresqll');  -- typo
```

---

## สรุปบทที่ 98

### Full-Text Search: สรุปเปรียบเทียบ

| Feature | PostgreSQL | MySQL | SQL Server |
|---------|-----------|-------|------------|
| Data Type | tsvector | FULLTEXT | nvarchar |
| Index | GIN/GiST | FULLTEXT | Full-Text Index |
| Query Language | tsquery | MATCH...AGAINST | CONTAINS/FREETEXT |
| Relevance | ts_rank() | MATCH score | CONTAINSTABLE.RANK |
| Highlight | ts_headline() | ไม่มี built-in | ไม่มี built-in |
| Languages | หลายภาษา | หลายภาษา | หลายภาษา |

### Best Practices
1. ใช้ **GIN index** เสมอสำหรับ tsvector columns
2. **setweight** A/B/C/D เพื่อ boost title มากกว่า body
3. ใช้ **websearch_to_tsquery** สำหรับ user input (ป้องกัน syntax errors)
4. เก็บ **precomputed tsvector** สำหรับ large tables
5. ใช้ **pg_trgm** ร่วมกันสำหรับ fuzzy matching

ในบทถัดไปเราจะเรียนรู้เกี่ยวกับ Analytical Functions และ OLAP Operations ซึ่งใช้สำหรับ Business Intelligence
