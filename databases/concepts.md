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
-- Продажи по регионам за год (в ClickHouse — доли секунды/секунды на миллиардах строк)
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
- **Query time** - время выполнения запроса (от долей секунды до минут на тяжёлых отчётах)
- **Compression ratio** - степень сжатия данных

**БД для OLAP:**
- ClickHouse, Amazon Redshift, Google BigQuery, Snowflake
- PostgreSQL (не оптимален, но можно для небольших объемов)

**Пример: Data Warehouse для e-commerce**
```sql
-- Fact table (звездообразная схема): ключи измерений + числовые метрики
CREATE TABLE fact_sales (
    sale_id BIGINT,
    date_id INT,           -- FK to dim_date
    user_id INT,           -- FK to dim_users
    product_id INT,        -- FK to dim_products
    quantity INT,
    amount DECIMAL(10,2)
) ENGINE = MergeTree()     -- синтаксис ClickHouse
ORDER BY (date_id, product_id);

-- Dimension tables — денормализованы (в этом смысл звезды)
dim_date: date_id, date, year, quarter, month, week, day_of_week
dim_users: user_id, email, name, registration_date, segment, region
dim_products: product_id, name, category, subcategory, brand
-- category и subcategory лежат прямо в dim_products, а не в отдельных таблицах.
-- Snowflake schema = те же измерения, но нормализованные:
-- dim_products → dim_subcategories → dim_categories (меньше дублей, больше JOIN)
```

> На практике в ClickHouse часто идут дальше звезды и делают одну широкую плоскую таблицу (region, category прямо в факте): JOIN'ы там дорогие, а колоночное хранение и сжатие делают дублирование дешёвым. Но это уже не star schema, а wide/flat table.

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
3. **Geo-distribution** - replicas в разных регионах (низкая latency)

> ⚠️ **Реплика — НЕ бэкап.** Случайный `DELETE FROM users` или `DROP TABLE` мгновенно реплицируется на все реплики. Бэкап — это копия **на момент времени**, к которой можно откатиться: `pg_dump`, PITR (базовый бэкап + архив WAL), снапшоты. Реплика защищает от падения сервера, бэкап — от ошибки человека или приложения. Частый вопрос-ловушка.

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
  | (асинхронно, обычно миллисекунды задержки)
  ↓
Replica: INSERT INTO users ... ✅ (под нагрузкой или при долгих запросах на реплике — секунды и больше)
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
- ✅ Нет потери подтверждённых данных при падении master
- ⚠️ Но это НЕ значит «replica всегда актуальна»: в PostgreSQL при `synchronous_commit = on` master ждёт, пока WAL **записан на диск** реплики, а не **применён**. Чтение с реплики сразу после коммита может вернуть старые данные. Гарантию «прочитаю своё» даёт только `synchronous_commit = remote_apply`
- ❌ Медленнее (ждем replica)
- ❌ Если replica недоступна - пишущие запросы зависнут

**Semi-synchronous репликация (MySQL):**
- Master ждет подтверждение от хотя бы 1 replica, что событие **получено** (записано в relay log), а не применено
- Баланс между скоростью и безопасностью
- ⚠️ При таймауте (`rpl_semi_sync_source_timeout`) **молча откатывается в async** — гарантии пропадают именно тогда, когда что-то пошло не так

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
```bash
# На master (postgresql.conf)
# Синхронность включает именно этот параметр; имя должно совпадать
# с application_name в primary_conninfo реплики
synchronous_standby_names = 'replica1'

# synchronous_commit = on — значение по умолчанию: ждать, пока WAL записан на диск реплики
# synchronous_commit = remote_apply — ждать, пока реплика ПРИМЕНИЛА изменения
#   (чтение с реплики сразу увидит коммит, но коммит медленнее)
```

**Logical Replication (PostgreSQL 10+):**
- Репликация на уровне таблиц/строк (не весь WAL)
- Можно реплицировать только часть БД
- Разные версии PostgreSQL (master 12, replica 14) → используется для апгрейда мажорной версии почти без простоя
- ⚠️ **Схема (DDL) не реплицируется**: таблицы на подписчике нужно создать заранее, а `ALTER TABLE` применять на обеих сторонах вручную

```sql
-- На master
CREATE PUBLICATION my_pub FOR TABLE users, orders;

-- На replica (таблицы users и orders уже должны существовать)
CREATE SUBSCRIPTION my_sub 
CONNECTION 'host=master_ip dbname=mydb user=replication' 
PUBLICATION my_pub;
```

---

### MySQL: Replication

> С MySQL 8.0.22+ терминология master/slave заменена на **source/replica**: `CHANGE REPLICATION SOURCE TO`, `START REPLICA`, `SHOW REPLICA STATUS`. Старые команды (`CHANGE MASTER TO`, `START SLAVE`) объявлены устаревшими и в новых версиях (8.4) удалены.

**Asynchronous Replication (по умолчанию):**
```sql
-- На source (my.cnf)
server-id = 1
log-bin = mysql-bin
binlog_format = ROW  -- по умолчанию с 5.7; STATEMENT/MIXED — устаревшие варианты

-- На replica (my.cnf)
server-id = 2
relay-log = relay-bin
read_only = 1

-- Подключить replica к source
CHANGE REPLICATION SOURCE TO
  SOURCE_HOST='source_ip',
  SOURCE_USER='replication',
  SOURCE_PASSWORD='password',
  SOURCE_LOG_FILE='mysql-bin.000001',
  SOURCE_LOG_POS=154;

START REPLICA;
SHOW REPLICA STATUS\G
```

**Semi-synchronous Replication:**
```sql
-- На source
INSTALL PLUGIN rpl_semi_sync_source SONAME 'semisync_source.so';
SET GLOBAL rpl_semi_sync_source_enabled = 1;
SET GLOBAL rpl_semi_sync_source_timeout = 1000;  -- 1 sec, потом fallback в async

-- На replica
INSTALL PLUGIN rpl_semi_sync_replica SONAME 'semisync_replica.so';
SET GLOBAL rpl_semi_sync_replica_enabled = 1;
```

**GTID (Global Transaction ID) - MySQL 5.6+:**
```sql
-- my.cnf
gtid_mode = ON
enforce_gtid_consistency = ON

-- Упрощает failover (не нужно помнить binlog position)
CHANGE REPLICATION SOURCE TO
  SOURCE_HOST='new_source_ip',
  SOURCE_USER='replication',
  SOURCE_PASSWORD='password',
  SOURCE_AUTO_POSITION = 1;  -- автоматически найдет позицию через GTID
```

---

### Failover - автоматическое переключение на replica

**Проблема:** Master упал, нужно промотировать replica в новый master.

**Решения:**

**1. Ручной failover:**
```bash
# На replica
pg_ctl promote  # PostgreSQL

# MySQL (8.0.22+)
STOP REPLICA;
RESET REPLICA ALL;
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
    // После записи в рамках того же HTTP-запроса/джобы все чтения идут на master.
    // Решает главную проблему replication lag: «записал — и сразу не увидел»
    'sticky' => true,
    'driver' => 'mysql',
    'database' => 'mydb',
    ...
],

// Использование
DB::table('users')->get();  // автоматически READ с replica
DB::table('users')->insert([...]);  // WRITE на master

// Принудительно с master (если нужны свежие данные)
DB::table('users')->useWritePdo()->get();   // Query Builder
User::onWriteConnection()->find($id);       // Eloquent
```

> `sticky` работает только в пределах одного запроса или джобы. Если пользователь создал заказ и *следующим* запросом открыл список заказов, этот запрос может уйти на отстающую реплику. Для таких экранов читай с master явно.

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

-- Удаление старых данных: в PostgreSQL нет DROP PARTITION, партиция — обычная таблица
DROP TABLE events_2025_01;  -- удалить январь 2025 (мгновенно, в отличие от DELETE)
-- или сначала отцепить, а потом удалить/заархивировать:
ALTER TABLE events DETACH PARTITION events_2025_01;

-- Без DEFAULT-партиции INSERT вне всех диапазонов упадёт с ошибкой
CREATE TABLE events_default PARTITION OF events DEFAULT;
```

> ⚠️ Частый подвох: PRIMARY KEY и UNIQUE на партиционированной таблице **обязаны включать ключ партиционирования**. `PRIMARY KEY (id)` здесь не создастся, нужен `PRIMARY KEY (id, created_at)`. Уникальность только по `id` глобально не гарантируется.

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
-- hash(user_id) % 4 определяет партицию (встроенная хеш-функция PostgreSQL, не сам user_id % 4)
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

-- Добавить новую партицию: ADD PARTITION после MAXVALUE не сработает (ошибка),
-- поэтому «разрезаем» pmax
ALTER TABLE events REORGANIZE PARTITION pmax INTO (
    PARTITION p2026 VALUES LESS THAN (2027),
    PARTITION pmax VALUES LESS THAN MAXVALUE
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
- ❌ Нет транзакций между shards из коробки (2PC/XA возможен, но блокирующий — обычно выбирают Saga)
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
```
shard = hash(user_id) % num_shards

Пример: user_id = 12345 → hash(12345) % 4 = 2 → Shard 2
```

**Плюсы:**
- ✅ Равномерное распределение **ключей**
- ✅ Нет hotspot от последовательных id (новые пользователи разлетаются по всем shards)
- ⚠️ Но хеш не спасает от **горячего ключа**: если на одного пользователя (знаменитость, популярный товар) приходится 50% запросов, его shard всё равно перегружен (см. «Проблемы шардирования»)

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
- ✅ Добавление/удаление shard = переезжает только ~1/N данных (соседний участок кольца), а не всё
- ✅ Используется в Cassandra, Amazon Dynamo (и DynamoDB), Riak

**Virtual nodes:** с несколькими точками на кольце участки получаются разного размера, и распределение неравномерное. Поэтому каждый физический сервер ставят на кольцо много раз (сотни виртуальных нод): нагрузка выравнивается, а при падении сервера его данные расходятся по многим соседям, а не валятся на одного.

> ⚠️ Redis Cluster использует **не** consistent hashing, а фиксированные **16384 hash slots**: `CRC16(key) % 16384`. Слоты распределены между нодами и переносятся между ними при решардинге. Идея похожая (двигаем куски, а не перехешируем всё), но механизм другой.

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
- ✅ Гибкость (можно перенести конкретного user на другой shard)
- ✅ Контролируемое размещение (крупные клиенты/VIP — на выделенный shard)

**Минусы:**
- ❌ Дополнительный lookup запрос (обычно кэшируется)
- ❌ Lookup таблица = single point of failure, если её не реплицировать

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

-- Причина: горячий ключ (celebrity user на Shard 3)
-- Перехеширование и split shard НЕ помогут: один ключ не разрежешь, он всё равно попадёт на один shard
-- Решения:
--   1. Кэш перед БД (Redis) — основная нагрузка горячего ключа уходит из базы
--   2. "Засолить" ключ: писать в user:123:0..N, при чтении агрегировать (для счётчиков, лайков)
--   3. Вынести горячего клиента на выделенный shard (directory-based)
```

Если же hotspot из-за **плохого shard key** (например, range по последовательному id), тогда помогает смена ключа или стратегии (hash), то есть перешардирование.

**4. Перешардирование (re-sharding):**
- Изменение количества shards требует миграции данных
- Downtime или сложная миграция с dual-write

---

## 🔺 CAP теорема

### Что такое CAP?

Три свойства распределённой системы:

1. **C**onsistency - согласованность (линеаризуемость: каждое чтение видит последнюю подтверждённую запись, система ведёт себя как одна копия данных)
2. **A**vailability - доступность (каждый запрос к **живому** узлу получает не-ошибочный ответ)
3. **P**artition tolerance - устойчивость к разделению сети (система продолжает работать при разрыве связи между узлами)

**Теорема:** если сеть разделилась (P), нужно выбрать **C или A**: либо отказать в запросе, либо ответить, возможно, устаревшими данными. Пока сеть в порядке, можно иметь и C, и A.

> ⚠️ Популярная формулировка «можно гарантировать только 2 из 3» вводит в заблуждение: P в распределённой системе не выбирают, разрывы сети неизбежны. Выбор всегда между **CP** и **AP** — и только на время partition.
>
> Ещё одно различие: A в CAP ≠ high availability (99.99%). CP-система может иметь отличный uptime, если при разрывах большинство узлов обычно остаётся вместе.

---

### CP - Consistency + Partition Tolerance

**Приоритет: согласованность > доступность**

- ✅ Все узлы видят одинаковые данные (strong consistency)
- ❌ Если узел недоступен - запросы к нему отклоняются (система частично недоступна)

**Примеры:**
- **MongoDB** (primary-secondary, `w: majority`)
  - Если primary недоступен, replicas read-only (нет записей до failover)
- **HBase**
- **etcd, ZooKeeper, Consul** (консенсус Raft/ZAB)
- **CockroachDB, Spanner** (NewSQL)

> ⚠️ **Redis Cluster — не CP.** Репликация асинхронная, и при failover кластер может потерять подтверждённые записи; документация Redis прямо говорит, что strong consistency нет. Даже `WAIT` этого не меняет.

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

W + R > N  =>  кворумы пересекаются: чтение задевает хотя бы одну реплику с последней подтверждённой записью (2 + 2 > 3)
W + R <= N  =>  eventual consistency
```

> ⚠️ `W + R > N` — это **не** строгая консистентность (линеаризуемость). При конкурентных записях, last-write-wins по часам с рассинхроном и sloppy quorum (hinted handoff) аномалии возможны. Формулировка для собеседования: «почти всегда видим последнюю запись, но гарантии линеаризуемости нет».

**Cassandra пример:**
```sql
-- Consistency level задаётся на запрос в драйвере, а в cqlsh — отдельной командой
-- (синтаксис USING CONSISTENCY удалён ещё в CQL3)
CONSISTENCY QUORUM;

-- Write: ждем подтверждения от 2 из 3 реплик
INSERT INTO users (id, name) VALUES (1, 'John');

-- Read: читаем с 2 из 3 реплик, берём версию с самым свежим timestamp
SELECT * FROM users WHERE id = 1;
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
- **PostgreSQL (одиночный сервер — вне CAP; кластер с синхронной репликацией — ближе к CP по записи)**:
  - Синхронная репликация = нет потери подтверждённых записей
  - Если синхронная replica недоступна - write блокируется (consistency > availability)
  - Но чтения с реплик при partition продолжаются и могут быть устаревшими, а failover внешний (Patroni), не часть протокола → в CAP PostgreSQL-кластер ложится неидеально
  - ACID транзакции, сложные JOIN
  
- **Cassandra (AP)**:
  - Eventual consistency (все узлы в конце концов синхронизируются)
  - Узлы могут быть недоступны - система продолжает работать
  - Tunable consistency (можно настроить quorum)
  - Нет JOIN и ACID-транзакций; есть только lightweight transactions (`IF NOT EXISTS`, на Paxos) для одной партиции

---

## 🎓 Ключевые точки для собеседования

### OLTP vs OLAP:
- ✅ OLTP = транзакции (INSERT/UPDATE/DELETE), row-oriented, нормализация, PostgreSQL/MySQL
- ✅ OLAP = аналитика (агрегации), column-oriented, денормализация, ClickHouse/Redshift
- ✅ Разные паттерны использования = разные БД

### Репликация:
- ✅ Master-Slave для HA и read scaling
- ✅ Реплика — НЕ бэкап (DELETE/DROP реплицируется мгновенно)
- ✅ Асинхронная vs Синхронная vs Semi-sync (semi-sync при таймауте молча падает в async)
- ✅ Синхронная репликация защищает от потери данных, но не от stale reads (кроме `remote_apply`)
- ✅ Replication lag - проблема stale reads; в Laravel — `'sticky' => true`
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
- ✅ Нет JOIN/транзакций между shards (из коробки; 2PC блокирующий → Saga)
- ✅ Shard key критичен (hotspots, rehashing); hash спасает от последовательных id, но не от горячего ключа
- ✅ Redis Cluster = 16384 hash slots, не consistent hashing
- ✅ Application-level vs Middleware (Vitess, Citus) vs Built-in (MongoDB)

### CAP теорема:
- ✅ При partition выбираем CP (отказать) или AP (ответить, возможно устаревшими данными); не «2 из 3»
- ✅ A в CAP ≠ high availability
- ✅ PostgreSQL с sync replication — ближе к CP по записи, но реплики могут отдавать устаревшие чтения
- ✅ Cassandra = AP с eventual consistency
- ✅ Quorum: W + R > N — кворумы пересекаются, но это не линеаризуемость
