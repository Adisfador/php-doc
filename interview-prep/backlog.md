# Темы для дополнения документации

Этот файл для накопления новых тем, вопросов и концепций, которые нужно добавить в основную документацию.

---

## 🗄️ Базы данных

### Темы для добавления:
- [ ] 

### Вопросы для изучения:
- [ ] 

---

## 🐘 PHP

### Темы для добавления:
- [ ] 

### Вопросы для изучения:
- [ ] 

---

## 🔴 Laravel

### Темы для добавления:
- [ ] 

### Вопросы для изучения:
- [ ] 

---

## 🏗️ Архитектура и паттерны

### Темы для добавления:
- [x] **Code Smells (Запахи кода)** - признаки проблемного кода, требующего рефакторинга
  - Long Method
  - Large Class
  - Duplicated Code
  - Feature Envy
  - Data Clumps
  - Primitive Obsession
  - Switch Statements
  - Lazy Class
  - Speculative Generality
  - Temporary Field
  - Message Chains
  - Middle Man
  - Inappropriate Intimacy
  - Alternative Classes with Different Interfaces
  - Incomplete Library Class
  - Data Class
  - Refused Bequest
  - Comments (как запах)

### Вопросы для изучения:
- [ ] 

---

## 🌐 API и интеграции

### Темы для добавления:
- [ ] 

### Вопросы для изучения:
- [ ] 

---

## 📮 Очереди и брокеры

### Темы для добавления:
- [ ] 

### Вопросы для изучения:
- [ ] 

---

## 🌐 Сетевые протоколы

### Темы для добавления:
- [ ] 

### Вопросы для изучения:
- [ ] 

---

## 🐳 DevOps и инфраструктура

### Темы для добавления:
- [ ] 

### Вопросы для изучения:
- [ ] 

---

## 📊 Мониторинг и observability

### Темы для добавления:
- [ ] 

### Вопросы для изучения:
- [ ] 

---

## ✅ Тестирование

### Темы для добавления:
- [ ] 

### Вопросы для изучения:
- [ ] 

---

## 🔧 Real-time коммуникация

### Темы для добавления:
- [ ] 

### Вопросы для изучения:
- [ ] 

---

## 🚀 Roadmap для Meta (FAANG)

### System Design (КРИТИЧНО - 40% собеса)
- [ ] **Load Balancing и масштабирование**
  - Nginx upstream, health checks
  - Cloud load balancers (AWS ALB/NLB, GCP LB)
  - L4 vs L7 балансировка
  - Sticky sessions, session affinity
  - Horizontal vs Vertical scaling
  - Stateless application design
  - Auto-scaling strategies

- [ ] **Caching strategies**
  - Multi-layer caching (browser, CDN, application, database)
  - Redis patterns (cache-aside, read-through, write-through, write-behind)
  - Cache invalidation strategies
  - Cache stampede problem
  - Memcached vs Redis
  - CDN (CloudFront, Cloudflare, Fastly)
  - Edge caching

- [ ] **Database Scaling**
  - Sharding strategies (horizontal partitioning)
  - Replication (master-slave, master-master)
  - Read replicas
  - Partitioning (range, hash, list)
  - Consistent hashing
  - Database federation
  - CQRS (Command Query Responsibility Segregation)

- [ ] **Message Queues и Event-Driven**
  - RabbitMQ advanced patterns (work queues, pub/sub, routing, topics)
  - Apache Kafka (topics, partitions, consumer groups, offset management)
  - Event sourcing
  - Saga pattern для distributed transactions
  - Dead Letter Queue (DLQ)
  - Idempotency patterns
  - Exactly-once delivery

- [ ] **Microservices Design**
  - Service decomposition strategies
  - API Gateway pattern
  - Service mesh (Istio, Linkerd концептуально)
  - Inter-service communication (sync vs async)
  - Circuit breaker pattern
  - Service discovery
  - Distributed tracing
  - Saga vs 2PC

- [ ] **Rate Limiting и защита**
  - Token bucket algorithm
  - Leaky bucket algorithm
  - Fixed window vs Sliding window
  - Distributed rate limiting (Redis)
  - DDoS protection
  - API throttling

- [ ] **Real-world System Design примеры**
  - Design Instagram/Facebook News Feed
  - Design WhatsApp/Messenger
  - Design Twitter timeline
  - Design URL shortener (bit.ly)
  - Design Dropbox/Google Drive
  - Design YouTube/Netflix
  - Design Uber/Lyft
  - Design notification system
  - Design search autocomplete
  - Design web crawler

### Distributed Systems (для Senior)
- [ ] **CAP Theorem глубоко**
  - Consistency, Availability, Partition tolerance
  - CP vs AP systems
  - BASE vs ACID
  - Eventual consistency patterns
  - Strong consistency patterns

- [ ] **Consensus Algorithms (концептуально)**
  - Raft protocol
  - Paxos (basic understanding)
  - Leader election
  - Log replication
  - Split-brain problem

- [ ] **Distributed Transactions**
  - Two-Phase Commit (2PC)
  - Saga pattern
  - Compensation logic
  - Distributed locks (Redis, ZooKeeper)

- [ ] **Time and Ordering**
  - Lamport timestamps
  - Vector clocks
  - Happened-before relation
  - Causality

- [ ] **Data Replication**
  - Single-leader replication
  - Multi-leader replication
  - Leaderless replication
  - Conflict resolution
  - Read-your-writes consistency
  - Monotonic reads

### Алгоритмы (40% собеса Meta)
- [ ] **Blind 75 - ВСЕ задачи обязательно**
  - Arrays & Hashing (9 задач)
  - Two Pointers (3 задачи)
  - Stack (3 задачи)
  - Binary Search (1 задача)
  - Sliding Window (4 задачи)
  - Linked List (6 задач)
  - Trees (11 задач)
  - Tries (1 задача)
  - Heap/Priority Queue (3 задачи)
  - Backtracking (3 задачи)
  - Graphs (6 задач)
  - Dynamic Programming (11 задач)
  - Greedy (2 задачи)
  - Intervals (2 задачи)

- [ ] **NeetCode 150 (расширение Blind 75)**
  - Focus на Medium/Hard задачи
  - Повторение паттернов

- [ ] **Meta Tagged на LeetCode**
  - Отфильтровать по компании Meta
  - Решить топ-50 часто задаваемых
  - Focus на Medium/Hard

- [ ] **Практика на время**
  - 45 минут на задачу Medium
  - 60 минут на задачу Hard
  - Объяснение решения вслух

### Behavioral Interview (20% собеса)
- [ ] **STAR метод подготовка**
  - Situation, Task, Action, Result
  - 10+ историй из опыта
  - Конфликты в команде
  - Технические решения
  - Лидерство и влияние
  - Неудачи и уроки

- [ ] **Meta Leadership Principles**
  - Move Fast
  - Focus on Impact
  - Be Bold
  - Be Open
  - Build Social Value

- [ ] **Типичные вопросы Meta**
  - "Tell me about a time you disagreed with a teammate"
  - "Describe a technical challenge you faced"
  - "Tell me about a project you're most proud of"
  - "How do you handle tight deadlines?"
  - "Describe a time you had to learn something quickly"

### Timeline и метрики готовности
- [ ] **Месяц 1-2: System Design**
  - Прочитать "System Design Interview" (Alex Xu)
  - Прочитать "Designing Data-Intensive Applications" (Martin Kleppmann)
  - 20+ mock system design problems
  - Рисовать диаграммы архитектур

- [ ] **Месяц 3-5: Алгоритмы**
  - Blind 75 (все 75 задач)
  - NeetCode 150 (добавить 75 задач)
  - Meta tagged (еще 50 задач)
  - Итого: 200+ задач LeetCode

- [ ] **Месяц 6: Mock Interviews**
  - Pramp.com (5+ sessions)
  - Interviewing.io (5+ sessions)
  - LeetCode mock interviews
  - System Design mock с друзьями/коллегами

- [ ] **Continuous: Behavioral**
  - Писать истории STAR
  - Практиковать ответы вслух
  - Записывать видео с ответами

### Ресурсы для подготовки
- [ ] **Книги**
  - System Design Interview Vol 1 & 2 (Alex Xu)
  - Designing Data-Intensive Applications (Martin Kleppmann)
  - Cracking the Coding Interview (Gayle McDowell)

- [ ] **Онлайн курсы**
  - Grokking the System Design Interview (Educative)
  - Grokking the Advanced System Design (Educative)
  - AlgoExpert System Design

- [ ] **YouTube каналы**
  - ByteByteGo (System Design)
  - NeetCode (Algorithms)
  - Exponent (Behavioral + System Design)
  - Gaurav Sen (System Design)

- [ ] **Практические платформы**
  - LeetCode Premium (Meta tagged)
  - Pramp (mock interviews)
  - Interviewing.io (mock interviews с инженерами)

### Текущий статус готовности
```
Theory (Docs):        ████████░░ 80%
├─ PHP/Laravel:       ██████████ 95%
├─ Databases:         ██████████ 95%
├─ Patterns:          █████████░ 85%
├─ System Design:     ██░░░░░░░░ 20%
└─ Distributed Sys:   █░░░░░░░░░ 10%

Practice:             ██░░░░░░░░ 20%
├─ Algorithms:        █░░░░░░░░░ 15%
├─ System Design:     ██░░░░░░░░ 20%
└─ Behavioral:        ██░░░░░░░░ 25%

Overall Meta Ready:   ████░░░░░░ 40%
Target: 80%+ за 6 месяцев
```

---

## Общие заметки

### Интересные статьи/ресурсы:
- 

### Вопросы с собеседований:
- 

---

## Как использовать этот файл

1. **Добавляй** новые темы по мере появления вопросов
2. **Группируй** по категориям
3. **Отмечай** галочкой когда тема добавлена в основную документацию
4. **Переноси** изученные темы в соответствующие разделы основной доки

**Формат добавления:**
```markdown
- [ ] **Название темы** - краткое описание
  - Подтема 1
  - Подтема 2
```
