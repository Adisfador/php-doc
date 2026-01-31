# Индексы в базах данных (Deep Dive)

## Типы индексов

### 1. B-Tree индекс (по умолчанию)

**Структура:** Balanced Tree (сбалансированное дерево)

```
            [50]
           /    \
       [20,35]  [70,90]
       /  |  \   /  |  \
    [10][30][40][60][80][100]
```

**Когда использовать:**
- Сравнения: `=`, `<`, `>`, `<=`, `>=`, `BETWEEN`
- `LIKE 'text%'` (начинается с)
- `ORDER BY`
- `IS NULL` / `IS NOT NULL`

**Примеры:**
```sql
-- Создание B-Tree индекса
CREATE INDEX idx_users_email ON users(email);

-- Composite (составной) индекс
CREATE INDEX idx_orders_user_date ON orders(user_id, created_at);

-- Partial (частичный) индекс
CREATE INDEX idx_active_users ON users(email) WHERE is_active = true;

-- Covering index (включает дополнительные столбцы)
CREATE INDEX idx_users_email_include ON users(email) INCLUDE (name, created_at);
```

**Принцип работы:**
1. Root node → Internal nodes → Leaf nodes
2. Leaf nodes содержат TID (указатели на строки в таблице)
3. Leaf nodes связаны (для range queries)

**Размер индекса:**
- Зависит от количества строк + размера индексируемых колонок
- Обычно ~10-30% от размера таблицы

### 2. Hash индекс

**Структура:** Hash table

```
hash('alice@mail.com') → bucket 123 → [TID1, TID5]
hash('bob@mail.com')   → bucket 456 → [TID2]
```

**Когда использовать:**
- ТОЛЬКО для `=` (equality)
- НЕ работает для: `<`, `>`, `LIKE`, `ORDER BY`

**PostgreSQL:**
```sql
CREATE INDEX idx_users_email_hash ON users USING HASH(email);
```

**Внимание:** В PostgreSQL Hash индексы редко используются, т.к. B-Tree почти так же быстр для `=`.

### 3. GiST (Generalized Search Tree)

**Назначение:** Универсальный индекс для сложных типов данных

**Использование:**
- Геометрические типы (PostGIS)
- Full-text search
- Range types
- ltree (иерархии)

```sql
-- Геометрия
CREATE INDEX idx_locations_point ON locations USING GIST(coordinates);

SELECT * FROM locations 
WHERE coordinates <-> point(55.7558, 37.6173) < 1000;  -- В радиусе 1км

-- Full-text search
CREATE INDEX idx_articles_fts ON articles USING GIST(to_tsvector('english', content));

SELECT * FROM articles 
WHERE to_tsvector('english', content) @@ to_tsquery('postgresql & index');

-- Range types
CREATE INDEX idx_events_period ON events USING GIST(event_period);

SELECT * FROM events 
WHERE event_period && '[2024-01-01, 2024-01-31]'::daterange;
```

### 4. GIN (Generalized Inverted Index)

**Назначение:** Для составных значений (массивы, JSONB, full-text)

**Структура:**
```
Inverted index:
'postgresql' → [doc1, doc5, doc10]
'database'   → [doc1, doc3, doc8]
'index'      → [doc1, doc5, doc9]
```

**Использование:**
```sql
-- JSONB
CREATE INDEX idx_users_metadata ON users USING GIN(metadata jsonb_path_ops);

SELECT * FROM users WHERE metadata @> '{"country": "Russia"}';

-- Массивы
CREATE INDEX idx_posts_tags ON posts USING GIN(tags);

SELECT * FROM posts WHERE tags @> ARRAY['postgresql', 'database'];

-- Full-text search (лучше чем GiST для FTS)
CREATE INDEX idx_articles_fts ON articles USING GIN(to_tsvector('english', content));
```

**GIN vs GiST для full-text:**
- **GIN:** быстрее для поиска, медленнее для вставки, больше размер
- **GiST:** медленнее для поиска, быстрее для вставки, меньше размер

### 5. SP-GiST (Space-Partitioned GiST)

**Назначение:** Для неравномерно распределенных данных

**Использование:**
- IP адреса (inet)
- Телефонные номера
- Данные с префиксами

```sql
CREATE INDEX idx_logs_ip ON logs USING SPGIST(ip_address);

SELECT * FROM logs WHERE ip_address << '192.168.1.0/24';
```

### 6. BRIN (Block Range Index)

**Назначение:** Для очень больших таблиц с физически отсортированными данными

**Структура:**
```
Pages 0-999:    [min=1, max=1000]
Pages 1000-1999: [min=1001, max=2000]
Pages 2000-2999: [min=2001, max=3000]
```

**Когда использовать:**
- Огромные таблицы (TB)
- Данные вставляются в хронологическом порядке (логи, time-series)
- Столбцы коррелируют с физическим порядком

```sql
CREATE INDEX idx_logs_created ON logs USING BRIN(created_at);

-- Очень маленький размер индекса (в тысячи раз меньше B-Tree)
```

**Плюсы:**
- Минимальный размер
- Быстрая вставка
- Низкое потребление памяти

**Минусы:**
- Медленнее для точечных запросов
- Требует корреляции с физическим порядком

### 7. Bloom Filter Index (PostgreSQL extension)

**Назначение:** Для multi-column queries с OR условиями

```sql
CREATE EXTENSION bloom;

CREATE INDEX idx_users_bloom ON users 
USING bloom(country, city, age, gender);

-- Эффективно для:
SELECT * FROM users 
WHERE country = 'Russia' OR city = 'Moscow' OR age = 25;
```

---

## Составные индексы (Composite/Multi-column)

### Правило левого префикса (Leftmost Prefix Rule)

```sql
CREATE INDEX idx_users ON users(country, city, age);

-- ✅ Использует индекс
SELECT * FROM users WHERE country = 'Russia';
SELECT * FROM users WHERE country = 'Russia' AND city = 'Moscow';
SELECT * FROM users WHERE country = 'Russia' AND city = 'Moscow' AND age = 25;

-- ❌ НЕ использует индекс
SELECT * FROM users WHERE city = 'Moscow';
SELECT * FROM users WHERE age = 25;
SELECT * FROM users WHERE city = 'Moscow' AND age = 25;
```

**Порядок колонок важен!**

### Как выбрать порядок колонок

1. **Самая селективная колонка первой:**
```sql
-- Плохо (city = 10000 значений, gender = 2 значения)
CREATE INDEX idx_bad ON users(gender, city);

-- Хорошо
CREATE INDEX idx_good ON users(city, gender);
```

2. **Частота использования:**
```sql
-- Если чаще запросы WHERE country = ?
CREATE INDEX idx_users ON users(country, city, age);
```

3. **Сортировка:**
```sql
-- Для ORDER BY created_at DESC
CREATE INDEX idx_posts ON posts(user_id, created_at DESC);
```

---

## Частичные индексы (Partial Index)

Индексируют только часть строк.

```sql
-- Индекс только активных пользователей
CREATE INDEX idx_active_users ON users(email) WHERE is_active = true;

-- Использование
SELECT * FROM users WHERE email = 'test@mail.com' AND is_active = true;
-- ✅ Использует индекс

SELECT * FROM users WHERE email = 'test@mail.com';
-- ❌ НЕ использует этот индекс (нет условия is_active = true)
```

**Преимущества:**
- Меньше размер индекса
- Быстрее вставка/обновление
- Целевая оптимизация

**Примеры использования:**
```sql
-- Только неудаленные записи
CREATE INDEX idx_posts_active ON posts(id) WHERE deleted_at IS NULL;

-- Только премиум пользователи
CREATE INDEX idx_premium_users ON users(created_at) WHERE subscription_type = 'premium';

-- Только ошибки в логах
CREATE INDEX idx_error_logs ON logs(created_at) WHERE level = 'ERROR';
```

---

## Expression Index (Функциональный индекс)

Индекс на результате выражения/функции.

```sql
-- Поиск без учета регистра
CREATE INDEX idx_users_email_lower ON users(LOWER(email));

SELECT * FROM users WHERE LOWER(email) = 'test@mail.com';

-- Извлечение из JSON
CREATE INDEX idx_users_country ON users((metadata->>'country'));

SELECT * FROM users WHERE metadata->>'country' = 'Russia';

-- Вычисляемое поле
CREATE INDEX idx_users_full_name ON users((first_name || ' ' || last_name));

SELECT * FROM users WHERE (first_name || ' ' || last_name) = 'John Doe';
```

---

## Covering Index (Index-Only Scan)

Индекс содержит все необходимые данные → можно не читать таблицу.

```sql
-- PostgreSQL 11+: INCLUDE
CREATE INDEX idx_users_email_include ON users(email) 
INCLUDE (name, created_at);

-- Запрос
SELECT name, created_at FROM users WHERE email = 'test@mail.com';
-- Index-Only Scan! Не читает heap таблицу
```

**Преимущества:**
- Быстрее (меньше I/O)
- Эффективнее кеширование

**Как проверить:**
```sql
EXPLAIN (ANALYZE, BUFFERS) 
SELECT name FROM users WHERE email = 'test@mail.com';

-- Смотрим: Index-Only Scan vs Index Scan
```

---

## Уникальные индексы

```sql
-- Уникальный индекс
CREATE UNIQUE INDEX idx_users_email_unique ON users(email);

-- Частичный уникальный (soft delete паттерн)
CREATE UNIQUE INDEX idx_users_email_active ON users(email) 
WHERE deleted_at IS NULL;

-- Составной уникальный
CREATE UNIQUE INDEX idx_order_items ON order_items(order_id, product_id);
```

---

## Оптимизация индексов

### 1. Анализ использования индексов

```sql
-- PostgreSQL: неиспользуемые индексы
SELECT 
    schemaname,
    tablename,
    indexname,
    idx_scan,
    idx_tup_read,
    idx_tup_fetch,
    pg_size_pretty(pg_relation_size(indexrelid)) AS index_size
FROM pg_stat_user_indexes
WHERE idx_scan = 0
ORDER BY pg_relation_size(indexrelid) DESC;

-- MySQL
SELECT 
    OBJECT_SCHEMA,
    OBJECT_NAME,
    INDEX_NAME
FROM performance_schema.table_io_waits_summary_by_index_usage
WHERE INDEX_NAME IS NOT NULL
    AND COUNT_STAR = 0
    AND OBJECT_SCHEMA != 'mysql'
ORDER BY OBJECT_SCHEMA, OBJECT_NAME;
```

### 2. Bloat (раздутость индексов)

После множества UPDATE/DELETE индексы раздуваются.

```sql
-- PostgreSQL: проверка bloat
SELECT
    schemaname,
    tablename,
    indexname,
    pg_size_pretty(pg_relation_size(indexrelid)) AS size,
    100 * (pg_relation_size(indexrelid)::float / 
           NULLIF(pg_relation_size(tablename::regclass), 0)) AS bloat_ratio
FROM pg_stat_user_indexes
ORDER BY pg_relation_size(indexrelid) DESC;

-- Исправление: REINDEX
REINDEX INDEX idx_users_email;
REINDEX TABLE users;

-- Без блокировки (PostgreSQL 12+)
REINDEX INDEX CONCURRENTLY idx_users_email;
```

### 3. Fillfactor

Резервное место на странице индекса для UPDATE (уменьшает bloat).

```sql
-- 90% заполнение (10% резерв)
CREATE INDEX idx_users_email ON users(email) WITH (fillfactor = 90);
```

**Когда использовать:**
- Частые UPDATE индексированных колонок
- По умолчанию 90 для B-Tree

### 4. Statistics Target

Определяет точность статистики для query planner.

```sql
-- По умолчанию 100
ALTER TABLE users ALTER COLUMN email SET STATISTICS 1000;

-- Пересчитать статистику
ANALYZE users;
```

---

## Мониторинг и диагностика

### EXPLAIN ANALYZE

```sql
EXPLAIN (ANALYZE, BUFFERS, VERBOSE) 
SELECT * FROM users WHERE email = 'test@mail.com';
```

**Что смотреть:**
- `Seq Scan` vs `Index Scan` vs `Index-Only Scan`
- `Actual time` (реальное время)
- `Buffers` (сколько блоков прочитано)
- `Rows` (планируемые vs фактические строки)

### pg_stat_statements (PostgreSQL)

```sql
CREATE EXTENSION pg_stat_statements;

-- Самые медленные запросы
SELECT 
    query,
    calls,
    total_exec_time,
    mean_exec_time,
    max_exec_time
FROM pg_stat_statements
ORDER BY mean_exec_time DESC
LIMIT 10;
```

### Missing Indexes (PostgreSQL)

```sql
-- Таблицы со много Seq Scans
SELECT 
    schemaname,
    tablename,
    seq_scan,
    seq_tup_read,
    idx_scan,
    seq_tup_read / seq_scan AS avg_seq_tup
FROM pg_stat_user_tables
WHERE seq_scan > 0
ORDER BY seq_tup_read DESC
LIMIT 20;
```

---

## Best Practices

### ✅ DO

1. **Индексируйте foreign keys**
```sql
CREATE INDEX idx_orders_user_id ON orders(user_id);
```

2. **Используйте WHERE часто**
```sql
CREATE INDEX idx_orders_status ON orders(status) WHERE status != 'completed';
```

3. **Covering indexes для часто используемых SELECT**
```sql
CREATE INDEX idx_users_lookup ON users(email) INCLUDE (name, role);
```

4. **Composite для частых комбинаций**
```sql
CREATE INDEX idx_logs_user_date ON logs(user_id, created_at DESC);
```

5. **Partial для soft deletes**
```sql
CREATE UNIQUE INDEX idx_email_active ON users(email) WHERE deleted_at IS NULL;
```

### ❌ DON'T

1. **Не индексируйте низкоселективные колонки**
```sql
-- Плохо (gender: только 2-3 значения)
CREATE INDEX idx_bad ON users(gender);
```

2. **Не создавайте дубликаты**
```sql
-- Дубликат (первая колонка idx_composite)
CREATE INDEX idx_user_id ON orders(user_id);
CREATE INDEX idx_composite ON orders(user_id, status);  -- Уже включает user_id
```

3. **Не индексируйте всё**
- Индексы замедляют INSERT/UPDATE/DELETE
- Занимают место
- Нужно поддерживать

---

## Laravel Migrations

```php
Schema::table('users', function (Blueprint $table) {
    // B-Tree (по умолчанию)
    $table->index('email');
    
    // Уникальный
    $table->unique('email');
    
    // Составной
    $table->index(['country', 'city']);
    
    // Partial (через raw)
    DB::statement('CREATE INDEX idx_active_users ON users(email) WHERE is_active = true');
    
    // Full-text (MySQL)
    $table->fullText('content');
    
    // Spatial (MySQL)
    $table->spatialIndex('coordinates');
});
```

---

## Ключевые вопросы для интервью

- Какие типы индексов вы знаете?
- Когда использовать B-Tree, а когда Hash?
- Что такое Covering Index и зачем он нужен?
- Объясните Leftmost Prefix Rule для составных индексов
- В каком порядке размещать колонки в составном индексе?
- Что такое Index-Only Scan?
- Когда использовать BRIN индекс?
- Чем GIN отличается от GiST?
- Что такое partial index и когда его использовать?
- Как найти неиспользуемые индексы?
- Что такое index bloat и как с ним бороться?
