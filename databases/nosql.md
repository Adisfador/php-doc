# NoSQL базы данных

Детальный обзор популярных NoSQL решений: Redis, MongoDB, ClickHouse, Elasticsearch. Когда использовать, архитектура, примеры.

---

## 🎯 NoSQL vs SQL

### Зачем NoSQL?

**Ограничения реляционных БД:**
- ❌ Жесткая схема (ALTER TABLE на миллиардах строк = downtime)
- ❌ Вертикальное масштабирование (один мощный сервер дорого)
- ❌ JOIN медленный на больших данных
- ❌ ACID транзакции = блокировки = меньше throughput

**Преимущества NoSQL:**
- ✅ Гибкая/отсутствующая схема (schema-less или schema-flexible)
- ✅ Горизонтальное масштабирование (добавить серверов легко)
- ✅ Высокая пропускная способность (тысячи запросов/сек)
- ✅ Специализация (документы, графы, time-series, key-value)

### CAP теорема

**CAP треугольник** - можно выбрать только 2 из 3:
- **C**onsistency - согласованность (все видят одинаковые данные)
- **A**vailability - доступность (система всегда отвечает)
- **P**artition tolerance - устойчивость к разделению сети (система продолжает работать, когда узлы потеряли связь друг с другом)

**Почему P - не "бонус", а данность:**
- P - это не "система распределённая", а "система переживает обрыв связи между своими узлами"
- Сам по себе P ничего не даёт пользователю - он описывает, что система **не развалилась**, когда сеть порвалась
- Как только серверов больше одного (реплики, шарды, дата-центры) - обрыв сети рано или поздно **случится** (упал свитч, лаг между ДЦ, GC-пауза выглядит как недоступность)
- Отказаться от P в распределённой системе нельзя - это значит "при любом сетевом сбое вся система ложится"
- Поэтому реальный вопрос CAP: **что жертвуем в момент разрыва - C или A?**

**Пример:** 2 узла (Москва и Франкфурт), связь между ними пропала, клиент пишет в Москву:
- **CP** - Москва отказывает в записи (ошибка/таймаут), т.к. не может синхронизироваться с Франкфуртом → данные согласованы, но система недоступна
- **AP** - Москва принимает запись, Франкфурт продолжает отдавать старые данные → система отвечает, но узлы временно расходятся (потом синхронизируются - eventual consistency)

**Примеры:**
- **CA** - традиционные RDBMS (PostgreSQL, MySQL) в single-node
- **CP** - MongoDB, HBase (consistency > availability). Redis Cluster строгим CP не является: асинхронная репликация, при failover возможна потеря подтверждённых записей
- **AP** - Cassandra, DynamoDB, Riak (availability > consistency)

**В реальности:**
- Partition (разделение сети) неизбежно в распределенных системах
- Выбор всегда между **CP** или **AP**
- SQL при синхронной репликации тяготеет к **CP**; при асинхронной - согласованность между репликами eventual
- NoSQL делится: **CP** (MongoDB, HBase) и **AP** (Cassandra, DynamoDB) с eventual consistency

---

## 🔴 Redis - In-Memory Data Store

### Что такое Redis?

**Redis** (REmote DIctionary Server) - in-memory key-value хранилище:
- Все данные в RAM (диск только для persistence)
- Скорость: ~100,000 операций/сек на одном ядре
- Атомарные операции на структурах данных
- Single-threaded (6.0+ - многопоточный I/O)
- Pub/Sub messaging
- Lua скрипты для сложной логики

### Структуры данных

**1. String - простой key-value:**
```redis
# SET/GET
SET user:123:name "John Doe"
GET user:123:name  # "John Doe"

# Атомарный инкремент (счетчики)
SET page:views 0
INCR page:views  # 1
INCRBY page:views 5  # 6
DECR page:views  # 5

# Expiration (TTL)
SET session:abc123 "user_data" EX 3600  # истекает через 1 час
TTL session:abc123  # 3599, 3598, ... -2 (expired)

# SET if Not eXists (блокировка)
SET lock:resource "locked" NX EX 10  # установить только если не существует
# Если вернул OK - блокировка получена, если nil - уже заблокировано
```

**Laravel cache:**
```php
// String под капотом
Cache::put('key', 'value', now()->addMinutes(10));
Cache::get('key');  // "value"
Cache::increment('views');
```

**2. Hash - объект с полями:**
```redis
# HSET/HGET
HSET user:123 name "John" email "john@example.com" age 30
HGET user:123 name  # "John"
HGETALL user:123  # {"name": "John", "email": "john@example.com", "age": 30}

# Атомарный инкремент поля
HINCRBY user:123 age 1  # 31

# Множественные операции
HMSET user:123 name "John" email "john@example.com"
HMGET user:123 name email  # ["John", "john@example.com"]

# Проверка существования поля
HEXISTS user:123 name  # 1 (true)

# Удалить поле
HDEL user:123 age
```

**Когда использовать Hash:**
- Объекты с несколькими полями (user, product, post)
- Избегать JSON сериализации (быстрее, меньше памяти)
- Атомарные операции на полях

**3. List - упорядоченный список:**
```redis
# LPUSH/RPUSH - добавить в начало/конец
LPUSH queue:tasks "task1"  # слева
RPUSH queue:tasks "task2"  # справа

# LPOP/RPOP - извлечь с начала/конца
LPOP queue:tasks  # "task1"
RPOP queue:tasks  # "task2"

# LRANGE - получить диапазон
RPUSH logs "log1" "log2" "log3"
LRANGE logs 0 -1  # все элементы ["log1", "log2", "log3"]
LRANGE logs 0 1  # первые 2 ["log1", "log2"]

# LTRIM - обрезать список (сохранить только диапазон)
LTRIM logs -100 -1  # сохранить последние 100 элементов (0 99 оставил бы первые 100)

# BLPOP - блокирующий POP (для очередей)
BLPOP queue:tasks 0  # ждать бесконечно пока не появится элемент
```

**Применение:**
- Очереди задач (FIFO: RPUSH + LPOP)
- Стеки (LIFO: LPUSH + LPOP)
- Логи с ограничением (RPUSH + LTRIM)
- Timeline (последние N постов)

**Laravel queues:**
```php
// Redis queue использует Lists
Queue::push(new SendEmail($user));
// RPUSH queues:default "{job_data}"
```

**4. Set - неупорядоченное множество уникальных элементов:**
```redis
# SADD - добавить элементы
SADD tags:post:123 "php" "laravel" "redis"
SADD tags:post:456 "php" "mysql" "docker"

# SMEMBERS - все элементы
SMEMBERS tags:post:123  # ["php", "laravel", "redis"]

# SISMEMBER - проверка наличия
SISMEMBER tags:post:123 "php"  # 1 (true)

# SCARD - количество элементов
SCARD tags:post:123  # 3

# Операции множеств
SINTER tags:post:123 tags:post:456  # пересечение ["php"]
SUNION tags:post:123 tags:post:456  # объединение ["php", "laravel", "redis", "mysql", "docker"]
SDIFF tags:post:123 tags:post:456   # разность ["laravel", "redis"]

# SREM - удалить элемент
SREM tags:post:123 "redis"

# SPOP - извлечь случайный элемент
SPOP tags:post:123  # "php" (удаляет из set)

# SRANDMEMBER - получить случайный (не удаляет)
SRANDMEMBER tags:post:123  # "laravel"
```

**Применение:**
- Тэги, категории
- Уникальные посетители (IP адреса)
- Отношения many-to-many (friends, followers)
- Операции множеств (общие друзья, похожие теги)

**5. Sorted Set (ZSet) - отсортированное множество с score:**
```redis
# ZADD - добавить с score
ZADD leaderboard 100 "user1" 200 "user2" 150 "user3"

# ZRANGE - получить по рангу (по возрастанию score)
ZRANGE leaderboard 0 -1 WITHSCORES
# ["user1", "100", "user3", "150", "user2", "200"]

# ZREVRANGE - по убыванию
ZREVRANGE leaderboard 0 2  # топ-3 ["user2", "user3", "user1"]

# ZRANK - ранг элемента
ZRANK leaderboard "user3"  # 1 (второе место, 0-based)

# ZSCORE - получить score
ZSCORE leaderboard "user2"  # 200

# ZINCRBY - атомарный инкремент score
ZINCRBY leaderboard 50 "user1"  # 150

# ZRANGEBYSCORE - диапазон по score
ZRANGEBYSCORE leaderboard 100 200  # score от 100 до 200

# ZREMRANGEBYRANK - удалить по рангу
ZREMRANGEBYRANK leaderboard 0 0  # удалить последнее место

# ZCARD - количество элементов
ZCARD leaderboard  # 3
```

**Применение:**
- Leaderboards (рейтинги игроков)
- Приоритетные очереди (score = priority)
- Временные ленты (score = timestamp)
- Автодополнение (score = relevance)
- Rate limiting (score = timestamp)

**Laravel example:**
```php
// Rate limiting с Sorted Set
// member должен быть уникальным, иначе повторные запросы перезапишут друг друга
Redis::zadd("rate_limit:user:123", microtime(true), uniqid('', true));
// Удалить старые записи (> 1 минуты назад)
Redis::zremrangebyscore("rate_limit:user:123", 0, time() - 60);
// Проверить количество за последнюю минуту
$count = Redis::zcard("rate_limit:user:123");
if ($count > 100) {
    abort(429, 'Too Many Requests');
}
```

### Persistence (сохранение на диск)

**RDB (Redis Database) - snapshot:**
```redis
# Настройка в redis.conf:
save 900 1      # сохранить если 1 изменение за 15 минут
save 300 10     # или 10 изменений за 5 минут
save 60 10000   # или 10000 изменений за 1 минуту

# Команды:
SAVE        # синхронный snapshot (блокирует)
BGSAVE      # background snapshot (fork процесс)
```

**Плюсы RDB:**
- ✅ Компактный (один файл dump.rdb)
- ✅ Быстрое восстановление
- ✅ Минимальное влияние на производительность

**Минусы RDB:**
- ❌ Можем потерять данные между snapshot'ами (до 15 минут)
- ❌ Fork может быть медленным на больших данных

**AOF (Append Only File) - лог команд:**
```redis
# Настройка в redis.conf:
appendonly yes
appendfilename "appendonly.aof"

# Синхронизация:
appendfsync always      # fsync на каждую команду (медленно, безопасно)
appendfsync everysec    # fsync раз в секунду (баланс) - РЕКОМЕНДУЕТСЯ
appendfsync no          # полагаться на ОС (быстро, небезопасно)
```

**Плюсы AOF:**
- ✅ Лучшая durability (потеря макс 1 секунды с everysec)
- ✅ Лог читаемый (можно редактировать)

**Минусы AOF:**
- ❌ Файл больше RDB
- ❌ Восстановление медленнее (replay команд)

**Рекомендация: RDB + AOF вместе** (Redis 4.0+):
```redis
# redis.conf
appendonly yes
save 900 1
aof-use-rdb-preamble yes  # AOF файл с RDB snapshot + новые команды
```

### Репликация и Redis Cluster

Это **два альтернативных режима**, а не слои одной схемы:

|          | Replication + Sentinel                        | Redis Cluster                                  |
| -------- | --------------------------------------------- | ---------------------------------------------- |
| Данные   | Все на одном master, реплики - полные копии   | Поделены между несколькими master              |
| Запись   | Один master                                   | Несколько master, каждый - свои слоты          |
| Failover | Отдельные процессы Sentinel                   | Встроен (голосуют master'ы)                    |
| Зачем    | Отказоустойчивость (+ чтение, если настроить) | Объём данных и запись не влезают в один сервер |

**Replication + Sentinel:**
```redis
# На каждой реплике:
replicaof 192.168.1.1 6379  # IP и порт master
```
- Master асинхронно шлёт поток команд репликам
- Реплики read-only = **запись запрещена, читать можно**. Но Redis сам чтение не распределяет - приложение должно явно слать чтение на реплики (данные могут отставать)
- **Sentinel - отдельный процесс** (`redis-sentinel`, тот же пакет), минимум 3 штуки для голосования большинством
- Sentinel следят за master, при падении промотят реплику; клиент сначала спрашивает у Sentinel "кто сейчас master?"

**Redis Cluster - шардирование + репликация:**
- Ключ → слот: `CRC16(имя ключа) % 16384` - **автоматически для любых ключей**, настраивать ничего не нужно
- Минимум 3 master (рекомендуется 6: 3 master + 3 replica)
- Реплики по умолчанию - **горячий резерв**: на чтение отвечают `MOVED` к master. Читать с них - только после `READONLY` в соединении
- Клиент должен уметь в кластер: не та нода → `MOVED 3999 10.0.0.2:6379` → редирект (phpredis `RedisCluster`, Predis умеют)
- Нет multi-key операций, если ключи в разных слотах
- Добавили ноду → перенести слоты вручную: `redis-cli --cluster reshard` / `rebalance` (кластер работает во время переноса)

```bash
# 1. На каждом из 6 серверов: cluster-enabled yes в redis.conf → запустить redis-server
# 2. ОДИН раз с любой машины, видящей все ноды:
redis-cli --cluster create \
  10.0.0.1:6379 10.0.0.2:6379 10.0.0.3:6379 \
  10.0.0.4:6379 10.0.0.5:6379 10.0.0.6:6379 \
  --cluster-replicas 1
# Состояние кластера каждая нода хранит в nodes.conf
```

**CAP: ни CP, ни AP** (и Cluster, и Sentinel):
- Не C: репликация асинхронная → master подтвердил запись и упал до репликации → после failover запись потеряна
- Не A: master в меньшинстве перестаёт принимать запись после `cluster-node-timeout`
- Уменьшить окно потерь: `min-replicas-to-write 1`, `WAIT 1 100` - но это не делает Redis CP
- Поэтому Redis - для кеша, сессий, очередей, счётчиков, а не для денег

```redis
# Hash tags для multi-key операций в Cluster
# Ключи с одинаковым {hash_tag} попадут на один узел
SET {user:123}:name "John"
SET {user:123}:email "john@example.com"
# Оба ключа на одном узле (hash по "user:123")
```

### Использование Redis в Laravel

**1. Кеширование:**
```php
use Illuminate\Support\Facades\Cache;

// Простой кеш
Cache::put('key', 'value', now()->addMinutes(10));
Cache::get('key');
Cache::forget('key');

// Remember (кеш или вычислить)
$users = Cache::remember('users:active', 3600, function () {
    return DB::table('users')->where('active', true)->get();
});

// Cache Tags (только Redis, Memcached)
Cache::tags(['users', 'active'])->put('key', 'value', 600);
Cache::tags(['users', 'active'])->get('key');
Cache::tags(['users'])->flush();  // очистить все с тэгом 'users'
```

**2. Очереди:**
```php
// config/queue.php
'redis' => [
    'driver' => 'redis',
    'connection' => 'default',
    'queue' => env('REDIS_QUEUE', 'default'),
],

// Dispatch job
ProcessPodcast::dispatch($podcast);

// Worker
php artisan queue:work redis
```

**3. Broadcasting (WebSockets):**
```php
// config/broadcasting.php
'redis' => [
    'driver' => 'redis',
    'connection' => 'default',
],

// Event
broadcast(new OrderShipped($order));

// Redis Pub/Sub под капотом
```

**4. Session:**
```php
// config/session.php
'driver' => env('SESSION_DRIVER', 'redis'),
```

**5. Rate Limiting:**
```php
use Illuminate\Support\Facades\RateLimiter;

// Под капотом Cache: increment + TTL (не Sorted Set)
RateLimiter::attempt(
    'send-message:'.$user->id,
    $perMinute = 5,
    function() {
        // Send message...
    }
);
```

---

## 📄 MongoDB - Document Database

### Что такое MongoDB?

**MongoDB** - документо-ориентированная NoSQL БД:
- Документы в формате BSON (Binary JSON)
- Коллекции вместо таблиц (collection ≈ table)
- Гибкая схема (разные документы могут иметь разные поля)
- Горизонтальное масштабирование (sharding)
- Репликация (replica sets)

### Структура данных

**Документ (Document):**
```json
{
  "_id": ObjectId("507f1f77bcf86cd799439011"),
  "name": "John Doe",
  "email": "john@example.com",
  "age": 30,
  "address": {
    "street": "123 Main St",
    "city": "New York",
    "country": "USA"
  },
  "tags": ["php", "mongodb", "nosql"],
  "created_at": ISODate("2026-01-28T10:30:00Z")
}
```

**_id** - уникальный идентификатор (автоматически генерируется, 12 байт):
- 4 байта - timestamp
- 5 байт - random value
- 3 байта - counter

### CRUD операции

**Create (Insert):**
```javascript
// insertOne
db.users.insertOne({
  name: "John Doe",
  email: "john@example.com",
  age: 30
})

// insertMany
db.users.insertMany([
  { name: "Alice", email: "alice@example.com" },
  { name: "Bob", email: "bob@example.com" }
])
```

**Read (Find):**
```javascript
// findOne - первый документ
db.users.findOne({ email: "john@example.com" })

// find - все документы (курсор)
db.users.find({ age: { $gte: 25 } })  // age >= 25

// Проекция (выбрать только нужные поля)
db.users.find(
  { age: { $gte: 25 } },
  { name: 1, email: 1, _id: 0 }  // 1 = включить, 0 = исключить
)

// Вложенные документы
db.users.find({ "address.city": "New York" })

// Массивы
db.users.find({ tags: "php" })  // содержит "php"
db.users.find({ tags: { $all: ["php", "mongodb"] } })  // содержит все

// Операторы сравнения
$eq, $ne, $gt, $gte, $lt, $lte, $in, $nin

// Логические операторы
db.users.find({
  $and: [
    { age: { $gte: 25 } },
    { age: { $lte: 40 } }
  ]
})

db.users.find({
  $or: [
    { city: "New York" },
    { city: "Los Angeles" }
  ]
})

// Сортировка, лимит, skip
db.users.find().sort({ age: -1 }).limit(10).skip(20)
// -1 = DESC, 1 = ASC
```

**Update:**
```javascript
// updateOne - обновить первый документ
db.users.updateOne(
  { email: "john@example.com" },  // filter
  { $set: { age: 31 } }            // update
)

// updateMany - обновить все подходящие
db.users.updateMany(
  { age: { $lt: 18 } },
  { $set: { status: "minor" } }
)

// Операторы обновления:
$set      // установить значение
$unset    // удалить поле
$inc      // инкремент
$push     // добавить в массив
$pull     // удалить из массива
$addToSet // добавить в массив если нет (уникальность)

// Примеры:
db.users.updateOne(
  { _id: ObjectId("...") },
  {
    $inc: { age: 1 },                   // age + 1
    $push: { tags: "new-tag" },          // добавить в массив
    $unset: { old_field: "" }            // удалить поле
  }
)
// $push и $addToSet на одно поле в одном update нельзя (conflict) - отдельный запрос:
db.users.updateOne(
  { _id: ObjectId("...") },
  { $addToSet: { tags: "unique-tag" } }  // добавить только если нет
)

// replaceOne - заменить весь документ
db.users.replaceOne(
  { _id: ObjectId("...") },
  { name: "New Name", email: "new@example.com" }
)

// upsert - insert если не найден
db.users.updateOne(
  { email: "new@example.com" },
  { $set: { name: "New User" } },
  { upsert: true }
)
```

**Delete:**
```javascript
// deleteOne
db.users.deleteOne({ email: "john@example.com" })

// deleteMany
db.users.deleteMany({ age: { $lt: 18 } })

// drop collection
db.users.drop()
```

### Индексы

**Создание индексов:**
```javascript
// Single field index
db.users.createIndex({ email: 1 })  // 1 = ASC, -1 = DESC

// Compound index (порядок важен!)
db.users.createIndex({ city: 1, age: -1 })

// Unique index
db.users.createIndex({ email: 1 }, { unique: true })

// Text index (full-text search)
db.posts.createIndex({ title: "text", content: "text" })

// Поиск с text index
db.posts.find({ $text: { $search: "mongodb tutorial" } })

// Посмотреть индексы
db.users.getIndexes()

// Удалить индекс
db.users.dropIndex("email_1")
```

### Aggregation Pipeline

**Мощный инструмент для аналитики:**
```javascript
// Аналог SQL: SELECT city, AVG(age) FROM users GROUP BY city
db.users.aggregate([
  { $group: {
      _id: "$city",                // GROUP BY city
      avgAge: { $avg: "$age" },    // AVG(age)
      count: { $sum: 1 }           // COUNT(*)
    }
  },
  { $sort: { avgAge: -1 } },       // ORDER BY avgAge DESC
  { $limit: 10 }                   // LIMIT 10
])

// Стадии (stages):
$match     // фильтрация (WHERE)
$group     // группировка (GROUP BY)
$sort      // сортировка (ORDER BY)
$limit     // лимит (LIMIT)
$skip      // пропуск (OFFSET)
$project   // проекция (SELECT)
$lookup    // JOIN
$unwind    // развернуть массив

// Пример с $lookup (JOIN):
db.orders.aggregate([
  {
    $lookup: {
      from: "users",           // JOIN с users
      localField: "user_id",   // orders.user_id
      foreignField: "_id",     // users._id
      as: "user"               // алиас
    }
  },
  { $unwind: "$user" },        // развернуть массив user
  {
    $project: {
      order_id: "$_id",
      user_name: "$user.name",
      total: 1
    }
  }
])

// Пример сложной агрегации:
db.orders.aggregate([
  // WHERE created_at >= '2026-01-01'
  { $match: {
      created_at: { $gte: ISODate("2026-01-01") }
    }
  },
  // JOIN users
  { $lookup: {
      from: "users",
      localField: "user_id",
      foreignField: "_id",
      as: "user"
    }
  },
  { $unwind: "$user" },
  // GROUP BY user.city
  { $group: {
      _id: "$user.city",
      total_revenue: { $sum: "$total" },
      order_count: { $sum: 1 },
      avg_order: { $avg: "$total" }
    }
  },
  // HAVING total_revenue > 10000
  { $match: {
      total_revenue: { $gt: 10000 }
    }
  },
  // ORDER BY total_revenue DESC
  { $sort: { total_revenue: -1 } },
  // LIMIT 10
  { $limit: 10 }
])
```

### Репликация (Replica Set)

**Replica Set - группа MongoDB серверов:**
- **Primary** - принимает записи, синхронизирует на secondaries
- **Secondary** - реплики, могут отдавать чтение (опционально)
- **Arbiter** - участвует в голосовании (не хранит данные)

**Автоматический failover:**
- При падении Primary, secondaries голосуют за новый Primary
- Клиенты автоматически переключаются на новый Primary

```javascript
// Инициализация Replica Set
rs.initiate({
  _id: "myReplicaSet",
  members: [
    { _id: 0, host: "mongo1:27017" },
    { _id: 1, host: "mongo2:27017" },
    { _id: 2, host: "mongo3:27017" }
  ]
})

// Статус
rs.status()

// Добавить secondary
rs.add("mongo4:27017")
```

### Sharding - горизонтальное масштабирование

**Sharding** - разделение данных между серверами:
- **Shard** - часть данных (каждый shard - replica set)
- **Config servers** - метаданные о шардах
- **Mongos** - роутер запросов к нужным шардам

**Shard key** - поле для распределения данных:
```javascript
// Включить sharding для БД
sh.enableSharding("mydb")

// Выбрать shard key и создать sharded collection
sh.shardCollection("mydb.users", { "city": 1 })
// Документы с одинаковым city попадут на один shard
// (антипример: низкая кардинальность -> "горячие" шарды)

// Или hash-based sharding (равномерное распределение)
sh.shardCollection("mydb.users", { "_id": "hashed" })
```

**Выбор shard key критичен:**
- ✅ Равномерное распределение данных
- ✅ Запросы содержат shard key (иначе broadcast на все шарды)
- ⚠️ Менять shard key тяжело: с 4.4 можно дополнить (`refineCollectionShardKey`), с 5.0 есть `reshardCollection` (дорогая операция)

### MongoDB в PHP/Laravel

**PHP драйвер (mongodb extension):**
```php
// Установка: pecl install mongodb
// composer require mongodb/mongodb

$client = new MongoDB\Client("mongodb://localhost:27017");
$collection = $client->mydb->users;

// Insert
$collection->insertOne([
    'name' => 'John Doe',
    'email' => 'john@example.com',
    'age' => 30
]);

// Find
$users = $collection->find(['age' => ['$gte' => 25]]);
foreach ($users as $user) {
    echo $user['name'];
}

// Update
$collection->updateOne(
    ['email' => 'john@example.com'],
    ['$set' => ['age' => 31]]
);
```

**Laravel MongoDB (mongodb/laravel-mongodb; бывший jenssegers/mongodb, заброшен):**
```php
// composer require mongodb/laravel-mongodb

// config/database.php
'mongodb' => [
    'driver' => 'mongodb',
    'host' => env('DB_HOST', '127.0.0.1'),
    'port' => env('DB_PORT', 27017),
    'database' => env('DB_DATABASE', 'homestead'),
    'username' => env('DB_USERNAME', 'homestead'),
    'password' => env('DB_PASSWORD', 'secret'),
],

// Model
use MongoDB\Laravel\Eloquent\Model;

class User extends Model
{
    protected $connection = 'mongodb';
    protected $collection = 'users';
}

// Использование
$user = User::where('age', '>', 25)->first();
$user->age = 31;
$user->save();
```

### Когда использовать MongoDB?

**✅ Хорошо подходит:**
- Гибкая схема (документы с разными полями)
- Иерархические данные (вложенные документы)
- Каталоги продуктов (атрибуты отличаются)
- Логи, события (write-heavy)
- Контент-управление (CMS)
- Профили пользователей (разные поля)

**❌ НЕ подходит:**
- Транзакции критичны (banking) - используй SQL
- Сложные JOIN запросы - MongoDB плохо с ними
- Много relations - SQL лучше
- Требуется строгая схема - SQL

---

## 📊 ClickHouse - Columnar Database for Analytics

### Что такое ClickHouse?

**ClickHouse** - колоночная СУБД для OLAP (Online Analytical Processing):
- Разработана Yandex для аналитики (Яндекс.Метрика)
- Колоночное хранение (column-oriented)
- Скорость: миллиарды строк в секунду
- Сжатие данных (10x-100x)
- SQL-like синтаксис

### Row-oriented vs Column-oriented

**Row-oriented (PostgreSQL, MySQL):**
```
Row 1: [id=1, name="John", age=30, city="NY"]
Row 2: [id=2, name="Alice", age=25, city="LA"]
Row 3: [id=3, name="Bob", age=35, city="NY"]
```
- Читает всю строку (даже если нужна 1 колонка)
- ✅ Хорош для OLTP (SELECT * WHERE id = 1)
- ❌ Плох для аналитики (COUNT, SUM, AVG по миллионам строк)

**Column-oriented (ClickHouse):**
```
id:   [1, 2, 3]
name: ["John", "Alice", "Bob"]
age:  [30, 25, 35]
city: ["NY", "LA", "NY"]
```
- Читает только нужные колонки
- Лучше сжатие (одинаковый тип данных в колонке)
- ✅ Отлично для аналитики (SELECT AVG(age) FROM users)
- ❌ Медленный UPDATE/DELETE (перезапись колонок)

### Создание таблицы и вставка

```sql
-- Создание таблицы
CREATE TABLE events (
    date Date,
    user_id UInt32,
    event_type String,
    page_url String,
    created_at DateTime
) ENGINE = MergeTree()
PARTITION BY toYYYYMM(date)
ORDER BY (date, user_id);

-- Вставка (batch insert рекомендуется)
INSERT INTO events VALUES
    ('2026-01-28', 123, 'page_view', '/home', '2026-01-28 10:00:00'),
    ('2026-01-28', 456, 'click', '/products', '2026-01-28 10:05:00');

-- Batch insert из файла (CSV)
cat data.csv | clickhouse-client --query="INSERT INTO events FORMAT CSV"
```

### MergeTree Engine

**MergeTree** - основной engine для аналитики:
- Данные сортируются по `ORDER BY`
- Партиционирование по `PARTITION BY` (опционально)
- Индекс разреженный (sparse index)
- Background слияние частей (merge)

```sql
CREATE TABLE page_views (
    date Date,
    user_id UInt64,
    page String,
    duration UInt32,
    created_at DateTime
) ENGINE = MergeTree()
PARTITION BY toYYYYMM(date)  -- партиции по месяцам
ORDER BY (date, user_id)     -- сортировка + primary key
TTL date + INTERVAL 90 DAY;  -- удалить данные старше 90 дней
```

**ORDER BY = Primary Key** (в MergeTree):
- Одна запись индекса на каждые 8192 строк (granule) - значение ORDER BY первой строки гранулы
- Sparse index (разреженный) - не на каждую строку, а на блоки
- Быстрый поиск по ORDER BY колонкам

### Запросы для аналитики

**Агрегация (скорость!):**
```sql
-- COUNT, AVG, SUM - миллионы строк в секунду
SELECT 
    toDate(created_at) as date,
    COUNT(*) as page_views,
    uniq(user_id) as unique_users  -- приблизительный COUNT DISTINCT (adaptive sampling; uniqHLL12 - HyperLogLog, uniqExact - точный)
FROM events
WHERE created_at >= '2026-01-01'
GROUP BY date
ORDER BY date;

-- Процентили
SELECT 
    quantile(0.5)(duration) as median,     -- медиана
    quantile(0.95)(duration) as p95,       -- 95-й процентиль
    quantile(0.99)(duration) as p99
FROM page_views;

-- Window functions (с ClickHouse 21.1+)
SELECT 
    user_id,
    event_type,
    created_at,
    ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY created_at) as row_num
FROM events;
```

**JOIN (ограничен!):**
```sql
-- LEFT JOIN (правая таблица должна быть маленькой)
SELECT 
    e.user_id,
    u.name,
    COUNT(*) as event_count
FROM events e
LEFT JOIN users u ON e.user_id = u.id
WHERE e.date >= '2026-01-01'
GROUP BY e.user_id, u.name;

-- ClickHouse не для JOIN! Лучше денормализовать данные
```

**Временные ряды:**
```sql
-- Группировка по временным интервалам
SELECT 
    toStartOfHour(created_at) as hour,
    COUNT(*) as events_per_hour
FROM events
WHERE created_at >= now() - INTERVAL 24 HOUR
GROUP BY hour
ORDER BY hour;

-- toStartOfMinute, toStartOfHour, toStartOfDay, toStartOfWeek, toStartOfMonth
```

### Materialized Views

**Пре-агрегация для скорости:**
```sql
-- Создать таблицу для пре-агрегации
CREATE TABLE daily_stats (
    date Date,
    page String,
    page_views SimpleAggregateFunction(sum, UInt64),  -- иначе при merge не суммируется
    unique_users AggregateFunction(uniq, UInt32)      -- тип совпадает с events.user_id
) ENGINE = AggregatingMergeTree()
ORDER BY (date, page);

-- Materialized View (автоматически заполняется)
CREATE MATERIALIZED VIEW daily_stats_mv TO daily_stats AS
SELECT 
    toDate(created_at) as date,
    page,
    count() as page_views,
    uniqState(user_id) as unique_users  -- AggregateFunction state
FROM events
GROUP BY date, page;

-- Запрос к пре-агрегированным данным (мгновенно)
SELECT 
    page,
    SUM(page_views) as total_views,
    uniqMerge(unique_users) as total_unique_users
FROM daily_stats
WHERE date >= '2026-01-01'
GROUP BY page;
```

### Replica и Sharding

**ReplicatedMergeTree - репликация:**
```sql
-- На каждой реплике
CREATE TABLE events_replica (
    date Date,
    user_id UInt32,
    event_type String
) ENGINE = ReplicatedMergeTree('/clickhouse/tables/{shard}/events', '{replica}')
PARTITION BY toYYYYMM(date)
ORDER BY (date, user_id);

-- Автоматическая синхронизация между репликами через ZooKeeper
```

**Distributed - sharding:**
```sql
-- Локальная таблица на каждом сервере
CREATE TABLE events_local (...) ENGINE = MergeTree() ...;

-- Distributed таблица (роутер)
CREATE TABLE events_all AS events_local
ENGINE = Distributed(cluster_name, database, events_local, rand());

-- Запросы к events_all распределяются между шардами
SELECT COUNT(*) FROM events_all;
-- ClickHouse параллельно запросит все шарды и объединит результат
```

### Когда использовать ClickHouse?

**✅ Отлично подходит:**
- Аналитика (dashboards, metrics, KPIs)
- Логи (application logs, web server logs)
- Time-series данные (метрики, события)
- Большие объемы данных (миллиарды строк)
- Append-only workload (вставки >> обновления)

**❌ НЕ подходит:**
- OLTP (частые UPDATE/DELETE)
- Транзакции
- Много JOIN'ов
- Маленькие объемы данных (< миллионы строк)

---

## 🔍 Elasticsearch - Search and Analytics Engine

### Что такое Elasticsearch?

**Elasticsearch** - распределенный поисковый движок:
- Полнотекстовый поиск (full-text search)
- Построен на Apache Lucene
- RESTful API (JSON)
- Горизонтальное масштабирование (sharding)
- Near real-time поиск

### Основные концепции

**Index** (аналог БД/таблицы) - коллекция документов:
```json
POST /users/_doc/1
{
  "name": "John Doe",
  "email": "john@example.com",
  "age": 30,
  "skills": ["php", "elasticsearch", "laravel"]
}
```

**Document** - JSON объект (аналог строки):
- `_id` - уникальный ID
- `_source` - сами данные
- `_index` - имя индекса

**Mapping** (аналог схемы) - типы полей:
```json
PUT /users
{
  "mappings": {
    "properties": {
      "name": { "type": "text" },        // полнотекстовый поиск
      "email": { "type": "keyword" },    // exact match, aggregations (подполя .keyword нет!)
      "age": { "type": "integer" },
      "skills": { "type": "text" }
    }
  }
}
```

**Типы полей:**
- **text** - полнотекстовый поиск (анализируется, токенизируется)
- **keyword** - точное совпадение, сортировка, агрегация (не анализируется)
- **integer**, **long**, **float**, **double** - числа
- **date** - даты
- **boolean** - true/false
- **geo_point**, **geo_shape** - геоданные

**Mapping - это не "какие поля разрешены", а "как индексировать поле":**
- Без явного mapping ES угадывает типы по первому документу (dynamic mapping): строка → `text` + подполе `.keyword`
- Лишние поля по умолчанию добавляются сами; запретить - `"dynamic": "strict"`
- Тип/анализатор существующего поля **изменить нельзя** → новый индекс + `_reindex`. Добавлять новые поля можно

### Создание индекса

`POST /users/_doc/1` создаст индекс автоматически, но с дефолтными настройками и угаданными типами (`"age": "30"` строкой → `text`). В проекте индекс создают **явно** (шарды, анализаторы задаются только при создании) или через **index template** для автосоздаваемых индексов (`logs-*`).

**Анализатор** - цепочка обработки текста перед индексацией:
```
"Разработчики PHP!" → char_filter (чистка) → tokenizer (слова) → filters
  lowercase → stop (убрать "и", "в") → synonyms → stemmer → ["разработчик", "php"]
```
Стеммер даёт морфологию: "разработчик" найдёт "разработчиков".

```json
PUT /vacancies
{
  "settings": {
    "number_of_shards": 1,
    "analysis": {
      "filter": {
        "ru_stop":     { "type": "stop", "stopwords": "_russian_" },
        "ru_stemmer":  { "type": "stemmer", "language": "russian" },
        "my_synonyms": { "type": "synonym", "synonyms": ["php, пхп", "js, javascript"] }
      },
      "analyzer": {
        "ru_text": {
          "tokenizer": "standard",
          "filter": ["lowercase", "ru_stop", "my_synonyms", "ru_stemmer"]
        }
      }
    }
  },
  "mappings": {
    "properties": {
      "title":  {
        "type": "text", "analyzer": "ru_text",
        "fields": { "raw": { "type": "keyword" } }   // title.raw - точный поиск/сортировка
      },
      "city":   { "type": "keyword" },
      "salary": { "type": "integer" }
    }
  }
}
```
Попроще - встроенный `"analyzer": "russian"`.

- **Опечатки** - не в индексе, а в запросе: `"match": { "title": { "query": "разрабочик", "fuzziness": "AUTO" } }`
- **Автодополнение** - в индексе: тип `search_as_you_type` или анализатор `edge_ngram` (префиксы "ра", "раз", "разр"...)

### Индексы внутри индекса

`CREATE INDEX` нет - **индексируется всё автоматически**, структура зависит от типа:

| Тип              | Структура                                       | Для чего                            |
| ---------------- | ----------------------------------------------- | ----------------------------------- |
| `text`           | Инвертированный индекс (слово → документы)      | Полнотекстовый поиск                |
| `keyword`        | Инвертированный индекс + doc_values (колоночно) | Точный поиск, сортировка, агрегации |
| числа, даты, geo | BKD-дерево                                      | Диапазоны, гео                      |

Логика **обратная SQL** - не добавляешь индексы, а отключаешь лишнее (экономия диска, быстрее запись):
```json
"description": { "type": "text", "index": false },       // хранится, но не ищется
"tags":        { "type": "keyword", "doc_values": false }, // без сортировки/агрегаций
"raw_payload": { "type": "object", "enabled": false }     // просто хранить, не разбирать
```
`index.sort.field` - физический порядок документов на диске (отдалённый аналог кластерного индекса).

### CRUD операции

**Create/Update (indexing):**
```json
// Создать с автогенерацией ID
POST /users/_doc
{
  "name": "Alice",
  "email": "alice@example.com"
}

// Создать с конкретным ID
PUT /users/_doc/2
{
  "name": "Bob",
  "email": "bob@example.com"
}

// Partial update
POST /users/_update/2
{
  "doc": {
    "age": 25
  }
}

// Upsert
POST /users/_update/3
{
  "doc": { "name": "Charlie" },
  "doc_as_upsert": true
}
```

**Read (search):**
```json
// Get by ID
GET /users/_doc/1

// Match all
GET /users/_search
{
  "query": {
    "match_all": {}
  }
}

// Full-text search (по полю text)
GET /users/_search
{
  "query": {
    "match": {
      "name": "john doe"  // токенизируется, ищет "john" OR "doe"
    }
  }
}

// Match phrase (фраза целиком)
GET /users/_search
{
  "query": {
    "match_phrase": {
      "name": "john doe"  // "john" AND "doe" рядом
    }
  }
}

// Term query (exact match для keyword)
GET /users/_search
{
  "query": {
    "term": {
      "email": "john@example.com"  // точное совпадение (email - keyword; для text-поля было бы email.keyword)
    }
  }
}

// Range query
GET /users/_search
{
  "query": {
    "range": {
      "age": {
        "gte": 25,
        "lte": 40
      }
    }
  }
}

// Bool query (AND, OR, NOT)
GET /users/_search
{
  "query": {
    "bool": {
      "must": [                          // AND
        { "match": { "skills": "php" } }
      ],
      "should": [                        // OR (boost score)
        { "match": { "skills": "laravel" } }
      ],
      "must_not": [                      // NOT
        { "term": { "status": "banned" } }
      ],
      "filter": [                        // AND (no scoring)
        { "range": { "age": { "gte": 18 } } }
      ]
    }
  }
}
```

**Delete:**
```json
// Delete by ID
DELETE /users/_doc/1

// Delete by query
POST /users/_delete_by_query
{
  "query": {
    "match": {
      "status": "inactive"
    }
  }
}
```

### Aggregations (аналитика)

```json
// Группировка (аналог GROUP BY)
GET /users/_search
{
  "size": 0,  // не возвращать документы, только агрегации
  "aggs": {
    "by_age": {
      "terms": {
        "field": "age"
      }
    }
  }
}

// Range aggregation
GET /users/_search
{
  "size": 0,
  "aggs": {
    "age_ranges": {
      "range": {
        "field": "age",
        "ranges": [
          { "from": 0, "to": 18 },
          { "from": 18, "to": 30 },
          { "from": 30 }
        ]
      }
    }
  }
}

// Stats (COUNT, MIN, MAX, AVG, SUM)
GET /users/_search
{
  "size": 0,
  "aggs": {
    "age_stats": {
      "stats": {
        "field": "age"
      }
    }
  }
}

// Nested aggregations
GET /orders/_search
{
  "size": 0,
  "aggs": {
    "by_status": {
      "terms": { "field": "status" },
      "aggs": {
        "total_amount": {
          "sum": { "field": "amount" }
        }
      }
    }
  }
}
```

### Relevance и Scoring

**BM25 (с Elasticsearch 5.0; ранее TF-IDF):**
- Документы ранжируются по релевантности
- Основан на идее TF-IDF: частота термина в документе (TF) vs редкость термина в индексе (IDF)
- BM25 добавляет насыщение TF и нормализацию по длине поля

```json
// Поиск с объяснением score
GET /posts/_search
{
  "query": {
    "match": {
      "title": "elasticsearch tutorial"
    }
  },
  "explain": true  // объяснение score
}

// Boosting (увеличить вес поля)
GET /posts/_search
{
  "query": {
    "multi_match": {
      "query": "elasticsearch",
      "fields": ["title^3", "content"]  // title в 3 раза важнее
    }
  }
}

// Function score (custom scoring)
GET /posts/_search
{
  "query": {
    "function_score": {
      "query": { "match": { "title": "tutorial" } },
      "functions": [
        {
          "field_value_factor": {
            "field": "views",  // учесть количество просмотров
            "modifier": "log1p"
          }
        }
      ]
    }
  }
}
```

### Elasticsearch в Laravel

**Laravel Scout (laravel/scout):**
```php
// composer require laravel/scout
// composer require matchish/laravel-scout-elasticsearch

// config/scout.php
'driver' => env('SCOUT_DRIVER', 'elasticsearch'),

// Model
use Laravel\Scout\Searchable;

class Post extends Model
{
    use Searchable;

    public function toSearchableArray()
    {
        return [
            'title' => $this->title,
            'content' => $this->content,
            'author' => $this->author->name,
        ];
    }
}

// Индексация
$post = Post::create([...]);  // автоматически индексируется

// Или вручную
$post->searchable();

// Поиск
$posts = Post::search('laravel tutorial')->get();

// С фильтрами
$posts = Post::search('tutorial')
    ->where('status', 'published')
    ->orderBy('created_at', 'desc')
    ->paginate(10);

// Переиндексация всех
php artisan scout:import "App\Models\Post"
```

### Когда использовать Elasticsearch?

**✅ Отлично подходит:**
- Полнотекстовый поиск (сайты, e-commerce, документация)
- Автодополнение (autocomplete, suggestions)
- Поиск с фильтрами (фасетный поиск)
- Логи и мониторинг (ELK stack: Elasticsearch + Logstash + Kibana)
- Аналитика в реальном времени

**❌ НЕ подходит:**
- OLTP (транзакции)
- Строгая консистентность (eventual consistency)
- Первичное хранилище данных (не замена БД)

---

## 📊 Сравнение NoSQL решений

| Аспект               | Redis                                       | MongoDB                                        | ClickHouse                                            | Elasticsearch                   |
| -------------------- | ------------------------------------------- | ---------------------------------------------- | ----------------------------------------------------- | ------------------------------- |
| **Тип**              | Key-Value, In-Memory                        | Document                                       | Columnar                                              | Document, Search                |
| **Схема**            | Schema-less                                 | Flexible schema                                | Fixed schema                                          | Dynamic mapping                 |
| **Persistence**      | RDB/AOF                                     | WiredTiger + journal                           | MergeTree parts                                       | Lucene segments                 |
| **Скорость записи**  | ⚡⚡⚡ Очень высокая                        | ⚡⚡ Высокая                                   | ⚡⚡ Высокая (batch)                                  | ⚡ Средняя                      |
| **Скорость чтения**  | ⚡⚡⚡ Мгновенная                           | ⚡⚡ Быстрая                                   | ⚡⚡⚡ Аналитика быстрая                              | ⚡⚡ Поиск быстрый              |
| **Масштабирование**  | Replication + Sentinel, Cluster             | Replica sets, sharding                         | Shards + ReplicatedMergeTree                          | Shards + replicas               |
| **Транзакции**       | ⚠️ MULTI/EXEC, Lua (изоляция без rollback)  | ✅ ACID (4.0 replica set, 4.2 шарды)           | ❌ Нет (атомарен INSERT одного блока)                 | ❌ Нет                          |
| **JOIN**             | ❌ Нет                                      | ⚠️ Ограничен ($lookup)                         | ✅ SQL JOIN (правая таблица в памяти)                 | ⚠️ Только nested / parent-child |
| **Full-text search** | ⚠️ Модуль RediSearch (Redis Stack / 8+)     | ⚠️ Базовый $text (полный - Atlas Search)       | ⚠️ Простой (LIKE, токен-индексы)                      | ✅ Лучший                       |
| **Use cases**        | Cache, sessions, queues, pub/sub            | Flexible data, catalogs, CMS                   | Analytics, logs, metrics                              | Search, autocomplete, logs      |
| **RAM usage**        | ⚡⚡⚡ Все в RAM                            | ⚡⚡ Кеш WiredTiger (индексы + горячие данные) | ⚡ Данные на диске, но тяжёлые GROUP BY/JOIN едят RAM | ⚡⚡ JVM heap + page cache ОС   |
| **Размер данных**    | ⚠️ Ограничен RAM (Cluster - суммой RAM нод) | ✅ Терабайты                                   | ✅ Петабайты                                          | ✅ Терабайты                    |

---

## 🎓 Для собеседования: ключевые точки

### Redis:
- ✅ 5 структур данных (String, Hash, List, Set, Sorted Set) + применение
- ✅ Persistence: RDB vs AOF vs оба
- ✅ Использование в Laravel (cache, queues, sessions, broadcasting)
- ✅ Single-threaded + атомарность
- ✅ Redis Cluster для sharding

### MongoDB:
- ✅ Document model, BSON, гибкая схема
- ✅ CRUD операции, operators ($set, $inc, $push)
- ✅ Aggregation Pipeline (stages, $lookup)
- ✅ Replica Set (Primary/Secondary/Arbiter)
- ✅ Sharding (shard key критичен)
- ✅ Когда использовать vs SQL

### ClickHouse:
- ✅ Columnar storage vs row-oriented
- ✅ MergeTree engine, ORDER BY = primary key
- ✅ Materialized Views для пре-агрегации
- ✅ OLAP vs OLTP
- ✅ Append-only, плохо для UPDATE/DELETE

### Elasticsearch:
- ✅ Full-text search, inverted index (термин -> список документов; основа Lucene)
- ✅ Mapping (text vs keyword)
- ✅ Query DSL (match, term, bool)
- ✅ Relevance scoring (BM25, развитие TF-IDF)
- ✅ Aggregations для аналитики
- ✅ Laravel Scout интеграция
