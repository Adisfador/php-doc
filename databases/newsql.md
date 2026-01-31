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
- Open source (бесплатная и enterprise версии)

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
-- Обычный PostgreSQL SQL!
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email STRING UNIQUE NOT NULL,
    username STRING,
    created_at TIMESTAMP DEFAULT now()
);

-- Индексы работают как в Postgres
CREATE INDEX idx_users_email ON users(email);

-- ACID транзакции
BEGIN;
    INSERT INTO users (email, username) VALUES ('test@example.com', 'testuser');
    UPDATE accounts SET balance = balance - 100 WHERE user_id = 'test-id';
COMMIT;

-- Geo-partitioning для latency optimization
ALTER TABLE users PARTITION BY LIST (region) (
    PARTITION us VALUES IN ('us-east', 'us-west'),
    PARTITION eu VALUES IN ('eu-west', 'eu-central'),
    PARTITION asia VALUES IN ('asia-pacific')
);
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

**Описание:** Первая NewSQL БД от Google, вдохновение для CockroachDB.

**Особенности:**
- TrueTime API (атомные часы для timestamp)
- External consistency (строже чем serializable)
- Managed service (только в Google Cloud)
- Автоматическое шардирование
- Multi-region из коробки

**Пример:**
```sql
-- Google Cloud SQL диалект
CREATE TABLE users (
    user_id STRING(36) NOT NULL,
    email STRING(255) NOT NULL,
    created_at TIMESTAMP NOT NULL OPTIONS (allow_commit_timestamp=true)
) PRIMARY KEY (user_id);

-- Read-only transactions для снижения latency
@{use_additional_parallelism=true}
SELECT * FROM users WHERE region = 'us-east';
```

**Цены:** Очень дорого (~$0.90 за узел/час + хранение)

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

**Описание:** In-memory NewSQL для высокопроизводительных workloads.

**Особенности:**
- Все данные в RAM (очень быстро)
- Stored procedures на Java
- Deterministic execution
- Партиционирование по ключу

---

## 🔄 Как работает распределенность в NewSQL

### Распределенные транзакции (2PC - Two-Phase Commit)

```
Client                Node 1 (Coordinator)     Node 2           Node 3
  │                          │                   │                │
  ├─ BEGIN                   │                   │                │
  ├─ INSERT users ...───────>│                   │                │
  │                          ├─ Prepare ────────>│                │
  │                          ├─ Prepare ─────────────────────────>│
  │                          │                   │                │
  │                          │<─── OK ───────────┤                │
  │                          │<─── OK ───────────────────────────┤
  ├─ COMMIT ────────────────>│                   │                │
  │                          ├─ Commit ──────────>│                │
  │                          ├─ Commit ──────────────────────────>│
  │<─ OK ────────────────────┤                   │                │
```

**Фазы:**
1. **Prepare Phase** - все узлы готовятся к коммиту
2. **Commit Phase** - если все OK, все узлы коммитят

**Проблемы:**
- Блокировки держатся дольше
- Если coordinator упал = транзакция висит
- Решение: 3PC (Three-Phase Commit) или Raft consensus

### Consensus протоколы: Raft

```
Node 1 (Leader)        Node 2 (Follower)      Node 3 (Follower)
     │                        │                        │
     ├─ Write request         │                        │
     ├─ AppendEntries ───────>│                        │
     ├─ AppendEntries ────────────────────────────────>│
     │                        │                        │
     │<─── ACK ───────────────┤                        │
     │<─── ACK ───────────────────────────────────────┤
     ├─ Commit (majority OK)  │                        │
     ├─ Notify ──────────────>│                        │
     ├─ Notify ──────────────────────────────────────>│
```

**Ключевые моменты:**
- Только leader обрабатывает writes
- Majority (кворум) должны согласиться
- Автоматический leader election при падении

---

## 📊 Сравнение NewSQL баз данных

| Характеристика | CockroachDB | Google Spanner | YugabyteDB | TiDB |
|----------------|-------------|----------------|------------|------|
| **SQL диалект** | PostgreSQL | Google SQL | PostgreSQL | MySQL |
| **Open Source** | ✅ (BSL) | ❌ | ✅ (Apache 2.0) | ✅ (Apache 2.0) |
| **On-premise** | ✅ | ❌ | ✅ | ✅ |
| **Cloud** | ✅ | ✅ (только GCP) | ✅ | ✅ |
| **Consistency** | Serializable | External | Serializable | Snapshot Isolation |
| **Geo-replication** | ✅ | ✅ | ✅ | ✅ |
| **HTAP** | ⚠️ Ограничено | ⚠️ | ✅ | ✅ |
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
   Простой CRUD, < 100GB данных
   → Overkill, используй PostgreSQL
   → Излишняя сложность
   ```

2. **Очень низкая latency**
   ```
   HFT (High-Frequency Trading), gaming
   → Distributed consensus добавляет задержки (5-50ms)
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

```php
// .env
DB_CONNECTION=pgsql  // CockroachDB использует PostgreSQL protocol
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
    'options' => [
        // CockroachDB retries транзакции автоматически
        PDO::ATTR_EMULATE_PREPARES => true,
    ],
],
```

```php
// Миграции - обычные Laravel
Schema::create('orders', function (Blueprint $table) {
    $table->uuid('id')->primary();
    $table->foreignUuid('user_id')->constrained();
    $table->decimal('total', 10, 2);
    $table->timestamp('created_at')->useCurrent();
    
    // Geo-partitioning для оптимизации
    // ALTER TABLE orders PARTITION BY LIST (region) ...
});
```

```php
// Модели - обычный Eloquent
class Order extends Model
{
    protected $keyType = 'string'; // UUID
    public $incrementing = false;
    
    protected static function booted()
    {
        static::creating(function ($order) {
            $order->id = Str::uuid();
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
            
            // ACID гарантируется даже если узлы в разных регионах!
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
-- CockroachDB Admin UI: http://localhost:8080

-- Посмотреть распределение данных по узлам
SHOW RANGES FROM TABLE orders;

-- Метрики репликации
SELECT * FROM crdb_internal.cluster_replication_status;

-- Активные транзакции
SELECT * FROM crdb_internal.node_transactions;

-- Slow queries
SELECT query, count, avg_latency_seconds
FROM crdb_internal.statement_statistics
WHERE avg_latency_seconds > 1
ORDER BY avg_latency_seconds DESC;
```

---

## ⚖️ Миграция с PostgreSQL на CockroachDB

### 1. Совместимость

**Что работает из коробки:**
- ✅ Базовый SQL (SELECT, INSERT, UPDATE, DELETE)
- ✅ JOINs, подзапросы, CTEs
- ✅ Индексы (B-Tree)
- ✅ Foreign keys
- ✅ Транзакции
- ✅ JSON/JSONB

**Что требует изменений:**
- ⚠️ SERIAL → UUID (рекомендуется)
- ⚠️ Некоторые PostgreSQL расширения (PostGIS частично)
- ⚠️ Stored procedures (ограничены)
- ⚠️ INTERLEAVE (deprecated в новых версиях)

### 2. Пошаговая миграция

```sql
-- 1. Экспорт из PostgreSQL
pg_dump -h localhost -U postgres mydb > dump.sql

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
    email STRING(255)  -- CockroachDB использует STRING, не VARCHAR
);

-- 3. Импорт в CockroachDB
cockroach sql --insecure < adapted_dump.sql

-- 4. Добавление geo-partitioning (опционально)
ALTER TABLE users PARTITION BY LIST (region) (
    PARTITION us VALUES IN ('us-east', 'us-west'),
    PARTITION eu VALUES IN ('eu-west')
);
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

### 2. Geo-partitioning для latency

```sql
-- Приложение в США и Европе
ALTER TABLE users PARTITION BY LIST (region) (
    PARTITION us VALUES IN ('us-east', 'us-west'),
    PARTITION eu VALUES IN ('eu-west', 'eu-central')
);

-- Пользователи из США → данные в US регионе
-- Пользователи из EU → данные в EU регионе
-- Latency снижается на 10x!
```

### 3. Read-only optimization

```sql
-- Для read-heavy workloads
SET TRANSACTION AS OF SYSTEM TIME '-5s';
SELECT * FROM products WHERE category = 'electronics';

-- Читает слегка устаревшие данные (5 сек назад)
-- Не ждет подтверждения от всех реплик
-- Latency ниже, throughput выше
```

### 4. Follower reads

```sql
-- Настройка в CockroachDB
ALTER TABLE products CONFIGURE ZONE USING num_replicas = 5;

-- Чтение с ближайшей реплики (не только leader)
SET enable_follower_reads = on;
SELECT * FROM products WHERE id = 'abc-123';
-- Читает с локальной реплики → меньше latency
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

-- ✅ Хорошо: батчи по 100-500 строк
BEGIN;
    INSERT INTO logs VALUES (...);  -- × 500
COMMIT;
-- Повторить 20 раз
```

### 3. Retry logic

```php
// CockroachDB может вернуть "retry transaction" error
DB::transaction(function () {
    // your logic
}, 5);  // автоматически retry до 5 раз

// Или вручную
use Illuminate\Database\QueryException;

$maxRetries = 5;
$attempt = 0;

while ($attempt < $maxRetries) {
    try {
        DB::transaction(function () {
            Order::create([...]);
        });
        break;  // успех
    } catch (QueryException $e) {
        if (str_contains($e->getMessage(), 'restart transaction')) {
            $attempt++;
            usleep(100000 * $attempt);  // exponential backoff
            continue;
        }
        throw $e;
    }
}
```

---

## 🆚 NewSQL vs PostgreSQL vs NoSQL

| Критерий | PostgreSQL | CockroachDB (NewSQL) | MongoDB (NoSQL) |
|----------|------------|----------------------|-----------------|
| **SQL** | ✅ Полный | ✅ PostgreSQL совместимый | ❌ Свой язык |
| **ACID** | ✅ Да | ✅ Да (distributed) | ⚠️ Ограниченно |
| **Схема** | ✅ Строгая | ✅ Строгая | ❌ Schema-less |
| **Масштабирование** | ⚠️ Вертикальное | ✅ Горизонтальное | ✅ Горизонтальное |
| **Репликация** | ⚠️ Вручную (streaming) | ✅ Автоматическая | ✅ Автоматическая |
| **Multi-region** | ❌ Сложно | ✅ Из коробки | ✅ Из коробки |
| **Consistency** | ✅ Strong | ✅ Strong | ⚠️ Eventual (по умолчанию) |
| **Latency** | 🟢 Низкая | 🟡 Средняя | 🟢 Низкая |
| **Сложность** | 🟢 Простая | 🔴 Высокая | 🟡 Средняя |
| **Цена** | 💰 Низкая | 💰💰 Средняя/Высокая | 💰 Низкая |

### Когда что выбрать:

**PostgreSQL:**
- Традиционное веб-приложение
- Данные помещаются на 1 сервер (< 1TB)
- Не критична geographic distribution
- Бюджет ограничен

**CockroachDB (NewSQL):**
- Глобальное приложение (пользователи везде)
- Критична высокая доступность
- Растущие данные (>1TB)
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

1. **NewSQL = SQL + Распределенность** - понимай эту концепцию
2. **ACID в распределенной системе** - как это работает (2PC, Raft)
3. **Когда использовать** - глобальные приложения, high availability
4. **Trade-offs** - latency выше, сложность выше, но масштабируемость лучше
5. **CockroachDB** - знай хотя бы одну NewSQL БД детально
6. **Отличия от NoSQL** - strong consistency vs eventual consistency
7. **Migration path** - как мигрировать с PostgreSQL на CockroachDB

**Ключевые вопросы:**
- В чем разница между NewSQL и NoSQL? → Consistency и SQL
- Почему не все используют NewSQL? → Сложность и latency
- Как работают distributed transactions? → 2PC или Raft consensus
- Когда выбрать CockroachDB вместо PostgreSQL? → Global scale, HA
то