# NewSQL базы данных

NewSQL - это категория современных СУБД, которые объединяют лучшее из SQL и NoSQL миров: строгая консистентность и SQL запросы традиционных баз + горизонтальное масштабирование NoSQL.

---

## 🎯 Что такое NewSQL?

**NewSQL = SQL + Распределенность**

```
Traditional SQL         NoSQL               NewSQL
(PostgreSQL)           (MongoDB)           (CockroachDB)
     │                     │                     │
     ├─ SQL ✅             ├─ SQL ❌             ├─ SQL ✅
     ├─ ACID ✅            ├─ ACID ⚠️            ├─ ACID ✅
     ├─ Схема ✅           ├─ Схема ❌           ├─ Схема ✅
     ├─ Масштаб. ❌        ├─ Масштаб. ✅        ├─ Масштаб. ✅
     └─ Распред. ❌        └─ Распред. ✅        └─ Распред. ✅
```

### Ключевые характеристики:

1. **SQL синтаксис** - стандартный SQL, привычный разработчикам
2. **ACID транзакции** - полная поддержка, даже в распределенной среде
3. **Горизонтальное масштабирование** - добавляй сервера, не апгрейдь железо
4. **Автоматическая репликация** - данные копируются между узлами
5. **High availability** - работает даже при падении узлов
6. **Strong consistency** - не eventual, а строгая консистентность

---

## 📦 Основные NewSQL базы данных

### 1. CockroachDB

**Описание:** PostgreSQL-совместимая распределенная БД, вдохновленная Google Spanner.

**Особенности:**
- PostgreSQL wire protocol (подключаешься как к Postgres)
- Автоматический sharding и rebalancing
- Multi-region deployment
- Serializable isolation по умолчанию
- Не open source с ноября 2024: проприетарная лицензия (Enterprise бесплатен для малого бизнеса); раньше ядро было под BSL

**Архитектура:**
```
┌─────────────────────────────────────────────────────┐
│              CockroachDB Cluster                     │
├─────────────┬─────────────┬─────────────┬───────────┤
│   Node 1    │   Node 2    │   Node 3    │   Node 4  │
│   (US-East) │   (US-West) │   (EU)      │   (Asia)  │
├─────────────┼─────────────┼─────────────┼───────────┤
│   Range 1   │   Range 1   │   Range 1   │           │
│   Range 2   │   Range 2   │             │   Range 2 │
│             │   Range 3   │   Range 3   │   Range 3 │
└─────────────┴─────────────┴─────────────┴───────────┘
        ↓               ↓               ↓
   3-way replication (по умолчанию)
```

**Пример использования:**
```sql
-- Почти обычный PostgreSQL (STRING — собственный тип CockroachDB, синоним TEXT/VARCHAR)
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email STRING UNIQUE NOT NULL,  -- UNIQUE сам создаёт индекс по email
    username STRING,
    created_at TIMESTAMP DEFAULT now()
);

-- Индексы работают как в Postgres
CREATE INDEX idx_users_username ON users(username);

-- ACID транзакции
BEGIN;
    INSERT INTO users (email, username) VALUES ('test@example.com', 'testuser');
    UPDATE accounts SET balance = balance - 100
    WHERE user_id = 'a0eebc99-9c0b-4ef8-bb6d-6bb9bd380a11';
COMMIT;

-- Multi-region (v21.1+): строка хранится в регионе пользователя
-- Имена регионов берутся из --locality, с которым запущены узлы
ALTER DATABASE myapp SET PRIMARY REGION "us-east1";
ALTER DATABASE myapp ADD REGION "europe-west1";
ALTER DATABASE myapp ADD REGION "asia-northeast1";

ALTER TABLE users SET LOCALITY REGIONAL BY ROW;
-- Добавляет скрытую колонку crdb_region, по ней строка "живёт" в своём регионе
```

**Когда использовать:**
- ✅ Глобальное приложение (пользователи по всему миру)
- ✅ Критична доступность (99.99%+)
- ✅ Нужен SQL + горизонтальное масштабирование
- ✅ Финансовые данные, транзакции

**Когда НЕ использовать:**
- ❌ Маленький проект (overkill)
- ❌ Очень низкая latency критична (distributed = больше задержек)
- ❌ Ограниченный бюджет (дороже одного Postgres)

### 2. Google Spanner

**Описание:** Глобально распределённая SQL БД от Google (статья 2012). Не первая NewSQL (раньше были H-Store/VoltDB, NuoDB), но самая влиятельная: вдохновила CockroachDB и YugabyteDB.

**Особенности:**
- TrueTime API (атомные часы + GPS дают время с гарантированной погрешностью)
- Репликация внутри шарда через Paxos (не Raft)
- External consistency (строже чем serializable)
- Managed service (только в Google Cloud)
- Автоматическое шардирование
- Multi-region из коробки

**Пример:**
```sql
-- GoogleSQL диалект (у Spanner есть и PostgreSQL-интерфейс)
CREATE TABLE users (
    user_id STRING(36) NOT NULL,
    email STRING(255) NOT NULL,
    created_at TIMESTAMP NOT NULL OPTIONS (allow_commit_timestamp=true)
) PRIMARY KEY (user_id);

CREATE INDEX idx_users_email ON users(email);

-- Table hint: принудительно использовать индекс
SELECT * FROM users@{FORCE_INDEX=idx_users_email}
WHERE email = 'test@example.com';

-- Снижение latency чтений: read-only транзакции со stale read
-- (например, exact staleness 10s) читают с ближайшей реплики без блокировок.
-- Задаётся параметрами транзакции в клиентской библиотеке, а не в SQL.
```

**Цены:** Дорого: ~$0.90 за узел/час (≈ $650/мес) + хранение. Минимальный инстанс — 100 processing units (1/10 узла)

### 3. YugabyteDB

**Описание:** Open-source NewSQL, PostgreSQL и Cassandra совместимая.

**Особенности:**
- Два API: YSQL (PostgreSQL) и YCQL (Cassandra)
- Автоматический failover
- Built-in sharding
- Open source (Apache 2.0)
- Можно развернуть on-premise

**Пример:**
```sql
-- PostgreSQL-совместимый YSQL
CREATE TABLE products (
    id SERIAL PRIMARY KEY,
    name VARCHAR(255),
    price DECIMAL(10,2),
    category_id INT
);

-- Sharding по hash ключа (автоматически)
-- Данные распределяются по tablets
```

### 4. TiDB

**Описание:** MySQL-совместимая NewSQL от PingCAP.

**Особенности:**
- MySQL wire protocol (подключаешься как к MySQL)
- HTAP (Hybrid Transactional/Analytical Processing)
- TiKV для хранения (key-value движок)
- TiFlash для аналитики
- Open source

**Пример:**
```sql
-- Обычный MySQL синтаксис
CREATE TABLE orders (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    user_id BIGINT NOT NULL,
    total DECIMAL(10,2),
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_user_id (user_id)
);

-- Работает даже JOIN и транзакции
BEGIN;
    INSERT INTO orders (user_id, total) VALUES (123, 99.99);
    UPDATE inventory SET stock = stock - 1 WHERE product_id = 456;
COMMIT;
```

### 5. VoltDB

**Описание:** In-memory NewSQL для высокопроизводительных workloads (сейчас называется Volt Active Data). Вырос из академического H-Store.

**Особенности:**
- Все данные в RAM (очень быстро)
- Stored procedures на Java
- Deterministic execution
- Партиционирование по ключу

---

## 🔄 Как работает распределенность в NewSQL

### Распределенные транзакции (2PC - Two-Phase Commit)

```
Client              Node 1 (Coordinator)     Node 2           Node 3
  │                        │                   │                │
  ├─ BEGIN ───────────────>│                   │                │
  ├─ INSERT / UPDATE ─────>├─ запись + lock ──>│                │
  │                        ├─ запись + lock ───────────────────>│
  ├─ COMMIT ──────────────>│                   │                │
  │                        ├─ Prepare ────────>│                │   фаза 1
  │                        ├─ Prepare ─────────────────────────>│
  │                        │<── Yes (vote) ────┤                │
  │                        │<── Yes (vote) ────────────────────┤
  │                        ├─ Commit ─────────>│                │   фаза 2
  │                        ├─ Commit ──────────────────────────>│
  │<─ OK ──────────────────┤                   │                │
```

**Фазы (начинаются только после COMMIT от клиента):**
1. **Prepare Phase** - координатор спрашивает участников; каждый durable сохраняет изменения и голосует Yes/No. Проголосовав Yes, участник уже не может передумать
2. **Commit Phase** - если все Yes → Commit всем, если хоть один No → Abort всем

**Проблемы классического 2PC:**
- Блокировки держатся минимум 2 сетевых round-trip'а
- Блокирующий протокол: если coordinator упал после Prepare, участники висят с блокировками, не зная исхода
- 3PC теоретически снимает блокировку, но ломается при network partition → на практике не используется

### Как это решают в NewSQL: 2PC поверх Raft/Paxos

```
   Шард A (Raft-группа)          Шард B (Raft-группа)
  leader + 2 реплики            leader + 2 реплики
          │                              │
          └──── 2PC между шардами ───────┘
```

- **Raft/Paxos** - отказоустойчивость *внутри* шарда (реплики договариваются о логе)
- **2PC** - атомарность *между* шардами
- Это не альтернативы, а два слоя: каждый участник 2PC (и запись о статусе транзакции) сам реплицирован через Raft → падение одного узла не подвешивает транзакцию, новый лидер знает её статус
- Spanner: 2PC поверх Paxos. CockroachDB: оптимизация Parallel Commits (коммит за ~1 round-trip вместо 2)

### Consensus протоколы: Raft

```
Client    Node 1 (Leader)         Node 2 (Follower)      Node 3 (Follower)
  │             │                        │                        │
  ├─ Write ────>│                        │                        │
  │             ├─ append в свой лог     │                        │
  │             ├─ AppendEntries ───────>│                        │
  │             ├─ AppendEntries ────────────────────────────────>│
  │             │<─── ACK ───────────────┤                        │
  │             ├─ committed: 2 из 3 (leader + Node 2) = кворум   │
  │<─── OK ─────┤  (ACK от Node 3 ждать не нужно)                 │
  │             ├─ следующий AppendEntries/heartbeat ────────────>│
  │             │  несёт leaderCommit → followers применяют запись│
```

**Ключевые моменты:**
- Только leader обрабатывает writes
- Majority (кворум) должны согласиться, причём leader сам входит в кворум: при 3 узлах хватает 1 ACK
- Отдельного сообщения "commit" нет: followers узнают commit index из следующего AppendEntries
- Кластер из 3 узлов переживает падение 1, из 5 — падение 2
- Автоматический leader election при падении лидера

---

## 📊 Сравнение NewSQL баз данных

| Характеристика | CockroachDB | Google Spanner | YugabyteDB | TiDB |
|----------------|-------------|----------------|------------|------|
| **SQL диалект** | PostgreSQL | GoogleSQL / PostgreSQL | PostgreSQL (+ Cassandra API) | MySQL |
| **Open Source** | ❌ (проприетарная с 2024) | ❌ | ✅ (Apache 2.0) | ✅ (Apache 2.0) |
| **On-premise** | ✅ | ❌ | ✅ | ✅ |
| **Cloud** | ✅ | ✅ (только GCP) | ✅ | ✅ |
| **Consistency (по умолчанию)** | Serializable | External consistency | Snapshot (Serializable опционально) | Snapshot Isolation |
| **Geo-replication** | ✅ | ✅ | ✅ | ✅ |
| **HTAP** | ⚠️ Ограничено | ⚠️ (Data Boost) | ❌ (нет колоночного движка) | ✅ (TiFlash) |
| **Цена** | Средняя | Очень высокая | Низкая | Низкая |

---

## 🚀 Когда использовать NewSQL

### ✅ Хорошие сценарии:

1. **Глобальные приложения**
   ```
   Пользователи в США, Европе, Азии
   → CockroachDB с geo-partitioning
   → Latency оптимизируется для каждого региона
   ```

2. **High availability критична**
   ```
   E-commerce, финтех, критическая инфраструктура
   → Выдерживает падение целого дата-центра
   → 99.99%+ uptime
   ```

3. **Растущие данные**
   ```
   Начали с 10GB, растем до 10TB
   → Горизонтальное масштабирование (добавляй узлы)
   → Не нужно переписывать на NoSQL
   ```

4. **Сложные транзакции**
   ```
   Финансы, банкинг, инвентаризация
   → ACID гарантии даже в распределенной системе
   → Не eventual consistency как в Cassandra
   ```

### ❌ Плохие сценарии:

1. **Маленький проект**
   ```
   Простой CRUD, данные спокойно влезают на 1 сервер
   → Overkill, используй PostgreSQL
   → Излишняя сложность
   ```

2. **Очень низкая latency**
   ```
   HFT (High-Frequency Trading), gaming
   → Distributed consensus добавляет задержки (1-5ms внутри региона, 50-200ms+ между регионами)
   → Используй in-memory БД или Redis
   ```

3. **Ограниченный бюджет**
   ```
   Стартап, MVP
   → 3-5 узлов = дороже чем 1 мощный Postgres
   → Начни с Postgres, мигрируй при необходимости
   ```

4. **Аналитика (OLAP)**
   ```
   Data warehouse, BI, отчеты
   → Используй ClickHouse или BigQuery
   → NewSQL оптимизированы для OLTP
   ```

---

## 🏗️ Архитектура приложения с NewSQL

### Laravel + CockroachDB

```dotenv
# .env — CockroachDB говорит по PostgreSQL wire protocol, поэтому драйвер pgsql
# (комментарии в .env только через #, а "//" в значении сломает парсинг)
DB_CONNECTION=pgsql
DB_HOST=cockroach-lb.example.com
DB_PORT=26257
DB_DATABASE=myapp
DB_USERNAME=root
DB_PASSWORD=secret
```

```php
// config/database.php
'pgsql' => [
    'driver' => 'pgsql',
    'url' => env('DATABASE_URL'),
    'host' => env('DB_HOST', 'localhost'),
    'port' => env('DB_PORT', '26257'),
    'database' => env('DB_DATABASE', 'forge'),
    'username' => env('DB_USERNAME', 'forge'),
    'password' => env('DB_PASSWORD', ''),
    'charset' => 'utf8',
    'prefix' => '',
    'schema' => 'public',
    'sslmode' => 'require',
    // CockroachDB сам ретраит только простые случаи (отдельный statement вне явной транзакции).
    // Явная транзакция при конфликте падает с SQLSTATE 40001 → ретраить в приложении (см. Retry logic)
],
```

```php
// Миграции - обычные Laravel
Schema::create('orders', function (Blueprint $table) {
    $table->uuid('id')->primary();
    $table->foreignUuid('user_id')->constrained();
    $table->decimal('total', 10, 2);
    $table->timestamps(); // Eloquent по умолчанию пишет и created_at, и updated_at
});

// Multi-region — отдельной миграцией:
// DB::statement('ALTER TABLE orders SET LOCALITY REGIONAL BY ROW');
```

```php
// Модели - обычный Eloquent
class Order extends Model
{
    protected $keyType = 'string'; // UUID
    public $incrementing = false;
    protected $fillable = ['user_id', 'total']; // без этого create() кинет MassAssignmentException
    
    protected static function booted()
    {
        static::creating(function ($order) {
            // Случайный UUIDv4 → записи равномерно по ranges.
            // Осторожно с трейтом HasUuids: он генерирует упорядоченные по времени UUID
            // (orderedUuid / v7) → все INSERT в одну range → hotspot.
            // Если нужен HasUuids — переопредели newUniqueId() на (string) Str::uuid()
            $order->id ??= (string) Str::uuid();
        });
    }
    
    // Транзакции работают как обычно
    public static function createWithPayment($userId, $total)
    {
        return DB::transaction(function () use ($userId, $total) {
            $order = Order::create([
                'user_id' => $userId,
                'total' => $total,
            ]);
            
            // ACID гарантируется даже если узлы в разных регионах
            // (но cross-region коммит стоит сотни ms)
            // Payment тоже должен иметь $fillable
            Payment::create([
                'order_id' => $order->id,
                'amount' => $total,
            ]);
            
            return $order;
        });
    }
}
```

### Мониторинг

```sql
-- DB Console (Admin UI): http://localhost:8080
-- Там же: SQL Activity (медленные запросы), Replication (under-replicated / unavailable ranges)

-- Распределение данных по узлам (ranges, реплики, leaseholder)
SHOW RANGES FROM TABLE orders;

-- Активные транзакции и запросы по всему кластеру
SHOW TRANSACTIONS;
SHOW STATEMENTS;

-- Slow queries: статистика хранится в JSON-колонках
-- (структура JSON может меняться между версиями — проще смотреть в DB Console)
SELECT metadata->>'query' AS query,
       (statistics->'statistics'->>'cnt')::INT AS cnt,
       (statistics->'statistics'->'svcLat'->>'mean')::FLOAT AS avg_latency_sec
FROM crdb_internal.statement_statistics
ORDER BY avg_latency_sec DESC
LIMIT 10;
```

---

## ⚖️ Миграция с PostgreSQL на CockroachDB

### 1. Совместимость

**Что работает из коробки:**
- ✅ Базовый SQL (SELECT, INSERT, UPDATE, DELETE)
- ✅ JOINs, подзапросы, CTEs
- ✅ Индексы (B-Tree, а также inverted/GIN для JSONB и массивов)
- ✅ Foreign keys
- ✅ Транзакции
- ✅ JSON/JSONB

**Что требует изменений:**
- ⚠️ SERIAL → UUID (рекомендуется; SERIAL в CockroachDB по умолчанию = `unique_rowid()`, а не sequence)
- ⚠️ PostgreSQL расширения в основном не поддерживаются (часть PostGIS-функций встроена)
- ⚠️ Функции и процедуры PL/pgSQL (поддерживаются с v23.2, но не всё)
- ⚠️ INTERLEAVE удалён в v21.2 (если был — переписать схему)

### 2. Пошаговая миграция

```bash
# 1. Экспорт из PostgreSQL
pg_dump -h localhost -U postgres --no-owner --no-privileges mydb > dump.sql
```

```sql
-- 2. Адаптация схемы для CockroachDB
-- Замена SERIAL на UUID
-- До:
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    email VARCHAR(255)
);

-- После:
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email VARCHAR(255)  -- VARCHAR можно оставить: в CockroachDB это алиас STRING
);
```

```bash
# 3. Импорт в CockroachDB (--insecure только для локального кластера)
# Для больших объёмов — MOLT Fetch или IMPORT INTO из CSV
cockroach sql --insecure --database=mydb < adapted_dump.sql
```

```sql
-- 4. Multi-region (опционально)
ALTER DATABASE mydb SET PRIMARY REGION "us-east1";
ALTER DATABASE mydb ADD REGION "europe-west1";
ALTER TABLE users SET LOCALITY REGIONAL BY ROW;
```

---

## 🔧 Оптимизация производительности

### 1. Используй UUID вместо SERIAL

```sql
-- Плохо: SERIAL создает hotspot (все INSERT в одну range)
CREATE TABLE orders (
    id SERIAL PRIMARY KEY,
    user_id INT
);

-- Хорошо: UUID распределяет данные равномерно
CREATE TABLE orders (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID
);
```

### 2. Multi-region таблицы для latency

```sql
-- Приложение в США и Европе (регионы = --locality узлов)
ALTER DATABASE myapp SET PRIMARY REGION "us-east1";
ALTER DATABASE myapp ADD REGION "europe-west1";

-- Каждая строка живёт в своём регионе (скрытая колонка crdb_region)
ALTER TABLE users SET LOCALITY REGIONAL BY ROW;

-- Справочники: читают все, пишут редко → быстрые чтения в любом регионе
ALTER TABLE countries SET LOCALITY GLOBAL;

-- Пользователь из EU читает и пишет свою строку локально:
-- единицы ms вместо ~100ms на межконтинентальный round-trip.
-- Цена: запись в GLOBAL таблицы медленнее.
```

> Старый способ — `PARTITION BY LIST` + zone configs. Колонка партиционирования там должна быть префиксом primary key, а настройка сложнее. Для новых проектов используй multi-region SQL выше.

### 3. Read-only optimization

```sql
-- Для read-heavy запросов, которым подходят слегка устаревшие данные
SELECT * FROM products AS OF SYSTEM TIME '-5s'
WHERE category = 'electronics';

-- Или для всей транзакции (SET TRANSACTION ... работает только внутри BEGIN)
BEGIN AS OF SYSTEM TIME '-5s';
    SELECT ...;
COMMIT;

-- Читает снимок данных 5 сек назад:
-- - не конфликтует с пишущими транзакциями (нет ожиданий блокировок и retry)
-- - может обслуживаться ближайшей репликой (follower read), а не leaseholder'ом
-- Latency ниже, throughput выше
```

### 4. Follower reads

```sql
-- Обычно читает и пишет только leaseholder range'а.
-- Follower read: читаем достаточно старый снимок с ближайшей реплики
SELECT * FROM products
AS OF SYSTEM TIME follower_read_timestamp()
WHERE id = 'a0eebc99-9c0b-4ef8-bb6d-6bb9bd380a11';
-- follower_read_timestamp() ≈ now() - ~4.8s: данные чуть устаревшие, latency локальная

-- Если нужны свежие (не stale) чтения из любого региона — GLOBAL таблицы
ALTER TABLE products SET LOCALITY GLOBAL;
```

---

## 🎓 Лучшие практики

### 1. Дизайн схемы

```sql
-- ✅ Хорошо: UUID для равномерного распределения
CREATE TABLE events (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL,
    event_type STRING NOT NULL,
    created_at TIMESTAMP DEFAULT now(),
    INDEX idx_user_events (user_id, created_at DESC)
);

-- ❌ Плохо: SERIAL создает hotspot
CREATE TABLE events (
    id SERIAL PRIMARY KEY,  -- Все INSERT идут в конец таблицы
    user_id INT,
    created_at TIMESTAMP
);
```

### 2. Размер транзакций

```sql
-- ❌ Плохо: огромная транзакция
BEGIN;
    -- 10,000 INSERT подряд
    INSERT INTO logs VALUES (...);  -- × 10,000
COMMIT;
-- Держит блокировки долго, может упасть

-- ✅ Хорошо: батчи по 100-500 строк одним multi-row INSERT
INSERT INTO logs (level, message) VALUES
    ('info', '...'),
    ('warn', '...'),
    ...;  -- 500 строк в одном statement
-- Повторить 20 раз. Каждый statement — отдельная implicit-транзакция,
-- её CockroachDB при конфликте может ретраить сам
```

### 3. Retry logic

```php
// При конфликте CockroachDB отдаёт SQLSTATE 40001 ("restart transaction")
// DB::transaction ретраит такие ошибки сам (40001 = concurrency error)
DB::transaction(function () {
    // your logic
}, 5);  // до 5 попыток, но без паузы между ними

// Вручную — с экспоненциальным backoff
use Illuminate\Database\QueryException;

$maxRetries = 5;

for ($attempt = 1; ; $attempt++) {
    try {
        DB::transaction(function () {
            Order::create([...]);
        });
        break; // успех
    } catch (QueryException $e) {
        // Проверяем SQLSTATE, а не текст сообщения
        $isRetryable = $e->getCode() === '40001';

        if (!$isRetryable || $attempt >= $maxRetries) {
            throw $e; // чужая ошибка или попытки кончились — не глотаем молча
        }

        // 100ms, 200ms, 400ms, 800ms + jitter
        usleep(100_000 * 2 ** ($attempt - 1) + random_int(0, 50_000));
    }
}
```

---

## 🆚 NewSQL vs PostgreSQL vs NoSQL

| Критерий | PostgreSQL | CockroachDB (NewSQL) | MongoDB (NoSQL) |
|----------|------------|----------------------|-----------------|
| **SQL** | ✅ Полный | ✅ PostgreSQL совместимый | ❌ Свой язык |
| **ACID** | ✅ Да | ✅ Да (distributed) | ⚠️ Multi-document с 4.0, но дороже и с ограничениями |
| **Схема** | ✅ Строгая | ✅ Строгая | ❌ Schema-less |
| **Масштабирование** | ⚠️ Вертикальное | ✅ Горизонтальное | ✅ Горизонтальное |
| **Репликация** | ⚠️ Вручную (streaming) | ✅ Автоматическая | ✅ Автоматическая |
| **Multi-region** | ❌ Сложно | ✅ Из коробки | ✅ Из коробки |
| **Consistency** | ✅ Strong | ✅ Strong | ⚠️ Strong при чтении с primary (по умолчанию), eventual при чтении с secondary |
| **Latency** | 🟢 Низкая | 🟡 Средняя | 🟢 Низкая |
| **Сложность** | 🟢 Простая | 🔴 Высокая | 🟡 Средняя |
| **Цена** | 💰 Низкая | 💰💰 Средняя/Высокая | 💰 Низкая |

### Когда что выбрать:

**PostgreSQL:**
- Традиционное веб-приложение
- Данные помещаются на 1 сервер (единицы TB — нормально для Postgres)
- Не критична geographic distribution
- Бюджет ограничен

**CockroachDB (NewSQL):**
- Глобальное приложение (пользователи везде)
- Критична высокая доступность
- Данные или запись не влезают в 1 сервер (десятки TB+)
- Нужен SQL + распределенность

**MongoDB (NoSQL):**
- Гибкая схема данных
- Очень high throughput writes
- Eventual consistency приемлема
- Document-oriented данные

---

## 📚 Ресурсы для изучения

### Документация:
- [CockroachDB Docs](https://www.cockroachlabs.com/docs/)
- [Google Spanner Docs](https://cloud.google.com/spanner/docs)
- [YugabyteDB Docs](https://docs.yugabyte.com/)
- [TiDB Docs](https://docs.pingcap.com/)

### Статьи и видео:
- [Google Spanner Paper](https://research.google/pubs/pub39966/) - оригинальная статья
- [CockroachDB Architecture](https://www.cockroachlabs.com/docs/stable/architecture/overview.html)
- [CAP Theorem in NewSQL](https://blog.yugabyte.com/cap-theorem-and-distributed-databases/)

### Курсы:
- [CockroachDB University](https://university.cockroachlabs.com/) - бесплатные курсы

---

## 🎯 Главное для собеседования

1. **NewSQL = SQL + ACID + горизонтальное масштабирование из коробки.** PostgreSQL тоже масштабируется (реплики, партиционирование, Citus, шардинг в приложении), но руками. NewSQL режет данные на куски, раскладывает по нодам и ребалансирует сам.

2. **Почему не всегда NewSQL.** Каждая запись ждёт кворум по сети (даже на одной ноде NewSQL медленнее PostgreSQL), Serializable по умолчанию → транзакции падают с конфликтом и требуют retry, дороже эксплуатация (минимум 3 ноды), неполная совместимость с PostgreSQL. Одного PostgreSQL хватает большинству проектов → «начни с Postgres, мигрируй, когда упрёшься».

3. **Raft/Paxos — консенсус.** Реплики договариваются об одном логе, запись закоммичена при подтверждении большинства. Отсюда нечётное число нод: 3 ноды переживают падение 1, 5 — падение 2, а 2 ноды хуже одной (падение любой останавливает запись).

4. **2PC** даёт атомарность между шардами. Классический 2PC блокирующий: координатор упал после Prepare → участники висят с блокировками. В NewSQL 2PC работает поверх Raft: решение о транзакции (transaction record) реплицировано → её может довести до конца любой, кто наткнулся на её блокировку.

5. **CAP: CockroachDB и Spanner — CP.** При разрыве сети меньшинство отказывает в записи (и в консистентном чтении). High availability (99.99%) ≠ буква A в CAP: большинство обычно остаётся вместе и система сама переизбирает лидеров. Отдельные чтения можно перевести на доступность ценой устаревших данных.

6. **Serializable vs External consistency.** Serializable — результат эквивалентен *какому-то* последовательному порядку. External consistency (strict serializability) — порядку, *совпадающему с реальным временем*. Spanner добивается этого через TrueTime (атомные часы) + commit wait; CockroachDB без атомных часов гарантирует это только для одного ключа.

7. **Hotspot.** В распределённой БД последовательные id (SERIAL, UUIDv7, `HasUuids`) все пишутся в конец пространства ключей → в один range на одной ноде → нужен случайный UUIDv4 (или hash-sharded индекс). В PostgreSQL наоборот: последовательные id лучше для B-tree, случайный UUIDv4 фрагментирует индекс.

8. **Для PHP-кода.** Транзакции ретраятся → внутри них не должно быть побочных эффектов (письма, HTTP, `dispatch()`), иначе они выполнятся несколько раз. Между сервисами — saga (компенсации вместо отката) и outbox (сообщение в той же транзакции, что и данные), консьюмеры идемпотентны. Этот пункт спрашивают и без всякого NewSQL — в любой теме про микросервисы и очереди.

**Ключевые вопросы:**
- В чем разница между NewSQL и NoSQL? → Strong consistency + SQL + ACID vs обычно eventual consistency и свой язык запросов
- Почему не все используют NewSQL? → Latency на каждой записи, retry, сложность и цена, а одного PostgreSQL хватает большинству
- Как работают distributed transactions? → 2PC между шардами, каждый участник реплицирован через Raft/Paxos (поэтому падение координатора не блокирует транзакцию)
- Когда выбрать CockroachDB вместо PostgreSQL? → Данные или запись не влезают в один сервер, нужен переживаемый отказ датацентра без ручного failover, глобальные пользователи / требования хранить данные в регионе
- Почему 2 ноды хуже одной? → Кворум из 2 — это 2, падение любой останавливает запись
