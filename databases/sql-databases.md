# SQL базы данных: PostgreSQL & MySQL

Детальное сравнение двух популярных RDBMS, синтаксис, диалекты, архитектурные отличия.

---

## 🎯 Основные отличия

### Архитектура и философия

**PostgreSQL** (объектно-реляционная БД):
- Полная поддержка ACID из коробки
- Строгое соблюдение SQL стандартов
- Расширяемость: custom types, operators, functions
- Process-per-connection модель (один процесс на соединение)
- MVCC для конкурентного доступа (Multi-Version Concurrency Control)
- Write-Ahead Logging (WAL) для долговечности

**MySQL/MariaDB** (реляционная БД):
- Модульная архитектура с pluggable storage engines
- InnoDB - основной engine (ACID, transactions, foreign keys)
- MyISAM - legacy, только для чтения (no transactions, table-level locks)
- Thread-per-connection модель
- Большой фокус на скорость чтения
- Проще в администрировании для базовых задач

### Когда что выбрать

**PostgreSQL подходит для:**
- Сложные аналитические запросы
- JSON/JSONB данные (document store внутри RDBMS)
- Геоданные (PostGIS extension)
- Full-text search встроенный
- Строгая консистентность и data integrity
- Проекты с частыми схемными изменениями

**MySQL подходит для:**
- Простые CRUD операции с высокой нагрузкой
- Read-heavy приложения (репликация master-slave)
- Проекты с жесткими требованиями к простоте настройки
- Легаси-системы, требующие обратной совместимости
- Веб-приложения с массовым трафиком (WordPress, Drupal legacy)

---

## 📊 Типы данных

### Числовые типы

**PostgreSQL:**
```sql
-- Целые числа
SMALLINT          -- 2 байта, -32768 to 32767
INTEGER / INT     -- 4 байта, -2147483648 to 2147483647
BIGINT            -- 8 байт, -9223372036854775808 to 9223372036854775807
SERIAL            -- AUTO_INCREMENT аналог (INT с sequence)
BIGSERIAL         -- BIGINT с sequence

-- Числа с плавающей точкой
REAL              -- 4 байта, 6 decimal digits precision
DOUBLE PRECISION  -- 8 байт, 15 decimal digits precision
NUMERIC(p, s)     -- точные вычисления (деньги!), p - precision, s - scale
DECIMAL(p, s)     -- синоним NUMERIC

-- Пример: деньги ВСЕГДА в NUMERIC
CREATE TABLE products (
    id SERIAL PRIMARY KEY,
    price NUMERIC(10, 2) NOT NULL CHECK (price >= 0)  -- 99999999.99 max
);
```

**MySQL:**
```sql
-- Целые числа (можно UNSIGNED для удвоения положительного диапазона)
TINYINT           -- 1 байт, -128 to 127 (UNSIGNED: 0 to 255)
SMALLINT          -- 2 байта
MEDIUMINT         -- 3 байта (только в MySQL!)
INT / INTEGER     -- 4 байта
BIGINT            -- 8 байт

-- AUTO_INCREMENT
id INT AUTO_INCREMENT PRIMARY KEY

-- Числа с плавающей точкой
FLOAT             -- 4 байта
DOUBLE            -- 8 байт
DECIMAL(p, s)     -- точные вычисления

-- Пример
CREATE TABLE products (
    id INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    price DECIMAL(10, 2) NOT NULL CHECK (price >= 0)
);
```

### Строковые типы

**PostgreSQL:**
```sql
-- Строки (нет разницы в производительности между типами!)
CHAR(n)           -- фиксированная длина, дополняется пробелами
VARCHAR(n)        -- переменная длина, limit n
TEXT              -- неограниченная длина (используй TEXT вместо VARCHAR без причины)

-- Пример: TEXT быстрее VARCHAR в PostgreSQL
CREATE TABLE posts (
    id SERIAL PRIMARY KEY,
    title VARCHAR(255) NOT NULL,  -- явный лимит для валидации
    content TEXT NOT NULL          -- неограниченный текст
);

-- Бинарные данные
BYTEA             -- binary data (хранение файлов, НЕ рекомендуется для больших)
```

**MySQL:**
```sql
-- Строки
CHAR(n)           -- фиксированная, max 255
VARCHAR(n)        -- переменная, max 65535 (зависит от row size limit)
TINYTEXT          -- max 255 bytes
TEXT              -- max 65535 bytes (~64KB)
MEDIUMTEXT        -- max 16MB
LONGTEXT          -- max 4GB

-- Пример
CREATE TABLE posts (
    id INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    title VARCHAR(255) NOT NULL,
    content LONGTEXT NOT NULL
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;

-- Бинарные
BLOB, TINYBLOB, MEDIUMBLOB, LONGBLOB
BINARY(n), VARBINARY(n)
```

**⚠️ ВАЖНО для MySQL:**
```sql
-- ВСЕГДА используй utf8mb4, НЕ utf8 (utf8 = 3-byte, не поддерживает emoji!)
DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci

-- utf8mb4_unicode_ci - case-insensitive, правильная сортировка
-- utf8mb4_bin - case-sensitive, бинарное сравнение
```

### Дата и время

**PostgreSQL:**
```sql
DATE              -- дата без времени: 2026-01-28
TIME              -- время без даты: 14:30:00
TIME WITH TIME ZONE  -- время с timezone
TIMESTAMP         -- дата + время: 2026-01-28 14:30:00
TIMESTAMP WITH TIME ZONE (TIMESTAMPTZ)  -- с timezone (рекомендуется!)
INTERVAL          -- временной интервал: '1 day', '3 hours'

-- Пример: ВСЕГДА используй TIMESTAMPTZ
CREATE TABLE events (
    id SERIAL PRIMARY KEY,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    starts_at TIMESTAMPTZ NOT NULL,
    duration INTERVAL NOT NULL
);

-- Операции с датами
SELECT NOW();                           -- 2026-01-28 14:30:00+00
SELECT NOW() + INTERVAL '1 day';        -- 2026-01-29 14:30:00+00
SELECT AGE('2026-01-28', '2020-01-01'); -- 6 years 27 days
SELECT EXTRACT(YEAR FROM NOW());        -- 2026
```

**MySQL:**
```sql
DATE              -- 1000-01-01 to 9999-12-31
TIME              -- -838:59:59 to 838:59:59
DATETIME          -- 1000-01-01 00:00:00 to 9999-12-31 23:59:59
TIMESTAMP         -- 1970-01-01 00:00:01 UTC to 2038-01-19 (Year 2038 problem!)
YEAR              -- 1901 to 2155

-- Пример
CREATE TABLE events (
    id INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    starts_at DATETIME NOT NULL
);

-- Операции
SELECT NOW();                           -- 2026-01-28 14:30:00
SELECT DATE_ADD(NOW(), INTERVAL 1 DAY); -- 2026-01-29 14:30:00
SELECT YEAR(NOW());                     -- 2026
```

**⚠️ TIMESTAMP в MySQL:**
- Хранится в UTC, конвертируется в timezone сессии
- Year 2038 problem (переполнение 32-bit UNIX timestamp)
- Для дат после 2038 используй DATETIME

### JSON типы

**PostgreSQL:**
```sql
JSON              -- текстовое хранение, валидация при вставке
JSONB             -- бинарное хранение, ИНДЕКСИРУЕТСЯ, БЫСТРЕЕ (используй JSONB!)

-- Пример: JSONB для гибких данных
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    email VARCHAR(255) NOT NULL UNIQUE,
    profile JSONB NOT NULL DEFAULT '{}'::jsonb
);

-- Вставка
INSERT INTO users (email, profile) VALUES 
('user@test.com', '{"age": 30, "city": "Moscow", "skills": ["PHP", "PostgreSQL"]}');

-- Запросы к JSONB
SELECT * FROM users WHERE profile->>'city' = 'Moscow';           -- ->> вернет TEXT
SELECT * FROM users WHERE (profile->'age')::int > 25;            -- -> вернет JSONB
SELECT * FROM users WHERE profile @> '{"city": "Moscow"}';       -- containment operator
SELECT * FROM users WHERE profile ? 'age';                       -- key exists
SELECT * FROM users WHERE profile->'skills' @> '"PHP"';          -- array contains

-- Индекс на JSONB (GIN - Generalized Inverted Index)
CREATE INDEX idx_users_profile ON users USING GIN (profile);

-- Обновление части JSONB
UPDATE users 
SET profile = jsonb_set(profile, '{age}', '31')
WHERE email = 'user@test.com';
```

**MySQL 5.7+:**
```sql
JSON              -- бинарное хранение с MySQL 5.7, индексируется через generated columns

-- Пример
CREATE TABLE users (
    id INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    email VARCHAR(255) NOT NULL UNIQUE,
    profile JSON NOT NULL
);

-- Вставка
INSERT INTO users (email, profile) VALUES 
('user@test.com', '{"age": 30, "city": "Moscow", "skills": ["PHP", "MySQL"]}');

-- Запросы
SELECT * FROM users WHERE JSON_EXTRACT(profile, '$.city') = 'Moscow';
SELECT * FROM users WHERE profile->>'$.city' = 'Moscow';  -- MySQL 8.0+ shorthand
SELECT * FROM users WHERE JSON_CONTAINS(profile, '"PHP"', '$.skills');

-- Индекс через generated column (MySQL 5.7+)
ALTER TABLE users 
ADD COLUMN city VARCHAR(100) AS (JSON_UNQUOTE(JSON_EXTRACT(profile, '$.city'))) STORED,
ADD INDEX idx_city (city);

-- Обновление
UPDATE users 
SET profile = JSON_SET(profile, '$.age', 31)
WHERE email = 'user@test.com';
```

### Специальные типы PostgreSQL

```sql
-- UUID
UUID              -- 128-bit identifier, 36 chars

CREATE TABLE sessions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id INT NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Массивы (поддержка native arrays!)
INTEGER[]         -- массив целых чисел
TEXT[]            -- массив строк

CREATE TABLE posts (
    id SERIAL PRIMARY KEY,
    title TEXT NOT NULL,
    tags TEXT[] NOT NULL DEFAULT '{}'  -- массив тегов
);

INSERT INTO posts (title, tags) VALUES 
('PHP Interview', ARRAY['php', 'interview', 'backend']);

-- Запросы к массивам
SELECT * FROM posts WHERE 'php' = ANY(tags);              -- содержит элемент
SELECT * FROM posts WHERE tags @> ARRAY['php', 'backend']; -- содержит все элементы
SELECT * FROM posts WHERE tags && ARRAY['php', 'laravel']; -- есть пересечение

-- Индекс на массив (GIN)
CREATE INDEX idx_posts_tags ON posts USING GIN (tags);

-- ENUM (есть в обоих, но в PostgreSQL это custom type)
CREATE TYPE order_status AS ENUM ('pending', 'processing', 'completed', 'cancelled');

CREATE TABLE orders (
    id SERIAL PRIMARY KEY,
    status order_status NOT NULL DEFAULT 'pending'
);

-- Геометрия (с расширением PostGIS)
POINT, LINE, POLYGON, GEOMETRY, GEOGRAPHY

-- IP адреса
INET              -- IPv4 or IPv6 address
CIDR              -- network address

CREATE TABLE logs (
    id SERIAL PRIMARY KEY,
    ip INET NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

SELECT * FROM logs WHERE ip << '192.168.1.0/24';  -- IP в подсети
```

---

## 🔧 DDL - Data Definition Language

### CREATE TABLE

**PostgreSQL:**
```sql
CREATE TABLE users (
    id SERIAL PRIMARY KEY,                        -- AUTO_INCREMENT
    email VARCHAR(255) NOT NULL UNIQUE,
    username VARCHAR(50) NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    
    -- Constraint с именем для читаемых ошибок
    CONSTRAINT users_email_check CHECK (email ~* '^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Z|a-z]{2,}$'),
    CONSTRAINT users_username_length CHECK (LENGTH(username) >= 3)
);

-- Индексы
CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_username ON users(username);
CREATE INDEX idx_users_created_at ON users(created_at);  -- для сортировки

-- Partial index (только активные)
CREATE INDEX idx_users_active ON users(email) WHERE is_active = TRUE;

-- Комментарии (важно для документации схемы)
COMMENT ON TABLE users IS 'User accounts';
COMMENT ON COLUMN users.is_active IS 'Soft delete flag';
```

**MySQL:**
```sql
CREATE TABLE users (
    id INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    email VARCHAR(255) NOT NULL UNIQUE,
    username VARCHAR(50) NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    is_active TINYINT(1) NOT NULL DEFAULT 1,      -- BOOLEAN = TINYINT(1)
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    
    -- Constraints
    CONSTRAINT users_email_check CHECK (email REGEXP '^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\\.[A-Z|a-z]{2,}$'),
    CONSTRAINT users_username_length CHECK (CHAR_LENGTH(username) >= 3),
    
    INDEX idx_users_email (email),
    INDEX idx_users_username (username),
    INDEX idx_users_created_at (created_at)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;

-- Комментарии
ALTER TABLE users COMMENT = 'User accounts';
ALTER TABLE users MODIFY COLUMN is_active TINYINT(1) NOT NULL DEFAULT 1 COMMENT 'Soft delete flag';
```

### Foreign Keys

**PostgreSQL:**
```sql
CREATE TABLE posts (
    id SERIAL PRIMARY KEY,
    user_id INT NOT NULL,
    title VARCHAR(255) NOT NULL,
    content TEXT NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    
    -- Foreign key с каскадным удалением
    CONSTRAINT fk_posts_user FOREIGN KEY (user_id) 
        REFERENCES users(id) 
        ON DELETE CASCADE        -- удалить все посты при удалении пользователя
        ON UPDATE CASCADE        -- обновить user_id при изменении id пользователя
);

-- Другие варианты:
-- ON DELETE RESTRICT - запретить удаление если есть связанные записи (по умолчанию)
-- ON DELETE SET NULL - установить NULL
-- ON DELETE SET DEFAULT - установить значение по умолчанию
-- ON DELETE NO ACTION - то же что RESTRICT

CREATE INDEX idx_posts_user_id ON posts(user_id);  -- ОБЯЗАТЕЛЬНО для FK!
```

**MySQL:**
```sql
CREATE TABLE posts (
    id INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    user_id INT UNSIGNED NOT NULL,
    title VARCHAR(255) NOT NULL,
    content LONGTEXT NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    
    CONSTRAINT fk_posts_user FOREIGN KEY (user_id) 
        REFERENCES users(id) 
        ON DELETE CASCADE 
        ON UPDATE CASCADE,
    
    INDEX idx_posts_user_id (user_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

**⚠️ ВАЖНО:**
- Всегда создавай индекс на FK колонку (иначе медленный DELETE родителя)
- InnoDB автоматически создает индекс на FK, НО лучше явно

### ALTER TABLE

**PostgreSQL:**
```sql
-- Добавить колонку
ALTER TABLE users ADD COLUMN phone VARCHAR(20);
ALTER TABLE users ADD COLUMN avatar_url TEXT;

-- Добавить с DEFAULT и NOT NULL сразу
ALTER TABLE users ADD COLUMN role VARCHAR(20) NOT NULL DEFAULT 'user';

-- Изменить тип колонки
ALTER TABLE users ALTER COLUMN phone TYPE VARCHAR(30);

-- Изменить тип с преобразованием
ALTER TABLE users ALTER COLUMN age TYPE INTEGER USING age::integer;

-- Установить/удалить DEFAULT
ALTER TABLE users ALTER COLUMN role SET DEFAULT 'user';
ALTER TABLE users ALTER COLUMN role DROP DEFAULT;

-- Установить/удалить NOT NULL
ALTER TABLE users ALTER COLUMN phone SET NOT NULL;
ALTER TABLE users ALTER COLUMN phone DROP NOT NULL;

-- Переименовать колонку
ALTER TABLE users RENAME COLUMN phone TO phone_number;

-- Переименовать таблицу
ALTER TABLE users RENAME TO accounts;

-- Удалить колонку
ALTER TABLE users DROP COLUMN avatar_url;

-- Добавить constraint
ALTER TABLE users ADD CONSTRAINT users_phone_unique UNIQUE (phone);
ALTER TABLE users ADD CONSTRAINT users_age_check CHECK (age >= 18);

-- Удалить constraint
ALTER TABLE users DROP CONSTRAINT users_phone_unique;
```

**MySQL:**
```sql
-- Добавить колонку
ALTER TABLE users ADD COLUMN phone VARCHAR(20);

-- Добавить с позицией
ALTER TABLE users ADD COLUMN role VARCHAR(20) NOT NULL DEFAULT 'user' AFTER email;
ALTER TABLE users ADD COLUMN id_new INT FIRST;

-- Изменить колонку (MODIFY - без переименования)
ALTER TABLE users MODIFY COLUMN phone VARCHAR(30);
ALTER TABLE users MODIFY COLUMN phone VARCHAR(30) NOT NULL;

-- Изменить с переименованием (CHANGE)
ALTER TABLE users CHANGE COLUMN phone phone_number VARCHAR(30);

-- Переименовать колонку (MySQL 8.0+)
ALTER TABLE users RENAME COLUMN phone TO phone_number;

-- Переименовать таблицу
ALTER TABLE users RENAME TO accounts;
RENAME TABLE users TO accounts;  -- альтернатива

-- Удалить колонку
ALTER TABLE users DROP COLUMN avatar_url;

-- Добавить/удалить constraint
ALTER TABLE users ADD CONSTRAINT users_phone_unique UNIQUE (phone);
ALTER TABLE users DROP CONSTRAINT users_phone_unique;
-- или для индексов:
ALTER TABLE users DROP INDEX users_phone_unique;
```

---

## 📝 DML - Data Manipulation Language

### INSERT

**PostgreSQL:**
```sql
-- Простая вставка
INSERT INTO users (email, username, password_hash) 
VALUES ('user@test.com', 'testuser', 'hashed_password');

-- Множественная вставка
INSERT INTO users (email, username, password_hash) 
VALUES 
    ('user1@test.com', 'user1', 'hash1'),
    ('user2@test.com', 'user2', 'hash2'),
    ('user3@test.com', 'user3', 'hash3');

-- RETURNING - вернуть вставленные данные (очень полезно!)
INSERT INTO users (email, username, password_hash) 
VALUES ('user@test.com', 'testuser', 'hashed') 
RETURNING id, created_at;

-- INSERT ... ON CONFLICT (UPSERT в PostgreSQL)
INSERT INTO users (email, username, password_hash) 
VALUES ('user@test.com', 'testuser', 'hashed')
ON CONFLICT (email) DO UPDATE SET
    username = EXCLUDED.username,
    updated_at = NOW();

-- ON CONFLICT DO NOTHING - игнорировать дубликат
INSERT INTO users (email, username, password_hash) 
VALUES ('user@test.com', 'testuser', 'hashed')
ON CONFLICT (email) DO NOTHING;

-- INSERT из SELECT
INSERT INTO archived_users (email, username, created_at)
SELECT email, username, created_at 
FROM users 
WHERE created_at < NOW() - INTERVAL '5 years';
```

**MySQL:**
```sql
-- Простая вставка
INSERT INTO users (email, username, password_hash) 
VALUES ('user@test.com', 'testuser', 'hashed_password');

-- Множественная
INSERT INTO users (email, username, password_hash) 
VALUES 
    ('user1@test.com', 'user1', 'hash1'),
    ('user2@test.com', 'user2', 'hash2');

-- INSERT IGNORE - игнорировать ошибки дубликатов
INSERT IGNORE INTO users (email, username, password_hash) 
VALUES ('user@test.com', 'testuser', 'hashed');

-- ON DUPLICATE KEY UPDATE (UPSERT в MySQL)
INSERT INTO users (email, username, password_hash) 
VALUES ('user@test.com', 'testuser', 'hashed')
ON DUPLICATE KEY UPDATE 
    username = VALUES(username),
    updated_at = CURRENT_TIMESTAMP;

-- MySQL 8.0.19+ - alias для VALUES()
INSERT INTO users (email, username, password_hash) 
VALUES ('user@test.com', 'testuser', 'hashed') AS new
ON DUPLICATE KEY UPDATE 
    username = new.username,
    updated_at = CURRENT_TIMESTAMP;

-- REPLACE - удалить и вставить (осторожно с FK!)
REPLACE INTO users (email, username, password_hash) 
VALUES ('user@test.com', 'testuser', 'hashed');

-- INSERT из SELECT
INSERT INTO archived_users (email, username, created_at)
SELECT email, username, created_at 
FROM users 
WHERE created_at < DATE_SUB(NOW(), INTERVAL 5 YEAR);
```

### UPDATE

**PostgreSQL:**
```sql
-- Простое обновление
UPDATE users 
SET username = 'newname' 
WHERE id = 1;

-- Обновление нескольких полей
UPDATE users 
SET 
    username = 'newname',
    updated_at = NOW()
WHERE id = 1;

-- UPDATE с JOIN (PostgreSQL синтаксис FROM)
UPDATE posts 
SET user_email = users.email
FROM users
WHERE posts.user_id = users.id;

-- RETURNING - вернуть обновленные данные
UPDATE users 
SET is_active = FALSE 
WHERE last_login < NOW() - INTERVAL '1 year'
RETURNING id, email, last_login;

-- UPDATE с подзапросом
UPDATE users 
SET post_count = (
    SELECT COUNT(*) FROM posts WHERE posts.user_id = users.id
);
```

**MySQL:**
```sql
-- Простое обновление
UPDATE users 
SET username = 'newname' 
WHERE id = 1;

-- UPDATE с JOIN
UPDATE posts 
INNER JOIN users ON posts.user_id = users.id
SET posts.user_email = users.email;

-- UPDATE с LIMIT (MySQL специфика, нет в PostgreSQL)
UPDATE users 
SET is_active = 0 
WHERE last_login < DATE_SUB(NOW(), INTERVAL 1 YEAR)
LIMIT 1000;  -- обновить только 1000 записей
```

### DELETE

**PostgreSQL:**
```sql
-- Простое удаление
DELETE FROM users WHERE id = 1;

-- DELETE с USING (JOIN для PostgreSQL)
DELETE FROM posts 
USING users
WHERE posts.user_id = users.id 
  AND users.is_active = FALSE;

-- RETURNING
DELETE FROM users 
WHERE last_login < NOW() - INTERVAL '2 years'
RETURNING id, email;

-- Truncate - быстрая очистка таблицы (сброс AUTO_INCREMENT)
TRUNCATE TABLE logs;
TRUNCATE TABLE logs RESTART IDENTITY;  -- сбросить SERIAL
TRUNCATE TABLE logs CASCADE;            -- удалить связанные данные в FK таблицах
```

**MySQL:**
```sql
-- Простое удаление
DELETE FROM users WHERE id = 1;

-- DELETE с JOIN
DELETE posts 
FROM posts
INNER JOIN users ON posts.user_id = users.id
WHERE users.is_active = 0;

-- DELETE с LIMIT
DELETE FROM logs 
WHERE created_at < DATE_SUB(NOW(), INTERVAL 1 MONTH)
LIMIT 10000;

-- Truncate
TRUNCATE TABLE logs;  -- сброс AUTO_INCREMENT, быстрее чем DELETE
```

---

## 🔍 DQL - Data Query Language (SELECT)

### Window Functions (мощь PostgreSQL!)

**PostgreSQL:**
```sql
-- ROW_NUMBER - нумерация строк в разбивке
SELECT 
    user_id,
    title,
    created_at,
    ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY created_at DESC) as row_num
FROM posts;

-- Выбрать последний пост каждого пользователя
WITH ranked_posts AS (
    SELECT 
        *,
        ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY created_at DESC) as rn
    FROM posts
)
SELECT * FROM ranked_posts WHERE rn = 1;

-- RANK и DENSE_RANK
SELECT 
    username,
    score,
    RANK() OVER (ORDER BY score DESC) as rank,            -- 1, 2, 2, 4
    DENSE_RANK() OVER (ORDER BY score DESC) as dense_rank -- 1, 2, 2, 3
FROM users;

-- LAG и LEAD - доступ к предыдущей/следующей строке
SELECT 
    date,
    revenue,
    LAG(revenue, 1) OVER (ORDER BY date) as prev_day_revenue,
    LEAD(revenue, 1) OVER (ORDER BY date) as next_day_revenue,
    revenue - LAG(revenue, 1) OVER (ORDER BY date) as revenue_diff
FROM daily_stats;

-- NTILE - разбить на N групп
SELECT 
    user_id,
    total_purchases,
    NTILE(4) OVER (ORDER BY total_purchases DESC) as quartile  -- 1, 2, 3, 4
FROM user_stats;

-- Агрегатные функции как window functions
SELECT 
    user_id,
    order_date,
    amount,
    SUM(amount) OVER (PARTITION BY user_id ORDER BY order_date) as running_total,
    AVG(amount) OVER (PARTITION BY user_id) as user_avg_amount
FROM orders;
```

**MySQL 8.0+** (поддержка window functions добавлена):
```sql
-- То же самое, синтаксис идентичен PostgreSQL
SELECT 
    user_id,
    title,
    created_at,
    ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY created_at DESC) as row_num
FROM posts;
```

**⚠️ MySQL < 8.0:**
```sql
-- Нет window functions! Используй переменные (медленно, неудобно)
SELECT 
    @row_number := @row_number + 1 AS row_num,
    user_id,
    title
FROM posts, (SELECT @row_number := 0) AS init
ORDER BY created_at DESC;
```

### CTE - Common Table Expressions (WITH)

**PostgreSQL:**
```sql
-- Простой CTE
WITH active_users AS (
    SELECT * FROM users WHERE is_active = TRUE
)
SELECT * FROM active_users WHERE created_at > NOW() - INTERVAL '1 month';

-- Множественные CTE
WITH 
    active_users AS (
        SELECT id, username FROM users WHERE is_active = TRUE
    ),
    user_posts AS (
        SELECT user_id, COUNT(*) as post_count 
        FROM posts 
        GROUP BY user_id
    )
SELECT 
    u.username,
    COALESCE(p.post_count, 0) as post_count
FROM active_users u
LEFT JOIN user_posts p ON u.id = p.user_id;

-- Рекурсивный CTE (иерархии, деревья)
WITH RECURSIVE category_tree AS (
    -- Base case: корневые категории
    SELECT id, name, parent_id, 0 as level, ARRAY[id] as path
    FROM categories
    WHERE parent_id IS NULL
    
    UNION ALL
    
    -- Recursive case: дочерние категории
    SELECT c.id, c.name, c.parent_id, ct.level + 1, ct.path || c.id
    FROM categories c
    INNER JOIN category_tree ct ON c.parent_id = ct.id
)
SELECT * FROM category_tree ORDER BY path;

-- Пример: последовательность чисел
WITH RECURSIVE numbers AS (
    SELECT 1 as n
    UNION ALL
    SELECT n + 1 FROM numbers WHERE n < 10
)
SELECT * FROM numbers;  -- 1, 2, 3, ..., 10
```

**MySQL 8.0+:**
```sql
-- Тот же синтаксис CTE
WITH active_users AS (
    SELECT * FROM users WHERE is_active = 1
)
SELECT * FROM active_users;

-- Рекурсивный CTE
WITH RECURSIVE category_tree AS (
    SELECT id, name, parent_id, 0 as level
    FROM categories
    WHERE parent_id IS NULL
    
    UNION ALL
    
    SELECT c.id, c.name, c.parent_id, ct.level + 1
    FROM categories c
    INNER JOIN category_tree ct ON c.parent_id = ct.id
)
SELECT * FROM category_tree;
```

### LATERAL JOIN (PostgreSQL 9.3+)

**PostgreSQL:**
```sql
-- LATERAL - для каждой строки левой таблицы выполнить подзапрос
-- Аналог CROSS APPLY в SQL Server

-- Получить 3 последних поста каждого пользователя
SELECT 
    u.username,
    p.title,
    p.created_at
FROM users u
CROSS JOIN LATERAL (
    SELECT title, created_at
    FROM posts
    WHERE posts.user_id = u.id
    ORDER BY created_at DESC
    LIMIT 3
) p;

-- LEFT JOIN LATERAL - вернуть users даже без постов
SELECT 
    u.username,
    p.title
FROM users u
LEFT JOIN LATERAL (
    SELECT title
    FROM posts
    WHERE posts.user_id = u.id
    ORDER BY created_at DESC
    LIMIT 1
) p ON TRUE;
```

**MySQL 8.0.14+** (поддержка LATERAL добавлена):
```sql
-- То же самое
SELECT 
    u.username,
    p.title
FROM users u
JOIN LATERAL (
    SELECT title
    FROM posts
    WHERE posts.user_id = u.id
    ORDER BY created_at DESC
    LIMIT 1
) p;
```

### Full-Text Search

**PostgreSQL:**
```sql
-- Встроенный full-text search (очень мощный!)

-- tsvector - документ для поиска
-- tsquery - запрос для поиска

-- Простой поиск
SELECT * FROM posts
WHERE to_tsvector('english', title || ' ' || content) @@ to_tsquery('english', 'postgresql & performance');

-- Создать колонку для FTS
ALTER TABLE posts ADD COLUMN search_vector tsvector;

-- Заполнить
UPDATE posts 
SET search_vector = to_tsvector('english', title || ' ' || content);

-- Индекс GIN для FTS (ОБЯЗАТЕЛЬНО!)
CREATE INDEX idx_posts_search ON posts USING GIN (search_vector);

-- Автоматическое обновление через trigger
CREATE FUNCTION posts_search_vector_update() RETURNS trigger AS $$
BEGIN
    NEW.search_vector := to_tsvector('english', NEW.title || ' ' || NEW.content);
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER posts_search_vector_update_trigger
BEFORE INSERT OR UPDATE ON posts
FOR EACH ROW EXECUTE FUNCTION posts_search_vector_update();

-- Поиск с рангом релевантности
SELECT 
    title,
    ts_rank(search_vector, query) as rank
FROM posts, to_tsquery('english', 'postgresql & performance') query
WHERE search_vector @@ query
ORDER BY rank DESC;

-- Поиск с подсветкой (highlighting)
SELECT 
    title,
    ts_headline('english', content, to_tsquery('postgresql'), 'MaxWords=50') as snippet
FROM posts
WHERE search_vector @@ to_tsquery('postgresql');
```

**MySQL 5.6+ (InnoDB FTS):**
```sql
-- Создать FULLTEXT индекс
CREATE TABLE posts (
    id INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    title VARCHAR(255) NOT NULL,
    content LONGTEXT NOT NULL,
    FULLTEXT INDEX ft_title_content (title, content)
) ENGINE=InnoDB;

-- Или добавить к существующей
ALTER TABLE posts ADD FULLTEXT INDEX ft_title_content (title, content);

-- Boolean mode search
SELECT * FROM posts
WHERE MATCH(title, content) AGAINST('+mysql -oracle' IN BOOLEAN MODE);

-- Natural language mode (по умолчанию)
SELECT * FROM posts
WHERE MATCH(title, content) AGAINST('mysql performance');

-- С рангом
SELECT 
    title,
    MATCH(title, content) AGAINST('mysql performance') as relevance
FROM posts
WHERE MATCH(title, content) AGAINST('mysql performance')
ORDER BY relevance DESC;

-- Query expansion mode
SELECT * FROM posts
WHERE MATCH(title, content) AGAINST('database' WITH QUERY EXPANSION);

-- ⚠️ Ограничения MySQL FTS:
-- - Минимальная длина слова: 4 символа (ft_min_word_len, надо менять в конфиге)
-- - Нет поддержки фраз в кавычках в natural language mode
-- - Меньше возможностей чем PostgreSQL FTS
```

---

## 🎨 Продвинутые конструкции

### Массивы (только PostgreSQL)

```sql
-- Создание массива
SELECT ARRAY[1, 2, 3, 4, 5];
SELECT '{1, 2, 3}'::integer[];

-- Доступ к элементам (1-based indexing!)
SELECT (ARRAY[10, 20, 30])[1];  -- 10
SELECT (ARRAY[10, 20, 30])[1:2];  -- {10, 20}

-- Функции для массивов
SELECT array_length(ARRAY[1,2,3], 1);  -- 3
SELECT array_append(ARRAY[1,2], 3);    -- {1,2,3}
SELECT array_prepend(0, ARRAY[1,2]);   -- {0,1,2}
SELECT array_remove(ARRAY[1,2,3,2], 2); -- {1,3}
SELECT array_position(ARRAY['a','b','c'], 'b');  -- 2

-- Развернуть массив в строки
SELECT unnest(ARRAY[1, 2, 3, 4, 5]) as num;

-- Агрегация в массив
SELECT 
    user_id,
    array_agg(title ORDER BY created_at DESC) as post_titles
FROM posts
GROUP BY user_id;
```

### EXPLAIN и EXPLAIN ANALYZE

**PostgreSQL:**
```sql
-- EXPLAIN - план выполнения (без выполнения запроса)
EXPLAIN 
SELECT * FROM users WHERE email = 'test@test.com';

-- EXPLAIN ANALYZE - план + реальное выполнение + статистика
EXPLAIN ANALYZE
SELECT * FROM users WHERE email = 'test@test.com';

-- С подробностями
EXPLAIN (ANALYZE, BUFFERS, VERBOSE, FORMAT JSON)
SELECT u.username, COUNT(p.id) as post_count
FROM users u
LEFT JOIN posts p ON u.id = p.user_id
GROUP BY u.id, u.username;

-- Что смотреть:
-- - Seq Scan vs Index Scan (Seq Scan плохо на больших таблицах)
-- - actual time vs cost (если большая разница - статистика устарела)
-- - rows - сколько строк планируется vs фактически
-- - Buffers - сколько данных прочитано (shared hit = кеш, shared read = диск)
```

**MySQL:**
```sql
-- EXPLAIN
EXPLAIN
SELECT * FROM users WHERE email = 'test@test.com';

-- EXPLAIN FORMAT=JSON (MySQL 5.6+)
EXPLAIN FORMAT=JSON
SELECT * FROM users WHERE email = 'test@test.com';

-- EXPLAIN ANALYZE (MySQL 8.0.18+) - с реальным выполнением
EXPLAIN ANALYZE
SELECT * FROM users WHERE email = 'test@test.com';

-- Что смотреть:
-- - type: ALL (плохо, seq scan), index, range, ref, eq_ref, const
-- - possible_keys и key - какие индексы доступны и используются
-- - rows - сколько строк будет просканировано
-- - Extra: Using index (covering index, хорошо), Using filesort (плохо), Using temporary (плохо)
```

---

## ⚡ Производительность и оптимизация

### Индексы: разница в синтаксисе

**PostgreSQL:**
```sql
-- B-Tree (по умолчанию)
CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_name ON users USING BTREE (username);

-- Hash (только для равенства, меньше размер)
CREATE INDEX idx_users_id_hash ON users USING HASH (id);

-- GIN (Generalized Inverted Index) - для JSONB, arrays, full-text
CREATE INDEX idx_posts_tags ON posts USING GIN (tags);
CREATE INDEX idx_users_profile ON users USING GIN (profile);

-- GiST (Generalized Search Tree) - для геометрии, диапазонов
CREATE INDEX idx_events_daterange ON events USING GIST (daterange);

-- BRIN (Block Range Index) - для очень больших таблиц с естественным порядком
CREATE INDEX idx_logs_created_at ON logs USING BRIN (created_at);

-- Partial index - индекс только для части данных
CREATE INDEX idx_users_active ON users(email) WHERE is_active = TRUE;

-- Expression index - индекс на выражение
CREATE INDEX idx_users_lower_email ON users(LOWER(email));

-- Composite index
CREATE INDEX idx_posts_user_created ON posts(user_id, created_at DESC);

-- Covering index (INCLUDE в PostgreSQL 11+)
CREATE INDEX idx_users_email_covering ON users(email) INCLUDE (username, created_at);
```

**MySQL:**
```sql
-- B-Tree (по умолчанию, единственный для InnoDB в большинстве случаев)
CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_email_btree ON users(email) USING BTREE;

-- Hash (только для MEMORY engine, не для InnoDB!)
CREATE INDEX idx_users_hash ON users(email) USING HASH;  -- работает только в MEMORY

-- FULLTEXT (для FTS)
CREATE FULLTEXT INDEX ft_posts_content ON posts(title, content);

-- Spatial index (для геометрии)
CREATE SPATIAL INDEX idx_locations_point ON locations(coordinates);

-- Composite
CREATE INDEX idx_posts_user_created ON posts(user_id, created_at DESC);

-- Covering index (через все колонки в индексе)
CREATE INDEX idx_users_email_covering ON users(email, username, created_at);

-- ⚠️ Нет partial indexes и expression indexes до MySQL 8.0.13
-- MySQL 8.0.13+ - functional index (expression index)
CREATE INDEX idx_users_lower_email ON users((LOWER(email)));

-- MySQL 8.0.13+ - invisible index
CREATE INDEX idx_users_username ON users(username) INVISIBLE;
```

### VACUUM и ANALYZE (PostgreSQL специфика)

```sql
-- VACUUM - удалить "мертвые" строки (deleted/updated)
VACUUM users;
VACUUM FULL users;  -- полная перестройка таблицы (блокирует таблицу)

-- ANALYZE - обновить статистику планировщика
ANALYZE users;

-- VACUUM ANALYZE - оба сразу
VACUUM ANALYZE users;

-- Автоматический VACUUM (autovacuum)
-- Включен по умолчанию, работает в фоне
-- Настройки в postgresql.conf:
-- autovacuum = on
-- autovacuum_vacuum_scale_factor = 0.2
-- autovacuum_analyze_scale_factor = 0.1

-- Посмотреть статистику VACUUM
SELECT 
    schemaname,
    relname,
    last_vacuum,
    last_autovacuum,
    last_analyze,
    last_autoanalyze,
    n_dead_tup
FROM pg_stat_user_tables
ORDER BY n_dead_tup DESC;
```

### OPTIMIZE TABLE (MySQL специфика)

```sql
-- Дефрагментация и обновление статистики
OPTIMIZE TABLE users;

-- Для InnoDB:
-- - Rebuilds table (перестройка таблицы)
-- - Reclaims unused space
-- - Updates index statistics

-- Автоматическая статистика
-- innodb_stats_auto_recalc = ON (по умолчанию)
```

---

## 🔒 Транзакции: отличия

### Isolation Levels

**PostgreSQL:**
```sql
-- Поддерживаемые уровни:
-- - READ UNCOMMITTED (ведет себя как READ COMMITTED)
-- - READ COMMITTED (по умолчанию)
-- - REPEATABLE READ (нет phantom reads в PostgreSQL!)
-- - SERIALIZABLE (полная изоляция)

-- Установить уровень изоляции
BEGIN TRANSACTION ISOLATION LEVEL SERIALIZABLE;
SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;

-- Проверить текущий уровень
SHOW transaction_isolation;

-- ⚠️ В PostgreSQL REPEATABLE READ предотвращает phantom reads!
-- Это сильнее чем в MySQL REPEATABLE READ
```

**MySQL (InnoDB):**
```sql
-- Поддерживаемые уровни:
-- - READ UNCOMMITTED (грязное чтение возможно)
-- - READ COMMITTED
-- - REPEATABLE READ (по умолчанию, но phantom reads возможны!)
-- - SERIALIZABLE

-- Установить уровень изоляции
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;
START TRANSACTION;

-- Проверить
SELECT @@transaction_isolation;

-- ⚠️ В MySQL REPEATABLE READ НЕ предотвращает phantom reads полностью
-- Next-key locks используются для предотвращения некоторых phantom reads
```

### Блокировки

**PostgreSQL:**
```sql
-- Row-level locks
SELECT * FROM users WHERE id = 1 FOR UPDATE;         -- exclusive lock, блокирует SELECT FOR UPDATE/SHARE
SELECT * FROM users WHERE id = 1 FOR NO KEY UPDATE;  -- не блокирует FK references
SELECT * FROM users WHERE id = 1 FOR SHARE;          -- shared lock, можно читать
SELECT * FROM users WHERE id = 1 FOR KEY SHARE;      -- для FK, не блокирует обновление non-key columns

-- SKIP LOCKED (PostgreSQL 9.5+) - пропустить заблокированные строки
SELECT * FROM tasks 
WHERE status = 'pending' 
ORDER BY created_at 
LIMIT 1 
FOR UPDATE SKIP LOCKED;

-- NOWAIT - вернуть ошибку если заблокировано
SELECT * FROM users WHERE id = 1 FOR UPDATE NOWAIT;

-- Table-level locks
LOCK TABLE users IN ACCESS EXCLUSIVE MODE;  -- самая строгая блокировка
```

**MySQL:**
```sql
-- Row-level locks (InnoDB)
SELECT * FROM users WHERE id = 1 FOR UPDATE;    -- exclusive lock
SELECT * FROM users WHERE id = 1 LOCK IN SHARE MODE;  -- shared lock (deprecated)
SELECT * FROM users WHERE id = 1 FOR SHARE;     -- MySQL 8.0+ синоним

-- SKIP LOCKED (MySQL 8.0+)
SELECT * FROM tasks 
WHERE status = 'pending' 
ORDER BY created_at 
LIMIT 1 
FOR UPDATE SKIP LOCKED;

-- NOWAIT (MySQL 8.0+)
SELECT * FROM users WHERE id = 1 FOR UPDATE NOWAIT;

-- Table-level locks
LOCK TABLES users WRITE;  -- exclusive
LOCK TABLES users READ;   -- shared
UNLOCK TABLES;
```

---

## 🎓 Когда выбирать PostgreSQL vs MySQL

### PostgreSQL выбирай если:
- ✅ Сложные аналитические запросы (window functions, CTE, LATERAL)
- ✅ JSON/JSONB документы (document database внутри RDBMS)
- ✅ Полнотекстовый поиск (встроенный FTS мощнее MySQL)
- ✅ Геоданные (PostGIS - лучший выбор для GIS)
- ✅ Массивы, custom types, расширения (pgcrypto, uuid-ossp, hstore)
- ✅ Строгая консистентность и data integrity
- ✅ MVCC без gap locks (меньше блокировок)
- ✅ Partial indexes, expression indexes, covering indexes
- ✅ Поддержка SQL стандартов (строгая)

### MySQL выбирай если:
- ✅ Простые CRUD с огромной нагрузкой на чтение
- ✅ Репликация master-slave из коробки проще
- ✅ Нужна простота администрирования
- ✅ Легаси-системы и совместимость (WordPress, Drupal)
- ✅ Clustering (Galera Cluster, MySQL Cluster)
- ✅ Меньший footprint памяти для маленьких БД

### Laravel и популярные CMS:
- **Laravel** - отлично работает с обоими (Query Builder абстрагирует)
- **WordPress, Drupal, Joomla** - только MySQL (легаси-код завязан)
- **Symfony, Django, Rails** - часто используют PostgreSQL

---

## 📚 Ресурсы для изучения

1. **PostgreSQL Documentation** - https://www.postgresql.org/docs/
2. **MySQL Documentation** - https://dev.mysql.com/doc/
3. **Use The Index, Luke** - https://use-the-index-luke.com/ (индексы для обоих)
4. **Postgres Weekly** - https://postgresweekly.com/
5. **High Performance MySQL** (книга O'Reilly)
6. **PostgreSQL: Up and Running** (книга O'Reilly)

---

## 🔥 Важные различия для собеседования

| Аспект | PostgreSQL | MySQL |
|--------|-----------|-------|
| **AUTO_INCREMENT** | `SERIAL` / `IDENTITY` | `AUTO_INCREMENT` |
| **LIMIT** | `LIMIT 10 OFFSET 20` | `LIMIT 20, 10` или `LIMIT 10 OFFSET 20` |
| **String concat** | `\|\|` или `concat()` | `concat()` |
| **BOOLEAN** | `TRUE`/`FALSE` (настоящий boolean) | `TINYINT(1)` (0/1) |
| **UPSERT** | `ON CONFLICT DO UPDATE` | `ON DUPLICATE KEY UPDATE` |
| **RETURNING** | ✅ Есть | ❌ Нет |
| **CTE (WITH)** | ✅ С 8.4 | ✅ С 8.0 |
| **Window Functions** | ✅ С 8.4 | ✅ С 8.0 |
| **LATERAL JOIN** | ✅ С 9.3 | ✅ С 8.0.14 |
| **Arrays** | ✅ Native support | ❌ Нет (JSON workaround) |
| **JSON** | `JSON` + `JSONB` (бинарный, индексируемый) | `JSON` (бинарный с 5.7) |
| **Full-Text Search** | ✅ Очень мощный | ✅ Базовый (InnoDB FTS) |
| **Partial indexes** | ✅ Есть | ❌ Нет |
| **Expression indexes** | ✅ Есть | ✅ С 8.0.13 (functional) |
| **Репликация** | Streaming, Logical | Async/Semi-sync, GTID, Group Replication |
| **VACUUM** | ✅ Обязательно | ❌ Не нужно |
| **Storage Engine** | ❌ Один (PostgreSQL) | ✅ Множество (InnoDB, MyISAM, Memory) |

---

---

## 🔥 Частые вопросы на собеседованиях

**Q: Чем PostgreSQL JSONB отличается от MySQL JSON?**
A: JSONB - бинарный формат, индексируется через GIN, операторы @>, ->, ->>, быстрее для запросов. MySQL JSON тоже бинарный с 5.7, но индексы только через generated columns.

**Q: Как сделать UPSERT в PostgreSQL vs MySQL?**
A: PostgreSQL - `ON CONFLICT DO UPDATE`, MySQL - `ON DUPLICATE KEY UPDATE`. PostgreSQL мощнее (можно указать конкретный constraint).

**Q: Что такое RETURNING в PostgreSQL?**
A: Возвращает вставленные/обновлённые данные (`RETURNING id, created_at`). В MySQL нет, используй `LAST_INSERT_ID()`.

**Q: Зачем PostgreSQL нужен VACUUM, а MySQL нет?**
A: PostgreSQL использует MVCC с tuple versioning (старые версии остаются). MySQL InnoDB использует undo logs + purge thread (автоматическая очистка).

**Q: Чем REPEATABLE READ в PostgreSQL отличается от MySQL?**
A: PostgreSQL предотвращает phantom reads на RR (благодаря MVCC), MySQL - нет (используется next-key locking, но не полная защита).

**Q: Когда использовать PostgreSQL vs MySQL?**
A: PostgreSQL - сложные запросы, JSON, full-text search, strict consistency. MySQL - простые CRUD, read-heavy, простота настройки, WordPress/легаси.

---

## 🎓 Для собеседования: ключевые точки

1. **AUTO_INCREMENT** - PostgreSQL: SERIAL/IDENTITY, MySQL: AUTO_INCREMENT
2. **MVCC** - PostgreSQL: tuple versioning + VACUUM, MySQL: undo logs + purge
3. **JSON** - PostgreSQL: JSONB + GIN индексы, MySQL: JSON + generated columns
4. **UPSERT** - PostgreSQL: ON CONFLICT, MySQL: ON DUPLICATE KEY  
5. **RETURNING** - PostgreSQL: встроенный, MySQL: LAST_INSERT_ID()
6. **Window Functions** - оба с 8.0+, PostgreSQL раньше (8.4)
7. **Arrays** - PostgreSQL: native, MySQL: нет (используй JSON)
8. **Full-Text Search** - PostgreSQL: мощный (tsvector), MySQL: базовый (FULLTEXT)
9. **Partial Indexes** - PostgreSQL: да, MySQL: нет
10. **REPEATABLE READ** - PostgreSQL: предотвращает phantom reads, MySQL: нет
11. **Репликация** - PG: streaming/logical, MySQL: async/semi-sync/GTID
12. **Storage Engine** - PostgreSQL: один, MySQL: InnoDB/MyISAM/Memory

**Главное:** Понимай trade-offs между MVCC реализациями, знай когда выбрать какую БД, умей читать EXPLAIN ANALYZE.
