# Концепции баз данных

OLTP vs OLAP, CAP теорема, репликация, шардирование, партиционирование - фундаментальные концепции для Middle/Senior.

---

## 🎯 OLTP vs OLAP

### OLTP - Online Transaction Processing

**Назначение:** Обработка транзакций в реальном времени.

**Характеристики:**
- 🔹 Много мелких транзакций (INSERT, UPDATE, DELETE)
- 🔹 Нормализация (3NF) - избегаем дублирования
- 🔹 Row-oriented хранение
- 🔹 Индексы для быстрого поиска (B-Tree)
- 🔹 ACID гарантии
- 🔹 Вертикальное масштабирование (мощный сервер)

**Примеры запросов:**
```sql
-- Получить заказ по ID (milliseconds)
SELECT * FROM orders WHERE id = 123;

-- Создать пользователя
INSERT INTO users (email, name) VALUES ('user@example.com', 'John');

-- Обновить статус заказа
UPDATE orders SET status = 'shipped' WHERE id = 123;

-- Удалить сессию
DELETE FROM sessions WHERE token = 'abc123';
```

**Метрики производительности:**
- **TPS** (Transactions Per Second) - транзакций в секунду
- **Latency** - задержка ответа (< 100ms хорошо)
- **Concurrent users** - одновременных пользователей

**БД для OLTP:**
- PostgreSQL, MySQL, Oracle, SQL Server
- MongoDB (NoSQL для OLTP)

**Пример: E-commerce приложение**
```sql
-- Таблицы нормализованы
users: id, email, name
orders: id, user_id, status, created_at
order_items: id, order_id, product_id, quantity, price
products: id, name, price

-- Частые операции:
-- - Создание заказа (INSERT в orders + order_items)
-- - Проверка наличия товара (SELECT FROM products WHERE id = ?)
-- - Обновление профиля пользователя (UPDATE users SET ...)
```

---

### OLAP - Online Analytical Processing

**Назначение:** Аналитика, отчеты, business intelligence.

**Характеристики:**
- 🔹 Редкие, но сложные запросы (агрегации, JOIN)
- 🔹 Денормализация (star schema, snowflake schema)
- 🔹 Column-oriented хранение
- 🔹 Batch обработка данных
- 🔹 Read-heavy (чтение >> записи)
- 🔹 Горизонтальное масштабирование (MPP - Massively Parallel Processing)

**Примеры запросов:**
```sql
-- Продажи по регионам за год (minutes)
SELECT 
    region,
    SUM(amount) as total_sales,
    COUNT(*) as order_count,
    AVG(amount) as avg_order
FROM orders
WHERE created_at >= '2025-01-01'
GROUP BY region
ORDER BY total_sales DESC;

-- Топ-10 продуктов по выручке
SELECT 
    p.name,
    SUM(oi.quantity * oi.price) as revenue
FROM order_items oi
JOIN products p ON oi.product_id = p.id
GROUP BY p.id, p.name
ORDER BY revenue DESC
LIMIT 10;

-- Когортный анализ (retention)
SELECT 
    DATE_TRUNC('month', first_order_date) as cohort,
    DATE_TRUNC('month', order_date) as month,
    COUNT(DISTINCT user_id) as users
FROM (
    SELECT 
        user_id,
        MIN(created_at) OVER (PARTITION BY user_id) as first_order_date,
        created_at as order_date
    FROM orders
) cohorts
GROUP BY cohort, month
ORDER BY cohort, month;
```

**Метрики производительности:**
- **Scan speed** - скорость сканирования (GB/sec)
- **Query time** - время выполнения запроса (minutes/hours)
- **Compression ratio** - степень сжатия данных

**БД для OLAP:**
- ClickHouse, Amazon Redshift, Google BigQuery, Snowflake
- PostgreSQL (не оптимален, но можно для небольших объемов)

**Пример: Data Warehouse для e-commerce**
```sql
-- Denormalized fact table (звездообразная схема)
CREATE TABLE fact_sales (
    sale_id BIGINT,
    date_id INT,           -- FK to dim_date
    user_id INT,           -- FK to dim_users
    product_id INT,        -- FK to dim_products
    quantity INT,
    amount DECIMAL(10,2),
    region VARCHAR(50),    -- денормализовано
    category VARCHAR(100)  -- денормализовано
) ENGINE = MergeTree()
ORDER BY (date_id, region);

-- Dimension tables
dim_date: date_id, date, year, quarter, month, week, day_of_week
dim_users: user_id, email, name, registration_date, segment
dim_products: product_id, name, category, subcategory, brand
```

**Star Schema:**
```
        dim_users
             |
             |
fact_sales --+-- dim_products
             |
             |
          dim_date
```

---

### OLTP vs OLAP: сравнение

| Аспект | OLTP | OLAP |
|--------|------|------|
| **Назначение** | Транзакции | Аналитика |
| **Запросы** | Простые, частые | Сложные, редкие |
| **Операции** | INSERT, UPDATE, DELETE | SELECT с агрегациями |
| **Данные** | Текущие (последние дни/месяцы) | Исторические (годы) |
| **Нормализация** | 3NF (избегаем дубликатов) | Денормализованы (star schema) |
| **Хранение** | Row-oriented | Column-oriented |
| **Индексы** | B-Tree на PK/FK | Sparse index, partitioning |
| **Размер** | Гигабайты - терабайты | Терабайты - петабайты |
| **Пользователи** | Тысячи (concurrent) | Десятки (analysts) |
| **Latency** | Миллисекунды | Секунды - минуты |
| **Примеры БД** | PostgreSQL, MySQL, Oracle | ClickHouse, Redshift, BigQuery |

---

### Hybrid: HTAP (Hybrid Transaction/Analytical Processing)

**Попытка объединить OLTP + OLAP в одной БД:**
- TiDB, MemSQL/SingleStore, SAP HANA
- Row + Column хранение одновременно
- Пока не массово распространены

---

## 🔄 Репликация (Replication)

### Зачем нужна репликация?

1. **High Availability (HA)** - отказоустойчивость (если master упал, failover на replica)
2. **Read Scaling** - распределить чтение на replicas (master пишет, replicas читают)
3. **Backup** - актуальная копия данных
4. **Geo-distribution** - replicas в разных регионах (низкая latency)

---

### Master-Slave (Primary-Replica)

**Архитектура:**
```
        Master (Primary)
        /      |      \
       /       |       \
  Replica1  Replica2  Replica3
```

**Как работает:**
1. Приложение пишет в **Master**
2. Master записывает изменения в **Write-Ahead Log (WAL)** или **Binary Log**
3. Replicas читают WAL/binlog и применяют изменения
4. Приложение читает с **Replicas** (снижает нагрузку на Master)

**Асинхронная репликация (по умолчанию):**
```
Master: INSERT INTO users ... (commit) ✅
  |
  | (асинхронно, секунды задержки)
  ↓
Replica: INSERT INTO users ... ✅ (через 1-5 сек)
```

**Проблемы:**
- ⚠️ **Replication lag** - задержка (replica отстает от master)
- ⚠️ **Stale reads** - чтение устаревших данных с replica
- ⚠️ Если master упал до синхронизации - данные потеряны на replica

**Синхронная репликация:**
```
Master: INSERT INTO users ... (ждет подтверждения от replica) ⏳
  |
  ↓
Replica: INSERT INTO users ... ✅
  |
  ↓
Master: commit ✅
```

**Trade-offs:**
- ✅ Нет потери данных (replica всегда актуальна)
- ❌ Медленнее (ждем replica)
- ❌ Если replica недоступна - пишущие запросы зависнут

**Semi-synchronous репликация (MySQL):**
- Master ждет подтверждение от хотя бы 1 replica
- Баланс между скоростью и безопасностью

---

### PostgreSQL: Streaming Replication

**Physical Replication** - копирование WAL файлов:
```bash
# На master (postgresql.conf)
wal_level = replica
max_wal_senders = 3
wal_keep_size = 1GB

# На replica (postgresql.conf)
hot_standby = on  # replica может отдавать read-only запросы

# Создать replica
pg_basebackup -h master_ip -D /var/lib/postgresql/data -U replication -P -v -R
```

**Synchronous Replication:**
```sql
-- На master (postgresql.conf)
synchronous_commit = on
synchronous_standby_names = 'replica1'

-- Master будет ждать подтверждения от replica1 перед commit
```

**Logical Replication (PostgreSQL 10+):**
- Репликация на уровне таблиц/строк (не весь WAL)
- Можно реплицировать только часть БД
- Разные версии PostgreSQL (master 12, replica 14)

```sql
-- На master
CREATE PUBLICATION my_pub FOR TABLE users, orders;

-- На replica
CREATE SUBSCRIPTION my_sub 
CONNECTION 'host=master_ip dbname=mydb user=replication' 
PUBLICATION my_pub;
```

---

### MySQL: Replication

**Asynchronous Replication (по умолчанию):**
```sql
-- На master (my.cnf)
server-id = 1
log-bin = mysql-bin
binlog_format = ROW  -- или STATEMENT, MIXED

-- На replica (my.cnf)
server-id = 2
relay-log = relay-bin
read_only = 1

-- Подключить replica к master
CHANGE MASTER TO
  MASTER_HOST='master_ip',
  MASTER_USER='replication',
  MASTER_PASSWORD='password',
  MASTER_LOG_FILE='mysql-bin.000001',
  MASTER_LOG_POS=154;

START SLAVE;
SHOW SLAVE STATUS\G
```

**Semi-synchronous Replication:**
```sql
-- На master
INSTALL PLUGIN rpl_semi_sync_master SONAME 'semisync_master.so';
SET GLOBAL rpl_semi_sync_master_enabled = 1;
SET GLOBAL rpl_semi_sync_master_timeout = 1000;  -- 1 sec

-- На replica
INSTALL PLUGIN rpl_semi_sync_slave SONAME 'semisync_slave.so';
SET GLOBAL rpl_semi_sync_slave_enabled = 1;
```

**GTID (Global Transaction ID) - MySQL 5.6+:**
```sql
-- my.cnf
gtid_mode = ON
enforce_gtid_consistency = ON

-- Упрощает failover (не нужно помнить binlog position)
CHANGE MASTER TO
  MASTER_HOST='new_master_ip',
  MASTER_USER='replication',
  MASTER_PASSWORD='password',
  MASTER_AUTO_POSITION = 1;  -- автоматически найдет позицию через GTID
```

---

### Failover - автоматическое переключение на replica

**Проблема:** Master упал, нужно промотировать replica в новый master.

**Решения:**

**1. Ручной failover:**
```bash
# На replica
pg_ctl promote  # PostgreSQL

# MySQL
STOP SLAVE;
RESET SLAVE ALL;
SET GLOBAL read_only = 0;  # теперь можно писать
```

**2. Автоматический failover:**

**PostgreSQL: Patroni + etcd**
- Patroni мониторит master и replicas
- При падении master автоматически промотит replica
- etcd - distributed configuration store

**MySQL: Orchestrator или MHA (Master High Availability)**
- Автоматический failover за секунды
- Переконфигурирует replicas на новый master

**Redis Sentinel:**
```bash
# Sentinel мониторит master
sentinel monitor mymaster 127.0.0.1 6379 2  # quorum = 2
sentinel down-after-milliseconds mymaster 5000
sentinel failover-timeout mymaster 10000

# При падении master промотит replica (автоматически)
```

---

### Read Replicas в приложении (Laravel)

```php
// config/database.php
'mysql' => [
    'read' => [
        'host' => ['192.168.1.2', '192.168.1.3'],  // replicas
    ],
    'write' => [
        'host' => ['192.168.1.1'],  // master
    ],
    'driver' => 'mysql',
    'database' => 'mydb',
    ...
],

// Использование
DB::table('users')->get();  // автоматически READ с replica
DB::table('users')->insert([...]);  // WRITE на master

// Принудительно с master (если нужны свежие данные)
DB::table('users')->onWriteConnection()->get();
```

---

## 🗂️ Партиционирование (Partitioning)

### Что такое партиционирование?

**Partitioning** - разбиение большой таблицы на **физически отдельные части (партиции)**.
- Каждая партиция = отдельная таблица/файл
- Логически это одна таблица (SELECT FROM users работает)
- Улучшает производительность (сканируются только нужные партиции)
- Упрощает обслуживание (удалить старые данные = DROP PARTITION)

---

### PostgreSQL: Declarative Partitioning (10+)

**Range Partitioning** - по диапазону значений:
```sql
-- Партиции по дате (самый популярный случай)
CREATE TABLE events (
    id SERIAL,
    user_id INT,
    event_type VARCHAR(50),
    created_at TIMESTAMP NOT NULL
) PARTITION BY RANGE (created_at);

-- Создать партиции
CREATE TABLE events_2025_01 PARTITION OF events
    FOR VALUES FROM ('2025-01-01') TO ('2025-02-01');

CREATE TABLE events_2025_02 PARTITION OF events
    FOR VALUES FROM ('2025-02-01') TO ('2025-03-01');

CREATE TABLE events_2025_03 PARTITION OF events
    FOR VALUES FROM ('2025-03-01') TO ('2025-04-01');

-- INSERT автоматически роутится в нужную партицию
INSERT INTO events (user_id, event_type, created_at) 
VALUES (123, 'click', '2025-01-15');  -- попадет в events_2025_01

-- SELECT сканирует только нужные партиции (partition pruning)
SELECT * FROM events 
WHERE created_at >= '2025-01-10' AND created_at < '2025-01-20';
-- Сканирует ТОЛЬКО events_2025_01 (быстро!)

-- Удаление старых данных = DROP PARTITION (мгновенно)
DROP TABLE events_2024_01;  -- удалить январь 2024
```

**List Partitioning** - по списку значений:
```sql
CREATE TABLE users (
    id SERIAL,
    username VARCHAR(50),
    country VARCHAR(2) NOT NULL
) PARTITION BY LIST (country);

CREATE TABLE users_us PARTITION OF users FOR VALUES IN ('US');
CREATE TABLE users_eu PARTITION OF users FOR VALUES IN ('DE', 'FR', 'GB');
CREATE TABLE users_asia PARTITION OF users FOR VALUES IN ('CN', 'JP', 'IN');
```

**Hash Partitioning** - по хешу:
```sql
-- Равномерное распределение данных
CREATE TABLE logs (
    id BIGSERIAL,
    message TEXT,
    user_id INT NOT NULL
) PARTITION BY HASH (user_id);

CREATE TABLE logs_p0 PARTITION OF logs FOR VALUES WITH (MODULUS 4, REMAINDER 0);
CREATE TABLE logs_p1 PARTITION OF logs FOR VALUES WITH (MODULUS 4, REMAINDER 1);
CREATE TABLE logs_p2 PARTITION OF logs FOR VALUES WITH (MODULUS 4, REMAINDER 2);
CREATE TABLE logs_p3 PARTITION OF logs FOR VALUES WITH (MODULUS 4, REMAINDER 3);
-- user_id % 4 определяет партицию
```

---

### MySQL: Partitioning

**Range Partitioning:**
```sql
CREATE TABLE events (
    id INT,
    user_id INT,
    created_at DATETIME NOT NULL
)
PARTITION BY RANGE (YEAR(created_at)) (
    PARTITION p2023 VALUES LESS THAN (2024),
    PARTITION p2024 VALUES LESS THAN (2025),
    PARTITION p2025 VALUES LESS THAN (2026),
    PARTITION pmax VALUES LESS THAN MAXVALUE
);

-- Добавить новую партицию
ALTER TABLE events ADD PARTITION (
    PARTITION p2027 VALUES LESS THAN (2028)
);

-- Удалить старую
ALTER TABLE events DROP PARTITION p2023;
```

**Hash Partitioning:**
```sql
CREATE TABLE users (
    id INT,
    username VARCHAR(50)
)
PARTITION BY HASH(id) PARTITIONS 4;
```

**List Partitioning:**
```sql
CREATE TABLE orders (
    id INT,
    region VARCHAR(50)
)
PARTITION BY LIST COLUMNS(region) (
    PARTITION p_north VALUES IN ('US', 'CA'),
    PARTITION p_europe VALUES IN ('DE', 'FR', 'GB'),
    PARTITION p_asia VALUES IN ('CN', 'JP', 'IN')
);
```

---

### Когда использовать партиционирование?

**✅ Хорошо подходит:**
- Таблицы > 100GB
- Временные данные (логи, события, метрики)
- Регулярное удаление старых данных (DROP PARTITION быстрее DELETE)
- Запросы с фильтрацией по partition key (WHERE created_at >= ...)

**❌ Не подходит:**
- Маленькие таблицы (< 10GB)
- Нет естественного ключа для партиционирования
- Запросы без фильтрации по partition key (сканируются все партиции)

---

## 🌍 Шардирование (Sharding)

### Что такое шардирование?

**Sharding** - горизонтальное разделение данных между **разными серверами (shards)**.
- Партиционирование = одна БД, физически разные таблицы
- Шардирование = разные БД на разных серверах

**Зачем:**
- ✅ Масштабирование (один сервер не справляется)
- ✅ Распределение нагрузки (каждый shard обрабатывает свою часть)
- ✅ Больше данных (сумма дискового пространства shards)

**Недостатки:**
- ❌ Сложность (нужен роутинг запросов)
- ❌ Нет JOIN между shards
- ❌ Нет транзакций между shards
- ❌ Перешардирование сложное (изменить shard key)

---

### Стратегии шардирования

**1. Range-based Sharding** - по диапазону значений:
```
Shard 1: user_id 1 - 1,000,000
Shard 2: user_id 1,000,001 - 2,000,000
Shard 3: user_id 2,000,001 - 3,000,000
```

**Плюсы:**
- ✅ Простая логика
- ✅ Range queries эффективны (SELECT WHERE user_id BETWEEN ...)

**Минусы:**
- ❌ Неравномерное распределение (hotspot на новых user_id)
- ❌ Новые пользователи все на одном shard

**2. Hash-based Sharding** - по хешу:
```python
shard = hash(user_id) % num_shards

# Пример:
user_id = 12345
shard = hash(12345) % 4 = 2  # Shard 2
```

**Плюсы:**
- ✅ Равномерное распределение
- ✅ Нет hotspots

**Минусы:**
- ❌ Range queries неэффективны (broadcast на все shards)
- ❌ Перешардирование сложное (изменение num_shards = rehashing всех данных)

**3. Consistent Hashing** - для динамического количества shards:
```
Hash ring:
    0 ------- Shard1 ------- Shard2 ------- Shard3 ------- 2^32-1
    ^                                                          |
    |__________________________________________________________|

user_id -> hash(user_id) -> найти ближайший shard по часовой стрелке
```

**Плюсы:**
- ✅ Добавление/удаление shard = minimal rehashing
- ✅ Используется в Redis Cluster, Cassandra, DynamoDB

**4. Directory-based Sharding** - lookup таблица:
```sql
-- Таблица маппинга user -> shard
CREATE TABLE user_shard_map (
    user_id INT PRIMARY KEY,
    shard_id INT NOT NULL
);

-- Запрос:
-- 1. SELECT shard_id FROM user_shard_map WHERE user_id = 123
-- 2. Подключиться к shard_id
-- 3. SELECT * FROM users WHERE id = 123
```

**Плюсы:**
- ✅ Гибкость (можно менять shard для user)
- ✅ Неравномерное распределение (VIP users на быстрый shard)

**Минусы:**
- ❌ Дополнительный lookup запрос
- ❌ Lookup таблица = single point of failure

---

### Реализация шардирования

**Уровень приложения (Application-level sharding):**
```php
// Laravel: определить shard по user_id
class ShardManager
{
    public function getShardConnection($userId)
    {
        $shardId = $userId % 4;  // 4 shards
        
        return match($shardId) {
            0 => DB::connection('shard0'),
            1 => DB::connection('shard1'),
            2 => DB::connection('shard2'),
            3 => DB::connection('shard3'),
        };
    }
}

// Использование
$shard = app(ShardManager::class)->getShardConnection($userId);
$user = $shard->table('users')->where('id', $userId)->first();

// config/database.php
'shard0' => ['host' => '192.168.1.1', ...],
'shard1' => ['host' => '192.168.1.2', ...],
'shard2' => ['host' => '192.168.1.3', ...],
'shard3' => ['host' => '192.168.1.4', ...],
```

**Middleware/Proxy уровень:**
- **Vitess** (для MySQL) - sharding middleware от YouTube
- **Citus** (для PostgreSQL) - distributed extension
- **ProxySQL** - query router с sharding

**Встроенное шардирование:**
- **MongoDB** - автоматический sharding
- **Redis Cluster** - 16384 hash slots
- **Cassandra** - consistent hashing

---

### MongoDB Sharding

```javascript
// Включить sharding для БД
sh.enableSharding("mydb")

// Выбрать shard key
sh.shardCollection("mydb.users", { "user_id": 1 })
// Документы распределяются по user_id

// Hash-based sharding (равномерное)
sh.shardCollection("mydb.logs", { "_id": "hashed" })

// Статус sharding
sh.status()
```

**Shard key важен:**
- ❌ Плохой: sequential ID (новые данные на один shard)
- ✅ Хороший: hash(user_id), {country: 1, user_id: 1}

---

### Проблемы шардирования

**1. Нет JOIN между shards:**
```sql
-- Невозможно (users и posts на разных shards):
SELECT u.name, p.title
FROM users u
JOIN posts p ON u.id = p.user_id
WHERE u.id = 123;

-- Решение: денормализация (хранить user_name в posts)
```

**2. Нет транзакций между shards:**
```sql
-- Невозможно atomic update на 2 shards:
BEGIN;
UPDATE users SET balance = balance - 100 WHERE id = 123;  -- Shard 1
UPDATE users SET balance = balance + 100 WHERE id = 456;  -- Shard 2
COMMIT;

-- Решение: Saga pattern, eventual consistency
```

**3. Hotspots (неравномерная нагрузка):**
```
Shard 1: 1000 req/sec
Shard 2: 100 req/sec   -- недогружен
Shard 3: 5000 req/sec  -- HOTSPOT!

-- Причина: плохой shard key (celebrity user на Shard 3)
-- Решение: rehashing, split shard
```

**4. Перешардирование (re-sharding):**
- Изменение количества shards требует миграции данных
- Downtime или сложная миграция с dual-write

---

## 🔺 CAP теорема

### Что такое CAP?

**CAP треугольник** - в распределенной системе можно гарантировать только **2 из 3**:

1. **C**onsistency - согласованность (все узлы видят одинаковые данные)
2. **A**vailability - доступность (система всегда отвечает, даже если узел упал)
3. **P**artition tolerance - устойчивость к разделению сети (система работает при разрыве связи между узлами)

**Теорема:** В реальных распределенных системах **Partition** неизбежен (сеть может упасть). Выбор всегда между **CP** или **AP**.

---

### CP - Consistency + Partition Tolerance

**Приоритет: согласованность > доступность**

- ✅ Все узлы видят одинаковые данные (strong consistency)
- ❌ Если узел недоступен - запросы к нему отклоняются (система частично недоступна)

**Примеры:**
- **MongoDB** (primary-secondary)
  - Если primary недоступен, replicas read-only (нет записей до failover)
- **HBase**
- **Redis Cluster** (в режиме синхронной репликации)

**Сценарий:**
```
Master (write) ---X--- Replica (read)
        |  Network partition
        
Приложение пишет в Master ✅
Приложение читает с Replica ❌ (отклонено, т.к. данные могут быть устаревшими)
```

---

### AP - Availability + Partition Tolerance

**Приоритет: доступность > согласованность**

- ✅ Система всегда отвечает (даже если узлы не синхронизированы)
- ❌ Данные могут быть несогласованными (eventual consistency)

**Примеры:**
- **Cassandra**
- **DynamoDB**
- **Riak**
- **CouchDB**

**Сценарий:**
```
Node1 ---X--- Node2
  |   Network partition
  
Приложение пишет в Node1 ✅ (данные на Node1)
Приложение читает с Node2 ✅ (старые данные, т.к. еще не синхронизировано)
-- Eventual consistency: через некоторое время Node2 получит изменения
```

**Quorum-based consistency (настраиваемая):**
```
N = 3 (всего replicas)
W = 2 (write quorum - подтверждение от 2 узлов)
R = 2 (read quorum - читать с 2 узлов)

W + R > N  =>  strong consistency (2 + 2 > 3)
W + R <= N  =>  eventual consistency
```

**Cassandra пример:**
```sql
-- Write с quorum
INSERT INTO users (id, name) VALUES (1, 'John') 
USING CONSISTENCY QUORUM;  -- ждем подтверждения от 2 из 3 узлов

-- Read с quorum
SELECT * FROM users WHERE id = 1 
USING CONSISTENCY QUORUM;  -- читаем с 2 из 3 узлов, сравниваем timestamps
```

---

### CA - Consistency + Availability (без Partition Tolerance)

**Только в single-node системах** (нет распределенности):
- **PostgreSQL** (один сервер)
- **MySQL** (один сервер)

**Как только добавляешь репликацию - выбор между CP или AP.**

---

### Для собеседования

**Вопрос:** "В чем разница между PostgreSQL с репликацией и Cassandra?"

**Ответ:**
- **PostgreSQL (CP)**:
  - Синхронная репликация = strong consistency
  - Если replica недоступна - write блокируется (consistency > availability)
  - ACID транзакции, сложные JOIN
  
- **Cassandra (AP)**:
  - Eventual consistency (все узлы в конце концов синхронизируются)
  - Узлы могут быть недоступны - система продолжает работать
  - Tunable consistency (можно настроить quorum)
  - Нет JOIN, нет транзакций

---

## 🎓 Ключевые точки для собеседования

### OLTP vs OLAP:
- ✅ OLTP = транзакции (INSERT/UPDATE/DELETE), row-oriented, нормализация, PostgreSQL/MySQL
- ✅ OLAP = аналитика (агрегации), column-oriented, денормализация, ClickHouse/Redshift
- ✅ Разные паттерны использования = разные БД

### Репликация:
- ✅ Master-Slave для HA и read scaling
- ✅ Асинхронная vs Синхронная vs Semi-sync
- ✅ Replication lag - проблема stale reads
- ✅ Failover: ручной vs автоматический (Patroni, Orchestrator, Sentinel)
- ✅ Physical vs Logical replication (PostgreSQL)

### Партиционирование:
- ✅ Разделение большой таблицы на меньшие части (партиции)
- ✅ Range, List, Hash партиционирование
- ✅ Partition pruning = сканирование только нужных партиций
- ✅ DROP PARTITION быстрее DELETE (удаление старых данных)

### Шардирование:
- ✅ Горизонтальное разделение данных между серверами
- ✅ Range, Hash, Consistent Hashing, Directory-based
- ✅ Нет JOIN/транзакций между shards
- ✅ Shard key критичен (hotspots, rehashing)
- ✅ Application-level vs Middleware (Vitess, Citus) vs Built-in (MongoDB)

### CAP теорема:
- ✅ В распределенных системах: CP (consistency) или AP (availability)
- ✅ PostgreSQL с sync replication = CP
- ✅ Cassandra = AP с eventual consistency
- ✅ Quorum: W + R > N = strong consistency
