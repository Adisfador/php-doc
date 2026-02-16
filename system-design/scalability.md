# Scalability - Масштабируемость систем

Паттерны и техники для создания масштабируемых систем.

---

## Load Balancing

**Load Balancer** - распределяет входящий трафик между несколькими серверами.

### Алгоритмы балансировки:

#### Round Robin
Последовательное распределение запросов.
```
Request 1 → Server 1
Request 2 → Server 2
Request 3 → Server 3
Request 4 → Server 1 (цикл)
```
**Когда:** все серверы одинаковой мощности.

#### Weighted Round Robin
С учётом мощности серверов.
```
Server 1 (weight=3) → получит 3 запроса
Server 2 (weight=1) → получит 1 запрос
```

#### Least Connections
Направляет на сервер с наименьшим количеством активных соединений.

**Когда:** разная длительность запросов.

#### IP Hash
```
hash(client_ip) % num_servers
```
**Плюс:** sticky sessions (один клиент → один сервер)  
**Минус:** неравномерное распределение

### L4 vs L7 Load Balancing

#### L4 (Transport Layer)
- Работает на уровне TCP/UDP
- Не смотрит на содержимое
- Быстрее, меньше overhead
- Примеры: HAProxy, AWS NLB

#### L7 (Application Layer)
- Работает на уровне HTTP/HTTPS
- Может анализировать URL, headers, cookies
- Content-based routing
- Примеры: Nginx, AWS ALB, Traefik

**Пример L7 routing:**
```nginx
location /api/ {
    proxy_pass http://api-servers;
}

location /static/ {
    proxy_pass http://cdn-servers;
}
```

### Health Checks

Проверка доступности серверов:
```nginx
upstream backend {
    server 192.168.1.10:80 max_fails=3 fail_timeout=30s;
    server 192.168.1.11:80 max_fails=3 fail_timeout=30s;
}
```

---

## Stateless Architecture

### Stateless vs Stateful

#### Stateful
```
Client → Server 1 (session stored)
Client → Server 2 (НЕТ доступа к session) ❌
```

#### Stateless
```
Client → Server 1 (session в Redis/DB)
Client → Server 2 (читает session из Redis) ✅
```

### Как сделать stateless:

1. **Сессии в внешнем хранилище** (Redis, Memcached)
2. **JWT tokens** вместо server-side sessions
3. **Stateless authentication** (OAuth tokens)

**Laravel пример:**
```php
// config/session.php
'driver' => 'redis', // вместо 'file'

// Теперь любой сервер может читать сессию
```

---

## Auto-Scaling

Автоматическое добавление/удаление серверов при изменении нагрузки.

### Метрики для scaling:

- **CPU utilization** > 70% → добавить сервер
- **Memory utilization** > 80% → добавить сервер
- **Request latency** > 500ms → добавить сервер
- **Queue depth** > 1000 → добавить worker

### Стратегии:

#### Reactive Scaling
Реакция на текущую нагрузку.
```
if avg(cpu) > 75% for 5 minutes:
    add_server()
```

#### Predictive Scaling
Прогнозирование на основе паттернов.
```
if hour == 9am (утренний пик):
    add_servers(count=5)
```

#### Scheduled Scaling
По расписанию.
```
Weekdays 9am-6pm: 10 servers
Weekdays 6pm-9am: 3 servers
Weekends: 2 servers
```

---

## Caching Layers

Многоуровневое кеширование для снижения нагрузки.

### Уровни кеширования:

```
Browser Cache (seconds-minutes)
    ↓
CDN Cache (minutes-hours)
    ↓
Application Cache / Redis (minutes-days)
    ↓
Database Query Cache (queries)
    ↓
Database (persistent storage)
```

### 1. Browser Cache
```http
Cache-Control: public, max-age=3600
ETag: "abc123"
```

### 2. CDN Cache
- CloudFront, Cloudflare, Fastly
- Кеширует static assets (CSS, JS, images)
- Edge locations близко к пользователям

### 3. Application Cache (Redis)
```php
$posts = Cache::remember('posts:latest', 3600, function () {
    return Post::latest()->limit(10)->get();
});
```

### 4. Database Cache
- Query result cache
- OpCache для compiled PHP code

---

## Database Scaling Strategies

### 1. Read Replicas

```
Master (writes) → replicates → Slave 1 (reads)
                             → Slave 2 (reads)
                             → Slave 3 (reads)
```

**Laravel настройка:**
```php
'mysql' => [
    'write' => ['host' => '192.168.1.10'],
    'read' => [
        ['host' => '192.168.1.11'],
        ['host' => '192.168.1.12'],
    ],
],
```

**Когда:** read-heavy приложения (90% read, 10% write).

### 2. Sharding (Horizontal Partitioning)

Разделение данных по разным БД.

**По user_id:**
```
users 1-100000    → shard_1
users 100001-200000 → shard_2
users 200001+       → shard_3
```

**Выбор shard:**
```php
$shardId = $userId % $numShards;
$db = "shard_{$shardId}";
```

### 3. CQRS (Command Query Responsibility Segregation)

Разделение read и write моделей.

```
Commands (writes) → Write DB (normalized)
                       ↓
                  Event stream
                       ↓
Queries (reads) ← Read DB (denormalized, optimized)
```

---

## Rate Limiting

Защита от перегрузки и абуза.

### Алгоритмы:

#### 1. Token Bucket
```
Bucket: вместимость 100 токенов
Скорость пополнения: 10 токенов/секунду

Request → забирает 1 токен
Если токенов нет → 429 Too Many Requests
```

**Пример (Redis):**
```php
$key = "rate_limit:user:{$userId}";
$current = Redis::incr($key);

if ($current === 1) {
    Redis::expire($key, 60); // 60 секунд
}

if ($current > 100) { // 100 requests/minute
    return response('Too many requests', 429);
}
```

#### 2. Leaky Bucket
Запросы обрабатываются с постоянной скоростью.

#### 3. Fixed Window
```
Minute 1: 100 requests allowed
Minute 2: reset counter, 100 requests allowed
```

#### 4. Sliding Window
Более точный, учитывает распределение во времени.

### Уровни rate limiting:

- **Per user:** 100 requests/minute
- **Per IP:** 1000 requests/minute
- **Global:** 100,000 requests/minute

---

## Content Delivery Network (CDN)

**CDN** - распределённая сеть серверов для доставки контента.

### Как работает:

```
User (New York) → Edge Server (New York) → Origin Server (London)
                      ↓ (cache hit)
                  Return cached content
```

### Преимущества:

- **Низкая latency** - контент ближе к пользователю
- **Разгрузка origin** - меньше запросов на основной сервер
- **DDoS защита** - CDN поглощает атаки

### Настройка:

```php
// Laravel Asset URL
'asset_url' => env('ASSET_URL', 'https://cdn.example.com'),
```

```html
<img src="{{ asset('images/logo.png') }}">
<!-- https://cdn.example.com/images/logo.png -->
```

### Cache-Control заголовки:

```nginx
location ~* \.(jpg|jpeg|png|gif|ico|css|js)$ {
    expires 1y;
    add_header Cache-Control "public, immutable";
}
```

---

## Geographic Distribution

### Multi-Region Deployment

Размещение серверов в разных географических регионах.

```
US West (Oregon)    → Users: West Coast USA
US East (Virginia)  → Users: East Coast USA, Europe
Asia (Singapore)    → Users: Asia Pacific
```

### GeoDNS Routing

DNS направляет пользователя на ближайший datacenter.

```
User in Europe → DNS → EU datacenter
User in Asia   → DNS → Asia datacenter
```

### Проблемы:

- **Data consistency** - синхронизация между регионами
- **Latency** - cross-region replication
- **Compliance** - GDPR, data residency laws

---

## Connection Pooling

Переиспользование соединений с БД вместо создания новых.

### Проблема без pooling:
```php
// Каждый request создаёт новое соединение
$pdo = new PDO(...); // expensive operation
```

### С pooling:
```
Application ← Connection Pool (10 connections) ← Database

Request 1 → берёт connection #1 → возвращает
Request 2 → берёт connection #1 (переиспользует)
```

**Laravel автоматически использует persistent connections.**

### Настройка:
```php
// config/database.php
'mysql' => [
    'options' => [
        PDO::ATTR_PERSISTENT => true, // persistent connections
    ],
],
```

---

## Async Processing

Вынос долгих операций в фоновые задачи.

### Synchronous (плохо):
```php
Route::post('/upload', function (Request $request) {
    $file = $request->file('video');
    
    // Блокирует пользователя на минуты
    processVideo($file); // 5 minutes
    
    return response('Done');
});
```

### Asynchronous (хорошо):
```php
Route::post('/upload', function (Request $request) {
    $file = $request->file('video');
    
    // Быстрый ответ
    ProcessVideo::dispatch($file);
    
    return response('Processing started');
});

// Job в очереди
class ProcessVideo implements ShouldQueue
{
    public function handle()
    {
        // Выполняется в фоне
        processVideo($this->file);
    }
}
```

---

## Resources

- [AWS Well-Architected Framework](https://aws.amazon.com/architecture/well-architected/)
- [Google Cloud Architecture](https://cloud.google.com/architecture)
- [High Scalability Blog](http://highscalability.com/)
