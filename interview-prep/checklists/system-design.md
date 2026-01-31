# System Design для собеседований

## Framework для System Design интервью

### Этапы (REASCH)
1. **Requirements** - требования
2. **Estimations** - оценки нагрузки
3. **API Design** - дизайн API
4. **Schema Design** - схема БД
5. **Components** - компоненты системы
6. **High-level Design** - архитектура

## 1. Requirements Clarification

### Функциональные требования
- Что система должна делать?
- Основные use cases?
- Какие операции: read-heavy или write-heavy?
- Кто пользователи?

### Нефункциональные требования
- [ ] **Scalability** - миллионы пользователей?
- [ ] **Availability** - SLA? 99.9%, 99.99%?
- [ ] **Consistency** - строгая или eventual?
- [ ] **Latency** - требования к скорости?
- [ ] **Durability** - не терять данные?

### Вопросы к интервьюеру
```
- Сколько пользователей? Daily Active Users?
- Объём данных? Сколько записей/запросов в секунду?
- Read/Write ratio?
- Нужны real-time updates?
- География пользователей?
- Какие самые важные features?
```

## 2. Capacity Estimations

### Traffic Estimates
```
Daily Active Users (DAU): 10M
Requests per user per day: 20
Total requests/day: 200M
Requests per second (QPS): 200M / 86400 ≈ 2,300 RPS
Peak QPS (3x): ≈ 7,000 RPS
```

### Storage Estimates
```
Average post size: 1KB
Posts per day: 5M
Storage per day: 5M * 1KB = 5GB/day
Storage per year: 5GB * 365 ≈ 1.8TB/year
With replication (3x): 5.4TB/year
```

### Bandwidth Estimates
```
Incoming data: 5GB/day ≈ 58KB/s
Outgoing data (read): 58KB/s * 100 (read/write ratio) ≈ 5.8MB/s
```

### Memory Estimates (Cache)
```
20% of requests hit 80% of data (80-20 rule)
Cache 20% of daily requests
2,300 RPS * 86400 * 0.2 * 1KB ≈ 40GB cache
```

## 3. API Design

### REST API примеры

#### User Service
```http
POST   /api/v1/users               # Register
POST   /api/v1/auth/login          # Login
GET    /api/v1/users/{id}          # Get user
PUT    /api/v1/users/{id}          # Update
DELETE /api/v1/users/{id}          # Delete
```

#### Post Service (Twitter-like)
```http
POST   /api/v1/posts               # Create post
GET    /api/v1/posts/{id}          # Get post
DELETE /api/v1/posts/{id}          # Delete
GET    /api/v1/feed                # User feed
GET    /api/v1/users/{id}/posts    # User posts
```

#### Request/Response examples
```json
POST /api/v1/posts
{
  "content": "Hello world",
  "media_urls": ["https://..."]
}

Response 201:
{
  "id": "123",
  "user_id": "456",
  "content": "Hello world",
  "created_at": "2024-01-01T12:00:00Z",
  "likes_count": 0
}
```

## 4. Database Schema Design

### SQL Schema (Twitter-like)
```sql
-- Users
CREATE TABLE users (
    id BIGSERIAL PRIMARY KEY,
    username VARCHAR(50) UNIQUE NOT NULL,
    email VARCHAR(255) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    created_at TIMESTAMP DEFAULT NOW(),
    INDEX idx_username (username),
    INDEX idx_email (email)
);

-- Posts
CREATE TABLE posts (
    id BIGSERIAL PRIMARY KEY,
    user_id BIGINT REFERENCES users(id),
    content TEXT NOT NULL,
    created_at TIMESTAMP DEFAULT NOW(),
    INDEX idx_user_created (user_id, created_at DESC)
);

-- Followers
CREATE TABLE followers (
    follower_id BIGINT REFERENCES users(id),
    followee_id BIGINT REFERENCES users(id),
    created_at TIMESTAMP DEFAULT NOW(),
    PRIMARY KEY (follower_id, followee_id),
    INDEX idx_followee (followee_id)
);

-- Likes
CREATE TABLE likes (
    user_id BIGINT REFERENCES users(id),
    post_id BIGINT REFERENCES posts(id),
    created_at TIMESTAMP DEFAULT NOW(),
    PRIMARY KEY (user_id, post_id),
    INDEX idx_post (post_id)
);
```

### NoSQL Schema (MongoDB)
```javascript
// Users collection
{
  "_id": ObjectId,
  "username": "john_doe",
  "email": "john@example.com",
  "profile": {
    "name": "John Doe",
    "bio": "...",
    "avatar_url": "..."
  },
  "stats": {
    "followers_count": 1234,
    "following_count": 567
  }
}

// Posts collection
{
  "_id": ObjectId,
  "user_id": ObjectId,
  "content": "Hello world",
  "media_urls": ["..."],
  "created_at": ISODate,
  "stats": {
    "likes_count": 42,
    "comments_count": 5
  }
}

// Timeline cache (Redis)
user:123:timeline -> [post_id1, post_id2, ...] (ZSET by timestamp)
```

## 5. Component Design

### Core Components

#### Load Balancer
- Nginx / HAProxy
- Health checks
- SSL termination
- Rate limiting

#### Application Servers
- Stateless PHP-FPM workers
- Horizontal scaling
- Auto-scaling groups

#### Cache Layer
- Redis cluster
- Cache-aside pattern
- TTL strategies
- Hot key handling

#### Database
- PostgreSQL / MySQL master-slave
- Read replicas
- Connection pooling
- Sharding if needed

#### Object Storage
- S3 / MinIO для media
- CDN для delivery

#### Message Queue
- RabbitMQ / Kafka
- Async processing
- Email, notifications

#### Search Service
- Elasticsearch
- Full-text search
- Hashtags, mentions

## 6. High-Level Architecture

### Basic Architecture (Twitter-like)
```
                          ┌─────────────┐
                          │   Client    │
                          └──────┬──────┘
                                 │
                          ┌──────▼──────┐
                          │     CDN     │
                          └──────┬──────┘
                                 │
                          ┌──────▼──────────┐
                          │ Load Balancer   │
                          └──────┬──────────┘
                                 │
                 ┌───────────────┼───────────────┐
                 │               │               │
          ┌──────▼─────┐  ┌─────▼──────┐ ┌─────▼──────┐
          │   Web       │  │   Web      │ │   Web      │
          │  Server 1   │  │  Server 2  │ │  Server 3  │
          └──────┬──────┘  └─────┬──────┘ └─────┬──────┘
                 │               │               │
                 └───────────────┼───────────────┘
                                 │
                 ┌───────────────┼───────────────┐
                 │               │               │
          ┌──────▼─────┐  ┌─────▼──────┐ ┌─────▼──────┐
          │   Cache    │  │  Message   │ │  Search    │
          │  (Redis)   │  │   Queue    │ │   (ES)     │
          └────────────┘  └────────────┘ └────────────┘
                                 │
                          ┌──────▼──────────┐
                          │   Database      │
                          │  (Master-Slave) │
                          └─────────────────┘
                                 │
                          ┌──────▼──────┐
                          │   Object    │
                          │  Storage    │
                          └─────────────┘
```

### Feed Generation Strategies

#### Pull Model (на лету)
```php
// Когда пользователь запрашивает feed
function getFeed(int $userId): array
{
    // 1. Получить followed users
    $followedIds = getFollowedUsers($userId);
    
    // 2. Получить их посты
    $posts = Post::whereIn('user_id', $followedIds)
        ->orderBy('created_at', 'DESC')
        ->limit(100)
        ->get();
    
    return $posts;
}

// Плюсы: актуальные данные, нет pre-computation
// Минусы: медленно для пользователей с большим количеством подписок
```

#### Push Model (фоновая генерация)
```php
// При создании поста
function createPost(int $userId, string $content): void
{
    $post = Post::create(['user_id' => $userId, 'content' => $content]);
    
    // Фоновая задача
    PushPostToFollowers::dispatch($post);
}

function pushToFollowers(Post $post): void
{
    $followers = getFollowers($post->user_id);
    
    foreach ($followers as $follower) {
        // Добавить в pre-generated feed (Redis)
        Redis::zadd("user:{$follower}:feed", $post->created_at, $post->id);
    }
}

// Плюсы: быстрое чтение
// Минусы: медленная запись для популярных пользователей
```

#### Hybrid Model
```php
// Для обычных: push model
// Для celebrities (>1M followers): pull model на лету
// Best of both worlds
```

## Common Design Patterns

### Rate Limiting
```
Algorithm: Token Bucket / Sliding Window

Redis implementation:
INCR user:123:requests:minute
EXPIRE user:123:requests:minute 60

if count > limit:
    return 429 Too Many Requests
```

### Pagination

#### Offset-based
```sql
SELECT * FROM posts 
ORDER BY created_at DESC 
LIMIT 20 OFFSET 40;

-- Проблема: медленно на больших offset
```

#### Cursor-based
```sql
SELECT * FROM posts 
WHERE created_at < '2024-01-01 12:00:00'
ORDER BY created_at DESC 
LIMIT 20;

-- Лучше для больших datasets
```

### Cache Strategies

#### Cache-Aside
```php
$data = Cache::get($key);
if (!$data) {
    $data = DB::query(...);
    Cache::put($key, $data, 3600);
}
return $data;
```

#### Write-Through
```php
DB::insert($data);
Cache::put($key, $data, 3600);
```

#### Write-Behind
```php
Cache::put($key, $data);
Queue::push(new WriteToDatabase($data)); // async
```

## Scalability Bottlenecks & Solutions

### Database
**Problem:** Database becomes bottleneck
**Solutions:**
- Read replicas
- Sharding (user_id based)
- Caching (Redis)
- Denormalization
- CQRS pattern

### Single Point of Failure
**Problem:** Master DB goes down
**Solutions:**
- Automatic failover
- Multi-AZ deployment
- Health checks

### Global Users
**Problem:** High latency for far users
**Solutions:**
- Multiple data centers
- GeoDNS routing
- CDN для static content
- Edge computing

## Trade-offs Discussion

### Consistency vs Availability (CAP)
- **Strongly consistent:** banking, inventory
- **Eventually consistent:** social media feeds, analytics

### Latency vs Accuracy
- **Real-time:** notifications, chat
- **Near real-time:** analytics, trending

### Cost vs Performance
- **High performance:** multi-region, много cache
- **Cost-effective:** single region, меньше redundancy

## Примеры систем для практики

### Tier 1 (базовый)
- URL Shortener (bit.ly)
- Pastebin
- Key-Value Store

### Tier 2 (средний)
- Twitter
- Instagram
- Uber
- WhatsApp

### Tier 3 (продвинутый)
- YouTube
- Netflix
- Google Drive
- Distributed Cache (Memcached)

## Советы для интервью

### Do's
✅ Задавай уточняющие вопросы
✅ Начинай с простого, потом усложняй
✅ Обсуждай trade-offs
✅ Используй цифры (estimations)
✅ Рисуй диаграммы
✅ Обсуждай bottlenecks и решения

### Don'ts
❌ Не прыгай сразу в детали
❌ Не делай assumptions без проверки
❌ Не игнорируй нефункциональные требования
❌ Не пытайся сделать идеальное решение сразу
❌ Не молчи - думай вслух
