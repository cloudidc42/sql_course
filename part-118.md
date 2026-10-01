# Part 118: Learning Management System (LMS)

## บทนำ (Introduction)

Learning Management System (LMS) เป็นระบบที่ใช้จัดการคอร์สออนไลน์ครบวงจร ตั้งแต่การสร้างบทเรียน ติดตามความก้าวหน้า ทำแบบทดสอบ ออกใบรับรอง ไปจนถึงระบบ Gamification เพื่อเพิ่มแรงจูงใจ

## 1. Complete LMS Schema

```sql
-- =========================================
-- LEARNING MANAGEMENT SYSTEM - COMPLETE DDL
-- =========================================

CREATE DATABASE lms_db
    CHARACTER SET utf8mb4
    COLLATE utf8mb4_unicode_ci;

USE lms_db;

-- =========================================
-- SECTION 1: USERS AND ROLES
-- =========================================

CREATE TABLE users (
    user_id         INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    username        VARCHAR(50) NOT NULL UNIQUE,
    email           VARCHAR(255) NOT NULL UNIQUE,
    password_hash   VARCHAR(255) NOT NULL,
    first_name      VARCHAR(100),
    last_name       VARCHAR(100),
    full_name       VARCHAR(200) GENERATED ALWAYS AS (CONCAT(first_name, ' ', last_name)) STORED,
    role            ENUM('student','instructor','admin') DEFAULT 'student',
    avatar_url      VARCHAR(500),
    bio             TEXT,
    is_active       BOOLEAN DEFAULT TRUE,
    created_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    last_login      DATETIME
) ENGINE=InnoDB;

-- =========================================
-- SECTION 2: COURSES
-- =========================================

CREATE TABLE categories (
    category_id     INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    parent_id       INT UNSIGNED,
    name            VARCHAR(200) NOT NULL,
    slug            VARCHAR(200) UNIQUE,
    FOREIGN KEY (parent_id) REFERENCES categories(category_id)
) ENGINE=InnoDB;

CREATE TABLE courses (
    course_id       INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    instructor_id   INT UNSIGNED NOT NULL,
    category_id     INT UNSIGNED,
    title           VARCHAR(300) NOT NULL,
    slug            VARCHAR(300) UNIQUE,
    description     TEXT,
    objectives      JSON COMMENT 'Array of learning objectives',
    requirements    JSON COMMENT 'Array of prerequisites',
    level           ENUM('beginner','intermediate','advanced','all') DEFAULT 'beginner',
    language        VARCHAR(10) DEFAULT 'th',
    thumbnail_url   VARCHAR(500),
    intro_video_url VARCHAR(500),
    price           DECIMAL(10,2) DEFAULT 0,
    original_price  DECIMAL(10,2),
    discount_pct    DECIMAL(5,2) GENERATED ALWAYS AS (
        CASE WHEN original_price > 0 
             THEN ROUND((1 - price/original_price) * 100, 1) 
             ELSE 0 END
    ) STORED,
    duration_minutes INT UNSIGNED DEFAULT 0,
    total_lessons   INT UNSIGNED DEFAULT 0,
    total_students  INT UNSIGNED DEFAULT 0,
    rating          DECIMAL(3,2) DEFAULT 0,
    rating_count    INT UNSIGNED DEFAULT 0,
    status          ENUM('draft','published','archived') DEFAULT 'draft',
    is_free         BOOLEAN GENERATED ALWAYS AS (price = 0) STORED,
    has_certificate BOOLEAN DEFAULT TRUE,
    created_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    published_at    DATETIME,
    FOREIGN KEY (instructor_id) REFERENCES users(user_id),
    FOREIGN KEY (category_id) REFERENCES categories(category_id) ON DELETE SET NULL,
    INDEX idx_instructor (instructor_id),
    INDEX idx_category (category_id),
    INDEX idx_status (status),
    FULLTEXT INDEX ft_course (title, description)
) ENGINE=InnoDB;

-- =========================================
-- SECTION 3: CURRICULUM
-- =========================================

CREATE TABLE sections (
    section_id      INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    course_id       INT UNSIGNED NOT NULL,
    title           VARCHAR(300) NOT NULL,
    description     TEXT,
    sort_order      INT DEFAULT 0,
    FOREIGN KEY (course_id) REFERENCES courses(course_id) ON DELETE CASCADE,
    INDEX idx_course_order (course_id, sort_order)
) ENGINE=InnoDB;

CREATE TABLE lessons (
    lesson_id       INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    section_id      INT UNSIGNED NOT NULL,
    course_id       INT UNSIGNED NOT NULL,
    title           VARCHAR(300) NOT NULL,
    lesson_type     ENUM('video','article','quiz','assignment','live','resource') DEFAULT 'video',
    content         LONGTEXT COMMENT 'Article content or video script',
    video_url       VARCHAR(500),
    video_duration_sec INT UNSIGNED DEFAULT 0,
    duration_text   VARCHAR(20) GENERATED ALWAYS AS (
        CONCAT(FLOOR(video_duration_sec / 60), ':', LPAD(MOD(video_duration_sec, 60), 2, '0'))
    ) STORED,
    sort_order      INT DEFAULT 0,
    is_preview      BOOLEAN DEFAULT FALSE COMMENT 'Free preview without enrollment',
    is_published    BOOLEAN DEFAULT TRUE,
    resources       JSON COMMENT 'Downloadable files',
    FOREIGN KEY (section_id) REFERENCES sections(section_id) ON DELETE CASCADE,
    FOREIGN KEY (course_id) REFERENCES courses(course_id) ON DELETE CASCADE,
    INDEX idx_section_order (section_id, sort_order),
    INDEX idx_course (course_id)
) ENGINE=InnoDB;

-- =========================================
-- SECTION 4: ENROLLMENTS
-- =========================================

CREATE TABLE enrollments (
    enrollment_id   INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    user_id         INT UNSIGNED NOT NULL,
    course_id       INT UNSIGNED NOT NULL,
    status          ENUM('active','completed','expired','cancelled') DEFAULT 'active',
    progress_pct    DECIMAL(5,2) DEFAULT 0,
    lessons_completed INT UNSIGNED DEFAULT 0,
    total_time_sec  INT UNSIGNED DEFAULT 0 COMMENT 'Total watch time',
    enrolled_at     TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    completed_at    DATETIME,
    last_accessed   DATETIME,
    payment_amount  DECIMAL(10,2) DEFAULT 0,
    payment_ref     VARCHAR(100),
    expiry_date     DATE,
    FOREIGN KEY (user_id) REFERENCES users(user_id),
    FOREIGN KEY (course_id) REFERENCES courses(course_id),
    UNIQUE KEY uk_enrollment (user_id, course_id),
    INDEX idx_user (user_id),
    INDEX idx_course (course_id),
    INDEX idx_status (status)
) ENGINE=InnoDB;

-- =========================================
-- SECTION 5: LESSON PROGRESS
-- =========================================

CREATE TABLE lesson_progress (
    progress_id     INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    enrollment_id   INT UNSIGNED NOT NULL,
    lesson_id       INT UNSIGNED NOT NULL,
    user_id         INT UNSIGNED NOT NULL,
    status          ENUM('not_started','in_progress','completed') DEFAULT 'not_started',
    watch_time_sec  INT UNSIGNED DEFAULT 0 COMMENT 'Total seconds watched',
    last_position_sec INT UNSIGNED DEFAULT 0 COMMENT 'Resume from this position',
    completion_pct  DECIMAL(5,2) DEFAULT 0,
    started_at      DATETIME,
    completed_at    DATETIME,
    FOREIGN KEY (enrollment_id) REFERENCES enrollments(enrollment_id) ON DELETE CASCADE,
    FOREIGN KEY (lesson_id) REFERENCES lessons(lesson_id),
    FOREIGN KEY (user_id) REFERENCES users(user_id),
    UNIQUE KEY uk_progress (enrollment_id, lesson_id),
    INDEX idx_user_lesson (user_id, lesson_id)
) ENGINE=InnoDB;

-- =========================================
-- SECTION 6: QUIZZES
-- =========================================

CREATE TABLE quizzes (
    quiz_id         INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    lesson_id       INT UNSIGNED NOT NULL,
    title           VARCHAR(300) NOT NULL,
    description     TEXT,
    time_limit_min  INT COMMENT 'NULL = no limit',
    passing_score   DECIMAL(5,2) DEFAULT 70,
    max_attempts    INT DEFAULT 3,
    shuffle_questions BOOLEAN DEFAULT TRUE,
    FOREIGN KEY (lesson_id) REFERENCES lessons(lesson_id) ON DELETE CASCADE
) ENGINE=InnoDB;

CREATE TABLE quiz_questions (
    question_id     INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    quiz_id         INT UNSIGNED NOT NULL,
    question_text   TEXT NOT NULL,
    question_type   ENUM('single_choice','multiple_choice','true_false','short_answer') DEFAULT 'single_choice',
    points          DECIMAL(5,2) DEFAULT 1,
    explanation     TEXT COMMENT 'Shown after answering',
    sort_order      INT DEFAULT 0,
    FOREIGN KEY (quiz_id) REFERENCES quizzes(quiz_id) ON DELETE CASCADE
) ENGINE=InnoDB;

CREATE TABLE question_options (
    option_id       INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    question_id     INT UNSIGNED NOT NULL,
    option_text     TEXT NOT NULL,
    is_correct      BOOLEAN DEFAULT FALSE,
    sort_order      INT DEFAULT 0,
    FOREIGN KEY (question_id) REFERENCES quiz_questions(question_id) ON DELETE CASCADE
) ENGINE=InnoDB;

CREATE TABLE quiz_attempts (
    attempt_id      INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    quiz_id         INT UNSIGNED NOT NULL,
    user_id         INT UNSIGNED NOT NULL,
    enrollment_id   INT UNSIGNED NOT NULL,
    attempt_number  INT DEFAULT 1,
    score           DECIMAL(5,2),
    max_score       DECIMAL(5,2),
    score_pct       DECIMAL(5,2),
    is_passed       BOOLEAN,
    time_taken_sec  INT UNSIGNED,
    started_at      DATETIME NOT NULL,
    submitted_at    DATETIME,
    FOREIGN KEY (quiz_id) REFERENCES quizzes(quiz_id),
    FOREIGN KEY (user_id) REFERENCES users(user_id),
    FOREIGN KEY (enrollment_id) REFERENCES enrollments(enrollment_id),
    INDEX idx_user_quiz (user_id, quiz_id)
) ENGINE=InnoDB;

CREATE TABLE quiz_answers (
    answer_id       INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    attempt_id      INT UNSIGNED NOT NULL,
    question_id     INT UNSIGNED NOT NULL,
    selected_option_ids JSON,
    text_answer     TEXT,
    is_correct      BOOLEAN,
    points_earned   DECIMAL(5,2) DEFAULT 0,
    FOREIGN KEY (attempt_id) REFERENCES quiz_attempts(attempt_id) ON DELETE CASCADE,
    FOREIGN KEY (question_id) REFERENCES quiz_questions(question_id)
) ENGINE=InnoDB;

-- =========================================
-- SECTION 7: CERTIFICATES
-- =========================================

CREATE TABLE certificates (
    cert_id         INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    user_id         INT UNSIGNED NOT NULL,
    course_id       INT UNSIGNED NOT NULL,
    enrollment_id   INT UNSIGNED NOT NULL,
    cert_number     VARCHAR(50) NOT NULL UNIQUE,
    issued_at       TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    expiry_date     DATE,
    FOREIGN KEY (user_id) REFERENCES users(user_id),
    FOREIGN KEY (course_id) REFERENCES courses(course_id),
    FOREIGN KEY (enrollment_id) REFERENCES enrollments(enrollment_id),
    INDEX idx_user (user_id),
    INDEX idx_cert_number (cert_number)
) ENGINE=InnoDB;

-- =========================================
-- SECTION 8: REVIEWS & RATINGS
-- =========================================

CREATE TABLE course_reviews (
    review_id       INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    course_id       INT UNSIGNED NOT NULL,
    user_id         INT UNSIGNED NOT NULL,
    rating          TINYINT NOT NULL CHECK (rating BETWEEN 1 AND 5),
    review_text     TEXT,
    is_visible      BOOLEAN DEFAULT TRUE,
    created_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (course_id) REFERENCES courses(course_id),
    FOREIGN KEY (user_id) REFERENCES users(user_id),
    UNIQUE KEY uk_review (course_id, user_id),
    INDEX idx_course (course_id, rating)
) ENGINE=InnoDB;

-- =========================================
-- SECTION 9: GAMIFICATION
-- =========================================

CREATE TABLE badges (
    badge_id        INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    name            VARCHAR(100) NOT NULL,
    description     TEXT,
    icon_url        VARCHAR(300),
    badge_type      ENUM('completion','streak','excellence','engagement','special') DEFAULT 'completion',
    criteria        JSON COMMENT 'Conditions to earn this badge'
) ENGINE=InnoDB;

CREATE TABLE user_badges (
    user_id         INT UNSIGNED NOT NULL,
    badge_id        INT UNSIGNED NOT NULL,
    earned_at       TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    course_id       INT UNSIGNED,
    PRIMARY KEY (user_id, badge_id),
    FOREIGN KEY (user_id) REFERENCES users(user_id),
    FOREIGN KEY (badge_id) REFERENCES badges(badge_id),
    FOREIGN KEY (course_id) REFERENCES courses(course_id) ON DELETE SET NULL
) ENGINE=InnoDB;

CREATE TABLE learning_streaks (
    user_id         INT UNSIGNED PRIMARY KEY,
    current_streak_days INT DEFAULT 0,
    longest_streak_days INT DEFAULT 0,
    last_activity_date DATE,
    total_days_active INT DEFAULT 0,
    FOREIGN KEY (user_id) REFERENCES users(user_id)
) ENGINE=InnoDB;
```

## 2. Sample Data

```sql
-- Users
INSERT INTO users (user_id, username, email, password_hash, first_name, last_name, role) VALUES
(1, 'instructor_somchai', 'somchai@lms.com', 'hash1', 'สมชาย', 'ใจดี', 'instructor'),
(2, 'instructor_malee', 'malee@lms.com', 'hash2', 'มาลี', 'เก่งกาจ', 'instructor'),
(3, 'student_vichai', 'vichai@lms.com', 'hash3', 'วิชัย', 'เรียนดี', 'student'),
(4, 'student_nisa', 'nisa@lms.com', 'hash4', 'นิสา', 'ขยัน', 'student'),
(5, 'student_pat', 'pat@lms.com', 'hash5', 'ปัทมา', 'เก่ง', 'student'),
(6, 'student_art', 'art@lms.com', 'hash6', 'อาร์ต', 'มุ่งมั่น', 'student');

-- Categories
INSERT INTO categories (category_id, parent_id, name, slug) VALUES
(1, NULL, 'Technology', 'technology'),
(2, NULL, 'Business', 'business'),
(3, 1, 'Programming', 'programming'),
(4, 1, 'Database', 'database'),
(5, 2, 'Marketing', 'marketing');

-- Courses
INSERT INTO courses (course_id, instructor_id, category_id, title, description, level, price, original_price, duration_minutes, total_lessons, total_students, rating, rating_count, status, published_at) VALUES
(1, 1, 4, 'MySQL Masterclass: ตั้งแต่พื้นฐานถึงขั้นสูง', 'เรียนรู้ MySQL ครบทุกด้าน ตั้งแต่พื้นฐาน SELECT ไปถึง Window Functions, Stored Procedures, Performance Tuning', 'beginner', 790, 1290, 1800, 45, 1234, 4.8, 956, 'published', '2024-01-01'),
(2, 1, 3, 'Python สำหรับนักวิเคราะห์ข้อมูล', 'Python, Pandas, NumPy, Matplotlib สำหรับ Data Analysis', 'intermediate', 990, 1490, 2400, 60, 876, 4.7, 654, 'published', '2024-01-15'),
(3, 2, 5, 'Digital Marketing ครบจบในคอร์สเดียว', 'SEO, SEM, Social Media Marketing, Email Marketing, Analytics', 'beginner', 690, 990, 1500, 35, 2341, 4.6, 1876, 'published', '2024-01-20'),
(4, 2, 3, 'React.js & Node.js Full Stack', 'พัฒนา Web Application แบบ Full Stack ด้วย React และ Node.js', 'intermediate', 1290, 1990, 3600, 80, 543, 4.9, 430, 'published', '2024-02-01');

-- Sections for Course 1
INSERT INTO sections (section_id, course_id, title, sort_order) VALUES
(1, 1, 'บทที่ 1: Introduction to MySQL', 1),
(2, 1, 'บทที่ 2: SELECT และ Filtering', 2),
(3, 1, 'บทที่ 3: Joins', 3),
(4, 1, 'บทที่ 4: Window Functions', 4);

-- Lessons
INSERT INTO lessons (lesson_id, section_id, course_id, title, lesson_type, video_duration_sec, sort_order, is_preview) VALUES
(1, 1, 1, 'แนะนำ MySQL และการติดตั้ง', 'video', 900, 1, TRUE),
(2, 1, 1, 'CREATE DATABASE และ TABLE', 'video', 1200, 2, FALSE),
(3, 2, 1, 'SELECT พื้นฐาน', 'video', 1500, 1, FALSE),
(4, 2, 1, 'WHERE, ORDER BY, LIMIT', 'video', 1800, 2, FALSE),
(5, 2, 1, 'แบบทดสอบ: SELECT พื้นฐาน', 'quiz', 0, 3, FALSE),
(6, 3, 1, 'INNER JOIN', 'video', 2100, 1, FALSE),
(7, 3, 1, 'LEFT/RIGHT JOIN', 'video', 1800, 2, FALSE),
(8, 4, 1, 'ROW_NUMBER, RANK, DENSE_RANK', 'video', 2400, 1, FALSE),
(9, 4, 1, 'LAG, LEAD, SUM OVER', 'video', 2100, 2, FALSE),
(10, 4, 1, 'แบบทดสอบ: Window Functions', 'quiz', 0, 3, FALSE);

-- Enrollments
INSERT INTO enrollments (enrollment_id, user_id, course_id, status, progress_pct, lessons_completed, total_time_sec, payment_amount, enrolled_at, last_accessed) VALUES
(1, 3, 1, 'active', 60, 6, 9600, 790, '2024-01-10 09:00:00', '2024-02-10 20:00:00'),
(2, 4, 1, 'completed', 100, 10, 18000, 790, '2024-01-05 10:00:00', '2024-02-05 18:00:00'),
(3, 5, 1, 'active', 30, 3, 4500, 790, '2024-01-20 14:00:00', '2024-02-08 15:00:00'),
(4, 3, 3, 'active', 50, 17, 7500, 690, '2024-01-15 11:00:00', '2024-02-09 19:00:00'),
(5, 6, 2, 'active', 25, 15, 6000, 990, '2024-01-25 09:00:00', '2024-02-10 17:00:00'),
(6, 4, 3, 'completed', 100, 35, 18000, 690, '2024-01-08 09:00:00', '2024-02-01 16:00:00');

-- Lesson Progress
INSERT INTO lesson_progress (enrollment_id, lesson_id, user_id, status, watch_time_sec, completion_pct, started_at, completed_at) VALUES
(1, 1, 3, 'completed', 910, 100, '2024-01-10 09:00:00', '2024-01-10 09:15:00'),
(1, 2, 3, 'completed', 1210, 100, '2024-01-10 09:20:00', '2024-01-10 09:40:00'),
(1, 3, 3, 'completed', 1520, 100, '2024-01-11 09:00:00', '2024-01-11 09:25:00'),
(1, 4, 3, 'completed', 1810, 100, '2024-01-11 09:30:00', '2024-01-11 10:00:00'),
(1, 5, 3, 'completed', 600, 100, '2024-01-11 10:05:00', '2024-01-11 10:15:00'),
(1, 6, 3, 'completed', 2110, 100, '2024-01-12 09:00:00', '2024-01-12 09:35:00'),
(1, 8, 3, 'in_progress', 1200, 50, '2024-02-10 19:00:00', NULL);

-- Quizzes
INSERT INTO quizzes (quiz_id, lesson_id, title, passing_score, max_attempts) VALUES
(1, 5, 'แบบทดสอบ SELECT พื้นฐาน', 70, 3),
(2, 10, 'แบบทดสอบ Window Functions', 70, 3);

-- Quiz Questions
INSERT INTO quiz_questions (question_id, quiz_id, question_text, question_type, points) VALUES
(1, 1, 'คำสั่งใดใช้ดึงข้อมูลทั้งหมดจากตาราง employees?', 'single_choice', 1),
(2, 1, 'WHERE clause ใช้ทำอะไร?', 'single_choice', 1),
(3, 1, 'ORDER BY ASC คือการเรียงจากน้อยไปมาก', 'true_false', 1),
(4, 2, 'Window Function ใดใช้กำหนดอันดับโดยไม่ข้ามเลข?', 'single_choice', 2);

INSERT INTO question_options (question_id, option_text, is_correct, sort_order) VALUES
(1, 'SELECT ALL FROM employees', FALSE, 1),
(1, 'SELECT * FROM employees', TRUE, 2),
(1, 'GET * FROM employees', FALSE, 3),
(1, 'FETCH * FROM employees', FALSE, 4),
(2, 'จำกัดจำนวน rows', FALSE, 1),
(2, 'กรองข้อมูลตามเงื่อนไข', TRUE, 2),
(2, 'จัดเรียงข้อมูล', FALSE, 3),
(3, 'True', TRUE, 1),
(3, 'False', FALSE, 2),
(4, 'RANK()', FALSE, 1),
(4, 'DENSE_RANK()', TRUE, 2),
(4, 'ROW_NUMBER()', FALSE, 3),
(4, 'NTILE()', FALSE, 4);

-- Quiz Attempts
INSERT INTO quiz_attempts (attempt_id, quiz_id, user_id, enrollment_id, attempt_number, score, max_score, score_pct, is_passed, time_taken_sec, started_at, submitted_at) VALUES
(1, 1, 3, 1, 1, 2.5, 3, 83.3, TRUE, 480, '2024-01-11 10:05:00', '2024-01-11 10:13:00'),
(2, 1, 4, 2, 1, 3, 3, 100, TRUE, 300, '2024-01-12 10:00:00', '2024-01-12 10:05:00'),
(3, 1, 5, 3, 1, 1.5, 3, 50, FALSE, 600, '2024-01-25 10:00:00', '2024-01-25 10:10:00'),
(4, 1, 5, 3, 2, 2, 3, 66.7, FALSE, 540, '2024-01-26 10:00:00', '2024-01-26 10:09:00'),
(5, 1, 5, 3, 3, 2.5, 3, 83.3, TRUE, 420, '2024-01-27 10:00:00', '2024-01-27 10:07:00');

-- Certificates
INSERT INTO certificates (cert_id, user_id, course_id, enrollment_id, cert_number, issued_at) VALUES
(1, 4, 1, 2, 'CERT-MYSQL-2024-001234', '2024-02-05 18:30:00'),
(2, 4, 3, 6, 'CERT-DKMKT-2024-001235', '2024-02-01 17:00:00');

-- Course Reviews
INSERT INTO course_reviews (course_id, user_id, rating, review_text, created_at) VALUES
(1, 4, 5, 'คอร์สดีมาก อธิบายชัดเจน มีตัวอย่างครบ ใช้งานได้จริงในชีวิตจริง แนะนำมากๆ', '2024-02-06 09:00:00'),
(3, 4, 5, 'เนื้อหาครบมาก อาจารย์สอนดี มีการ update ตามเทรนด์', '2024-02-02 10:00:00'),
(1, 3, 4, 'สอนดีครับ แต่บางบทอยากให้มีตัวอย่างเพิ่มขึ้น', '2024-02-10 21:00:00');

-- Badges
INSERT INTO badges (badge_id, name, description, badge_type) VALUES
(1, 'First Step', 'เรียนจบบทเรียนแรก', 'completion'),
(2, 'Course Finisher', 'เรียนจบคอร์สแรก', 'completion'),
(3, 'Quiz Master', 'ทำคะแนน 100% ในแบบทดสอบ', 'excellence'),
(4, '7-Day Streak', 'เรียนต่อเนื่อง 7 วัน', 'streak'),
(5, 'Speed Learner', 'เรียนจบคอร์สภายใน 7 วัน', 'special');

-- User Badges
INSERT INTO user_badges (user_id, badge_id, course_id, earned_at) VALUES
(4, 1, 1, '2024-01-05 11:00:00'),
(4, 2, 1, '2024-02-05 18:00:00'),
(4, 3, 1, '2024-01-12 10:06:00'),
(4, 5, 1, '2024-02-05 18:00:00');

-- Learning Streaks
INSERT INTO learning_streaks (user_id, current_streak_days, longest_streak_days, last_activity_date, total_days_active) VALUES
(3, 5, 12, '2024-02-10', 34),
(4, 0, 31, '2024-02-05', 62),
(5, 2, 8, '2024-02-10', 21),
(6, 1, 5, '2024-02-10', 16);
```

## 3. LMS Queries

### Query 1: Course Progress Dashboard

```sql
-- Dashboard ความก้าวหน้าของ Student
SELECT 
    u.first_name,
    u.last_name,
    c.title AS course_title,
    e.enrolled_at,
    e.progress_pct,
    e.lessons_completed,
    c.total_lessons,
    c.total_lessons - e.lessons_completed AS remaining_lessons,
    -- Time spent
    ROUND(e.total_time_sec / 3600, 1) AS hours_spent,
    -- Estimated time to complete
    ROUND(
        (c.duration_minutes - e.total_time_sec / 60) / 60, 
        1
    ) AS est_hours_remaining,
    e.last_accessed,
    DATEDIFF(NOW(), e.last_accessed) AS days_since_last_access,
    e.status,
    -- Alert if inactive
    CASE 
        WHEN DATEDIFF(NOW(), e.last_accessed) > 14 THEN 'INACTIVE - Send reminder'
        WHEN DATEDIFF(NOW(), e.last_accessed) > 7 THEN 'Low engagement'
        ELSE 'Active'
    END AS engagement_status
FROM enrollments e
JOIN users u ON e.user_id = u.user_id
JOIN courses c ON e.course_id = c.course_id
WHERE e.status = 'active'
ORDER BY e.last_accessed DESC;
```

---

### Query 2: Course Completion Rate by Instructor

```sql
-- Completion Rate และ Metrics ของแต่ละ Instructor
SELECT 
    u.full_name AS instructor_name,
    COUNT(DISTINCT c.course_id) AS total_courses,
    COUNT(DISTINCT e.enrollment_id) AS total_enrollments,
    COUNT(DISTINCT CASE WHEN e.status = 'completed' THEN e.enrollment_id END) AS completions,
    ROUND(
        COUNT(DISTINCT CASE WHEN e.status = 'completed' THEN e.enrollment_id END) 
        * 100.0 / NULLIF(COUNT(DISTINCT e.enrollment_id), 0),
        1
    ) AS completion_rate_pct,
    -- Average rating
    ROUND(AVG(cr.rating), 2) AS avg_rating,
    COUNT(DISTINCT cr.review_id) AS total_reviews,
    -- Revenue
    SUM(e.payment_amount) AS total_revenue,
    ROUND(AVG(e.payment_amount), 2) AS avg_revenue_per_enrollment,
    -- Student engagement
    ROUND(AVG(e.progress_pct), 1) AS avg_student_progress
FROM users u
JOIN courses c ON u.user_id = c.instructor_id AND c.status = 'published'
LEFT JOIN enrollments e ON c.course_id = e.course_id
LEFT JOIN course_reviews cr ON c.course_id = cr.course_id
WHERE u.role = 'instructor'
GROUP BY u.user_id, u.full_name
ORDER BY total_revenue DESC;
```

---

### Query 3: Quiz Performance Analysis

```sql
-- วิเคราะห์ผลการทำ Quiz
WITH attempt_stats AS (
    SELECT 
        qa.quiz_id,
        q.title AS quiz_title,
        l.title AS lesson_title,
        c.title AS course_title,
        COUNT(DISTINCT qa.user_id) AS students_attempted,
        COUNT(qa.attempt_id) AS total_attempts,
        ROUND(AVG(qa.score_pct), 1) AS avg_score_pct,
        MIN(qa.score_pct) AS min_score,
        MAX(qa.score_pct) AS max_score,
        SUM(CASE WHEN qa.is_passed THEN 1 ELSE 0 END) AS passed_count,
        -- First-attempt pass rate
        SUM(CASE WHEN qa.attempt_number = 1 AND qa.is_passed THEN 1 ELSE 0 END) AS first_attempt_pass,
        COUNT(CASE WHEN qa.attempt_number = 1 THEN 1 END) AS first_attempt_count,
        AVG(qa.time_taken_sec) AS avg_time_sec
    FROM quiz_attempts qa
    JOIN quizzes q ON qa.quiz_id = q.quiz_id
    JOIN lessons l ON q.lesson_id = l.lesson_id
    JOIN courses c ON l.course_id = c.course_id
    GROUP BY qa.quiz_id, q.title, l.title, c.title
)
SELECT 
    *,
    ROUND(passed_count * 100.0 / NULLIF(total_attempts, 0), 1) AS overall_pass_rate_pct,
    ROUND(first_attempt_pass * 100.0 / NULLIF(first_attempt_count, 0), 1) AS first_attempt_pass_rate,
    ROUND(avg_time_sec / 60, 1) AS avg_time_minutes,
    -- Difficulty level
    CASE 
        WHEN ROUND(passed_count * 100.0 / NULLIF(total_attempts, 0), 1) >= 80 THEN 'Easy'
        WHEN ROUND(passed_count * 100.0 / NULLIF(total_attempts, 0), 1) >= 60 THEN 'Medium'
        ELSE 'Hard'
    END AS difficulty_level
FROM attempt_stats
ORDER BY overall_pass_rate_pct ASC;
```

---

### Query 4: Student Learning Path Recommendation

```sql
-- แนะนำ Course ถัดไปสำหรับ Student
WITH student_profile AS (
    SELECT 
        e.user_id,
        GROUP_CONCAT(DISTINCT c.category_id) AS enrolled_categories,
        AVG(e.progress_pct) AS avg_progress,
        MAX(c.level) AS highest_level_enrolled,
        COUNT(DISTINCT e.course_id) AS courses_enrolled
    FROM enrollments e
    JOIN courses c ON e.course_id = c.course_id
    WHERE e.user_id = 3
    GROUP BY e.user_id
),
course_scores AS (
    SELECT 
        c.course_id,
        c.title,
        c.level,
        c.rating,
        c.total_students,
        c.price,
        -- Match score
        CASE WHEN c.category_id IN (
            SELECT category_id FROM enrollments e2 
            JOIN courses c2 ON e2.course_id = c2.course_id 
            WHERE e2.user_id = 3
        ) THEN 30 ELSE 0 END AS category_match_score,
        c.rating * 10 AS rating_score,
        LOG(c.total_students + 1) * 2 AS popularity_score,
        -- Not already enrolled
        CASE WHEN c.course_id NOT IN (
            SELECT course_id FROM enrollments WHERE user_id = 3
        ) THEN 1 ELSE 0 END AS is_new
    FROM courses c
    WHERE c.status = 'published'
)
SELECT 
    course_id,
    title,
    level,
    price,
    rating,
    total_students,
    ROUND(category_match_score + rating_score + popularity_score, 2) AS recommendation_score
FROM course_scores
WHERE is_new = 1
ORDER BY recommendation_score DESC
LIMIT 5;
```

---

### Query 5: Certificate Verification

```sql
-- ระบบตรวจสอบใบรับรอง (Certificate Verification API)
SELECT 
    cert.cert_number,
    cert.issued_at,
    cert.expiry_date,
    CASE WHEN cert.expiry_date IS NULL OR cert.expiry_date >= CURDATE() 
         THEN 'VALID' ELSE 'EXPIRED' END AS cert_status,
    -- Student info
    u.full_name AS student_name,
    -- Course info
    c.title AS course_title,
    c.level AS course_level,
    c.duration_minutes AS course_duration_min,
    -- Instructor
    inst.full_name AS instructor_name,
    -- Completion details
    e.completed_at,
    e.total_time_sec / 3600 AS hours_spent,
    -- Best quiz score
    MAX(qa.score_pct) AS best_quiz_score
FROM certificates cert
JOIN users u ON cert.user_id = u.user_id
JOIN courses c ON cert.course_id = c.course_id
JOIN users inst ON c.instructor_id = inst.user_id
JOIN enrollments e ON cert.enrollment_id = e.enrollment_id
LEFT JOIN quiz_attempts qa ON e.enrollment_id = qa.enrollment_id
WHERE cert.cert_number = 'CERT-MYSQL-2024-001234'
GROUP BY cert.cert_id, cert.cert_number, cert.issued_at, cert.expiry_date,
         u.full_name, c.title, c.level, c.duration_minutes,
         inst.full_name, e.completed_at, e.total_time_sec;
```

---

### Query 6: Gamification Leaderboard

```sql
-- Leaderboard ของ Course (Rankings ของ Students)
WITH student_scores AS (
    SELECT 
        e.user_id,
        e.course_id,
        -- Progress score
        e.progress_pct * 0.3 AS progress_score,
        -- Quiz performance
        COALESCE(AVG(qa.score_pct) * 0.3, 0) AS quiz_score,
        -- Engagement (watch time vs course duration)
        LEAST(
            e.total_time_sec / NULLIF(c.duration_minutes * 60, 0) * 100 * 0.2,
            20
        ) AS engagement_score,
        -- Speed bonus
        CASE 
            WHEN e.status = 'completed' AND 
                 DATEDIFF(e.completed_at, e.enrolled_at) <= 7 THEN 20
            WHEN e.status = 'completed' THEN 10
            ELSE 0
        END AS speed_bonus,
        COUNT(DISTINCT ub.badge_id) AS badges_earned
    FROM enrollments e
    JOIN courses c ON e.course_id = c.course_id
    LEFT JOIN quiz_attempts qa ON e.enrollment_id = qa.enrollment_id AND qa.is_passed = TRUE
    LEFT JOIN user_badges ub ON e.user_id = ub.user_id AND ub.course_id = e.course_id
    WHERE e.course_id = 1
    GROUP BY e.user_id, e.course_id, e.progress_pct, e.total_time_sec,
             c.duration_minutes, e.status, e.completed_at, e.enrolled_at
)
SELECT 
    RANK() OVER (ORDER BY progress_score + quiz_score + engagement_score + speed_bonus DESC) AS rank_pos,
    u.full_name AS student_name,
    u.avatar_url,
    ROUND(progress_score + quiz_score + engagement_score + speed_bonus, 1) AS total_score,
    ROUND(progress_score / 0.3, 1) AS progress_pct,
    ROUND(quiz_score / 0.3, 1) AS quiz_avg_pct,
    badges_earned,
    speed_bonus
FROM student_scores ss
JOIN users u ON ss.user_id = u.user_id
ORDER BY total_score DESC;
```

---

### Query 7: Content Gap Analysis

```sql
-- วิเคราะห์ Lessons ที่ Students Drop off (เรียนแล้วเลิก)
SELECT 
    l.lesson_id,
    l.title AS lesson_title,
    s.title AS section_title,
    l.sort_order,
    l.lesson_type,
    l.video_duration_sec / 60 AS duration_min,
    COUNT(lp.progress_id) AS students_started,
    SUM(CASE WHEN lp.status = 'completed' THEN 1 ELSE 0 END) AS students_completed,
    ROUND(
        SUM(CASE WHEN lp.status = 'completed' THEN 1 ELSE 0 END) * 100.0 /
        NULLIF(COUNT(lp.progress_id), 0),
        1
    ) AS completion_rate_pct,
    ROUND(AVG(lp.watch_time_sec) / 60, 1) AS avg_watch_time_min,
    ROUND(AVG(lp.completion_pct), 1) AS avg_completion_pct,
    -- Drop-off indicator
    CASE 
        WHEN ROUND(
            SUM(CASE WHEN lp.status = 'completed' THEN 1 ELSE 0 END) * 100.0 /
            NULLIF(COUNT(lp.progress_id), 0), 1
        ) < 50 THEN 'HIGH DROP-OFF'
        WHEN ROUND(
            SUM(CASE WHEN lp.status = 'completed' THEN 1 ELSE 0 END) * 100.0 /
            NULLIF(COUNT(lp.progress_id), 0), 1
        ) < 70 THEN 'MODERATE DROP-OFF'
        ELSE 'OK'
    END AS dropoff_status
FROM lessons l
JOIN sections s ON l.section_id = s.section_id
LEFT JOIN lesson_progress lp ON l.lesson_id = lp.lesson_id
WHERE l.course_id = 1
GROUP BY l.lesson_id, l.title, s.title, l.sort_order, l.lesson_type, l.video_duration_sec
ORDER BY l.sort_order;
```

---

### Query 8: Revenue and Enrollment Trends

```sql
-- Revenue Trends รายเดือนพร้อม Growth Rate
WITH monthly_revenue AS (
    SELECT 
        DATE_FORMAT(e.enrolled_at, '%Y-%m') AS month,
        COUNT(e.enrollment_id) AS enrollments,
        SUM(e.payment_amount) AS revenue,
        COUNT(DISTINCT e.user_id) AS new_students,
        COUNT(DISTINCT CASE WHEN e.status = 'completed' THEN e.user_id END) AS completions
    FROM enrollments e
    GROUP BY DATE_FORMAT(e.enrolled_at, '%Y-%m')
)
SELECT 
    month,
    enrollments,
    revenue,
    new_students,
    completions,
    -- Growth
    LAG(enrollments) OVER (ORDER BY month) AS prev_month_enrollments,
    ROUND(
        (enrollments - LAG(enrollments) OVER (ORDER BY month)) 
        * 100.0 / NULLIF(LAG(enrollments) OVER (ORDER BY month), 0),
        1
    ) AS enrollment_growth_pct,
    LAG(revenue) OVER (ORDER BY month) AS prev_month_revenue,
    ROUND(
        (revenue - LAG(revenue) OVER (ORDER BY month)) 
        * 100.0 / NULLIF(LAG(revenue) OVER (ORDER BY month), 0),
        1
    ) AS revenue_growth_pct,
    -- Running totals
    SUM(revenue) OVER (ORDER BY month) AS cumulative_revenue,
    SUM(enrollments) OVER (ORDER BY month) AS cumulative_enrollments
FROM monthly_revenue
ORDER BY month DESC;
```

---

### Query 9: Learning Streak Analytics

```sql
-- วิเคราะห์ Learning Streaks และ Engagement Patterns
SELECT 
    u.full_name AS student_name,
    ls.current_streak_days,
    ls.longest_streak_days,
    ls.total_days_active,
    ls.last_activity_date,
    DATEDIFF(CURDATE(), ls.last_activity_date) AS days_since_activity,
    -- Badge eligibility
    CASE 
        WHEN ls.current_streak_days >= 30 THEN '🏆 30-Day Streak!'
        WHEN ls.current_streak_days >= 7 THEN '🔥 7-Day Streak!'
        WHEN ls.current_streak_days >= 3 THEN '⚡ 3-Day Streak'
        ELSE 'Start your streak!'
    END AS streak_status,
    -- Engagement health
    COUNT(DISTINCT e.course_id) AS active_courses,
    ROUND(AVG(e.progress_pct), 1) AS avg_course_progress,
    SUM(e.total_time_sec) / 3600 AS total_hours_studied
FROM learning_streaks ls
JOIN users u ON ls.user_id = u.user_id
LEFT JOIN enrollments e ON u.user_id = e.user_id AND e.status = 'active'
GROUP BY u.user_id, u.full_name, ls.current_streak_days, ls.longest_streak_days,
         ls.total_days_active, ls.last_activity_date
ORDER BY ls.current_streak_days DESC;
```

---

### Query 10: Question Difficulty Analysis

```sql
-- วิเคราะห์ความยากของแต่ละข้อสอบ
WITH question_stats AS (
    SELECT 
        qq.question_id,
        qq.question_text,
        qq.question_type,
        qq.points,
        q.title AS quiz_title,
        COUNT(qa_ans.answer_id) AS total_answers,
        SUM(CASE WHEN qa_ans.is_correct THEN 1 ELSE 0 END) AS correct_answers,
        ROUND(
            SUM(CASE WHEN qa_ans.is_correct THEN 1 ELSE 0 END) * 100.0 / 
            NULLIF(COUNT(qa_ans.answer_id), 0), 
            1
        ) AS correct_rate_pct,
        ROUND(AVG(qa_ans.points_earned), 2) AS avg_points_earned
    FROM quiz_questions qq
    JOIN quizzes q ON qq.quiz_id = q.quiz_id
    LEFT JOIN quiz_answers qa_ans ON qq.question_id = qa_ans.question_id
    GROUP BY qq.question_id, qq.question_text, qq.question_type, qq.points, q.title
)
SELECT 
    question_id,
    LEFT(question_text, 100) AS question_preview,
    quiz_title,
    question_type,
    points,
    total_answers,
    correct_answers,
    correct_rate_pct,
    avg_points_earned,
    -- IRT Difficulty Index (p-value)
    CASE 
        WHEN correct_rate_pct >= 80 THEN 'EASY (>80%)'
        WHEN correct_rate_pct >= 50 THEN 'MEDIUM (50-80%)'
        WHEN correct_rate_pct >= 30 THEN 'HARD (30-50%)'
        ELSE 'VERY HARD (<30%)'
    END AS difficulty_level,
    -- Discrimination suggestion
    CASE 
        WHEN correct_rate_pct > 90 THEN 'Consider replacing - too easy'
        WHEN correct_rate_pct < 20 THEN 'Review content - may be confusing'
        ELSE 'Good discriminating power'
    END AS recommendation
FROM question_stats
ORDER BY correct_rate_pct ASC;
```

---

## แบบฝึกหัด (Challenge Exercises)

1. **SCORM Tracking**: ออกแบบ Schema รองรับ SCORM 2004 (xAPI/Tin Can) สำหรับ tracking interactions ละเอียด เช่น Hover, Click, Time-on-element

2. **Adaptive Learning**: เขียน Query สร้าง Adaptive Quiz ที่เลือกคำถามยากขึ้นถ้า Student ตอบถูกติดกัน 3 ข้อ (Item Response Theory แบบง่าย)

3. **Discussion Forum**: ออกแบบตาราง `course_discussions` สำหรับ Q&A Forum และเขียน Query หา Unanswered Questions ที่รออาจารย์ตอบนานสุด

4. **Cohort Retention**: เขียน Cohort Analysis Query ดู % ของ Students ที่ลงทะเบียนในแต่ละเดือน ยังกลับมาเรียนใน Week 2, 4, 8

5. **Assignment Grading**: ออกแบบ Schema สำหรับ Assignment (Peer Review) และเขียน Query คำนวณ Final Grade จาก Peer Reviews 3 คน (ตัดคะแนนสูงสุดและต่ำสุด)

6. **Live Session**: เพิ่มตาราง `live_sessions` และ `live_attendance` แล้วเขียน Query วิเคราะห์ Attendance Rate และ Average Watch Duration ของ Live Sessions

7. **Prerequisite Chain**: เขียน Recursive CTE ตรวจสอบว่า Student เรียนครบ Prerequisites ทั้งหมดก่อนจะลงทะเบียน Advanced Course ได้หรือไม่

8. **Revenue Optimization**: วิเคราะห์ว่า Price Point ไหน มี Conversion Rate และ Revenue สูงสุด (Price Sensitivity Analysis)

9. **Instructor Revenue Share**: คำนวณ Revenue Share ให้ Instructor แต่ละคน โดย Platform หัก 30% แล้วหาร Revenue ตาม Enrollment ที่เกิดจากแต่ละ Marketing Channel

10. **Learning Time Prediction**: สร้าง Query ทำนาย วันที่คาดว่า Student จะเรียนจบ โดยอิงจาก Average Daily Learning Time ของแต่ละคน
