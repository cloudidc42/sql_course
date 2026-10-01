# Part 117: Social Media Platform Database

## บทนำ (Introduction)

Social Media Platform เป็นระบบที่มีความซับซ้อนสูง ต้องรองรับ Users หลายล้านคน Posts, Reactions, Comments, Followers และ News Feed Algorithm บทนี้ออกแบบ Schema ครบถ้วน พร้อม Queries ที่ใช้จริงในระบบ Social Media

## 1. Complete Social Media Schema

```sql
-- =========================================
-- SOCIAL MEDIA PLATFORM - COMPLETE DDL
-- =========================================

CREATE DATABASE social_db
    CHARACTER SET utf8mb4
    COLLATE utf8mb4_unicode_ci;

USE social_db;

-- =========================================
-- SECTION 1: USERS
-- =========================================

CREATE TABLE users (
    user_id         BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    username        VARCHAR(50) NOT NULL UNIQUE,
    email           VARCHAR(255) NOT NULL UNIQUE,
    password_hash   VARCHAR(255) NOT NULL,
    display_name    VARCHAR(100),
    bio             TEXT,
    avatar_url      VARCHAR(500),
    cover_url       VARCHAR(500),
    website         VARCHAR(300),
    location        VARCHAR(200),
    birth_date      DATE,
    gender          ENUM('male','female','other','prefer_not_to_say'),
    is_verified     BOOLEAN DEFAULT FALSE,
    is_private      BOOLEAN DEFAULT FALSE,
    is_active       BOOLEAN DEFAULT TRUE,
    follower_count  INT UNSIGNED DEFAULT 0,
    following_count INT UNSIGNED DEFAULT 0,
    post_count      INT UNSIGNED DEFAULT 0,
    last_login_at   DATETIME,
    created_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_username (username),
    INDEX idx_email (email),
    FULLTEXT INDEX ft_user (display_name, bio)
) ENGINE=InnoDB;

-- ตาราง User Settings
CREATE TABLE user_settings (
    user_id             BIGINT UNSIGNED PRIMARY KEY,
    notify_likes        BOOLEAN DEFAULT TRUE,
    notify_comments     BOOLEAN DEFAULT TRUE,
    notify_follows      BOOLEAN DEFAULT TRUE,
    notify_mentions     BOOLEAN DEFAULT TRUE,
    notify_messages     BOOLEAN DEFAULT TRUE,
    language            VARCHAR(10) DEFAULT 'th',
    theme               ENUM('light','dark','auto') DEFAULT 'auto',
    FOREIGN KEY (user_id) REFERENCES users(user_id) ON DELETE CASCADE
) ENGINE=InnoDB;

-- =========================================
-- SECTION 2: FOLLOWS
-- =========================================

CREATE TABLE follows (
    follow_id       BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    follower_id     BIGINT UNSIGNED NOT NULL COMMENT 'คนที่กด Follow',
    following_id    BIGINT UNSIGNED NOT NULL COMMENT 'คนที่ถูก Follow',
    status          ENUM('pending','accepted') DEFAULT 'accepted',
    created_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (follower_id) REFERENCES users(user_id) ON DELETE CASCADE,
    FOREIGN KEY (following_id) REFERENCES users(user_id) ON DELETE CASCADE,
    UNIQUE KEY uk_follow (follower_id, following_id),
    INDEX idx_following (following_id),
    INDEX idx_follower (follower_id)
) ENGINE=InnoDB;

-- =========================================
-- SECTION 3: POSTS
-- =========================================

CREATE TABLE posts (
    post_id         BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    user_id         BIGINT UNSIGNED NOT NULL,
    post_type       ENUM('text','image','video','story','reel','poll','shared') DEFAULT 'text',
    content         TEXT,
    media_urls      JSON,
    thumbnail_url   VARCHAR(500),
    -- Sharing
    original_post_id BIGINT UNSIGNED,
    -- Metrics (denormalized for performance)
    like_count      INT UNSIGNED DEFAULT 0,
    comment_count   INT UNSIGNED DEFAULT 0,
    share_count     INT UNSIGNED DEFAULT 0,
    view_count      BIGINT UNSIGNED DEFAULT 0,
    save_count      INT UNSIGNED DEFAULT 0,
    -- Settings
    visibility      ENUM('public','followers','friends','private') DEFAULT 'public',
    allow_comments  BOOLEAN DEFAULT TRUE,
    -- Location
    latitude        DECIMAL(10,7),
    longitude       DECIMAL(10,7),
    location_name   VARCHAR(200),
    -- Metadata
    is_pinned       BOOLEAN DEFAULT FALSE,
    is_archived     BOOLEAN DEFAULT FALSE,
    is_deleted      BOOLEAN DEFAULT FALSE,
    created_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    FOREIGN KEY (user_id) REFERENCES users(user_id),
    FOREIGN KEY (original_post_id) REFERENCES posts(post_id) ON DELETE SET NULL,
    INDEX idx_user (user_id, created_at),
    INDEX idx_created (created_at),
    INDEX idx_visibility (visibility),
    FULLTEXT INDEX ft_content (content)
) ENGINE=InnoDB;

-- =========================================
-- SECTION 4: HASHTAGS
-- =========================================

CREATE TABLE hashtags (
    hashtag_id      INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    name            VARCHAR(100) NOT NULL UNIQUE,
    post_count      INT UNSIGNED DEFAULT 0,
    created_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_name (name)
) ENGINE=InnoDB;

CREATE TABLE post_hashtags (
    post_id         BIGINT UNSIGNED NOT NULL,
    hashtag_id      INT UNSIGNED NOT NULL,
    PRIMARY KEY (post_id, hashtag_id),
    FOREIGN KEY (post_id) REFERENCES posts(post_id) ON DELETE CASCADE,
    FOREIGN KEY (hashtag_id) REFERENCES hashtags(hashtag_id) ON DELETE CASCADE,
    INDEX idx_hashtag (hashtag_id, post_id)
) ENGINE=InnoDB;

-- =========================================
-- SECTION 5: REACTIONS & LIKES
-- =========================================

CREATE TABLE reactions (
    reaction_id     BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    user_id         BIGINT UNSIGNED NOT NULL,
    target_type     ENUM('post','comment','story') NOT NULL,
    target_id       BIGINT UNSIGNED NOT NULL,
    reaction_type   ENUM('like','love','haha','wow','sad','angry') DEFAULT 'like',
    created_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (user_id) REFERENCES users(user_id) ON DELETE CASCADE,
    UNIQUE KEY uk_reaction (user_id, target_type, target_id),
    INDEX idx_target (target_type, target_id),
    INDEX idx_user (user_id)
) ENGINE=InnoDB;

-- =========================================
-- SECTION 6: COMMENTS
-- =========================================

CREATE TABLE comments (
    comment_id      BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    post_id         BIGINT UNSIGNED NOT NULL,
    user_id         BIGINT UNSIGNED NOT NULL,
    parent_id       BIGINT UNSIGNED COMMENT 'สำหรับ Reply',
    content         TEXT NOT NULL,
    like_count      INT UNSIGNED DEFAULT 0,
    reply_count     INT UNSIGNED DEFAULT 0,
    is_deleted      BOOLEAN DEFAULT FALSE,
    created_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    FOREIGN KEY (post_id) REFERENCES posts(post_id) ON DELETE CASCADE,
    FOREIGN KEY (user_id) REFERENCES users(user_id) ON DELETE CASCADE,
    FOREIGN KEY (parent_id) REFERENCES comments(comment_id) ON DELETE SET NULL,
    INDEX idx_post (post_id, parent_id, created_at),
    INDEX idx_user (user_id)
) ENGINE=InnoDB;

-- =========================================
-- SECTION 7: MESSAGES (Direct Messages)
-- =========================================

CREATE TABLE conversations (
    conv_id         BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    conv_type       ENUM('direct','group') DEFAULT 'direct',
    name            VARCHAR(200) COMMENT 'Group chat name',
    created_by      BIGINT UNSIGNED,
    created_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (created_by) REFERENCES users(user_id) ON DELETE SET NULL
) ENGINE=InnoDB;

CREATE TABLE conversation_members (
    conv_id         BIGINT UNSIGNED NOT NULL,
    user_id         BIGINT UNSIGNED NOT NULL,
    role            ENUM('admin','member') DEFAULT 'member',
    last_read_at    DATETIME,
    joined_at       TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (conv_id, user_id),
    FOREIGN KEY (conv_id) REFERENCES conversations(conv_id) ON DELETE CASCADE,
    FOREIGN KEY (user_id) REFERENCES users(user_id) ON DELETE CASCADE,
    INDEX idx_user (user_id)
) ENGINE=InnoDB;

CREATE TABLE messages (
    message_id      BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    conv_id         BIGINT UNSIGNED NOT NULL,
    sender_id       BIGINT UNSIGNED NOT NULL,
    message_type    ENUM('text','image','video','audio','file','post_share','sticker') DEFAULT 'text',
    content         TEXT,
    media_url       VARCHAR(500),
    reply_to_id     BIGINT UNSIGNED,
    is_deleted      BOOLEAN DEFAULT FALSE,
    sent_at         TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (conv_id) REFERENCES conversations(conv_id) ON DELETE CASCADE,
    FOREIGN KEY (sender_id) REFERENCES users(user_id),
    FOREIGN KEY (reply_to_id) REFERENCES messages(message_id) ON DELETE SET NULL,
    INDEX idx_conv (conv_id, sent_at),
    INDEX idx_sender (sender_id)
) ENGINE=InnoDB;

-- =========================================
-- SECTION 8: NOTIFICATIONS
-- =========================================

CREATE TABLE notifications (
    notif_id        BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    user_id         BIGINT UNSIGNED NOT NULL COMMENT 'ผู้รับ',
    actor_id        BIGINT UNSIGNED COMMENT 'ผู้ทำ action',
    notif_type      ENUM('like','comment','follow','mention','share','reply','tag','poll_ended') NOT NULL,
    target_type     VARCHAR(20),
    target_id       BIGINT UNSIGNED,
    message         TEXT,
    is_read         BOOLEAN DEFAULT FALSE,
    created_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (user_id) REFERENCES users(user_id) ON DELETE CASCADE,
    FOREIGN KEY (actor_id) REFERENCES users(user_id) ON DELETE SET NULL,
    INDEX idx_user_unread (user_id, is_read, created_at),
    INDEX idx_created (created_at)
) ENGINE=InnoDB;

-- =========================================
-- SECTION 9: SAVED POSTS
-- =========================================

CREATE TABLE saved_posts (
    user_id         BIGINT UNSIGNED NOT NULL,
    post_id         BIGINT UNSIGNED NOT NULL,
    collection_name VARCHAR(100) DEFAULT 'All Saved',
    saved_at        TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (user_id, post_id),
    FOREIGN KEY (user_id) REFERENCES users(user_id) ON DELETE CASCADE,
    FOREIGN KEY (post_id) REFERENCES posts(post_id) ON DELETE CASCADE
) ENGINE=InnoDB;

-- =========================================
-- SECTION 10: POST VIEWS (for analytics)
-- =========================================

CREATE TABLE post_views (
    view_id         BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    post_id         BIGINT UNSIGNED NOT NULL,
    viewer_id       BIGINT UNSIGNED,
    source          ENUM('feed','explore','profile','search','hashtag','share') DEFAULT 'feed',
    duration_sec    INT UNSIGNED,
    viewed_at       TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (post_id) REFERENCES posts(post_id) ON DELETE CASCADE,
    FOREIGN KEY (viewer_id) REFERENCES users(user_id) ON DELETE SET NULL,
    INDEX idx_post (post_id, viewed_at),
    INDEX idx_viewer (viewer_id, viewed_at)
) ENGINE=InnoDB;
```

## 2. Sample Data

```sql
-- Users
INSERT INTO users (user_id, username, email, password_hash, display_name, bio, is_verified, follower_count, following_count, post_count) VALUES
(1, 'somchai_th', 'somchai@mail.com', 'hash1', 'สมชาย ใจดี', 'ชอบถ่ายรูป ท่องเที่ยว 📸', TRUE, 12500, 350, 487),
(2, 'malee_art', 'malee@mail.com', 'hash2', 'มาลี สร้างสรรค์', 'นักออกแบบกราฟิก 🎨 | Freelance', FALSE, 3200, 820, 234),
(3, 'tech_vichai', 'vichai@mail.com', 'hash3', 'วิชัย เทคโน', 'Software Engineer | Python | MySQL', FALSE, 890, 420, 156),
(4, 'foodie_nisa', 'nisa@mail.com', 'hash4', 'นิสา ฟู้ดดี้', 'รีวิวร้านอาหาร อร่อยทั่วไทย 🍜', TRUE, 45000, 200, 1200),
(5, 'travel_pat', 'pat@mail.com', 'hash5', 'ปัทมา แทรเวล', 'เดินทาง 30+ ประเทศ ✈️', TRUE, 78000, 150, 890),
(6, 'dev_art', 'artdev@mail.com', 'hash6', 'อาร์ต Dev', 'Full Stack Developer | React | Node', FALSE, 560, 310, 89),
(7, 'chef_kanya', 'kanya@mail.com', 'hash7', 'กัญญา เชฟ', 'Home Chef | สอนทำอาหาร', FALSE, 22000, 180, 567),
(8, 'fitness_ton', 'ton_fit@mail.com', 'hash8', 'ต้น ฟิตเนส', 'Personal Trainer | โค้ชสุขภาพ 💪', FALSE, 15000, 250, 430);

-- Follows
INSERT INTO follows (follower_id, following_id, status) VALUES
(1, 4, 'accepted'), (1, 5, 'accepted'), (1, 7, 'accepted'),
(2, 1, 'accepted'), (2, 4, 'accepted'), (2, 5, 'accepted'),
(3, 1, 'accepted'), (3, 6, 'accepted'), (3, 2, 'accepted'),
(4, 5, 'accepted'), (4, 7, 'accepted'),
(5, 4, 'accepted'), (5, 7, 'accepted'),
(6, 3, 'accepted'), (6, 1, 'accepted'),
(7, 4, 'accepted'), (7, 5, 'accepted'),
(8, 4, 'accepted'), (8, 7, 'accepted');

-- Hashtags
INSERT INTO hashtags (hashtag_id, name, post_count) VALUES
(1, 'อาหารไทย', 45000),
(2, 'ท่องเที่ยว', 120000),
(3, 'programming', 8900),
(4, 'ออกกำลังกาย', 34000),
(5, 'กราฟิกดีไซน์', 5600),
(6, 'รีวิวร้านอาหาร', 78000),
(7, 'เที่ยวไทย', 95000),
(8, 'สอนทำอาหาร', 12000);

-- Posts
INSERT INTO posts (post_id, user_id, post_type, content, like_count, comment_count, share_count, view_count, visibility, created_at) VALUES
(1, 4, 'image', 'ร้านข้าวมันไก่เจ้าอร่อย ถนนเยาวราช 🍗 บรรยากาศดี ราคาไม่แพง #อาหารไทย #รีวิวร้านอาหาร', 1250, 89, 45, 28000, 'public', '2024-02-01 12:00:00'),
(2, 5, 'image', 'พระอาทิตย์ตกที่เกาะสมุย งดงามมาก 🌅 #ท่องเที่ยว #เที่ยวไทย', 3400, 156, 234, 85000, 'public', '2024-02-02 18:30:00'),
(3, 3, 'text', 'เพิ่งเรียนรู้ Window Functions ใน SQL ชีวิตเปลี่ยนไปเลย! RANK(), DENSE_RANK(), LAG(), LEAD() มีประโยชน์มาก #programming', 320, 45, 78, 5600, 'public', '2024-02-03 10:00:00'),
(4, 7, 'video', 'สอนทำต้มยำกุ้งน้ำข้น สูตรโฮมเมด 🍤 #สอนทำอาหาร #อาหารไทย', 890, 67, 123, 45000, 'public', '2024-02-03 14:00:00'),
(5, 8, 'image', 'วันนี้ Workout หนักมาก 💪 Squat 120kg x 5 sets สู้ๆ #ออกกำลังกาย', 567, 34, 12, 12000, 'public', '2024-02-04 07:00:00'),
(6, 1, 'image', 'เที่ยวดอยอินทนนท์ ยอดดอยสูงสุดของไทย อากาศเย็นสบาย 🌿 #เที่ยวไทย #ท่องเที่ยว', 2100, 98, 67, 42000, 'public', '2024-02-05 09:00:00'),
(7, 4, 'image', 'Michelin Bib Gourmand ร้านก๋วยเตี๋ยวเรือ สุขุมวิท 38 🍜 ราคา 60 บาท คุ้มมาก #รีวิวร้านอาหาร', 4500, 234, 567, 95000, 'public', '2024-02-06 11:00:00'),
(8, 2, 'image', 'Branding Design โปรเจกต์ใหม่ 🎨 คอนเซ็ปต์ Nature + Modern #กราฟิกดีไซน์', 780, 56, 89, 15000, 'public', '2024-02-07 16:00:00');

-- Post Hashtags
INSERT INTO post_hashtags VALUES
(1, 1), (1, 6),
(2, 2), (2, 7),
(3, 3),
(4, 8), (4, 1),
(5, 4),
(6, 7), (6, 2),
(7, 6),
(8, 5);

-- Reactions
INSERT INTO reactions (user_id, target_type, target_id, reaction_type, created_at) VALUES
(1, 'post', 7, 'love', '2024-02-06 11:05:00'),
(2, 'post', 7, 'like', '2024-02-06 11:10:00'),
(3, 'post', 3, 'love', '2024-02-03 10:05:00'),
(4, 'post', 2, 'wow', '2024-02-02 18:35:00'),
(5, 'post', 7, 'like', '2024-02-06 12:00:00'),
(6, 'post', 3, 'like', '2024-02-03 10:15:00'),
(7, 'post', 7, 'love', '2024-02-06 13:00:00'),
(8, 'post', 5, 'like', '2024-02-04 07:10:00');

-- Comments
INSERT INTO comments (comment_id, post_id, user_id, parent_id, content, like_count, created_at) VALUES
(1, 7, 1, NULL, 'เคยไปกินแล้ว อร่อยมากครับ คิวยาวมากด้วย!', 45, '2024-02-06 11:20:00'),
(2, 7, 2, NULL, 'ราคา 60 บาทเท่านั้นเหรอ? ต้องไปลอง', 23, '2024-02-06 11:30:00'),
(3, 7, 1, 2, 'ใช่เลยครับ คุ้มมากๆ', 12, '2024-02-06 11:35:00'),
(4, 3, 6, NULL, 'เห็นด้วย! Window Functions เปลี่ยนวิธีคิดเลย', 18, '2024-02-03 10:20:00'),
(5, 3, 3, 4, 'ใช่เลย ROW_NUMBER() กับ PARTITION BY ทำให้ชีวิตง่ายขึ้นมาก', 25, '2024-02-03 10:30:00');

-- Notifications
INSERT INTO notifications (user_id, actor_id, notif_type, target_type, target_id, message, created_at) VALUES
(4, 1, 'like', 'post', 7, 'สมชาย ใจดี กด Like โพสต์ของคุณ', '2024-02-06 11:05:00'),
(4, 2, 'comment', 'post', 7, 'มาลี สร้างสรรค์ คอมเมนต์: "ราคา 60 บาทเท่านั้นเหรอ?"', '2024-02-06 11:30:00'),
(3, 6, 'comment', 'post', 3, 'อาร์ต Dev คอมเมนต์: "เห็นด้วย! Window Functions..."', '2024-02-03 10:20:00'),
(1, 2, 'follow', NULL, 2, 'มาลี สร้างสรรค์ เริ่มติดตามคุณ', '2024-01-15 09:00:00'),
(1, 3, 'follow', NULL, 3, 'วิชัย เทคโน เริ่มติดตามคุณ', '2024-01-20 14:00:00');
```

## 3. Social Media Queries

### Query 1: News Feed Algorithm

```sql
-- News Feed: Posts จากคนที่ Follow เรียงตาม Engagement Score
WITH following_posts AS (
    SELECT 
        p.*,
        u.display_name AS author_name,
        u.avatar_url,
        u.is_verified,
        -- Engagement Score สำหรับ Ranking
        (
            p.like_count * 1.0 +
            p.comment_count * 2.0 +
            p.share_count * 3.0 +
            -- Recency decay: คะแนนลดลงตามเวลา
            EXP(-0.1 * TIMESTAMPDIFF(HOUR, p.created_at, NOW())) * 100
        ) AS engagement_score,
        -- User interaction
        COALESCE(r.reaction_type, NULL) AS my_reaction,
        CASE WHEN sp.user_id IS NOT NULL THEN TRUE ELSE FALSE END AS is_saved
    FROM follows f
    JOIN posts p ON f.following_id = p.user_id
    JOIN users u ON p.user_id = u.user_id
    LEFT JOIN reactions r ON p.post_id = r.target_id 
        AND r.target_type = 'post' AND r.user_id = 1  -- Current user = 1
    LEFT JOIN saved_posts sp ON p.post_id = sp.post_id AND sp.user_id = 1
    WHERE f.follower_id = 1  -- Current user = 1
      AND p.visibility IN ('public', 'followers')
      AND p.is_deleted = FALSE
      AND p.is_archived = FALSE
)
SELECT 
    post_id,
    author_name,
    is_verified,
    post_type,
    LEFT(content, 200) AS content_preview,
    like_count,
    comment_count,
    share_count,
    view_count,
    my_reaction,
    is_saved,
    created_at,
    ROUND(engagement_score, 2) AS score
FROM following_posts
ORDER BY engagement_score DESC
LIMIT 20;
```

---

### Query 2: Trending Hashtags

```sql
-- Trending Hashtags ในช่วง 24 ชั่วโมงที่ผ่านมา
WITH recent_activity AS (
    SELECT 
        ph.hashtag_id,
        COUNT(DISTINCT ph.post_id) AS recent_posts,
        SUM(p.like_count + p.comment_count * 2 + p.share_count * 3) AS engagement_sum,
        SUM(p.view_count) AS total_views
    FROM post_hashtags ph
    JOIN posts p ON ph.post_id = p.post_id
    WHERE p.created_at >= DATE_SUB(NOW(), INTERVAL 24 HOUR)
      AND p.is_deleted = FALSE
      AND p.visibility = 'public'
    GROUP BY ph.hashtag_id
),
previous_day AS (
    SELECT 
        ph.hashtag_id,
        COUNT(DISTINCT ph.post_id) AS prev_posts
    FROM post_hashtags ph
    JOIN posts p ON ph.post_id = p.post_id
    WHERE p.created_at BETWEEN DATE_SUB(NOW(), INTERVAL 48 HOUR) 
                            AND DATE_SUB(NOW(), INTERVAL 24 HOUR)
    GROUP BY ph.hashtag_id
)
SELECT 
    h.name AS hashtag,
    h.post_count AS total_post_count,
    COALESCE(ra.recent_posts, 0) AS posts_last_24h,
    COALESCE(ra.engagement_sum, 0) AS engagement_24h,
    COALESCE(ra.total_views, 0) AS views_24h,
    COALESCE(pd.prev_posts, 0) AS posts_prev_24h,
    -- Trend Score = recent activity + velocity
    ROUND(
        COALESCE(ra.recent_posts, 0) * 10 +
        COALESCE(ra.engagement_sum, 0) * 0.01 +
        -- Velocity: growth rate vs previous period
        (COALESCE(ra.recent_posts, 0) - COALESCE(pd.prev_posts, 0)) * 5,
        2
    ) AS trend_score,
    CASE 
        WHEN COALESCE(pd.prev_posts, 0) = 0 THEN 'NEW'
        WHEN ra.recent_posts > pd.prev_posts * 2 THEN '🔥 VIRAL'
        WHEN ra.recent_posts > pd.prev_posts THEN '📈 RISING'
        ELSE '📊 STEADY'
    END AS trend_status
FROM hashtags h
LEFT JOIN recent_activity ra ON h.hashtag_id = ra.hashtag_id
LEFT JOIN previous_day pd ON h.hashtag_id = pd.hashtag_id
WHERE COALESCE(ra.recent_posts, 0) > 0
ORDER BY trend_score DESC
LIMIT 10;
```

---

### Query 3: User Analytics Dashboard

```sql
-- Analytics สำหรับ Creator Dashboard (User 4 = foodie_nisa)
WITH user_posts AS (
    SELECT 
        post_id,
        created_at,
        like_count,
        comment_count,
        share_count,
        view_count,
        WEEK(created_at) AS week_num,
        MONTH(created_at) AS month_num
    FROM posts
    WHERE user_id = 4 AND is_deleted = FALSE
),
monthly_stats AS (
    SELECT 
        DATE_FORMAT(created_at, '%Y-%m') AS month,
        COUNT(*) AS posts_count,
        SUM(like_count) AS total_likes,
        SUM(comment_count) AS total_comments,
        SUM(share_count) AS total_shares,
        SUM(view_count) AS total_views,
        ROUND(AVG(like_count), 1) AS avg_likes_per_post,
        -- Engagement Rate = (Likes + Comments + Shares) / Views * 100
        ROUND(
            SUM(like_count + comment_count + share_count) * 100.0 / 
            NULLIF(SUM(view_count), 0), 
            2
        ) AS engagement_rate_pct
    FROM user_posts
    GROUP BY DATE_FORMAT(created_at, '%Y-%m')
),
top_posts AS (
    SELECT 
        post_id,
        LEFT(p.content, 100) AS content_preview,
        like_count + comment_count * 2 + share_count * 3 AS performance_score,
        RANK() OVER (ORDER BY like_count + comment_count * 2 + share_count * 3 DESC) AS rank_pos
    FROM posts p
    WHERE user_id = 4 AND is_deleted = FALSE
)
SELECT 
    ms.*,
    LAG(ms.total_views) OVER (ORDER BY ms.month) AS prev_month_views,
    ROUND(
        (ms.total_views - LAG(ms.total_views) OVER (ORDER BY ms.month)) 
        * 100.0 / NULLIF(LAG(ms.total_views) OVER (ORDER BY ms.month), 0),
        1
    ) AS view_growth_pct
FROM monthly_stats ms
ORDER BY ms.month DESC;
```

---

### Query 4: Follower Network Analysis

```sql
-- วิเคราะห์ Network ของ User: Mutual Follows, Suggested Follows
WITH my_following AS (
    SELECT following_id 
    FROM follows 
    WHERE follower_id = 1 AND status = 'accepted'
),
my_followers AS (
    SELECT follower_id 
    FROM follows 
    WHERE following_id = 1 AND status = 'accepted'
),
-- Friends of Friends ที่ยังไม่ได้ Follow
fof AS (
    SELECT DISTINCT 
        f2.following_id AS suggested_user_id,
        COUNT(DISTINCT f1.following_id) AS mutual_count
    FROM my_following f1
    JOIN follows f2 ON f1.following_id = f2.follower_id
        AND f2.status = 'accepted'
    WHERE f2.following_id NOT IN (SELECT following_id FROM my_following)
      AND f2.following_id != 1
    GROUP BY f2.following_id
    ORDER BY mutual_count DESC
    LIMIT 10
)
SELECT 
    u.user_id,
    u.username,
    u.display_name,
    u.is_verified,
    u.follower_count,
    u.bio,
    fof.mutual_count AS mutual_connections,
    -- Already follower but not following back
    CASE WHEN u.user_id IN (SELECT follower_id FROM my_followers) 
         THEN TRUE ELSE FALSE END AS follows_you,
    'friend_of_friend' AS suggestion_reason
FROM fof
JOIN users u ON fof.suggested_user_id = u.user_id
WHERE u.is_active = TRUE
ORDER BY fof.mutual_count DESC;
```

---

### Query 5: Comment Thread with Nested Replies

```sql
-- ดึง Comments พร้อม Nested Replies (2 levels deep)
SELECT 
    c.comment_id,
    c.post_id,
    c.parent_id,
    c.content,
    c.like_count,
    c.reply_count,
    c.created_at,
    u.user_id,
    u.username,
    u.display_name,
    u.avatar_url,
    u.is_verified,
    -- Nested level
    CASE WHEN c.parent_id IS NULL THEN 0 ELSE 1 END AS depth,
    -- For ordering: top-level first, then replies under their parent
    COALESCE(c.parent_id, c.comment_id) AS thread_root,
    c.comment_id AS sort_key
FROM comments c
JOIN users u ON c.user_id = u.user_id
WHERE c.post_id = 7
  AND c.is_deleted = FALSE
ORDER BY thread_root ASC, depth ASC, c.created_at ASC;
```

---

### Query 6: Viral Post Detection

```sql
-- ตรวจหาโพสต์ที่กำลัง Viral (Engagement velocity สูง)
WITH hourly_engagement AS (
    SELECT 
        r.target_id AS post_id,
        COUNT(*) AS reactions_last_hour,
        HOUR(NOW()) AS current_hour
    FROM reactions r
    WHERE r.target_type = 'post'
      AND r.created_at >= DATE_SUB(NOW(), INTERVAL 1 HOUR)
    GROUP BY r.target_id
),
comment_velocity AS (
    SELECT 
        post_id,
        COUNT(*) AS comments_last_hour
    FROM comments
    WHERE created_at >= DATE_SUB(NOW(), INTERVAL 1 HOUR)
      AND is_deleted = FALSE
    GROUP BY post_id
)
SELECT 
    p.post_id,
    u.display_name AS author,
    LEFT(p.content, 150) AS content_preview,
    p.like_count,
    p.comment_count,
    p.share_count,
    p.view_count,
    COALESCE(he.reactions_last_hour, 0) AS reactions_last_1h,
    COALESCE(cv.comments_last_hour, 0) AS comments_last_1h,
    -- Velocity Score
    COALESCE(he.reactions_last_hour, 0) * 1 + 
    COALESCE(cv.comments_last_hour, 0) * 2 AS velocity_score,
    p.created_at,
    TIMESTAMPDIFF(HOUR, p.created_at, NOW()) AS hours_old
FROM posts p
JOIN users u ON p.user_id = u.user_id
LEFT JOIN hourly_engagement he ON p.post_id = he.post_id
LEFT JOIN comment_velocity cv ON p.post_id = cv.post_id
WHERE p.visibility = 'public'
  AND p.is_deleted = FALSE
  AND p.created_at >= DATE_SUB(NOW(), INTERVAL 48 HOUR)
  AND (COALESCE(he.reactions_last_hour, 0) + COALESCE(cv.comments_last_hour, 0)) > 0
ORDER BY velocity_score DESC
LIMIT 20;
```

---

### Query 7: User Engagement Score

```sql
-- คำนวณ User Engagement Score (เหมาะสำหรับ Account Health)
WITH activity_30d AS (
    SELECT 
        u.user_id,
        u.username,
        u.display_name,
        u.follower_count,
        u.post_count,
        -- Post activity
        COUNT(DISTINCT p.post_id) AS posts_30d,
        -- Received engagement
        SUM(p.like_count) AS total_likes_received,
        SUM(p.comment_count) AS total_comments_received,
        -- Given engagement
        COUNT(DISTINCT r.reaction_id) AS reactions_given_30d,
        COUNT(DISTINCT c.comment_id) AS comments_given_30d
    FROM users u
    LEFT JOIN posts p ON u.user_id = p.user_id 
        AND p.created_at >= DATE_SUB(NOW(), INTERVAL 30 DAY)
        AND p.is_deleted = FALSE
    LEFT JOIN reactions r ON u.user_id = r.user_id 
        AND r.created_at >= DATE_SUB(NOW(), INTERVAL 30 DAY)
    LEFT JOIN comments c ON u.user_id = c.user_id 
        AND c.created_at >= DATE_SUB(NOW(), INTERVAL 30 DAY)
        AND c.is_deleted = FALSE
    WHERE u.is_active = TRUE
    GROUP BY u.user_id, u.username, u.display_name, u.follower_count, u.post_count
)
SELECT 
    user_id,
    username,
    display_name,
    follower_count,
    posts_30d,
    total_likes_received,
    total_comments_received,
    reactions_given_30d,
    comments_given_30d,
    -- Engagement Rate = total engagement / followers
    ROUND(
        (total_likes_received + total_comments_received) * 100.0 / 
        NULLIF(follower_count, 0),
        2
    ) AS engagement_rate_pct,
    -- Overall Score (0-100)
    LEAST(100, ROUND(
        (posts_30d * 5) +
        (total_likes_received * 0.1) +
        (total_comments_received * 0.3) +
        (reactions_given_30d * 0.5) +
        (comments_given_30d * 1.0),
        0
    )) AS engagement_score
FROM activity_30d
ORDER BY engagement_score DESC;
```

---

### Query 8: Unread Notification Count by Type

```sql
-- นับ Unread Notifications แยกตาม Type
SELECT 
    u.user_id,
    u.display_name,
    -- Total unread
    SUM(CASE WHEN n.is_read = FALSE THEN 1 ELSE 0 END) AS total_unread,
    -- By type
    SUM(CASE WHEN n.notif_type = 'like' AND NOT n.is_read THEN 1 ELSE 0 END) AS likes,
    SUM(CASE WHEN n.notif_type = 'comment' AND NOT n.is_read THEN 1 ELSE 0 END) AS comments,
    SUM(CASE WHEN n.notif_type = 'follow' AND NOT n.is_read THEN 1 ELSE 0 END) AS follows,
    SUM(CASE WHEN n.notif_type = 'mention' AND NOT n.is_read THEN 1 ELSE 0 END) AS mentions,
    SUM(CASE WHEN n.notif_type = 'share' AND NOT n.is_read THEN 1 ELSE 0 END) AS shares,
    -- Latest notification time
    MAX(n.created_at) AS latest_at
FROM users u
LEFT JOIN notifications n ON u.user_id = n.user_id
WHERE u.user_id = 4  -- Current user
GROUP BY u.user_id, u.display_name;
```

---

### Query 9: Content Performance by Hour of Day

```sql
-- วิเคราะห์ว่า โพสต์ชั่วโมงไหน ได้ Engagement มากที่สุด
SELECT 
    HOUR(p.created_at) AS hour_of_day,
    COUNT(*) AS post_count,
    ROUND(AVG(p.like_count), 1) AS avg_likes,
    ROUND(AVG(p.comment_count), 1) AS avg_comments,
    ROUND(AVG(p.share_count), 1) AS avg_shares,
    ROUND(AVG(p.view_count), 0) AS avg_views,
    ROUND(AVG(p.like_count + p.comment_count + p.share_count), 1) AS avg_total_engagement,
    -- Best time indicator
    RANK() OVER (ORDER BY AVG(p.like_count + p.comment_count + p.share_count) DESC) AS rank_pos,
    CONCAT(LPAD(HOUR(p.created_at), 2, '0'), ':00 - ', LPAD(HOUR(p.created_at)+1, 2, '0'), ':00') AS time_slot
FROM posts p
WHERE p.user_id = 4
  AND p.is_deleted = FALSE
  AND p.created_at >= DATE_SUB(NOW(), INTERVAL 90 DAY)
GROUP BY HOUR(p.created_at)
ORDER BY avg_total_engagement DESC;
```

---

### Query 10: Mutual Followers and Common Interests

```sql
-- หา Common Interests ระหว่าง Users 2 คน
WITH user1_hashtags AS (
    SELECT ph.hashtag_id, COUNT(*) AS usage_count
    FROM posts p
    JOIN post_hashtags ph ON p.post_id = ph.post_id
    WHERE p.user_id = 1
    GROUP BY ph.hashtag_id
),
user2_hashtags AS (
    SELECT ph.hashtag_id, COUNT(*) AS usage_count
    FROM posts p
    JOIN post_hashtags ph ON p.post_id = ph.post_id
    WHERE p.user_id = 4
    GROUP BY ph.hashtag_id
)
SELECT 
    h.name AS hashtag,
    u1h.usage_count AS user1_usage,
    u2h.usage_count AS user2_usage,
    (u1h.usage_count + u2h.usage_count) AS combined_usage
FROM user1_hashtags u1h
JOIN user2_hashtags u2h ON u1h.hashtag_id = u2h.hashtag_id
JOIN hashtags h ON u1h.hashtag_id = h.hashtag_id
ORDER BY combined_usage DESC;
```

---

## แบบฝึกหัด (Challenge Exercises)

1. **Story Views**: ออกแบบตาราง Stories (content หายใน 24 ชั่วโมง) และเขียน Query ดึง Stories ที่ยังไม่หมดอายุของคนที่เรา Follow

2. **Explore Page**: เขียน Query สร้าง Explore Feed สำหรับ User ที่ยังไม่ Follow ใคร โดยแนะนำ Posts จาก Popular Hashtags

3. **Block/Mute Feature**: เพิ่มตาราง `user_blocks` และ `user_mutes` แล้วเขียน Query ที่ Filter Posts ออกจาก Feed ถ้า User ถูก Block/Mute

4. **Poll Post**: ออกแบบตาราง `post_polls` และ `poll_votes` แล้วเขียน Query ดึงผล Poll พร้อม % ของแต่ละตัวเลือก

5. **Reels Recommendation**: เขียน Recommendation Query สำหรับ Short Videos โดยใช้ Collaborative Filtering แบบง่าย (Users ที่ Like Reels เดียวกัน ชอบ Reels อะไรอีก?)

6. **Spam Detection**: เขียน Query ตรวจหา Accounts ที่น่าสงสัยว่าเป็น Spam (Post บ่อยผิดปกติ, Follow rate สูง, Engagement ต่ำ)

7. **Influencer Tier**: จัด Tier Influencer (Mega/Macro/Micro/Nano) จาก Follower Count และ Engagement Rate

8. **Conversation Inbox**: เขียน Query ดึง Conversation List สำหรับ Inbox โดยแสดง Last Message, Unread Count และ Online Status

9. **Hashtag Analytics**: สร้าง Hashtag Detail Page Query แสดง: Top Posts, Top Users ที่ใช้, Timeline ของการใช้งานช่วง 30 วัน

10. **Content Moderation**: เขียน Query ช่วย Flag โพสต์ที่มีการ Report มากผิดปกติ หรือ Engagement Pattern ผิดปกติ (Like ทะลุในเวลาสั้น)
