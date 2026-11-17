# Оптимизация и обслуживание БД

Детальный разбор оптимизации запросов, обслуживания, мониторинга производительности PostgreSQL и MySQL.

---

## 🎯 Методология оптимизации

### Цикл оптимизации

1. **Измерить** - найти медленные запросы (slow query log, pg_stat_statements)
2. **Анализировать** - EXPLAIN ANALYZE, понять узкое место
3. **Оптимизировать** - индексы, переписать запрос, изменить конфигурацию
4. **Проверить** - измерить снова, убедиться в улучшении
5. **Мониторить** - следить за метриками в production

### Правило оптимизации

**Не оптимизируй без измерений!** Преждевременная оптимизация - корень зла.

---

## 📊 EXPLAIN и EXPLAIN ANALYZE

### PostgreSQL: чтение плана выполнения

```sql
-- Базовый EXPLAIN - план БЕЗ выполнения
EXPLAIN
SELECT u.username, COUNT(p.id) as post_count
FROM users u
LEFT JOIN posts p ON u.id = p.user_id
WHERE u.created_at > '2025-01-01'
GROUP BY u.id, u.username;

-- Вывод:
-- GroupAggregate  (cost=1000.00..2000.00 rows=100 width=24)
--   Group Key: u.id
--   ->  Sort  (cost=900.00..950.00 rows=1000 width=16)
--         Sort Key: u.id
--         ->  Hash Left Join  (cost=100.00..800.00 rows=1000 width=16)
--               Hash Cond: (p.user_id = u.id)
--               ->  Seq Scan on posts p  (cost=0.00..500.00 rows=10000 width=8)
--               ->  Hash  (cost=50.00..50.00 rows=100 width=12)
--                     ->  Seq Scan on users u  (cost=0.00..50.00 rows=100 width=12)
--                           Filter: (created_at > '2025-01-01'::date)
```

**Что значат колонки:**
- **cost** - оценка стоимости (startup_cost..total_cost)
  - `cost=0.00..50.00` - начальная стоимость 0, общая 50
  - Условные единицы (не секунды!), для сравнения планов
- **rows** - оценка количества строк
- **width** - средний размер строки в байтах

**EXPLAIN ANALYZE - реальное выполнение:**
```sql
EXPLAIN ANALYZE
SELECT u.username, COUNT(p.id) as post_count
FROM users u
LEFT JOIN posts p ON u.id = p.user_id
WHERE u.created_at > '2025-01-01'
GROUP BY u.id, u.username;

-- Вывод:
-- GroupAggregate  (cost=1000.00..2000.00 rows=100 width=24) (actual time=12.345..45.678 rows=95 loops=1)
--   Group Key: u.id
--   ->  Sort  (cost=900.00..950.00 rows=1000 width=16) (actual time=10.123..11.234 rows=980 loops=1)
--         Sort Key: u.id
--         Sort Method: quicksort  Memory: 71kB
--         ->  Hash Left Join  (cost=100.00..800.00 rows=1000 width=16) (actual time=2.345..8.901 rows=980 loops=1)
--               Hash Cond: (p.user_id = u.id)
--               ->  Seq Scan on posts p  (cost=0.00..500.00 rows=10000 width=8) (actual time=0.012..3.456 rows=9876 loops=1)
--               ->  Hash  (cost=50.00..50.00 rows=100 width=12) (actual time=0.789..0.789 rows=95 loops=1)
--                     Buckets: 1024  Batches: 1  Memory Usage: 12kB
--                     ->  Seq Scan on users u  (cost=0.00..50.00 rows=100 width=12) (actual time=0.023..0.567 rows=95 loops=1)
--                           Filter: (created_at > '2025-01-01'::date)
--                           Rows Removed by Filter: 5
-- Planning Time: 0.234 ms
-- Execution Time: 46.123 ms
```

**actual time** - реальное время выполнения в миллисекундах:
- `actual time=12.345..45.678` - первая строка за 12ms, все строки за 45ms
- **loops** - сколько раз выполнялся узел

**EXPLAIN с BUFFERS (очень важно!):**
```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM users WHERE email = 'test@example.com';

-- Вывод:
-- Index Scan using idx_users_email on users  (cost=0.29..8.30 rows=1 width=123) (actual time=0.034..0.035 rows=1 loops=1)
--   Index Cond: ((email)::text = 'test@example.com'::text)
--   Buffers: shared hit=4
-- Planning:
--   Buffers: shared hit=16
-- Planning Time: 0.123 ms
-- Execution Time: 0.056 ms

-- Buffers:
-- shared hit = прочитано из shared_buffers (память PostgreSQL) - ХОРОШО
-- shared read = прочитано с диска - ПЛОХО
-- shared dirtied = изменено в памяти
-- shared written = записано на диск
```

**Цель:** `shared hit` максимальный, `shared read` минимальный.

**Почему нельзя просто увеличить `shared_buffers`:**
- Память нужна и другим: `work_mem` (на каждую операцию каждого соединения), `maintenance_work_mem`, процессы backend. Нехватка RAM = swap = хуже.
- Двойное кеширование: страница может лежать и в `shared_buffers`, и в page cache ОС. Поэтому дефолт ~25% RAM, остальное - кешу ОС. Кроме того, `shared read` не всегда означает реальное чтение с диска - данные могут прийти из кеша ОС (`effective_cache_size` сообщает об этом планировщику).
- Убывающая отдача: если горячий набор данных уже помещается в буфер, дальнейший рост hit ratio не даст. Холодные данные (одноразовый seq scan по большой таблице) дадут `read` при любом размере.
- Накладные расходы: больше грязных страниц между checkpoint (всплески записи), дороже управление буфером, дольше прогрев после рестарта.
- Не лечит причину: высокий `read` часто из-за отсутствия индекса (Seq Scan). Сначала индекс и запрос, потом память.

**EXPLAIN с FORMAT JSON:**
```sql
EXPLAIN (ANALYZE, BUFFERS, FORMAT JSON)
SELECT * FROM users WHERE id = 1;
```

Удобно для парсинга и визуализации (https://explain.dalibo.com/, https://tatiyants.com/pev/).

### Типы сканирований (Scan Types)

**Seq Scan** - последовательное сканирование всей таблицы:
```sql
-- ПЛОХО на больших таблицах
Seq Scan on users  (cost=0.00..1234.56 rows=50000 width=123)
```
Когда появляется:
- Нет подходящего индекса
- Выбирается >10-15% строк таблицы (индекс не эффективен)
- Таблица маленькая (< 1000 строк) - Seq Scan быстрее

**Index Scan** - чтение через индекс:
```sql
-- ХОРОШО
Index Scan using idx_users_email on users  (cost=0.29..8.30 rows=1 width=123)
  Index Cond: ((email)::text = 'test@example.com'::text)
```
Читает индекс + переходит в таблицу за данными.

**Index Only Scan** - чтение только индекса (covering index):
```sql
-- ОТЛИЧНО (не нужно читать таблицу)
Index Only Scan using idx_users_email_covering on users  (cost=0.29..4.30 rows=1 width=24)
  Index Cond: (email = 'test@example.com'::text)
  Heap Fetches: 0  -- сколько раз пришлось читать таблицу (0 = отлично)
```

**Bitmap Index Scan + Bitmap Heap Scan** - чтение через bitmap:
```sql
-- Для средних выборок (1-10% таблицы)
Bitmap Heap Scan on users  (cost=100.00..500.00 rows=1000 width=123)
  Recheck Cond: (created_at > '2025-01-01'::date)
  ->  Bitmap Index Scan on idx_users_created_at  (cost=0.00..99.75 rows=1000 width=0)
        Index Cond: (created_at > '2025-01-01'::date)
```
1. Сканирует индекс, строит bitmap страниц
2. Читает страницы таблицы в порядке физического расположения (эффективнее Index Scan для больших выборок)

### Типы соединений (Join Types)

**Nested Loop** - вложенный цикл:
```sql
Nested Loop  (cost=0.29..1000.00 rows=100 width=50)
  ->  Seq Scan on users u  (cost=0.00..50.00 rows=10 width=25)
  ->  Index Scan using idx_posts_user_id on posts p  (cost=0.29..95.00 rows=10 width=25)
        Index Cond: (user_id = u.id)
```
- Для каждой строки из внешней таблицы ищет в внутренней
- **Хорош** когда внешняя таблица маленькая + есть индекс на внутренней
- **Плох** когда обе таблицы большие

**Hash Join** - соединение через hash-таблицу:
```sql
Hash Join  (cost=100.00..800.00 rows=1000 width=50)
  Hash Cond: (p.user_id = u.id)
  ->  Seq Scan on posts p  (cost=0.00..500.00 rows=10000 width=25)
  ->  Hash  (cost=50.00..50.00 rows=100 width=25)
        ->  Seq Scan on users u  (cost=0.00..50.00 rows=100 width=25)
```
1. Строит hash-таблицу из меньшей таблицы (users)
2. Сканирует большую таблицу (posts), ищет совпадения в hash
- **Хорош** для больших таблиц без индексов, equality joins
- Требует памяти (work_mem)

**Merge Join** - соединение отсортированных наборов:
```sql
Merge Join  (cost=500.00..1500.00 rows=1000 width=50)
  Merge Cond: (p.user_id = u.id)
  ->  Sort  (cost=400.00..425.00 rows=10000 width=25)
        Sort Key: p.user_id
        ->  Seq Scan on posts p  (cost=0.00..300.00 rows=10000 width=25)
  ->  Sort  (cost=50.00..52.50 rows=100 width=25)
        Sort Key: u.id
        ->  Seq Scan on users u  (cost=0.00..45.00 rows=100 width=25)
```
- Сортирует обе таблицы, потом мерджит
- **Хорош** если данные уже отсортированы (индексы)
- **Плох** если нужна дорогая сортировка

### MySQL: чтение EXPLAIN

```sql
EXPLAIN
SELECT u.username, COUNT(p.id) as post_count
FROM users u
LEFT JOIN posts p ON u.id = p.user_id
WHERE u.created_at > '2025-01-01'
GROUP BY u.id, u.username;

-- Вывод (табличный формат):
-- +----+-------------+-------+------+---------------+------+---------+------+------+----------+----------------------------------------------+
-- | id | select_type | table | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra                                        |
-- +----+-------------+-------+------+---------------+------+---------+------+------+----------+----------------------------------------------+
-- |  1 | SIMPLE      | u     | ALL  | PRIMARY       | NULL | NULL    | NULL | 1000 |    33.33 | Using where; Using temporary; Using filesort |
-- |  1 | SIMPLE      | p     | ref  | user_id       | user_id | 4    | u.id |   10 |   100.00 | Using index                                  |
-- +----+-------------+-------+------+---------------+------+---------+------+------+----------+----------------------------------------------+
```

**Колонки:**
- **id** - порядок выполнения (одинаковый id = одновременно)
- **select_type** - тип SELECT (SIMPLE, PRIMARY, SUBQUERY, DERIVED, UNION)
- **table** - какая таблица
- **type** - тип доступа (ОТ ЛУЧШЕГО К ХУДШЕМУ):
  - **system** - таблица из 1 строки (лучший)
  - **const** - поиск по PRIMARY KEY или UNIQUE (1 строка)
  - **eq_ref** - уникальное соединение по PRIMARY/UNIQUE
  - **ref** - соединение по не-уникальному индексу
  - **range** - диапазон по индексу (BETWEEN, IN, >, <)
  - **index** - полное сканирование индекса
  - **ALL** - полное сканирование таблицы (ХУДШИЙ, избегай!)

**possible_keys** - какие индексы МОГУТ быть использованы
**key** - какой индекс РЕАЛЬНО использован (NULL = нет индекса)
**rows** - оценка количества строк для сканирования
**filtered** - процент строк, отфильтрованных WHERE (100% = все подошли)
**Extra** - дополнительная информация:
  - **Using index** - covering index (отлично!)
  - **Using where** - фильтрация на уровне сервера (норма)
  - **Using temporary** - создается временная таблица (плохо)
  - **Using filesort** - сортировка не по индексу (плохо на больших данных)
  - **Using index condition** - Index Condition Pushdown (хорошо)
  - **Using join buffer** - нет индекса для JOIN (плохо)

**EXPLAIN ANALYZE (MySQL 8.0.18+):**
```sql
EXPLAIN ANALYZE
SELECT * FROM users WHERE email = 'test@example.com'\G

-- Вывод (древовидный формат с реальным временем):
-- -> Index lookup on users using idx_users_email (email='test@example.com')  (cost=0.35 rows=1) (actual time=0.023..0.024 rows=1 loops=1)
```

**EXPLAIN FORMAT=JSON:**
```sql
EXPLAIN FORMAT=JSON
SELECT * FROM users WHERE id = 1\G
```

Визуализация: https://dev.mysql.com/doc/workbench/en/wb-performance-explain.html

---

## 🔎 Поиск медленных запросов

### PostgreSQL: pg_stat_statements

**Установка расширения:**
```sql
-- В postgresql.conf добавить:
shared_preload_libraries = 'pg_stat_statements'
pg_stat_statements.track = all
pg_stat_statements.max = 10000

-- Перезапуск PostgreSQL, потом:
CREATE EXTENSION pg_stat_statements;
```

**Поиск медленных запросов:**
```sql
-- Топ-10 самых медленных запросов по общему времени
SELECT 
    query,
    calls,
    total_exec_time,
    mean_exec_time,
    max_exec_time,
    rows
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 10;

-- Топ-10 по среднему времени
SELECT 
    query,
    calls,
    mean_exec_time,
    stddev_exec_time
FROM pg_stat_statements
ORDER BY mean_exec_time DESC
LIMIT 10;

-- Запросы с высокой вариативностью времени (проблемы с планировщиком?)
SELECT 
    query,
    calls,
    mean_exec_time,
    stddev_exec_time,
    stddev_exec_time / NULLIF(mean_exec_time, 0) as coefficient_of_variation
FROM pg_stat_statements
WHERE calls > 100
ORDER BY coefficient_of_variation DESC
LIMIT 10;

-- Сбросить статистику
SELECT pg_stat_statements_reset();
```

### MySQL: Slow Query Log

**Включить slow query log:**
```ini
# В my.cnf (или my.ini):
[mysqld]
slow_query_log = 1
slow_query_log_file = /var/log/mysql/slow-query.log
long_query_time = 1                # запросы дольше 1 секунды
log_queries_not_using_indexes = 1  # логировать запросы без индексов
```

```sql
-- Или динамически в runtime:
SET GLOBAL slow_query_log = 'ON';
SET GLOBAL long_query_time = 1;
SET GLOBAL log_queries_not_using_indexes = 'ON';
```

**Анализ с помощью mysqldumpslow:**
```bash
# Топ-10 медленных запросов
mysqldumpslow -s t -t 10 /var/log/mysql/slow-query.log

# -s t - сортировка по времени (t = time, c = count, l = lock time)
# -t 10 - топ 10

# Топ-10 по среднему времени
mysqldumpslow -s at -t 10 /var/log/mysql/slow-query.log

# Только SELECT запросы
mysqldumpslow -s t -g "SELECT" /var/log/mysql/slow-query.log
```

**Анализ с pt-query-digest (Percona Toolkit):**
```bash
pt-query-digest /var/log/mysql/slow-query.log

# Вывод в файл
pt-query-digest /var/log/mysql/slow-query.log > report.txt

# Топ-10 запросов
pt-query-digest --limit 10 /var/log/mysql/slow-query.log

# Только SELECT
pt-query-digest --filter '$event->{arg} =~ m/^select/i' /var/log/mysql/slow-query.log
```

**Performance Schema (MySQL 5.6+):**
```sql
-- Включить (в my.cnf):
performance_schema = ON

-- Топ-10 медленных запросов из Performance Schema
SELECT 
    DIGEST_TEXT as query,
    COUNT_STAR as exec_count,
    AVG_TIMER_WAIT / 1000000000000 as avg_time_sec,
    MAX_TIMER_WAIT / 1000000000000 as max_time_sec
FROM performance_schema.events_statements_summary_by_digest
ORDER BY AVG_TIMER_WAIT DESC
LIMIT 10;

-- Сбросить статистику
TRUNCATE TABLE performance_schema.events_statements_summary_by_digest;
```

### Что есть из коробки, а что нужно включать

**PostgreSQL:**

| Источник                     | Статус         |
|------------------------------|----------------|
| `pg_stat_user_tables`        | из коробки     |
| `pg_stat_user_indexes`       | из коробки     |
| `pg_statio_*`                | из коробки     |
| `pg_stat_activity`           | из коробки     |
| `pg_locks`                   | из коробки     |
| `pg_stat_statements`         | нужно включить |
| `log_min_duration_statement` | нужно включить |
| `auto_explain`               | нужно включить |

- `pg_stat_statements`: `shared_preload_libraries` + рестарт + `CREATE EXTENSION`
- `log_min_duration_statement`: по умолчанию `-1` (выключен), рестарт не нужен
- `auto_explain`: подключается как расширение, пишет планы медленных запросов в лог

**MySQL:**

| Источник                        | Статус               |
|---------------------------------|----------------------|
| Performance Schema (с 5.6.6)    | из коробки           |
| `sys` schema (с 5.7)            | из коробки           |
| `information_schema`            | из коробки           |
| `SHOW GLOBAL STATUS`            | из коробки           |
| `SHOW PROCESSLIST`              | из коробки           |
| Slow query log                  | нужно включить       |
| `log_queries_not_using_indexes` | опционально (шумный) |

- Slow query log: `slow_query_log = ON` (по умолчанию выключен)

Счётчики PostgreSQL копятся с момента последнего `pg_stat_reset()` (или рестарта после краха), поэтому вывод «индекс не используется» делай только на достаточно длинном периоде статистики.

### PostgreSQL: проблемные таблицы и индексы

```sql
-- Таблицы, где Seq Scan преобладает над Index Scan (кандидаты на индекс)
SELECT
    relname AS table_name,
    seq_scan,
    seq_tup_read,
    idx_scan,
    n_live_tup
FROM pg_stat_user_tables
WHERE seq_scan > COALESCE(idx_scan, 0)
  AND n_live_tup > 10000
ORDER BY seq_tup_read DESC
LIMIT 20;

-- Неиспользуемые индексы (не считая тех, что обеспечивают PK/UNIQUE)
SELECT
    s.relname AS table_name,
    s.indexrelname AS index_name,
    s.idx_scan,
    pg_size_pretty(pg_relation_size(s.indexrelid)) AS index_size
FROM pg_stat_user_indexes s
JOIN pg_index i ON i.indexrelid = s.indexrelid
WHERE s.idx_scan = 0
  AND NOT i.indisunique
ORDER BY pg_relation_size(s.indexrelid) DESC;
```

Осторожно с удалением: проверь, что статистика не была недавно сброшена, и что индекс не используется только на реплике (там своя статистика).

### MySQL: sys schema (готовые представления)

`sys` построена поверх Performance Schema и отвечает на типовые вопросы без ручных запросов:

```sql
-- Неиспользуемые индексы
SELECT * FROM sys.schema_unused_indexes;

-- Дублирующиеся (избыточные) индексы
SELECT * FROM sys.schema_redundant_indexes;

-- Таблицы, по которым идут полные сканы
SELECT * FROM sys.schema_tables_with_full_table_scans;

-- Запросы с полным сканом таблицы
SELECT * FROM sys.statements_with_full_table_scans LIMIT 10;

-- Запросы с временными таблицами на диске и сортировкой без индекса
SELECT * FROM sys.statements_with_temp_tables LIMIT 10;
SELECT * FROM sys.statements_with_sorting LIMIT 10;

-- Топ запросов по общему времени
SELECT * FROM sys.statement_analysis LIMIT 10;
```

`sys.schema_unused_indexes` строится на `performance_schema.table_io_waits_summary_by_index_usage`, а статистика сбрасывается при рестарте сервера, так что то же правило: смотри после долгого аптайма.

---

## 🚀 Стратегии оптимизации запросов

### 1. Используй индексы правильно

**Плохо:**
```sql
-- Seq Scan, медленно на больших таблицах
SELECT * FROM users WHERE email = 'test@example.com';
```

**Хорошо:**
```sql
-- Создать индекс
CREATE INDEX idx_users_email ON users(email);

-- Index Scan, быстро
SELECT * FROM users WHERE email = 'test@example.com';
```

**Composite index для покрытия запроса:**
```sql
-- Запрос использует user_id + created_at
SELECT * FROM posts 
WHERE user_id = 123 
ORDER BY created_at DESC 
LIMIT 10;

-- Composite index (порядок важен!)
CREATE INDEX idx_posts_user_created ON posts(user_id, created_at DESC);
-- user_id первый (WHERE), created_at второй (ORDER BY)
```

**Covering index (Index Only Scan):**
```sql
-- Запрос нужны только email и username
SELECT email, username FROM users WHERE email = 'test@example.com';

-- PostgreSQL: INCLUDE
CREATE INDEX idx_users_email_covering ON users(email) INCLUDE (username);

-- MySQL: добавить в индекс
CREATE INDEX idx_users_email_covering ON users(email, username);

-- Теперь Index Only Scan - не нужно читать таблицу
```

### 2. Избегай функций на индексированных колонках

**Плохо (индекс НЕ используется):**
```sql
SELECT * FROM users WHERE LOWER(email) = 'test@example.com';
-- Seq Scan, т.к. функция LOWER() ломает индекс
```

**Хорошо (expression index в PostgreSQL):**
```sql
CREATE INDEX idx_users_email_lower ON users(LOWER(email));

SELECT * FROM users WHERE LOWER(email) = 'test@example.com';
-- Index Scan on idx_users_email_lower
```

**Хорошо (MySQL 8.0.13+ functional index):**
```sql
CREATE INDEX idx_users_email_lower ON users((LOWER(email)));

SELECT * FROM users WHERE LOWER(email) = 'test@example.com';
```

**Или нормализуй данные:**
```sql
-- Хранить email в lowercase при вставке
INSERT INTO users (email) VALUES (LOWER('Test@Example.com'));

-- Поиск без функции
SELECT * FROM users WHERE email = 'test@example.com';
-- Index Scan on idx_users_email
```

### 3. Избегай SELECT *

**Плохо:**
```sql
SELECT * FROM users WHERE id = 1;
-- Читает все колонки (может быть 50+ колонок с TEXT/BLOB)
```

**Хорошо:**
```sql
SELECT id, email, username FROM users WHERE id = 1;
-- Читает только нужные колонки, меньше I/O
```

**Еще лучше (covering index):**
```sql
CREATE INDEX idx_users_id_covering ON users(id) INCLUDE (email, username);

SELECT id, email, username FROM users WHERE id = 1;
-- Index Only Scan (не читает таблицу вообще)
```

### 4. N+1 Query Problem

**Плохо (N+1 запросов):**
```php
// Laravel Eloquent
$users = User::all();  // 1 запрос

foreach ($users as $user) {
    echo $user->posts->count();  // N запросов (по 1 на каждого user)
}
// Итого: 1 + N запросов
```

**Хорошо (Eager Loading):**
```php
$users = User::with('posts')->get();  // 2 запроса (users + posts)

foreach ($users as $user) {
    echo $user->posts->count();  // без запросов, данные уже загружены
}
```

**SQL эквивалент:**
```sql
-- Плохо (N+1):
SELECT * FROM users;  -- 1 запрос
-- Потом для каждого user:
SELECT * FROM posts WHERE user_id = 1;
SELECT * FROM posts WHERE user_id = 2;
-- ... N запросов

-- Хорошо (JOIN или 2 запроса):
-- Вариант 1: JOIN
SELECT 
    u.id,
    u.username,
    COUNT(p.id) as post_count
FROM users u
LEFT JOIN posts p ON u.id = p.user_id
GROUP BY u.id, u.username;

-- Вариант 2: 2 отдельных запроса (Eloquent подход)
SELECT * FROM users;
SELECT * FROM posts WHERE user_id IN (1, 2, 3, ...);  -- все users одним запросом
```

### 5. LIMIT + OFFSET vs Cursor Pagination

**Плохо (глубокая пагинация с OFFSET):**
```sql
-- Страница 10000 (offset 100000 строк)
SELECT * FROM posts 
ORDER BY created_at DESC 
LIMIT 10 OFFSET 100000;

-- БД должна прочитать и пропустить 100000 строк!
-- Чем дальше страница, тем медленнее
```

**Хорошо (cursor-based pagination):**
```sql
-- Страница 1
SELECT * FROM posts 
ORDER BY created_at DESC, id DESC 
LIMIT 10;
-- Вернет id последней записи, например 9876543

-- Страница 2 (курсор = 9876543), сравнение пары (created_at, id)
-- работает корректно и когда created_at не уникален:
SELECT * FROM posts 
WHERE (created_at, id) < (
    SELECT created_at, id FROM posts WHERE id = 9876543
)
ORDER BY created_at DESC, id DESC 
LIMIT 10;

-- Индекс для эффективности:
CREATE INDEX idx_posts_created_id ON posts(created_at DESC, id DESC);
```

**Laravel cursor pagination:**
```php
// Вместо:
$posts = Post::paginate(10);  // OFFSET под капотом

// Используй:
$posts = Post::cursorPaginate(10);  // cursor-based
```

### 6. Используй EXISTS вместо COUNT

**Плохо (считает все строки):**
```sql
SELECT COUNT(*) FROM posts WHERE user_id = 123;
-- Если результат > 0, значит есть посты

IF (SELECT COUNT(*) FROM posts WHERE user_id = 123) > 0 THEN
    -- user has posts
END IF;
```

**Хорошо (останавливается после первой найденной):**
```sql
SELECT EXISTS(SELECT 1 FROM posts WHERE user_id = 123);
-- Вернет TRUE или FALSE, остановится после первой строки
```

**В Laravel:**
```php
// Плохо
if (Post::where('user_id', 123)->count() > 0) { }

// Хорошо
if (Post::where('user_id', 123)->exists()) { }
```

### 7. OR в WHERE и индексы

**Суть проблемы:** при `OR` нужны строки, подходящие под *любую* ветку, поэтому **каждая ветка должна уметь использовать индекс сама**. Если хотя бы одна ветка не может, СУБД всё равно придётся пройти всю таблицу, и она выбирает Seq Scan для всего запроса.

```sql
SELECT * FROM users 
WHERE email = 'test@example.com' 
   OR username = 'testuser';
```

**Отдельные индексы на каждую колонку** (`email` и `username`):
- PostgreSQL может объединить их через **BitmapOr** (два Bitmap Index Scan -> один Bitmap Heap Scan)
- MySQL может объединить их через **index_merge** (`Using union(...)` в EXPLAIN)
- Но планировщик не обязан так делать: при плохой оценке селективности, сложных условиях или `ORDER BY ... LIMIT` он может выбрать Seq Scan

**Составной индекс `(email, username)` при OR не помогает:**
- Ветка `email = ...` использует индекс (левый префикс)
- Ветка `username = ...` не может: `username` вторая колонка, а условия на `email` в этой ветке нет, значения `username` разбросаны по всему индексу
- Итог: одна ветка без индекса -> Seq Scan для всего запроса
- Порядок веток значения не имеет: `username = ... OR email = ...` ведёт себя так же

```sql
-- индекс (email, username)
WHERE email = 'a' OR email = 'b'                       -- ок: у обеих веток есть левый префикс
WHERE email = 'a' OR (email = 'b' AND username = 'x')  -- ок: у обеих веток есть левый префикс
WHERE email = 'a' OR username = 'x'                    -- плохо: ветка username без email
```

Правило: составной индекс при OR работает, только если **у каждой ветки есть его левый префикс**. Для веток по разным колонкам нужны отдельные индексы.

**UNION как альтернатива:** каждая часть планируется независимо и идёт через свой индекс, без зависимости от того, как планировщик обработает OR.
```sql
SELECT * FROM users WHERE email = 'test@example.com'
UNION
SELECT * FROM users WHERE username = 'testuser';
```
- Тоже подразумевает два отдельных индекса (составной не поможет и здесь)
- `UNION` убирает дубли (доп. сортировка/hash); `UNION ALL` дешевле, но вернёт строку дважды, если совпали обе ветки

**OR по одной колонке** заменяется на `IN`:

```sql
SELECT * FROM users WHERE id IN (1, 2, 3, 4, 5);
-- Index Scan, эффективно
```

### 8. Денормализация для производительности

**Плохо (JOIN на каждом запросе):**
```sql
SELECT 
    p.title,
    u.username,
    u.avatar_url
FROM posts p
JOIN users u ON p.user_id = u.id
WHERE p.id = 123;

-- JOIN на каждом чтении поста
```

**Хорошо (денормализация):**
```sql
-- Добавить username и avatar_url прямо в posts
ALTER TABLE posts 
ADD COLUMN username VARCHAR(50),
ADD COLUMN avatar_url TEXT;

-- Обновлять при изменении user (trigger или application logic)
-- Теперь чтение без JOIN:
SELECT title, username, avatar_url FROM posts WHERE id = 123;
```

**Trade-off:**
- ✅ Быстрее чтение (нет JOIN)
- ❌ Сложнее обновление (синхронизация)
- ❌ Больше места (дублирование данных)

**Когда использовать:**
- Read-heavy приложения (чтение >> записи)
- Данные редко меняются (username, avatar_url)
- Критична скорость чтения (новостные ленты, комментарии)

### 9. Partial Index (PostgreSQL)

**Для фильтрации частых запросов:**
```sql
-- Запрос: только активные пользователи
SELECT * FROM users WHERE is_active = TRUE AND email = 'test@example.com';

-- Partial index только для is_active = TRUE
CREATE INDEX idx_users_email_active ON users(email) WHERE is_active = TRUE;

-- Индекс меньше (только активные), быстрее
```

**Для мягкого удаления:**
```sql
-- Колонка deleted_at
CREATE INDEX idx_users_email_not_deleted ON users(email) WHERE deleted_at IS NULL;

-- Запросы без удаленных:
SELECT * FROM users WHERE email = 'test@example.com' AND deleted_at IS NULL;
-- Index Scan on idx_users_email_not_deleted
```

### 10. Batch операции вместо циклов

**Плохо (N запросов в цикле):**
```php
foreach ($userIds as $userId) {
    DB::table('users')->where('id', $userId)->update(['is_active' => false]);
}
// N UPDATE запросов
```

**Хорошо (один запрос):**
```php
DB::table('users')->whereIn('id', $userIds)->update(['is_active' => false]);

// UPDATE users SET is_active = false WHERE id IN (1, 2, 3, ...);
```

**Для INSERT:**
```sql
-- Плохо (N запросов):
INSERT INTO users (email) VALUES ('user1@test.com');
INSERT INTO users (email) VALUES ('user2@test.com');
-- ... N раз

-- Хорошо (bulk insert):
INSERT INTO users (email) VALUES 
    ('user1@test.com'),
    ('user2@test.com'),
    ('user3@test.com');
-- Один запрос, N строк
```

---

## 🗄️ VACUUM и ANALYZE (PostgreSQL)

### MVCC и "мертвые" строки

**PostgreSQL использует MVCC (Multi-Version Concurrency Control):**
- При UPDATE/DELETE старая версия строки НЕ удаляется сразу
- Остается "мертвая" строка (dead tuple) для старых транзакций
- Накопление dead tuples увеличивает размер таблицы и индексов
- Замедляет запросы (больше данных для сканирования)

**VACUUM очищает dead tuples:**
```sql
-- VACUUM - удаляет dead tuples, освобождает место для новых данных
VACUUM users;

-- VACUUM FULL - полная перестройка таблицы, возвращает место ОС
VACUUM FULL users;  -- БЛОКИРУЕТ таблицу! Не используй в production без downtime

-- VACUUM ANALYZE - VACUUM + обновление статистики
VACUUM ANALYZE users;

-- Для всей БД
VACUUM;
VACUUM ANALYZE;
```

**Autovacuum (автоматический VACUUM):**
```conf
# Включен по умолчанию в postgresql.conf:
autovacuum = on
autovacuum_max_workers = 3  # количество параллельных vacuum процессов
autovacuum_naptime = 1min   # интервал проверки
```

```sql
-- Для каждой таблицы autovacuum запускается когда:
-- dead_tuples > autovacuum_vacuum_threshold + autovacuum_vacuum_scale_factor * tuples
-- По умолчанию: 50 + 0.2 * tuples (20% таблицы изменено)

-- Посмотреть когда был последний vacuum:
SELECT 
    schemaname,
    relname as table_name,
    last_vacuum,
    last_autovacuum,
    n_dead_tup as dead_tuples,
    n_live_tup as live_tuples,
    n_dead_tup::float / NULLIF(n_live_tup, 0) as dead_ratio
FROM pg_stat_user_tables
ORDER BY n_dead_tup DESC;

-- Таблицы с большим количеством dead tuples нуждаются в VACUUM
```

**Настройка autovacuum для конкретной таблицы:**
```sql
-- Более агрессивный autovacuum для часто обновляемых таблиц
ALTER TABLE sessions SET (
    autovacuum_vacuum_scale_factor = 0.1,  -- vacuum при 10% изменений (вместо 20%)
    autovacuum_vacuum_threshold = 100      -- минимум 100 dead tuples
);

-- Отключить autovacuum (осторожно!)
ALTER TABLE logs SET (autovacuum_enabled = false);
```

### ANALYZE - обновление статистики планировщика

**Планировщик использует статистику для выбора плана:**
- Количество строк в таблице
- Распределение значений в колонках
- Корреляция между колонками
- NULL значения

**Устаревшая статистика = плохие планы запросов!**

```sql
-- ANALYZE - собрать статистику
ANALYZE users;

-- Auto-analyze (автоматический, включен по умолчанию)
-- Запускается после autovacuum_analyze_threshold + autovacuum_analyze_scale_factor * tuples
-- По умолчанию: 50 + 0.1 * tuples (10% изменений)

-- Посмотреть последний analyze:
SELECT 
    schemaname,
    relname,
    last_analyze,
    last_autoanalyze,
    n_mod_since_analyze  -- изменений с последнего analyze
FROM pg_stat_user_tables
ORDER BY n_mod_since_analyze DESC;

-- Статистика по колонке:
SELECT 
    attname as column_name,
    n_distinct,  -- количество уникальных значений (-1 = все уникальны)
    correlation  -- корреляция с физическим порядком (-1 to 1)
FROM pg_stats
WHERE tablename = 'users';
```

**Когда запускать ANALYZE вручную:**
- После bulk INSERT/UPDATE (загрузка данных)
- После DELETE большого количества строк
- Перед важными запросами (если знаешь что статистика устарела)

---

## 🔧 Настройки PostgreSQL для производительности

### Память

**postgresql.conf:**
```conf
# Shared buffers - кеш данных в памяти
shared_buffers = 4GB  # 25% RAM для dedicated сервера, 10-15% для shared

# Effective cache size - информация для планировщика о доступной ОС кеше
effective_cache_size = 12GB  # 50-75% RAM

# Work mem - память для сортировок и hash joins (НА ОПЕРАЦИЮ!)
work_mem = 64MB  # Осторожно! Много параллельных запросов = work_mem * connections
# Для аналитики можно больше (256MB-1GB)

# Maintenance work mem - для VACUUM, CREATE INDEX, ALTER TABLE
maintenance_work_mem = 1GB  # Можно больше (2-4GB)
```

**Как рассчитать:**
- **shared_buffers**: 25% RAM для dedicated, 10-15% для shared с другими приложениями
- **effective_cache_size**: 50-75% RAM (не аллоцируется, только информация для планировщика)
- **work_mem**: Осторожно! `work_mem * max_connections` не должно превышать RAM
  - Пример: 100 connections, work_mem=64MB => 6.4GB max
  - Лучше увеличивать для конкретных запросов: `SET work_mem = '256MB';`

### Checkpoint и WAL

```conf
# Write-Ahead Log (WAL) настройки
wal_buffers = 16MB           # буфер для WAL записей
min_wal_size = 1GB           # минимальный размер WAL
max_wal_size = 4GB           # максимальный (checkpoint когда достигнут)

# Checkpoint - сброс грязных страниц на диск
checkpoint_timeout = 15min   # максимальный интервал
checkpoint_completion_target = 0.9  # растянуть checkpoint на 90% интервала (меньше нагрузка)

# WAL писатель
wal_writer_delay = 200ms     # задержка записи WAL на диск
commit_delay = 0             # группировка коммитов (0 = отключено)
```

**Trade-offs:**
- Больше `max_wal_size` = реже checkpoint = меньше I/O, но дольше восстановление после краха
- Меньше `checkpoint_timeout` = чаще checkpoint = безопаснее, но больше I/O

### Connections и Workers

```conf
max_connections = 100        # максимум подключений (process-per-connection модель)
max_worker_processes = 8     # для параллельных запросов и утилит
max_parallel_workers_per_gather = 4  # параллельность на запрос
max_parallel_workers = 8     # общий лимит параллельных воркеров
```

**Connection Pooling (PgBouncer):**
- PostgreSQL fork процесс на каждое подключение (тяжело)
- Рекомендуется использовать PgBouncer для connection pooling
- `max_connections` в PostgreSQL можно уменьшить (20-50)
- PgBouncer управляет 1000+ клиентских соединений, использует 20-50 серверных

### Query Planning

```conf
# Random page cost (SSD vs HDD)
random_page_cost = 1.1       # SSD (дефолт 4.0 для HDD)
seq_page_cost = 1.0          # базовая стоимость seq scan

# Effective IO concurrency
effective_io_concurrency = 200  # SSD (дефолт 1 для HDD)

# Default statistics target
default_statistics_target = 100  # больше = точнее статистика, но дольше ANALYZE
```

---

## 🔧 Настройки MySQL (InnoDB) для производительности

### Память (my.cnf / my.ini)

```conf
[mysqld]
# InnoDB buffer pool - главный кеш (страницы данных и индексов)
innodb_buffer_pool_size = 8G  # 70-80% RAM для dedicated сервера

# InnoDB buffer pool instances - параллельность
innodb_buffer_pool_instances = 8  # 1GB на instance (8GB / 8 = 1GB)

# InnoDB log file size - размер redo log
innodb_log_file_size = 1G     # больше = меньше flushing, но дольше восстановление

# InnoDB flush method
innodb_flush_method = O_DIRECT  # Linux: избегает double buffering (OS cache + InnoDB)

# Query cache (deprecated в 5.7.20, УДАЛЁН в MySQL 8.0 - этих параметров там больше нет)
# query_cache_type = 0
# query_cache_size = 0
```

### Connections

```conf
max_connections = 150         # максимум подключений
max_connect_errors = 100000   # лимит ошибок подключения

# Thread cache
thread_cache_size = 8         # кеш потоков (избегать создания нового потока)

# Table cache
table_open_cache = 2000       # кеш открытых таблиц
table_definition_cache = 1000 # кеш определений таблиц
```

### InnoDB настройки

```conf
# InnoDB I/O capacity (SSD)
innodb_io_capacity = 2000     # IOPS для фоновых задач (flush, merge)
innodb_io_capacity_max = 4000 # максимум IOPS

# InnoDB flush log at transaction commit
innodb_flush_log_at_trx_commit = 1  # 1 = ACID (flush на каждый commit)
                                      # 0 = flush раз в секунду (быстрее, но можем потерять 1 сек транзакций)
                                      # 2 = flush в OS cache (средне)

# InnoDB file per table
innodb_file_per_table = 1     # каждая таблица в отдельном файле (удобнее управлять)

# InnoDB autoinc lock mode
innodb_autoinc_lock_mode = 2  # interleaved mode (быстрее для bulk insert, но не для statement-based replication)
```

### Временные таблицы

```conf
tmp_table_size = 64M          # максимальный размер временной таблицы в памяти
max_heap_table_size = 64M     # максимальный размер MEMORY таблицы
```

---

## 📊 Мониторинг производительности

### PostgreSQL: важные метрики

**1. Cache Hit Ratio (должен быть > 99%):**
```sql
SELECT 
    sum(heap_blks_read) as heap_read,
    sum(heap_blks_hit) as heap_hit,
    sum(heap_blks_hit) / NULLIF(sum(heap_blks_hit) + sum(heap_blks_read), 0) * 100 as cache_hit_ratio
FROM pg_statio_user_tables;

-- Если < 99%, увеличить shared_buffers
```

**2. Index Usage (каждая таблица должна использовать индексы):**
```sql
SELECT 
    schemaname,
    relname as table_name,
    indexrelname,
    idx_scan,           -- сколько раз индекс использован
    idx_tup_read,       -- строк прочитано через индекс
    idx_tup_fetch       -- строк извлечено из таблицы
FROM pg_stat_user_indexes
ORDER BY idx_scan;

-- Индексы с idx_scan = 0 не используются (можно удалить)
```

**3. Table Bloat (раздувание таблиц из-за dead tuples):**
```sql
SELECT 
    schemaname,
    relname as table_name,
    pg_size_pretty(pg_total_relation_size(relid)) as total_size,
    n_dead_tup,
    n_live_tup,
    round(n_dead_tup::numeric / NULLIF(n_live_tup, 0) * 100, 2) as dead_ratio
FROM pg_stat_user_tables
WHERE n_dead_tup > 1000
ORDER BY n_dead_tup DESC;

-- Если dead_ratio > 10-20%, нужен VACUUM
```

**4. Locks и блокировки:**
```sql
SELECT 
    locktype,
    database,
    relation::regclass,
    mode,
    granted,
    pid
FROM pg_locks
WHERE NOT granted  -- не полученные блокировки (ждут)
ORDER BY pid;

-- Активные запросы с блокировками
SELECT 
    pid,
    usename,
    application_name,
    state,
    query,
    now() - query_start as duration
FROM pg_stat_activity
WHERE state != 'idle'
  AND pid IN (SELECT pid FROM pg_locks WHERE NOT granted);
```

**5. Размеры таблиц и индексов:**
```sql
SELECT 
    schemaname,
    tablename,
    pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) as total_size,
    pg_size_pretty(pg_relation_size(schemaname||'.'||tablename)) as table_size,
    pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename) - pg_relation_size(schemaname||'.'||tablename)) as indexes_size
FROM pg_tables
ORDER BY pg_total_relation_size(schemaname||'.'||tablename) DESC
LIMIT 20;
```

### MySQL: важные метрики

**1. InnoDB Buffer Pool Hit Ratio (> 99%):**
```sql
SHOW GLOBAL STATUS LIKE 'Innodb_buffer_pool_read%';

-- Расчет:
-- hit_ratio = (Innodb_buffer_pool_read_requests - Innodb_buffer_pool_reads) / Innodb_buffer_pool_read_requests * 100
-- Если < 99%, увеличить innodb_buffer_pool_size
```

**2. Открытые таблицы:**
```sql
SHOW GLOBAL STATUS LIKE 'Open_tables';
SHOW GLOBAL STATUS LIKE 'Opened_tables';

-- Если Opened_tables постоянно растет, увеличить table_open_cache
```

**3. Временные таблицы на диске:**
```sql
SHOW GLOBAL STATUS LIKE 'Created_tmp%';

-- Created_tmp_disk_tables / Created_tmp_tables > 0.25 (25%) - плохо
-- Увеличить tmp_table_size и max_heap_table_size
```

**4. Slow queries:**
```sql
SHOW GLOBAL STATUS LIKE 'Slow_queries';

-- Должно быть минимально
```

**5. Размеры таблиц:**
```sql
SELECT 
    table_schema,
    table_name,
    table_rows,
    ROUND(data_length / 1024 / 1024, 2) as data_mb,
    ROUND(index_length / 1024 / 1024, 2) as index_mb,
    ROUND((data_length + index_length) / 1024 / 1024, 2) as total_mb
FROM information_schema.tables
WHERE table_schema NOT IN ('mysql', 'information_schema', 'performance_schema', 'sys')
ORDER BY (data_length + index_length) DESC
LIMIT 20;
```

---

## 🔌 Архитектура подключений: PostgreSQL vs MySQL

### Модели обработки подключений

**PostgreSQL: процессы (fork-based model)**
- Каждое подключение = отдельный процесс (fork от главного процесса)
- Полная изоляция памяти между процессами
- Используется механизм fork() операционной системы

**MySQL/InnoDB: потоки (thread-based model)**
- Каждое подключение = отдельный поток (thread) внутри одного процесса
- Общая память между потоками
- Более легковесная модель

### Технические различия

#### PostgreSQL: форки (процессы)

```
┌─────────────────────────────────────┐
│     Postmaster (главный процесс)    │
└─────────────────────────────────────┘
              │
    ┌─────────┼─────────┬─────────┐
    │         │         │         │
┌───▼───┐ ┌───▼───┐ ┌───▼───┐ ┌───▼───┐
│Process│ │Process│ │Process│ │Process│
│Client1│ │Client2│ │Client3│ │Client4│
│       │ │       │ │       │ │       │
│Memory │ │Memory │ │Memory │ │Memory │
│Isolated│ │Isolated│ │Isolated│ │Isolated│
└───────┘ └───────┘ └───────┘ └───────┘
```

**Характеристики:**
- 🔒 **Изоляция** - приватная память процесса (work_mem и др.), ошибка в запросе затрагивает только его backend. Но при аварийном падении backend (например, segfault) postmaster перезапускает все соединения и запускает crash recovery
- 💾 **Память** - у каждого процесса свои work_mem и temp_buffers; shared_buffers (кеш данных) при этом общий для всех процессов (shared memory)
- 🐌 **Создание** - fork() относительно медленный (копирование адресного пространства)
- 🔄 **Переключение** - context switch между процессами дороже, чем между потоками
- 📊 **Overhead** - больше потребление памяти на подключение (~10MB базовый overhead)
- 🛡️ **Надежность** - ошибка в одном запросе не влияет на другие; при аварийном крахе backend все соединения сбрасываются (crash recovery), данные остаются целыми благодаря WAL

#### MySQL: потоки (threads)

```
┌──────────────────────────────────────────────┐
│         MySQL Server (один процесс)          │
│  ┌────────┐  ┌────────┐  ┌────────┐         │
│  │Thread 1│  │Thread 2│  │Thread 3│         │
│  │Client 1│  │Client 2│  │Client 3│         │
│  └────────┘  └────────┘  └────────┘         │
│                                               │
│        Shared Memory (Buffer Pool)           │
│  ┌─────────────────────────────────────┐    │
│  │      InnoDB Buffer Pool             │    │
│  │      Query Cache (устарел)          │    │
│  │      Table Definitions              │    │
│  └─────────────────────────────────────┘    │
└──────────────────────────────────────────────┘
```

**Характеристики:**
- ⚡ **Скорость создания** - создание потока быстрее, чем fork()
- 💾 **Память** - меньше overhead на подключение (~256KB - 1MB)
- 🔄 **Переключение** - быстрое переключение контекста между потоками
- 🤝 **Общая память** - эффективное использование InnoDB buffer pool
- ⚠️ **Изоляция** - аварийное падение любого потока (segfault) убивает весь процесс mysqld; обычные ошибки запросов затрагивают только своё соединение
- 🧵 **Синхронизация** - необходимость в mutex/locks для доступа к общей памяти

### Преимущества и недостатки

#### PostgreSQL (процессы)

**Преимущества:**
- ✅ **Надежность** - изоляция процессов, ошибка в запросе не роняет сервер (аварийный крах backend сбрасывает все соединения, но данные целы благодаря WAL)
- ✅ **Безопасность** - память одного клиента не может быть прочитана другим
- ✅ **Предсказуемость** - каждый процесс имеет гарантированную память
- ✅ **Отладка** - легко подключиться к процессу (gdb -p PID)
- ✅ **Убить** - зависший запрос можно остановить без последствий для других (`pg_cancel_backend` / `pg_terminate_backend`)

**Недостатки:**
- ❌ **Memory overhead** - каждый процесс ~10MB базового потребления
- ❌ **Fork cost** - создание нового подключения медленнее
- ❌ **Scaling** - на тысячах подключений начинаются проблемы
- ❌ **Переключение** - context switch между процессами дороже

#### MySQL (потоки)

**Преимущества:**
- ✅ **Легковесность** - меньше памяти на подключение
- ✅ **Быстрое создание** - thread spawn быстрее fork()
- ✅ **Context switch** - быстрое переключение между потоками
- ✅ **Scaling** - лучше масштабируется на много подключений
- ✅ **Память** - эффективное использование shared buffer pool

**Недостатки:**
- ❌ **Изоляция** - аварийное падение любого потока (баг сервера, плагина, UDF) убивает весь процесс mysqld
- ❌ **Отладка** - сложнее отлаживать многопоточный код
- ❌ **Конкуренция** - mutex contention при высокой нагрузке

### Что на практике роняет сервер

Разница между моделями меньше, чем кажется: при настоящей аварии клиенты в обеих СУБД теряют соединения, и идёт crash recovery (WAL / redo log). Обычные ошибки запросов (синтаксис, deadlock, нарушение ограничений) не роняют ничего ни там, ни там.

**PostgreSQL (падает backend-процесс):**
- Баг в расширении на C (PostGIS, сторонние `.so` модули) или в самом сервере (segfault)
- OOM killer убил backend (SIGKILL)
- PANIC (не удалось записать WAL, обнаружена порча данных)
- Процессы делят `shared_buffers`, поэтому после аварийного выхода backend postmaster не может быть уверен в целостности shared memory и **перезапускает все backend'ы** (в логе: `terminating any other active server processes`)

**MySQL (падает весь mysqld):**
- Баг в сервере, плагине или UDF (пользовательская функция на C)
- Намеренная остановка InnoDB: `InnoDB: Assertion failure` при обнаружении порчи, либо `semaphore wait has lasted > 600 seconds`
- OOM killer: при большом `innodb_buffer_pool_size` mysqld самый большой процесс и типичная жертва

Реальное преимущество процессной модели PostgreSQL: можно штатно остановить один зависший запрос или соединение, не рискуя остальными. Но от аварийного рестарта всех соединений она не защищает.

### Как это влияет на разработчика

#### 1. Connection Pooling - КРИТИЧНО для PostgreSQL

**PostgreSQL:**
```php
// БЕЗ пулинга - плохо! Каждый запрос = fork()
$pdo = new PDO('pgsql:host=localhost;dbname=mydb', 'user', 'pass');
$pdo->query('SELECT * FROM users');
// При закрытии - процесс уничтожается

// С PgBouncer (connection pooler) - правильно!
// .env
DATABASE_URL=pgsql://user:pass@localhost:6432/mydb  # 6432 = PgBouncer

// Конфиг Laravel с пулом из 20 соединений:
// PgBouncer переиспользует процессы PostgreSQL
```

**MySQL:**
```php
// MySQL лучше справляется без пулинга, но пулинг все равно полезен
$pdo = new PDO('mysql:host=localhost;dbname=mydb', 'user', 'pass');
// Thread creation быстрее, но пулинг снижает overhead
```

**Вывод для разработчика:**
- 🚨 **PostgreSQL** - connection pooling ОБЯЗАТЕЛЕН в production (PgBouncer, pgpool-II)
- ✅ **MySQL** - пулинг полезен, но не так критичен
- 📱 **Веб-приложения** - используют десятки/сотни подключений → без пулинга PostgreSQL будет тормозить

#### 2. Максимальное количество подключений

**PostgreSQL:**
```sql
-- Типичный лимит
max_connections = 100  -- PostgreSQL по умолчанию

-- Проблема: 100 процессов по 10MB = 1GB только на процессы
-- Решение: PgBouncer держит 100 подключений к клиентам,
--          но использует только 10-20 backend процессов
```

**MySQL:**
```sql
-- Типичный лимит
max_connections = 151  -- MySQL по умолчанию

-- 151 поток по 1MB = ~150MB, более масштабируемо
-- Но все равно используйте пулинг!
```

**Вывод для разработчика:**
- PostgreSQL: держите `max_connections` небольшим (50-100), используйте PgBouncer
- MySQL: можно иметь больше подключений, но это не отменяет пулинг
- Laravel/PHP-FPM: количество workers × connections на worker → легко превысить лимит

#### 3. Long-running queries и deadlocks

**PostgreSQL:**
```sql
-- Зависший запрос - можно убить без последствий
SELECT pg_terminate_backend(pid)
FROM pg_stat_activity
WHERE state = 'active' AND query_start < NOW() - INTERVAL '5 minutes';

-- Процесс будет убит, другие продолжат работать
```

**MySQL:**
```sql
KILL QUERY 12345;       -- остановить только запрос, соединение остаётся
KILL CONNECTION 12345;  -- закрыть соединение (откатит его открытую транзакцию)

-- Обе команды штатные и безопасные для остальных соединений
```

**Вывод для разработчика:**
- В обеих СУБД зависший запрос или соединение останавливается штатной командой (`pg_cancel_backend` / `pg_terminate_backend`, `KILL QUERY` / `KILL CONNECTION`), остальные не страдают
- Нельзя делать `kill -9` самому процессу сервера (postgres backend или mysqld): в PostgreSQL это перезапустит все соединения, в MySQL уронит сервер
- Откат убитой долгой транзакции может занять заметное время

#### 4. Мониторинг

**PostgreSQL:**
```sql
-- Посмотреть активные процессы
SELECT pid, usename, application_name, client_addr, state, query
FROM pg_stat_activity
WHERE state = 'active';

-- Каждая строка = отдельный OS процесс (видно в htop/ps)
```

**MySQL:**
```sql
-- Посмотреть потоки
SHOW PROCESSLIST;

-- Все потоки внутри одного процесса mysqld
```

#### 5. Memory Management

**PostgreSQL:**
```sql
-- Каждый процесс получает свой work_mem
work_mem = 4MB  -- на процесс
-- 100 подключений с 1 sort = 400MB RAM!

-- Настройка критична!
```

**MySQL:**
```sql
-- Потоки шарят InnoDB buffer pool
innodb_buffer_pool_size = 4GB  -- общий для всех
sort_buffer_size = 256KB  -- на поток

-- Более предсказуемое потребление памяти
```

### Практические рекомендации для разработчика

#### Обязательно делать:

1. **PostgreSQL:**
   - ✅ Использовать PgBouncer в production
   - ✅ Настроить `max_connections` = 20-50 в PostgreSQL
   - ✅ PgBouncer `pool_mode = transaction` для веб-приложений
   - ✅ Мониторить количество active процессов

2. **MySQL:**
   - ✅ Использовать connection pooling (ProxySQL или application-level)
   - ✅ Настроить `max_connections` адекватно (200-500)
   - ✅ Мониторить thread cache hit rate

3. **Обе СУБД:**
   - ✅ В Laravel: правильно настроить database connections pool
   - ✅ Закрывать неиспользуемые подключения
   - ⚠️ Persistent connections (`PDO::ATTR_PERSISTENT`) используй осторожно: это не пул, а одно постоянное соединение на воркер FPM (расходует `max_connections`); состояние сессии (незавершённая транзакция, temp-таблицы, `SET`) может «протечь» в следующий запрос; соединение может оборваться по таймауту (`gone away`); с PgBouncer не нужно

#### Laravel конфигурация:

```php
// config/database.php
'connections' => [
    'pgsql' => [
        'driver' => 'pgsql',
        'host' => env('DB_HOST', 'localhost'),
        'port' => env('DB_PORT', '6432'),  // PgBouncer!
        'database' => env('DB_DATABASE', 'forge'),
        // ...
        'options' => [
            // Важно для PgBouncer!
            PDO::ATTR_EMULATE_PREPARES => true,
        ],
    ],
    
    'mysql' => [
        'driver' => 'mysql',
        // ...
        // MySQL справляется лучше, но пулинг все равно рекомендуется
    ],
],
```

### Итоговый ответ

**Касается ли это разработчика?**

**ДА, касается! Вот как:**

1. **Connection Pooling** - для PostgreSQL становится необходим, когда соединений сотни и больше
2. **Лимиты подключений** - нужно понимать, сколько можно держать (воркеры FPM + очереди + cron, умноженные на число серверов)
3. **Архитектура приложения** - короткие транзакции, правильный пулинг
4. **Масштабирование** - PostgreSQL требует больше внимания к connections
5. **Memory planning** - PostgreSQL требует больше памяти на подключение

**Простыми словами:**
- **PostgreSQL** - процесс на соединение, при сотнях и тысячах одновременных клиентов нужен пулер (PgBouncer)
- **MySQL** - поток на соединение, легче по ресурсам, но пулер (ProxySQL) тоже полезен при большом числе клиентов
- Для типичного Laravel-приложения разница не критична
- Разница зависит не от RPS, а от числа **одновременных соединений** (тысячи RPS на коротких запросах могут обслуживаться десятком соединений)

**Что делать разработчику:**
1. Посчитай бюджет соединений; при большом числе клиентов ставь пулер (PgBouncer для PostgreSQL)
2. Не держи долгие транзакции открытыми
3. Следи за количеством активных подключений (мониторинг!)
4. Знай лимиты своей БД и настраивай connection pool адекватно

---

## 🎓 Checklist для оптимизации

### Перед оптимизацией:
- ✅ Определить медленные запросы (slow query log, pg_stat_statements)
- ✅ Измерить текущую производительность (baseline)
- ✅ Понять бизнес-требования (приемлемое время ответа)

### Оптимизация запросов:
- ✅ EXPLAIN ANALYZE каждого медленного запроса
- ✅ Проверить наличие индексов на WHERE, JOIN, ORDER BY колонках
- ✅ Избегать SELECT *, использовать только нужные колонки
- ✅ N+1 query problem (eager loading)
- ✅ Cursor pagination вместо OFFSET для глубокой пагинации
- ✅ EXISTS вместо COUNT для проверки наличия
- ✅ Batch операции вместо циклов
- ✅ Covering indexes для популярных запросов

### Индексы:
- ✅ Composite indexes в правильном порядке (WHERE -> JOIN -> ORDER BY)
- ✅ Partial indexes для часто фильтруемых данных (PostgreSQL)
- ✅ Expression indexes для запросов с функциями
- ✅ Удалить неиспользуемые индексы (idx_scan = 0)

### Обслуживание (PostgreSQL):
- ✅ VACUUM ANALYZE регулярно (autovacuum включен?)
- ✅ Мониторить dead tuples (< 10-20% dead_ratio)
- ✅ Обновлять статистику после bulk операций

### Обслуживание (MySQL):
- ✅ OPTIMIZE TABLE для дефрагментации
- ✅ Мониторить InnoDB buffer pool hit ratio (> 99%)

### Конфигурация:
- ✅ shared_buffers / innodb_buffer_pool_size правильно настроены
- ✅ work_mem достаточно для сложных запросов
- ✅ Connection pooling (PgBouncer для PostgreSQL)

### После оптимизации:
- ✅ Измерить снова (сравнить с baseline)
- ✅ Мониторить в production (Prometheus, Grafana)
- ✅ Документировать изменения

---

**Главное для собеседования:**
1. **EXPLAIN ANALYZE** - умей читать и интерпретировать планы
2. **Индексы** - понимай B-Tree, когда добавлять, composite vs single
3. **VACUUM (PostgreSQL)** - зачем нужен, как работает MVCC
4. **N+1 problem** - знай как находить и исправлять
5. **Cursor pagination** - почему лучше OFFSET для больших данных
6. **Cache Hit Ratio** - главная метрика памяти (> 99%)
7. **Мониторинг** - pg_stat_statements, slow query log, Performance Schema
