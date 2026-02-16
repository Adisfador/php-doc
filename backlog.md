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

#### Критичные для Middle:
- [ ] **DateTime & DateTimeImmutable** → добавить в **core-basics.md** - работа с датами, timezones, DateInterval, DatePeriod, форматирование
- [ ] **Regular Expressions** → добавить в **core-basics.md** - preg_match, preg_replace, preg_match_all, patterns, modifiers, named groups, lookbehind/lookahead
- [ ] **File Operations детально** → добавить в **core-basics.md** - fopen/fread/fwrite/fclose, file_get_contents/file_put_contents с контекстом, file locking

#### Важные для Senior:
- [ ] **Fibers (PHP 8.1+)** → добавить в **internals.md** - асинхронность, Fiber::suspend/resume, use cases
- [ ] **HTTP Client** → добавить в **core-basics.md** - curl (curl_init, setopt, exec), stream_context_create, Guzzle basics
- [ ] **XML Processing** → добавить в **core-basics.md** - SimpleXML, DOMDocument, XMLReader/XMLWriter для больших файлов
- [ ] **JSON Processing** → добавить в **core-basics.md** - json_encode/decode, JsonSerializable interface, JSON_THROW_ON_ERROR, json_validate (PHP 8.3)

#### Можно добавить позже:
- [ ] **Multibyte Strings** → добавить в **core-basics.md** - mb_* functions, encodings (UTF-8, cp1251), mb_strlen vs strlen, mb_convert_encoding
- [ ] **Advanced Magic Methods** → дополнить **oop.md** - детальный разбор всех __method (__set_state, __debugInfo, __sleep/__wakeup, __serialize/__unserialize)
- [ ] **Memory Management** → добавить в **internals.md** - reference counting, circular references, WeakReference vs WeakMap, gc_collect_cycles
- [ ] **FFI (Foreign Function Interface)** → добавить в **internals.md** - вызов C библиотек из PHP, use cases

### Вопросы для изучения:
- [ ] 

---

## 🔴 Laravel

### Темы для добавления:

#### Критичные для Middle/Senior (приоритет 1):
- [ ] **Artisan Commands** - создание команд, аргументы/опции, интерактивность (ask, confirm, choice), прогресс-бары, таблицы
- [ ] **Mail & Notifications** - Mailable (markdown/view), Notification channels (mail/database/slack/broadcast), on-demand notifications, queueing
- [ ] **Task Scheduling** - scheduler (cron expressions), frequency, tasks, overlapping prevention, maintenance mode, webhooks
- [ ] **API Resources** - JsonResource, ResourceCollection, conditional attributes, nested resources, pagination, wrapping
- [ ] **File Storage** - filesystems config (local/public/s3), disk operations, streaming/chunked uploads, visibility, CDN integration

#### Важные для Senior (приоритет 2):
- [ ] **Localization (i18n)** - языковые файлы (PHP/JSON), trans() vs __(), pluralization, параметры, определение локали
- [ ] **Pagination** - paginate() vs simplePaginate() vs cursorPaginate(), customization, appending query strings, JSON responses
- [ ] **Sanctum** - API tokens (abilities/scopes), SPA authentication (CSRF), token expiration, mobile apps auth
- [ ] **Telescope** - installation/config, debugging в dev окружении, watchers (queries/requests/exceptions/jobs/mail), filtering, tag
- [ ] **Octane** - Swoole vs RoadRunner, performance benefits, tables/cache, concurrent tasks, intervals/ticks, dependency injection caveats

#### Можно добавить позже (приоритет 3):
- [ ] **Passport** - полноценный OAuth2 server, authorization code grant, client credentials, personal access tokens, scopes
- [ ] **Socialite** - OAuth authentication provider (Google, GitHub, Facebook, Twitter), stateless auth
- [ ] **Scout** - full-text search integration (Algolia/Meilisearch/Typesense), indexing, searching, pagination
- [ ] **Dusk** - end-to-end browser testing, headless Chrome, page objects, assertions, continuous integration
- [ ] **Vite** - modern asset bundling (замена Mix), HMR, CSS preprocessing, code splitting, production builds
- [ ] **Pint** - opinionated code style fixer на базе PHP-CS-Fixer, Laravel style preset, configuration
- [ ] **Sail** - Docker-based local development (MySQL/PostgreSQL/Redis/Mailhog/Selenium), customization, services
- [ ] **Folio** - page-based routing (альтернатива классическому routing), route model binding, middleware
- [ ] **Prompts** - красивые CLI prompts (text, password, select, multiselect, confirm, search, etc.)
- [ ] **Reverb** - первопартийный WebSocket server от Laravel, scaling, channels, presence

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
- [x] **Load Balancing и масштабирование** - реализовано в [system-design/scalability.md](system-design/scalability.md)
- [x] **Caching strategies** - реализовано в [system-design/scalability.md](system-design/scalability.md)
- [x] **Database Scaling** - реализовано в [system-design/scalability.md](system-design/scalability.md)
- [ ] **Message Queues и Event-Driven** - частично в [messaging/](messaging/)
  - Event sourcing
  - Saga pattern для distributed transactions
  - Exactly-once delivery
- [ ] **Microservices Design**
  - Service decomposition strategies
  - API Gateway pattern
  - Service mesh (Istio, Linkerd концептуально)
  - Circuit breaker pattern
  - Service discovery
  - Distributed tracing
- [x] **Rate Limiting и защита** - реализовано в [system-design/scalability.md](system-design/scalability.md)
- [ ] **Real-world System Design примеры** - создать [system-design/case-studies.md](system-design/case-studies.md)
  - Design Instagram/Facebook News Feed
  - Design WhatsApp/Messenger
  - Design Twitter timeline
  - Design URL shortener (bit.ly)
  - Design Dropbox/Google Drive
  - Design YouTube/Netflix
  - Design Uber/Lyft

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
